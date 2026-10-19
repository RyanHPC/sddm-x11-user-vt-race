# Root cause

SDDM 0.21.0, greeter running in the `x11-user` fallback. Source links point to the [`v0.21.0`](https://github.com/sddm/sddm/tree/v0.21.0) tag. Xorg links point to the `xorg-server-21.1.13` tag; the traces were taken on the installed 21.1.25 binary (see [limitations](limitations.md)).

## In one paragraph

When the login succeeds, the greeter exits and its helper terminates the greeter's Xorg. At the same time the user session asks the kernel to switch to the new VT. The kernel notifies the VT's owner (that same Xorg) and waits for it to release the VT, but the Xorg is already shutting down and never answers. The user session blocks in `VT_WAITACTIVE` until SDDM gives up.

## Sequence

| # | What happens | Where |
|---|---|---|
| 0 | `DisplayServer=wayland` but the greeter compositor is missing, so the daemon falls back to `x11-user` (a user-owned Xorg started by `sddm-helper-start-x11user`). | [`Seat.cpp#L58-L66`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Seat.cpp#L58-L66) |
| 1 | Authentication succeeds; the daemon emits `loginSucceeded` to the greeter. | [`Display.cpp#L465-L496`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Display.cpp#L465-L496), [`GreeterProxy.cpp#L198`](https://github.com/sddm/sddm/blob/v0.21.0/src/greeter/GreeterProxy.cpp#L198) |
| 2 | The greeter closes its window on `loginSucceeded`; with no windows left, Qt quits the application (default `quitOnLastWindowClosed`). | [`GreeterApp.cpp#L173`](https://github.com/sddm/sddm/blob/v0.21.0/src/greeter/GreeterApp.cpp#L173) |
| 3 | In `sddm-helper-start-x11user`, the greeter process finishing quits the app, and `aboutToQuit` calls `helper.stop()`. | [`HelperStartX11User.cpp#L57-L59`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/HelperStartX11User.cpp#L57-L59), [`#L71`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/HelperStartX11User.cpp#L71) |
| 4 | `stop()` sends SIGTERM to the greeter's Xorg and waits for it. | [`xorguserhelper.cpp#L68-L82`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/xorguserhelper.cpp#L68-L82) |
| 5 | In parallel, the user session (a Wayland session in my tests, so `vt_auto` is false) runs `VirtualTerminal::jumpToVt` in the child process before `exec`. It calls `VT_ACTIVATE` then `VT_WAITACTIVE`, and only retries on `EINTR`. | [`UserSession.cpp#L191`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/UserSession.cpp#L191), [`#L231`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/UserSession.cpp#L231), [`VirtualTerminal.cpp#L241-L252`](https://github.com/sddm/sddm/blob/v0.21.0/src/common/VirtualTerminal.cpp#L241-L252) |
| 6 | Because the child is stuck before `exec`, the helper's `waitForStarted()` (Qt default: 30 s) times out, which is the `Session started false` message 30 s later. | [`UserSession.cpp#L134`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/UserSession.cpp#L134), [`Display.cpp#L545-L549`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Display.cpp#L545-L549) |
| 7 | The helper is then terminated; for a user session `stop()` waits up to 60 s and then kills it, which is the `sddm-helper exited with 2` message. A new greeter starts. | [`UserSession.cpp#L146-L161`](https://github.com/sddm/sddm/blob/v0.21.0/src/helper/UserSession.cpp#L146-L161) |

The timings in steps 6 and 7 (30 s, then 60 s, 90 s in total) match what I measured; the processes involved are in [evidence](evidence.md).

## Why Xorg never answers

- The old VT is in `VT_PROCESS` mode with Xorg as owner. On `VT_ACTIVATE` the kernel sends it SIGUSR1 and waits for `VT_RELDISP`.
- The SIGUSR1 handler, `xf86VTRequest`, only sets `xf86Info.vtRequestsPending = TRUE`: [`VTsw_usl.c#L44-L50`](https://gitlab.freedesktop.org/xorg/xserver/-/blob/xorg-server-21.1.13/hw/xfree86/os-support/shared/VTsw_usl.c#L44-L50).
- The flag is only acted on by `xf86Wakeup`, the wakeup handler of the server's main loop, which calls `xf86VTSwitch` (and from it `xf86VTLeave` and `VT_RELDISP`): [`xf86Events.c#L217-L224`](https://gitlab.freedesktop.org/xorg/xserver/-/blob/xorg-server-21.1.13/hw/xfree86/common/xf86Events.c#L217-L224), [`#L499`](https://gitlab.freedesktop.org/xorg/xserver/-/blob/xorg-server-21.1.13/hw/xfree86/common/xf86Events.c#L499).
- After SIGTERM the main loop has already left. The traces show `GiveUp`, one `xf86Wakeup` and `CloseDownExtensions` within about 1 ms of the signal; the VT request arrives 24–42 ms later, sets the flag, and nothing reads it. Xorg exits about 80–106 ms after the SIGTERM.
- Result: no `xf86VTLeave`, no `VT_RELDISP`, no `complete_change_console` in the kernel.

Control with a live Xorg (manual switch to a TTY): SIGUSR1, then `xf86VTRequest`, `xf86VTLeave`, `VT_RELDISP` after about 11 ms.

## Why the other configurations do not hit it

- **`DisplayServer=x11`:** the greeter's Xorg belongs to the daemon and is stopped much later (at the end of the session in my logs), so it is alive and serves the request in 71–110 ms.
- **Wayland greeter (`weston`):** the VT owner is `systemd-logind`, which resets the VT mode shortly after the compositor is terminated, so the switch completes immediately. The window of time is similar; the mechanism is different. Only 5 logins, so this is not proof it can never race.

## Why the one-line patch hides it

On a successful session start the daemon stops the greeter by itself after 5 s ([`Display.cpp#L545-L549`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Display.cpp#L545-L549)). If the greeter does not quit on its own at step 2 (upstream commit `da0b64d`, `app.setQuitOnLastWindowClosed(false)`), the greeter's Xorg is still alive when `VT_ACTIVATE` arrives and serves it in 10–27 ms; it is terminated about 5 s later, as in the measurements. The race is still in the code; the order of events just no longer triggers it. That commit was written for another bug (#2211), so this is a side effect, not a fix.

## Direction for a proper fix (untested)

I have not written or tested a patch. Ideas, in the order I would try them:

1. Do not terminate the greeter's X server until the VT switch has completed (for example, only after the daemon reports the session as started), so the VT owner can always answer.
2. Make `jumpToVt` bounded: poll or time-limit `VT_WAITACTIVE` and retry `VT_ACTIVATE` instead of waiting until the helper is killed. A retry every 2 s was proposed for a related report in [#1443](https://github.com/sddm/sddm/issues/1443) and never merged.
3. Let the helper put the VT in `VT_AUTO` once the VT owner is known to be gone (the idea behind [PR #2204](https://github.com/sddm/sddm/pull/2204); by my reading of the code it does not cover this case, and I have not tested it).

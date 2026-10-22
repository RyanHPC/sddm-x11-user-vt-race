# Upstream fix: reading of commit `da0b64d` on `develop`

This page is a reading of SDDM's source code, not a test. I checked it on Oct 22, 2026 against SDDM `develop` at [`633c1bf`](https://github.com/sddm/sddm/commit/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550) (Oct 20, 2026) and the `v0.21.0` tag (Feb 21, 2024). I did not build or run anything from `develop`; my login counts are on 0.21.0 plus this single commit ([evidence](evidence.md#4-x11-user-fallback--one-line-patch)).

## The commit

| | |
|---|---|
| Commit | [`da0b64d`](https://github.com/sddm/sddm/commit/da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a), "greeter: Don't quit when the last window closes" |
| Author, date | Daniel Betz, Aug 12, 2026 |
| Why it was written | [#2211](https://github.com/sddm/sddm/issues/2211): with one display, unplugging the monitor closes the last window, the greeter exits with status 0, the daemon takes that for success and does not start a new greeter |
| Size | `src/greeter/GreeterApp.cpp`, 4 lines added: a comment, `app.setQuitOnLastWindowClosed(false);` and blank lines |
| In `develop` | yes (it is an ancestor of `origin/develop`) |
| In `v0.21.0` | no |
| In a release | no: the newest tag is `v0.21.0`, and `develop` has 61 commits after it |
| Patch in this repository | [`../patch/greeter-keep-alive.patch`](../patch/greeter-keep-alive.patch) has the same diff as the commit and applies to the `v0.21.0` tag (checked with `git apply --check`) |

## Order of events with the commit

Source links below point to `develop` at `633c1bf`.

| # | What happens | Where |
|---|---|---|
| 1 | The login succeeds and the greeter closes its window. Without the commit the greeter exits here, because it was the last window. With it, the greeter keeps running. | [`GreeterApp.cpp#L173`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/greeter/GreeterApp.cpp#L173), [`#L365-L366`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/greeter/GreeterApp.cpp#L365-L366) |
| 2 | In the user session's helper, `openSession()` runs first and only afterwards `sessionOpened(true)` sends `SESSION_STATUS`. | [`HelperApp.cpp#L174-L180`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/HelperApp.cpp#L174-L180) |
| 3 | `openSession()` ends in `session()->start()`, which starts the process and calls `waitForStarted()`. The child runs `setupChildProcess()` before `exec`, and that is where `jumpToVt` is called. | [`Backend.cpp#L122`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/Backend.cpp#L122), [`UserSession.cpp#L134`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/UserSession.cpp#L134), [`#L231-L232`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/UserSession.cpp#L231-L232) |
| 4 | The daemon receives the status and, if it is `true`, schedules `Greeter::stop` 5 s later. | [`Auth.cpp#L195-L198`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/auth/Auth.cpp#L195-L198), [`Display.cpp#L562-L566`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/daemon/Display.cpp#L562-L566) |
| 5 | `Greeter::stop` stops the greeter's helper; for the `x11-user` fallback this ends in `sddm-helper-start-x11user` quitting and stopping the greeter's Xorg with SIGTERM. This is the same SIGTERM that, without the commit, arrived at the same time as the VT request. | [`Greeter.cpp#L249-L266`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/daemon/Greeter.cpp#L249-L266), [`UserSession.cpp#L146`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/UserSession.cpp#L146), [`HelperStartX11User.cpp#L57-L59`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/HelperStartX11User.cpp#L57-L59), [`xorguserhelper.cpp#L68-L76`](https://github.com/sddm/sddm/blob/633c1bf5c08d6c8e5b8435a70edb281b9f7b1550/src/helper/xorguserhelper.cpp#L68-L76) |

So the greeter's Xorg is terminated only after the user session has been reported as started, and that report comes after `waitForStarted()` returned, that is, after the child went through `jumpToVt`. This relies on `waitForStarted()` returning only once the child has reached `exec`, which is how I understand Qt's `QProcess` to work on Unix; it matches what I measured on 0.21.0, where the 30 s `Session started false` is that call timing out while the child is stuck in the VT switch. In my traces with the commit applied, the VT request was served by the live Xorg in 10–27 ms and the Xorg got SIGTERM 4.8–5.3 s later.

## What differs between `v0.21.0` and `develop` on this path

I compared the files involved with `git diff v0.21.0 origin/develop`.

- **Unchanged:** the greeter-stop logic (`slotSessionStarted` and its 5 s timer), `HelperApp.cpp`, `Backend.cpp`, `Auth.cpp`, and the body of `jumpToVt`. The only change in `VirtualTerminal.cpp` is that `defaultVtPath` is no longer `static`.
- **Changed, but not in the race:** guards for seats without a VT (`seat()->canTTY()`, and `vtNumber > 0` around `jumpToVt` in `UserSession.cpp`), cursor and `xrdb` handling in `xorguserhelper.cpp`, and `Seat::displayStopped`.
- **Not read in full:** `Display.cpp` has 123 changed lines. I read the hunks that mention VT, terminal id or the greeter, not all of them.

## What this supports and what it does not

Supports: with `da0b64d`, the order of events is the one I measured with the patch on 0.21.0, and the code that terminates the greeter's Xorg is the same on `develop`.

Does not show:

- that `develop` passes my login test (I did not build it);
- what happens when the session start fails (`Session started false`): the 5 s timer is only armed on success, and I did not follow that path;
- other reasons for the greeter to exit before the session is reported as started: the same race may be possible there. `jumpToVt` still has no timeout; its loop only continues on `EINTR`;
- the log side effect described by the author of [PR #2204](https://github.com/sddm/sddm/pull/2204) (`HELPER_OTHER_ERROR` on every login): I did not trace it.

## Check it yourself

```sh
git clone --filter=blob:none https://github.com/sddm/sddm && cd sddm
git merge-base --is-ancestor da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a v0.21.0 || echo "not in v0.21.0"
git merge-base --is-ancestor da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a origin/develop && echo "in develop"
git tag --sort=-v:refname | head -3
git rev-list --count v0.21.0..origin/develop
git diff v0.21.0 origin/develop -- src/helper/UserSession.cpp src/common/VirtualTerminal.cpp
```

# Evidence

All data comes from one machine (see [Environment](../README.md#environment)), SDDM 0.21.0 unless stated. Raw logs are in [`../logs/`](../logs/); the scripts that produced them are in [`../scripts/`](../scripts/). Hostname and user name are redacted in the logs, and the labels of the older ones were translated to English.

## How it was measured

`bpftrace`, all events on one clock. Probes:

- `ioctl` entries for `VT_SETMODE` (0x5602), `VT_RELDISP` (0x5605), `VT_ACTIVATE` (0x5606), `VT_WAITACTIVE` (0x5607) and `KDSETMODE` (0x4b3a)
- `signal_generate` / `signal_deliver` for SIGTERM and SIGUSR1 (the VT switch signal)
- kprobes on the kernel's `change_console` and `complete_change_console`
- `do_exit` of `Xorg`, `weston` and `sddm-greeter`
- uprobes on the installed `/usr/lib/Xorg`: `GiveUp`, `xf86Wakeup` (logged only after SIGTERM), `xf86VTRequest`, `xf86VTLeave`, `CloseDownExtensions`, `ddxGiveUp`

Which script produced which log:

| Log | Script | Case |
|---|---|---|
| `vt-trace2.log` | `vt-trace2.bt` | cycle, Hyprland |
| `vt-trace3.log` | `vt-trace3.bt` | cycle, Hyprland |
| `vt-trace4.log` | `vt-trace4.bt` | cycle, Hyprland |
| `vt-trace5-x11.log` | `vt-trace4.bt` | `DisplayServer=x11` |
| `vt-trace6-plasma.log` | `vt-trace4.bt` | cycle, Plasma Wayland |
| `vt-trace7-weston.log` | `vt-trace7.bt` | Wayland greeter (weston) |
| `vt-trace8-patched.log` | `vt-trace7.bt` | `x11-user` fallback + one-line patch |
| `vt-trace9-pr2210.log` | `vt-trace4.bt` | `x11-user` fallback + the one-line change of sddm/sddm#2210 |

## 1. The cycle (original package, `x11-user` fallback, no delay)

Four captures, each on the first login attempted: three with Hyprland, one with Plasma (Wayland). Times are relative to the SIGTERM that `sddm-helper-start-x11user` sends to the greeter's Xorg (t = 0).

| Event | trace2 | trace3 | trace4 | trace6 (Plasma) |
|---|---|---|---|---|
| Kernel `change_console` (user session `VT_ACTIVATE`) | +37 ms | +32 ms | +24 ms | +27 ms |
| `xf86VTRequest` runs (SIGUSR1 delivered) | +40 ms | +42 ms | +37 ms | +28 ms |
| `xf86VTLeave` / `VT_RELDISP` | never | never | never | never |
| `ddxGiveUp` (Xorg shutting down) | n/a | +109 ms | +95 ms | +80 ms |
| Xorg exits | +95 ms | +133 ms | +106 ms | +83 ms |
| `complete_change_console` | not until the 2nd `VT_ACTIVATE` | same | same | same |
| 2nd `VT_ACTIVATE` completes immediately (no owner left) | +30.1 s | +30.0 s | +30.1 s | +30.1 s |

In traces 4 and 6, `GiveUp`, one `xf86Wakeup` and `CloseDownExtensions` all run within about 1 ms of the SIGTERM; in trace 3 (which only probed `xf86Wakeup`) the one `xf86Wakeup` runs in the same millisecond as the SIGTERM. In all three, `xf86Wakeup` is not called again after the VT request arrives. The first `xf86VTRequest` lines at the start of traces 3 and 4 belong to the greeter's Xorg acquiring its VT when it starts (the same SIGUSR1 handler is used for acquire and release). In trace 6 the user helper is killed at +90.2 s, when the next greeter's VT switch happens. (trace2 did not have the `xf86Wakeup`, `GiveUp` and `ddxGiveUp` probes.)

SDDM journal of two cycles:

```
Hyprland                                             Plasma (Wayland)
17:40:22 Session ... selected, VT 7                  02:15:07 login
17:40:22 Greeter stopped. HELPER_SUCCESS             02:15:37 Session started false      (+30 s)
17:40:52 Session started false      (+30 s)          02:16:37 Auth: sddm-helper exited with 2   (+90 s)
17:41:52 Auth: sddm-helper exited with 2  (+90 s)
17:41:52 Adding new display... (new greeter)
```

**Control, Xorg alive.** In the same captures the next greeter's Xorg, which is alive and in its main loop, goes `SIGUSR1` → `xf86VTRequest` → `xf86VTLeave` → `VT_RELDISP` → `complete_change_console` in 11 ms (trace6), 10 ms (trace3) and 17 ms (trace4). So the handler works when the main loop is running.

## 2. `DisplayServer=x11` (greeter's Xorg owned by the daemon)

Log: `vt-trace5-x11.log`, 9 logins. The greeter's Xorg stays alive during the login and handles the VT request in 71–110 ms (`xf86VTRequest` → `xf86VTLeave` → `complete_change_console`). The `sddm` daemon sends it SIGTERM only at the end of the session, 7–72 s after the switch (8 of the 9 have that SIGTERM in the log). None of the 9 shows the 30 s second `VT_ACTIVATE` that marks the cycle. (I did not save the SDDM journal of this run. I counted 11 logins while testing; the trace contains 9, so I report 9.)

## 3. Wayland greeter (`weston`)

Log: `vt-trace7-weston.log` + `sddm-journal-weston.log`. 5 successful logins (1 Plasma, 4 Hyprland; one more attempt failed on a wrong password). No fallback message, `Session started true` in the same second every time.

| Login | SIGTERM to `weston` → `VT_ACTIVATE` | `complete_change_console` |
|---|---|---|
| 1 | +47 ms | same ms as `change_console` |
| 2 | +30 ms | same ms |
| 3 | +33 ms | same ms |
| 4 | +28 ms | same ms |
| 5 | +27 ms | same ms |

No SIGUSR1 is sent to any VT owner. In the four logins I checked line by line, `systemd-logind` issues `KDSETMODE` and `VT_SETMODE` 4 ms and 24–33 ms after the SIGTERM, before or in the same millisecond as the helper's `VT_ACTIVATE`. The window of time is similar to the Xorg case; the mechanism is different. Five logins do not prove a Wayland greeter can never race.

## 4. `x11-user` fallback + one-line patch

Package: SDDM 0.21.0-7 built from the Arch `PKGBUILD` plus `git cherry-pick -n da0b64d` (`app.setQuitOnLastWindowClosed(false)`; it applies cleanly on v0.21.0), see [`../patch/`](../patch/). Same configuration that produces the cycle (`DisplayServer=wayland`, no compositor, no PAM delay). Logs: `vt-trace8-patched.log`, `sddm-journal-patched.log`.

11 logins with Hyprland, 11 × `Session started true`, 0 × `Session started false` (plus one failed password attempt at 04:12:33, not counted).

| # | Login (journal) | `VT_ACTIVATE` → `complete_change_console` | SIGTERM to greeter's Xorg, after `VT_ACTIVATE` |
|---|---|---|---|
| 1 | 04:11:02 | 11 ms | 4.77 s |
| 2 | 04:11:36 | 10 ms | 4.98 s |
| 3 | 04:11:46 | 27 ms | 5.29 s |
| 4 | 04:11:55 | 21 ms | 5.29 s |
| 5 | 04:12:07 | 25 ms | 5.28 s |
| 6 | 04:12:16 | 24 ms | 5.28 s |
| 7 | 04:12:25 | 24 ms | 5.00 s |
| 8 | 04:12:37 | 24 ms | 5.30 s |
| 9 | 04:12:46 | 24 ms | 4.79 s |
| 10 | 04:12:57 | 27 ms | 4.93 s |
| 11 | 04:13:06 | 23 ms | 4.97 s |

Every switch is served by the still-running Xorg (SIGUSR1 → `VT_RELDISP` → `complete_change_console`), and the Xorg is terminated about 5 s later, which matches the daemon stopping the greeter 5 s after a successful session start ([`Display.cpp#L545-L549`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Display.cpp#L545-L549)). The order of events is reversed compared with section 1, and the cycle disappears.

## 5. `x11-user` fallback + the change of sddm/sddm#2210

Package: SDDM 0.21.0-7.2 = the Arch `PKGBUILD` plus a `sed` that replaces `VirtualTerminal::jumpToVt(vtNumber, x11UserSession);` with `VirtualTerminal::jumpToVt(vtNumber, true);` in `src/helper/UserSession.cpp`. That is the whole change of [PR #2210](https://github.com/sddm/sddm/pull/2210) (on `develop` it sits behind an `if (vtNumber > 0)` that `v0.21.0` does not have). Same configuration as section 1 (`DisplayServer=wayland`, no `weston`, no `sleep` in PAM). One login, the first one after restarting SDDM, Hyprland. Logs: `vt-trace9-pr2210.log` and `sddm-journal-pr2210.log`. t = 0 is the SIGTERM that `sddm-helper-start-x11user` sends to the greeter's Xorg (pid 834982 in the trace).

| Event | t |
|---|---|
| `GiveUp` (SIGTERM handler) and `xf86Wakeup` | +0 ms |
| `CloseDownExtensions` (main loop already finished) | +1 ms |
| `change_console` (kernel receives `VT_ACTIVATE`), SIGUSR1 sent to Xorg | +26 ms |
| `xf86VTRequest` | +27 ms |
| `ddxGiveUp` | +51 ms |
| Xorg exit | +54 ms |
| second `change_console` + `complete_change_console` | +30.1 s |
| third `change_console` + `complete_change_console` (new greeter) | +90.4 s |

There is no `xf86VTLeave` and no `complete_change_console` after the first `change_console`: the request is never served, as in section 1. Journal: `Login` at 19:05:05, `Session started false` at 19:05:36, `Auth: sddm-helper exited with 2` at 19:06:36, then a new greeter.

Later in the same trace, the Xorg of the second greeter (pid 839674, still alive) serves a VT switch away from the greeter (there is no login in the journal) in about 2 ms: `xf86VTRequest`, `xf86VTLeave`, `complete_change_console`.

Result: one login tested, one cycle; same sequence and timings as the unmodified package. By reading the code (not tested further), a likely reason is that `fixVtMode` only acts on the current VT when it is in `VT_AUTO`, while the greeter's Xorg leaves it in `VT_PROCESS`, and the kernel still waits for that Xorg to answer.

## Summary

| Case | Logins | Cycles |
|---|---|---|
| `x11-user` fallback, original package, no delay | 4 captures | 4 (first login each time) |
| `DisplayServer=x11` | 9 | 0 |
| Wayland greeter (weston) | 5 | 0 |
| `x11-user` fallback + one-line patch | 11 | 0 |
| `x11-user` fallback + change of sddm/sddm#2210 | 1 (first login) | 1 |
| `x11-user` fallback + `sleep 1` in PAM | several, informal (not counted, no trace saved) | 0 observed |

The comparison between the original package and the patched one was made on different boots and days, with the same configuration. What this does and does not show is in [limitations](limitations.md).

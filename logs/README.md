# Logs

Raw data behind [docs/evidence.md](../docs/evidence.md). Hostname and user name are redacted (`host`, `user`). Which bpftrace script produced which trace is in the table in [docs/evidence.md](../docs/evidence.md).

## Traces

| File | Case |
|---|---|
| `vt-trace2.log`, `vt-trace3.log`, `vt-trace4.log` | `x11-user` fallback, original package, Hyprland (the cycle) |
| `vt-trace6-plasma.log` | `x11-user` fallback, original package, Plasma (Wayland) (the cycle) |
| `vt-trace5-x11.log` | `DisplayServer=x11` |
| `vt-trace7-weston.log` | Wayland greeter (`weston`) |
| `vt-trace8-patched.log` | `x11-user` fallback + one-line patch |
| `vt-trace9-pr2210.log` | `x11-user` fallback + the change of sddm/sddm#2210 (the cycle) |

## Journals

| File | What it contains |
|---|---|
| `sddm-journal2.log` | One capture of the cycle (Oct 17, 17:40 to 17:41): `Session started false` after ~30 s. |
| `sddm-journal-plasma.log` | SDDM journal from Oct 17 18:40 to Oct 18 03:24, with the Plasma (Wayland) capture. |
| `sddm-journal-weston.log` | Oct 18 from 03:25. The first greeter (03:25 to 03:37) still used the `x11-user` fallback, before `weston` was in use. The 5 `weston` logins are from 03:38 on, with no fallback message. |
| `sddm-journal-pr2210.log` | Oct 19, 19:04 to 19:06: first login with the #2210 change. `Login` 19:05:05, `Session started false` 19:05:36, `exited with 2` 19:06:36, then a new greeter. |
| `sddm-journal-pr2210-all.log` | Oct 19 19:05 to Oct 20 00:52: all 11 logins with the #2210 change (9 × `Session started false`, 2 × `true`; one wrong password at 00:41:28, not counted). Two boots. |
| `sddm-journal-control-07.log` | Oct 20 00:54 to 01:01: control with the unmodified 0.21.0-7 package (6 × `false`, 2 × `true`). Includes a reboot at 00:56:22; the last cycle was still running when the journal was exported. |
| `sddm-journal-patched.log` | Oct 18 from 03:25 to 04:13. It contains the `weston` period above (5 logins), then, after the restart at 04:10:53, the patched build (11 logins, 0 `Session started false`). |

## Reading the journals

- `Auth: sddm-helper exited with 255` is the normal stop of the greeter's helper. It is not the cycle. A plain `grep "exited with 2"` also matches these lines; use `grep "exited with 2$"` for the cycle's message.
- The cycle is `Session started false` ~30 s after the login, then `Auth: sddm-helper exited with 2` ~60 s later.
- `falling back to DisplayServer=x11-user` marks the path where the problem happens.

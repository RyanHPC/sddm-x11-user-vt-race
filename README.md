# SDDM login hangs ~30 s, then returns to the greeter

A race in SDDM 0.21.0's `x11-user` greeter fallback: the greeter's X server is shut down at the same moment the user session asks for the VT switch, and the switch is never served.

> **Investigation status:** root cause confirmed with traces on one machine, under the tested configuration. Not yet confirmed upstream or on other setups. See [Not verified](#not-verified).

## Symptom

After typing the password in SDDM, the login hangs. About 30 s later the journal shows `Session started false`; about 60 s after that, `Auth: sddm-helper exited with 2` (90 s in total), and a new greeter appears. The cycle repeats.

```
sddm[...]: Auth: sddm-helper exited successfully          <- the greeter's helper
sddm[...]: Greeter stopped. SDDM::Auth::HELPER_SUCCESS
        ... 30 s ...
sddm[...]: Session started false
        ... 60 s ...
sddm[...]: Auth: sddm-helper exited with 2
sddm[...]: Adding new display...                          <- new greeter
```

It happens with Hyprland and with Plasma (Wayland) sessions alike.

## When it happens

`DisplayServer=wayland` in `/etc/sddm.conf`, but the greeter's default compositor (`weston --shell=kiosk`) is not installed. SDDM logs:

```
Failed to launch the display server, falling back to DisplayServer=x11-user
```

and runs the greeter on a user-owned Xorg started by `sddm-helper-start-x11user`.

## Cause (short version)

1. On login the greeter closes its window and exits.
2. `sddm-helper-start-x11user` then sends SIGTERM to the greeter's Xorg.
3. In parallel, the user session helper calls `VT_ACTIVATE` + `VT_WAITACTIVE` to move to the new VT.
4. The kernel signals the old VT's owner (Xorg, in `VT_PROCESS` mode) with SIGUSR1 and waits for `VT_RELDISP`. But Xorg's main loop has already exited because of the SIGTERM: `xf86VTRequest` only sets a flag, and the actual switch (`xf86VTSwitch`) is only run from the main loop's wakeup handler. The request is never served, and Xorg exits ~100 ms later.
5. The helper blocks until SDDM gives up (30 s), kills it (60 s more), and starts a new greeter.

Details and source links: [docs/root-cause.md](docs/root-cause.md). Measured timelines: [docs/evidence.md](docs/evidence.md).

## What to do today

**Use an X11 greeter run by the daemon:**

```ini
# /etc/sddm.conf
[General]
DisplayServer=x11
```

With this the greeter's Xorg belongs to the SDDM daemon, stays alive during login, and serves the VT switch in ~70–110 ms.

This is what worked on my setup (9 logins, 0 cycles), not a guaranteed universal fix. To revert, set `DisplayServer` back to its previous value (or remove the line) and restart SDDM (log out first, it ends your session). Full steps to apply and revert: [fixes/solution-displayserver-x11.md](fixes/solution-displayserver-x11.md).

## Other options (not equivalent)

| Option | What it really is | Where |
|---|---|---|
| `DisplayServer=x11` | Avoids the faulty path. Does not fix SDDM. | [fixes/solution-displayserver-x11.md](fixes/solution-displayserver-x11.md) |
| Greeter stays alive after login (one-line change, upstream commit `da0b64d`) | Mitigation. Reverses the order of events so the cycle disappears, but the race is still in the code, only hidden by a ~5 s timer. | [fixes/mitigation-keep-greeter-alive.md](fixes/mitigation-keep-greeter-alive.md), [patch/](patch/) |
| `sleep 1` in the PAM session stack | Workaround. Only changes timing. | [fixes/workaround-pam-sleep.md](fixes/workaround-pam-sleep.md) |
| Proper fix in SDDM | Not written or tested. Idea: wait for the greeter's X server to exit before `VT_ACTIVATE`, or bound `VT_WAITACTIVE` and retry. | [docs/root-cause.md](docs/root-cause.md#direction-for-a-proper-fix-untested) |

## Evidence at a glance

| Test (all on SDDM 0.21.0, same machine) | Result |
|---|---|
| `x11-user` fallback, no delay, Hyprland (3 captures) + Plasma Wayland (1 capture) | Cycle on the first login in each capture; VT request arrives ~25–40 ms after the Xorg SIGTERM and is never served |
| `DisplayServer=x11` | 0 cycles in 9 logins (from the log); Xorg serves the VT request in 71–110 ms |
| Wayland greeter (`weston`) | 0 cycles in 5 logins; `systemd-logind` resets the VT mode after the compositor exits, switch completes immediately |
| `x11-user` fallback + one-line patch (greeter stays alive) | 0 cycles in 11 logins; Xorg serves the VT request in 10–27 ms and receives SIGTERM ~5 s later |
| `x11-user` fallback + `sleep 1` in PAM | No cycle in several informal logins; no trace or log saved |

The patch row is the closest thing to a causal test: it changes only the order of events (the greeter outlives the VT switch) and the cycle disappears. The `x11` row shows the same switch being served by a live Xorg, and the `weston` row shows a different VT owner (`systemd-logind`) completing it.

Raw data: [logs/](logs/). Scripts: [scripts/](scripts/).

## Environment

- Garuda Linux (Arch-based), Acer Nitro V15 (RTX 3050 Optimus + Intel)
- sddm 0.21.0-7, xorg-server 21.1.25-2, linux-zen 7.2.9.zen1-1, nvidia-open-dkms 615.78.08-1, hyprland 0.56.2-4
- Session: Hyprland (`start-hyprland`) and Plasma (Wayland)

## Related upstream issues

Same symptom reported in [sddm/sddm#2129](https://github.com/sddm/sddm/issues/2129), [#2043](https://github.com/sddm/sddm/issues/2043) and [#1624](https://github.com/sddm/sddm/issues/1624). [#1929](https://github.com/sddm/sddm/issues/1929) may be the same (same versions, 32 s then 60 s, user helper only starting after the timeout), but it does not say which `DisplayServer` was used. Related VT-switch discussions: [#1443](https://github.com/sddm/sddm/issues/1443), [PR #2204](https://github.com/sddm/sddm/pull/2204). I have not checked whether those reports have the same mechanism. If your journal (`journalctl -b -u sddm`) contains `falling back to DisplayServer=x11-user` right before the hang, it may be this one.

## Not verified

Details in [docs/limitations.md](docs/limitations.md). Mainly: other machines and setups, Wayland greeters beyond the 5 logins tested, the full upstream `master`, and a proper fix in SDDM (not written).

## Repository layout

```
README.md
docs/      root cause, evidence, limitations
fixes/     solution, mitigation, workaround
patch/     the one-line patch and how to build it
logs/      raw traces and journals (hostname and user redacted), see logs/README.md
scripts/   bpftrace scripts used for the traces
```

## About this investigation

This happened on my own laptop. I could have kept using it with the workaround, but I wanted to find the actual root cause, so I kept digging. I did it with the help of Claude (Anthropic's AI assistant), which I used to write the tracing scripts and analyse the results; I ran all the tests on my machine. Everything here comes from one machine and one configuration, so please verify before relying on it.

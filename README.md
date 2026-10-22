# SDDM login hangs ~30 s, then returns to the greeter

A race in SDDM 0.21.0's `x11-user` greeter fallback: the greeter's X server is shut down at the same moment the user session asks for the VT switch, and the switch is never served.

> **Investigation status:** root cause confirmed with traces on one machine, under the tested configuration. Not yet confirmed on other setups. See [Not verified](#not-verified). Reported upstream in [sddm/sddm#2225](https://github.com/sddm/sddm/issues/2225), which the maintainer closed as a likely duplicate of [#2043](https://github.com/sddm/sddm/issues/2043).
>
> **Update (Oct 22, 2026):** SDDM's `develop` branch already contains a change that removes this race: commit [`da0b64d`](https://github.com/sddm/sddm/commit/da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a), written for another bug ([#2211](https://github.com/sddm/sddm/issues/2211)). I did not know that when I published. I found out from the closing comment of [PR #2204](https://github.com/sddm/sddm/pull/2204), whose author reached the same conclusion for the Wayland greeter. No release after `v0.21.0` exists yet (checked in the git tags on Oct 22, 2026). What I measured here stands: the mechanism in the `x11-user` fallback, and that this commit makes the cycle disappear on 0.21.0. Earlier versions of this repository called the commit a "mitigation"; that was wrong, and this version corrects it.

## Symptom

After typing the password in SDDM, the login hangs. About 30 s later the journal shows `Session started false`; about 60 s after that, `Auth: sddm-helper exited with 2` (90 s in total), and a new greeter appears. The cycle repeats. On my machine it is intermittent: 6 of 8 logins on the unmodified package hit it and 2 logged in normally ([evidence](docs/evidence.md#6-control-original-package-same-configuration)).

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

If your distribution already ships an SDDM built from `develop` after commit `da0b64d` (Gentoo's `0.21.0_p20260819` snapshot is one, according to the author of PR #2204), you should not hit this at all. Once a release includes that commit, updating is the real fix.

## Other options (not equivalent)

| Option | What it really is | Where |
|---|---|---|
| `DisplayServer=x11` | Avoids the faulty path. Does not fix SDDM. | [fixes/solution-displayserver-x11.md](fixes/solution-displayserver-x11.md) |
| **Upstream fix:** greeter stays alive after login (commit `da0b64d` on `develop`, not in a release yet) | Removes the race as a side effect of a change made for another bug (#2211). On 0.21.0 plus this commit: 0 cycles in 11 logins, against 6 of 8 without it. It reverses the order of events: by my reading of the code the daemon stops the greeter only after the session start is reported, which is after the VT switch. Not tested on current `develop`. | [fixes/upstream-fix-greeter-keep-alive.md](fixes/upstream-fix-greeter-keep-alive.md), [docs/upstream-fix-analysis.md](docs/upstream-fix-analysis.md), [patch/](patch/) |
| `sleep 1` in the PAM session stack | Workaround. Only changes timing. | [fixes/workaround-pam-sleep.md](fixes/workaround-pam-sleep.md) |
| Extra safeguard in SDDM (optional) | For a greeter that dies for another reason: bound `VT_WAITACTIVE` and re-issue `VT_ACTIVATE`. The author of PR #2204 offered a separate PR for this; I have not written or tested it. | [docs/root-cause.md](docs/root-cause.md#other-safeguards-untested) |

## Evidence at a glance

| Test (all on SDDM 0.21.0, same machine) | Result |
|---|---|
| `x11-user` fallback, no delay, Hyprland (3 captures) + Plasma Wayland (1 capture) | Cycle on the first login in each capture; VT request arrives ~25–40 ms after the Xorg SIGTERM and is never served |
| `x11-user` fallback, original package, control run, right after (journal only) | Cycle in 6 of 8 logins; the other 2 logged in normally |
| `DisplayServer=x11` | 0 cycles in 9 logins (from the log); Xorg serves the VT request in 71–110 ms |
| Wayland greeter (`weston`) | 0 cycles in 5 logins; `systemd-logind` resets the VT mode after the compositor exits, switch completes immediately |
| `x11-user` fallback + upstream commit `da0b64d` (one line, greeter stays alive) | 0 cycles in 11 logins; Xorg serves the VT request in 10–27 ms and receives SIGTERM ~5 s later |
| `x11-user` fallback + change of [sddm/sddm#2210](https://github.com/sddm/sddm/pull/2210) | Cycle in 9 of 11 logins (the first one traced: same sequence and timings as the unmodified package); the other 2 logged in normally |
| `x11-user` fallback + `sleep 1` in PAM | No cycle in several informal logins; no trace or log saved |

The `da0b64d` row is the closest thing to a causal test: it changes only the order of events (the greeter outlives the VT switch) and the cycle disappears. The cycle is intermittent (about 3 logins in 4 on the unmodified package here), so 0 of 11 is unlikely to be luck: p ≈ 0.001 against the control run (one-sided Fisher exact test; the runs were on different days). The `x11` row shows the same switch being served by a live Xorg, and the `weston` row shows a different VT owner (`systemd-logind`) completing it.

Raw data: [logs/](logs/). Scripts: [scripts/](scripts/).

## Environment

- Garuda Linux (Arch-based), Acer Nitro V15 (RTX 3050 Optimus + Intel)
- sddm 0.21.0-7, xorg-server 21.1.25-2, linux-zen 7.2.9.zen1-1, nvidia-open-dkms 615.78.08-1, hyprland 0.56.2-4
- Session: Hyprland (`start-hyprland`) and Plasma (Wayland)

## Related upstream issues

Same symptom reported in [sddm/sddm#2129](https://github.com/sddm/sddm/issues/2129), [#2043](https://github.com/sddm/sddm/issues/2043) and [#1624](https://github.com/sddm/sddm/issues/1624). #2129 seems to mix at least two variants of the same symptom: most of its thread is about the Wayland greeter (`kwin_wayland` leaving the VT in `VT_PROCESS`, see [PR #2204](https://github.com/sddm/sddm/pull/2204)), which is a different trigger from the `x11-user` fallback described here. [#1929](https://github.com/sddm/sddm/issues/1929) may be the same (same versions, 32 s then 60 s, user helper only starting after the timeout), but it does not say which `DisplayServer` was used. Related VT-switch discussions: [#1443](https://github.com/sddm/sddm/issues/1443), [PR #2204](https://github.com/sddm/sddm/pull/2204) and [PR #2210](https://github.com/sddm/sddm/pull/2210). I have not checked whether those reports have the same mechanism. For my case: the one-line change of #2210 (`jumpToVt(vtNumber, true)`) did not stop the cycle (9 of 11 logins, [evidence](docs/evidence.md#5-x11-user-fallback--the-change-of-sddmsddm2210)). The author of #2204 closed it without merging after concluding that it does not fix the race (it leaves a VT that is already in `VT_PROCESS` alone) and that `da0b64d` does. For user sessions of type wayland or x11 the expression in #2204 (`x11UserSession || waylandUserSession`) also gives `true`, so I would expect the same result there as with #2210, but I did not build #2204. (In that code `x11UserSession` means an X11 user session, not `DisplayServer=x11-user`.) If your journal (`journalctl -b -u sddm`) contains `falling back to DisplayServer=x11-user` right before the hang, it may be this one.

## Not verified

Details in [docs/limitations.md](docs/limitations.md). Mainly: other machines and setups, Wayland greeters beyond the 5 logins tested, the full upstream `develop` branch (I tested 0.21.0 plus `da0b64d` only), and any safeguard beyond `da0b64d` (not written).

## Repository layout

```
README.md
docs/      root cause, evidence, limitations, upstream fix analysis
fixes/     solution, upstream fix, workaround
patch/     the upstream commit (da0b64d) and how to build it
logs/      raw traces and journals (hostname and user redacted), see logs/README.md
scripts/   bpftrace scripts used for the traces
```

## About this investigation

This happened on my own laptop. I could have kept using it with the workaround, but I wanted to find the actual root cause, so I kept digging. I did it with the help of Claude (Anthropic's AI assistant), which I used to write the tracing scripts and analyse the results; I ran all the tests on my machine. Everything here comes from one machine and one configuration, so please verify before relying on it.

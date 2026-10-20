# Limitations

## Confirmed on my machine

On one machine and one configuration (Acer Nitro V15, RTX 3050 Optimus + Intel, Garuda Linux, SDDM 0.21.0-7, xorg-server 21.1.25-2, linux-zen 7.2.9.zen1-1), with Hyprland and Plasma (Wayland) sessions:

- The cycle in the `x11-user` fallback is caused by the VT request reaching the greeter's Xorg after its main loop has already left, so it is never served ([root cause](root-cause.md), [evidence](evidence.md)).
- `DisplayServer=x11`, a Wayland greeter (`weston`) and the one-line patch each avoid it; the measurements are in [evidence](evidence.md).
- The cycle is intermittent here: 6 of 8 logins on the unmodified package (control run). The one-line change of #2210 did not stop it (9 of 11 logins).

## Not confirmed

- Other hardware, distributions or SDDM setups, and whether the reports in [sddm/sddm#2129](https://github.com/sddm/sddm/issues/2129) and [#2043](https://github.com/sddm/sddm/issues/2043) have the same mechanism.
- That every login problem I had on this machine before came from this race.
- How often it happens elsewhere. On my machine it was 6 of 8 logins in the control run (and 4 of 4 in the first captures).
- Wayland greeters other than `weston` 15.0.1 (5 logins tested).
- The patch on the full upstream `develop` branch, and a build of [PR #2210](https://github.com/sddm/sddm/pull/2210) or [PR #2204](https://github.com/sddm/sddm/pull/2204) themselves. I tested only the one-line change of #2210 on 0.21.0. For my sessions the expression in #2204 gives the same value, but I did not build it.
- The failure that #2210 describes (a VT signal reaching the session process after `exec`, the helper exiting with the signal number). I did not see it here: in my journals the helper exits with 2 or 255, never with a signal number.
- A proper fix in SDDM: I have not written or tested one. The ideas in [root-cause](root-cause.md#direction-for-a-proper-fix-untested) are untested.
- Why `sleep 1` in PAM helps: my explanation is an inference, not traced.
- The Xorg source links point to the upstream tag `xorg-server-21.1.25`, the installed version. I read `xf86VTRequest` and `xf86Wakeup` there, and `VTsw_usl.c` and `xf86Events.c` have no differences between 21.1.13 and 21.1.25. The Arch package adds one patch on top of the upstream tag (a `damage` revert), which does not touch these files.

## Notes

- The four captures of the cycle and the 11 patched logins were taken on different boots and days, with the same configuration. The first captures (trace2, trace3) used earlier versions of the tracing script, with fewer probes; see [evidence](evidence.md). The 11 patched logins (Oct 18), the 11 logins with the #2210 change (Oct 19–20) and the 8 control logins (Oct 20) are small samples, and only the traced logins show the mechanism; the others are counts from the journal.
- The patch, upstream commit `da0b64d`, was written for a different problem ([#2211](https://github.com/sddm/sddm/issues/2211)). Here it changes the order of events: the greeter's Xorg is still alive when the VT switch is requested.

## How this was made

I did this investigation with the help of Claude (Anthropic's AI assistant), which wrote the tracing scripts and helped analyse the results and the source code. I ran all the tests. The raw logs and scripts are included so the claims can be checked.

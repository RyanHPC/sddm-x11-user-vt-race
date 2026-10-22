# Upstream fix: keep the greeter alive after login

A one-line change in the greeter that is already on SDDM's `develop` branch. On my machine, 0.21.0 plus this single commit did not cycle (0 of 11 logins, against 6 of 8 on the unmodified package). **By my reading of the code it changes the order of events, not only the timing. It was written for another bug, and removes this race as a side effect.**

## The change

```cpp
// src/greeter/GreeterApp.cpp, right after creating QGuiApplication
app.setQuitOnLastWindowClosed(false);
```

This is upstream commit [`da0b64d`](https://github.com/sddm/sddm/commit/da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a) by Daniel Betz. It was written for a different problem ([#2211](https://github.com/sddm/sddm/issues/2211): the greeter exits when the last monitor is unplugged). The full patch is in [`../patch/greeter-keep-alive.patch`](../patch/greeter-keep-alive.patch); it applies cleanly on `v0.21.0`.

No release after `v0.21.0` exists yet (checked in the git tags on Oct 22, 2026); the code path is read in detail in [upstream-fix-analysis](../docs/upstream-fix-analysis.md). The author of [PR #2204](https://github.com/sddm/sddm/pull/2204) says Gentoo's `0.21.0_p20260819` snapshot is built from this commit.

## What it changes

Without it, the greeter exits on its own as soon as the login succeeds, and the greeter's Xorg is terminated right then, at the same time the user session asks for the VT switch (see [root cause](../docs/root-cause.md)).

With it, the greeter stays alive after the login. The daemon stops it 5 s after a successful session start ([`Display.cpp#L545-L549`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Display.cpp#L545-L549)), and the session start is reported only after the user process has started, which is after the VT switch (by my reading of the code). In my traces the switch is served by the live Xorg in 10–27 ms, and the Xorg is terminated 4.8–5.3 s later.

## Independent confirmation (Wayland greeter)

I learned after publishing that the author of [PR #2204](https://github.com/sddm/sddm/pull/2204) reached the same conclusion on a Wayland greeter. In his closing comment he says the greeter used to quit by itself about 100 ms after `LoginSucceeded`, so the VT jump landed in the middle of its teardown, and that with this commit the greeter and its compositor stay up until `slotSessionStarted()` stops them. He closed his own PR for that reason. His log excerpt shows the greeter outliving the VT switch by about 5 s. That is a different greeter and a different machine from mine; it supports the mechanism, it is not a measurement of the `x11-user` fallback.

## Result on my machine

11 logins, 0 cycles, same configuration in which the unmodified package cycled in 6 of 8 logins (control run, a different day) and in 4 of 4 first captures. Details: [evidence](../docs/evidence.md#4-x11-user-fallback--one-line-patch).

## Side effect in the logs

According to the author of PR #2204, with this commit the greeter is always stopped with SIGTERM, so the daemon logs `Greeter stopped.` with `HELPER_OTHER_ERROR` (the helper exits with 3) on every login, where it used to log `HELPER_SUCCESS`. He says the status is ignored, so it is harmless, but it can confuse someone reading the journal. I did not check this in my own logs.

## What it does not do

- It does not change the VT handling. If the greeter exits for another reason before the session start is reported, the same race may still be possible (untested).
- I tested it on 0.21.0 plus this single commit, not on the upstream `develop` branch.
- It changes how the greeter behaves in other situations (for example, when all screens disappear), which is the point of the upstream commit, but I did not test those.

## Build and install

See [`../patch/README.md`](../patch/README.md). To go back to the stock package: `sudo pacman -S sddm`.

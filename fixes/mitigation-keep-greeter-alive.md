# Mitigation: keep the greeter alive after login

A one-line change in the greeter that makes the cycle disappear. **It hides the race; it does not remove it.**

## The change

```cpp
// src/greeter/GreeterApp.cpp, right after creating QGuiApplication
app.setQuitOnLastWindowClosed(false);
```

This is upstream commit [`da0b64d`](https://github.com/sddm/sddm/commit/da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a) by Daniel Betz. It was written for a different problem ([#2211](https://github.com/sddm/sddm/issues/2211): the greeter exits when the last monitor is unplugged). The full patch is in [`../patch/greeter-keep-alive.patch`](../patch/greeter-keep-alive.patch); it applies cleanly on `v0.21.0`.

## What it changes

Without it, the greeter exits on its own as soon as the login succeeds, and the greeter's Xorg is terminated right then, at the same time the user session asks for the VT switch (see [root cause](../docs/root-cause.md)).

With it, the greeter stays alive after the login. The daemon stops it 5 s after a successful session start ([`Display.cpp#L545-L549`](https://github.com/sddm/sddm/blob/v0.21.0/src/daemon/Display.cpp#L545-L549)), long after the VT switch is done. In my traces the switch is served by the live Xorg in 10–27 ms, and the Xorg is terminated 4.8–5.3 s later.

## Result on my machine

11 logins, 0 cycles, same configuration that gave the cycle on the first login every time without the patch. Details: [evidence](../docs/evidence.md#4-x11-user-fallback--one-line-patch).

## Why it is not a fix

- The race is still in the code. It is hidden because the greeter's Xorg now outlives the VT switch by about 5 s; any change that makes the switch slower than that timer, or that stops the greeter earlier, would bring it back.
- It was tested on 0.21.0 plus this single commit, not on the upstream `develop` branch.
- It changes how the greeter behaves in other situations (for example, when all screens disappear), which is the point of the upstream commit, but I did not test those.

## Build and install

See [`../patch/README.md`](../patch/README.md). To go back to the stock package: `sudo pacman -S sddm`.

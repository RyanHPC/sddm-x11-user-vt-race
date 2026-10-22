# Workaround: `sleep 1` in the PAM session stack

A timing workaround. It made the problem go away in my informal tests (several logins, no trace or log saved) but does not address the cause. Prefer the [`DisplayServer=x11` solution](solution-displayserver-x11.md), or the [upstream fix](upstream-fix-greeter-keep-alive.md) once a release includes it.

## The line

In `/etc/pam.d/sddm`, among the `session` lines:

```
session optional pam_exec.so /usr/bin/sleep 1
```

## How I think it works (inference, not traced)

PAM opens the session in the user helper before it jumps to the new VT. A one-second pause there gives the greeter's Xorg (which needs about 100 ms to exit after its SIGTERM) time to be gone first. With no VT owner left, `VT_ACTIVATE` completes immediately instead of waiting for an answer that never comes.

## Cost and risks

- Adds about one second to every login.
- It is timing-based: it hides the race instead of removing it.
- A mistake in a PAM file can break logins. Back it up first and keep a TTY logged in while you test:

  ```sh
  sudo cp /etc/pam.d/sddm /etc/pam.d/sddm.bak
  ```

- `/etc/pam.d/sddm` is a packaged config file: an update can leave a `.pacnew` next to it, and you may need to re-apply the line by hand.

## Result

No cycle in my tests with this line. I did not count those logins, so I do not give a number.

## Revert

```sh
sudo cp /etc/pam.d/sddm.bak /etc/pam.d/sddm
```

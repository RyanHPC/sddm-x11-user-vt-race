# Solution: `DisplayServer=x11`

Run the greeter on an X server owned by the SDDM daemon instead of the `x11-user` fallback.

**This avoids the faulty path; it does not fix SDDM.** It is what worked on my setup (9 logins in the trace, 0 cycles). It is not a guaranteed fix for everyone.

## Why it works

With `DisplayServer=x11` the greeter's Xorg is started and stopped by the daemon. It stays alive during the login, so when the user session asks for the VT switch it answers within 71–110 ms (`xf86VTRequest` → `xf86VTLeave` → `VT_RELDISP`). The daemon stops it only at the end of the session. See [evidence](../docs/evidence.md#2-displayserverx11-greeters-xorg-owned-by-the-daemon).

## Requirements

An Xorg server installed (on Arch it is a dependency of `sddm`).

## Apply

1. Find where `DisplayServer` is set, and keep a backup:

   ```sh
   grep -rn DisplayServer /etc/sddm.conf /etc/sddm.conf.d/ /usr/lib/sddm/sddm.conf.d/ 2>/dev/null
   sudo cp /etc/sddm.conf /etc/sddm.conf.bak     # if you use /etc/sddm.conf
   ```

2. Set it in `/etc/sddm.conf` (or a file in `/etc/sddm.conf.d/`), under `[General]`:

   ```ini
   [General]
   DisplayServer=x11
   ```

3. Restart SDDM. This ends your graphical session, so save your work first (or just reboot):

   ```sh
   sudo systemctl restart sddm
   ```

## Check that it worked

```sh
journalctl -b -u sddm | grep -E "falling back|Session started"
```

You should see `Session started true` after each login and **no** `falling back to DisplayServer=x11-user` line.

## Revert

Put the previous value back (or delete the line, or restore the backup) and restart SDDM:

```sh
sudo cp /etc/sddm.conf.bak /etc/sddm.conf
sudo systemctl restart sddm
```

## Alternative

If you want a Wayland greeter, install its compositor (by default SDDM runs `weston --shell=kiosk`). In my tests a weston greeter did not hit the cycle (5 logins), because the VT owner is `systemd-logind` there. That sample is small; see [limitations](../docs/limitations.md).

# Patch: build SDDM 0.21.0 with the one-line change

[`greeter-keep-alive.patch`](greeter-keep-alive.patch) is upstream commit [`da0b64d`](https://github.com/sddm/sddm/commit/da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a) (author: Daniel Betz, see the header of the file). It applies cleanly on the `v0.21.0` tag. What it does and why it is only a mitigation: [fixes/mitigation-keep-greeter-alive.md](../fixes/mitigation-keep-greeter-alive.md).

This is how I built and tested it on Arch/Garuda. It replaces the system `sddm` package, so use it only if you are comfortable restoring it (the last step does that). A snapshot (for example with snapper) before installing is a good idea.

## Build

1. Get the Arch packaging and make sure it is the version you have installed (`pacman -Q sddm`):

   ```sh
   git clone https://gitlab.archlinux.org/archlinux/packaging/packages/sddm.git
   cd sddm
   git tag | tail -3
   grep -E '^(pkgver|pkgrel)' PKGBUILD
   ```

2. Mark the package as a local build and cherry-pick the commit in `prepare()`:

   ```sh
   sed -i 's/^pkgrel=7$/pkgrel=7.1/' PKGBUILD
   sed -i '/Fix build with cmake 4/a\  git -C $pkgname cherry-pick -n da0b64d8e2378c4bce16cbea6cac2ffc89ad1a4a # test: greeter does not quit on last window closed' PKGBUILD
   sed -n '/^prepare/,/^}/p' PKGBUILD
   ```

   `prepare()` should now contain two `cherry-pick` lines. (Adjust `pkgrel=7` if yours differs.)

3. Build. `-s` installs missing build dependencies, including the Qt5 ones, because the PKGBUILD also builds the Qt5 greeter:

   ```sh
   makepkg -s
   ```

4. Check that the change is in the compiled source:

   ```sh
   grep -n -B1 -A1 setQuitOnLastWindowClosed src/sddm/src/greeter/GreeterApp.cpp
   ```

## Install

```sh
sudo pacman -U sddm-0.21.0-7.1-x86_64.pkg.tar.zst
sudo systemctl restart sddm      # ends your session; or reboot
```

## Go back to the stock package

```sh
sudo pacman -S sddm              # pacman reports a downgrade to 0.21.0-7; that is expected
```

The build dependencies installed by `makepkg -s` stay on the system as orphans; remove them with `pacman -Rns` if you want.

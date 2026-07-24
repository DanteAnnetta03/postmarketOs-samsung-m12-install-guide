# postmarketOS on the Samsung Galaxy M12 — Build & Install Guide

A from-zero, step-by-step walkthrough for building and flashing a
working postmarketOS install on the Samsung Galaxy M12 (SM-M127F,
Exynos 850 / 3830), using the patched device port developed in these
two companion repositories:

- **[pmaports fork](https://github.com/DanteAnnetta03/pmaports)**,
  branch `samsung-m12` (kernel config, device package, install docs):
  the device port itself. See its
  `device/testing/device-samsung-m12/README.md` for *why* every change
  was made, current status, known issues, and the outstanding TODOs
  (Docker, no DRM/GPU/Wayland). This guide only covers *how* to go
  from a clean machine to a booted phone.
- **[pmbootstrap fork](https://github.com/DanteAnnetta03/pmbootstrap)**,
  branch `heimdall-dtbo` (build tool patches): a small set of fixes to
  `pmbootstrap` itself, without which the build in this guide will not
  complete. See its commit log for the exact bugs fixed.

This guide assumes no prior state — a clean Linux machine, a Galaxy
M12, a USB cable, and an SD card.

## About the platform note below

This guide was written and verified on:

```
Ubuntu 24.04.4 LTS, kernel 6.17, x86_64
Python 3.12, git 2.54
```

**This is informational, not a requirement.** Nothing here is
Ubuntu-specific except the exact package manager invocation
(`apt install ...`) in the dependency list — swap that for your
distro's equivalent (`dnf`, `pacman`, `zypper`, etc.) and everything
else applies the same way. pmbootstrap itself runs its actual build
inside an Alpine Linux chroot regardless of your host distro, so the
host OS only needs to provide the tools listed below, nothing more.

## Dependencies

### Host build tools

- `git`
- `python3` (3.9+; developed against 3.12)
- `sudo` access (pmbootstrap needs to create chroots, loop devices,
  and mount/unmount filesystems)
- A C toolchain + `libusb-1.0-0-dev` (or your distro's equivalent) —
  only needed to build `heimdall` from source, see below
- `parted`, `util-linux` (`losetup`, etc.) — pulled in by most base
  installs already; pmbootstrap itself will fail loudly and tell you
  what's missing if not

Debian/Ubuntu:
```sh
sudo apt install git python3 python3-pip build-essential libusb-1.0-0-dev \
    parted util-linux
```

### Phone-communication tools

- **`heimdall` >= 2.2.2, built from source.** Distro-packaged builds
  (e.g. `2.0.2` on Debian/Ubuntu 24.04) fail with libusb errors talking
  to this specific device — build your own:
  ```sh
  git clone https://git.sr.ht/~grimler/Heimdall
  cd Heimdall
  # follow the project's own build instructions (CMake-based)
  ```
- **Odin4** — Samsung's official Linux flashing tool, used to write
  `boot.img`/`dtbo.img`/`vbmeta.img` to the phone's internal
  partitions. Set up its udev rules per its own installer/docs so it
  can see the device in Download Mode without `sudo`.
- **`avbtool`** — used to build a `vbmeta.img` with verification
  disabled (the kernel here isn't signed with Samsung's keys):
  ```sh
  pip install --user avbtool
  ```
- **An SD card + reader on your PC.** Large data transfers to this
  device over USB (both `heimdall` and Odin4) are unreliable — a full
  rootfs image (1.7GB+) consistently fails partway through regardless
  of cable/port. This guide writes the rootfs to an SD card with `dd`
  instead of flashing it to internal storage.

## Step 1 — Get the patched pmbootstrap

```sh
git clone -b heimdall-dtbo https://github.com/DanteAnnetta03/pmbootstrap.git
cd pmbootstrap
```

All `pmbootstrap` commands below are run from inside this directory,
as `python3 -m pmbootstrap ...`.

## Step 2 — Get the pmaports fork with the M12 device port

`pmbootstrap` looks for its pmaports checkout at
`~/.local/var/pmbootstrap/cache_git/pmaports` by default. The M12
device port isn't in official/upstream pmaports, so put the fork there
yourself *before* running `pmbootstrap init`, so it uses this checkout
instead of cloning the official one:

```sh
mkdir -p ~/.local/var/pmbootstrap/cache_git
git clone -b samsung-m12 https://github.com/DanteAnnetta03/pmaports.git \
    ~/.local/var/pmbootstrap/cache_git/pmaports
```

Optionally add the official postmarketOS repo as a second remote,
useful for pulling in upstream fixes later:
```sh
cd ~/.local/var/pmbootstrap/cache_git/pmaports
git remote add upstream https://gitlab.com/postmarketOS/pmaports.git
```

## Step 3 — Initialize pmbootstrap

From the `pmbootstrap` directory:
```sh
python3 -m pmbootstrap init
```
Answer the prompts: vendor `samsung`, codename `m12`, channel
`testing`. For the user interface prompt, pick **none/console** or a
lightweight X11 window manager (e.g. i3) — do not pick anything that
requires GPU acceleration or Wayland; this device's kernel has no DRM
driver (see the pmaports README's "Status" section for details and
implications).

If `pmbootstrap init` complains about the pmaports checkout (branch
mismatch, dirty tree, etc.), that means it's not picking up the fork
from step 2 — double check the path and that channel `testing` exists
on the checked-out branch.

## Step 4 — Build and install

```sh
python3 -m pmbootstrap install
```
This builds `linux-samsung-m12`, `device-samsung-m12`, and every
package the device APKBUILD depends on, and produces (inside the
pmbootstrap work directory, typically
`~/.local/var/pmbootstrap/chroot_native/home/pmos/rootfs/` and
`.../images/`):

- `boot.img` — kernel + initramfs in Android boot image format
- `dtbo.img` — device tree overlay
- a raw rootfs image

This step takes a while (kernel compile + full rootfs). If you hit an
apk/build error here that looks like one of the bugs listed in the
pmbootstrap fork's commit log, that's why the patched fork is a
dependency and not optional.

## Step 5 — Build vbmeta.img

The generated boot image isn't signed with Samsung's keys, so AVB
verification must be explicitly disabled for the partition it targets
(`VBMETA_SYSTEM`, per this device's `deviceinfo`):
```sh
avbtool make_vbmeta_image --flags 2 -o vbmeta.img
```
(`--flags 2` = verification disabled.)

## Step 6 — Read your device's real partition sizes

```sh
heimdall print-pit
```
Partition sizes are given in sectors (× 512 bytes) and **vary by
firmware region/revision** — don't assume round numbers from someone
else's output. `boot.img` from step 4 must fit within the `BOOT`
partition's size. If it doesn't, see the pmaports README's "Why the
kernel config was changed" section — this device port already strips
the camera/ISP driver stack and uses `lzma` initfs compression to make
it fit; if you've added more to the kernel or device package, you may
need to trim further.

## Step 7 — Package boot.img / dtbo.img / vbmeta.img for Odin4

Odin4 flashes a `.tar.md5` — a plain tar archive with a trailing
md5sum line appended:
```sh
tar -H ustar -c boot.img dtbo.img vbmeta.img -f m12-boot.tar
md5sum -t m12-boot.tar >> m12-boot.tar
mv m12-boot.tar m12-boot.tar.md5
```
**Gotcha:** the md5sum line inside the file must reference the short
relative filename (`m12-boot.tar`), not an absolute path — Odin4 fails
to parse the file and rejects it otherwise. Run `md5sum` from inside
the same directory as the tar, as shown above.

## Step 8 — Flash boot/dtbo/vbmeta with Odin4

1. Put the phone in Download Mode (typically: power off, then hold
   Volume Down + Volume Up while connecting the USB cable, confirm
   with Volume Up).
2. Open Odin4, load `m12-boot.tar.md5` in the **AP** slot.
3. Start. This writes `boot.img → BOOT`, `dtbo.img → DTBO`,
   `vbmeta.img → VBMETA_SYSTEM`.

Do **not** attempt to also flash the rootfs through Odin4/heimdall —
that's the large transfer that's unreliable on this device. Use the SD
card instead (next step).

## Step 9 — Write the rootfs to an SD card

From your PC, with the SD card inserted in a reader:
```sh
lsblk   # identify the SD card's device node — double check this!
sudo dd if=<rootfs image from step 4> of=/dev/sdX bs=4M status=progress
sync
```
**This overwrites the entire card.** Confirm the device node with
`lsblk` before running `dd` — an unrelated disk will be silently
destroyed if you get this wrong.

`deviceinfo_external_storage="true"` is already set in this device
port, so the phone looks for a bootable rootfs on external (SD)
storage automatically.

## Step 10 — First boot

1. Insert the SD card into the phone.
2. Power on.
3. If the device reboots within a few seconds of reaching userspace
   in a loop, the `watchdog` service isn't running — this should not
   happen with an unmodified copy of this device package (it's
   enabled by default specifically to prevent this), but if you've
   customized the package, check `rc-status` once you get any shell
   access.
4. Log in (default postmarketOS on-device setup wizard / SSH, per
   however you configured `pmbootstrap init`).

### Quick sanity checklist

- Boots without rebooting in a loop.
- Console text is upright, not sideways (screen rotation applied at
  boot).
- `nmcli device` shows `wlan0` as `disconnected`/`connected`, not
  `unmanaged`.
- `loadkeys`/`consolefont` services show as started (`rc-status`).

If any of these fail, check the pmaports README's "Known issues" and
"Why the device package was changed" sections first — most boot-time
problems with this port have already been diagnosed there.

## What this guide does not cover

Desktop environment setup (window manager, shell, wallpaper, etc.) is
not device-specific once the device boots reliably — any generic
postmarketOS/Alpine guide applies from here on. Camera and Docker
support are known-incomplete; see the pmaports README's TODO section.

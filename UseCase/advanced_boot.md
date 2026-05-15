# Also read
[Installing Debian GNU/Linux from a Unix/Linux System](https://www.debian.org/releases/forky/amd64/apds03.en.html)

# Advanced Debian Boot And Minimal Install

These notes are for learning Linux in a system/server admin style instead of a normal desktop-user install flow.

## Debian Installer Image Directory

Debian installer directories such as:

```text
https://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/
```

Example top-level structure:

```text
MANIFEST
MANIFEST.udebs
MD5SUMS
SHA256SUMS
cdrom/
hd-media/
netboot/
udeb.list
```

This page is not a single `.iso` download because Debian exposes the installer building blocks:

- kernel
- initrd
- netboot files
- tiny boot images
- cdrom installer images
- manifests

This is useful for sysadmin workflows such as PXE boot, rescue boot, GRUB boot, serial console installs, and automated installs.

Top-level files:

| File | Purpose |
| --- | --- |
| `MANIFEST` | summary of installer image files |
| `MANIFEST.udebs` | installer component package list |
| `MD5SUMS` | old checksum list for integrity checks |
| `SHA256SUMS` | stronger checksum list for integrity checks |
| `udeb.list` | Debian installer `.udeb` component list |

Top-level directories:

| Folder | Purpose |
| --- | --- |
| `cdrom/` | normal ISO installer images |
| `netboot/` | PXE, tiny installer, kernel/initrd network boot |
| `hd-media/` | install using files stored on a disk |

`netboot/` is the most sysadmin-like path. It is small, flexible, and close to how servers, VMs, rescue systems, and bare-metal provisioning work.

## Install Without A USB Stick

If an existing Linux system still boots, you can download the Debian installer kernel and initrd to disk, then boot them from GRUB.

```sh
# create a place GRUB can read
sudo mkdir -p /boot/debian-installer

# download the Debian installer kernel
sudo curl -L -o /boot/debian-installer/linux https://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux

# download the Debian installer initrd
sudo curl -L -o /boot/debian-installer/initrd.gz https://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz
```

Then reboot, open the GRUB command line, and boot those local files:

```sh
# load the installer kernel from the disk partition GRUB can see
linux (hd0,gpt2)/boot/debian-installer/linux

# load the installer initrd
initrd (hd0,gpt2)/boot/debian-installer/initrd.gz

# boot the installer
boot
```

Adjust `(hd0,gpt2)` to match the partition that contains `/boot`.

GRUB itself usually cannot use `curl` or `wget` like Linux can. The normal pattern is:

```text
existing Linux or rescue shell downloads files
GRUB boots the local downloaded files
Debian installer starts in RAM
installer configures network and downloads packages
```

## If The OS Is Broken

This method can help when the installed OS cannot boot, but only in some cases.

If only GRUB works, the installer kernel and initrd must already be stored on a disk partition that GRUB can read.

```text
only GRUB works
-> installer files already exist under /boot/debian-installer/
-> GRUB can boot those local files
-> Debian installer starts
```

If only GRUB works and the installer files were not downloaded earlier, this usually will not work by itself. GRUB is a bootloader, not a normal Linux shell, so it usually cannot fetch files with `curl` or `wget`.

If the system drops into an initramfs or BusyBox shell, it might still be possible to recover without USB, but only if that environment has working network tools and drivers.

```text
OS fails before normal boot
-> initramfs shell appears
-> network tools exist
-> download installer files to /boot
-> reboot
-> boot them from GRUB
```

Minimal initramfs environments often lack `curl`, `wget`, Wi-Fi firmware, or full networking tools. Wired ethernet is much more likely to work than Wi-Fi.

Best recovery preparation is to download the installer files while the OS still works.

```sh
# create a recovery installer directory
sudo mkdir -p /boot/debian-installer

# download the installer kernel before the system breaks
sudo curl -L -o /boot/debian-installer/linux https://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux

# download the installer initrd before the system breaks
sudo curl -L -o /boot/debian-installer/initrd.gz https://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz
```

Quick rule:

```text
GRUB only + installer files already saved
-> likely usable

GRUB only + no installer files
-> usually need USB, PXE, or another rescue boot

initramfs only + working network tools
-> maybe usable

initramfs only + no network tools
-> usually need USB, PXE, or another rescue boot
```

## How Internet Works Before The OS Is Installed

When the installer boots, it is already running a tiny Linux system in RAM:

```text
UEFI/BIOS
-> GRUB
-> Linux kernel
-> initrd/initramfs
-> temporary RAM filesystem
-> installer
```

That temporary environment can load drivers, start networking, use DHCP, and download packages before the final installed OS exists.

Wired ethernet is usually easier than Wi-Fi in minimal installers because Wi-Fi often needs firmware and extra configuration.

## GRUB Loopback ISO Boot

Another useful trick is booting an ISO already stored on disk.

```sh
# make an ISO visible to GRUB as a loop device
loopback loop (hd0,gpt2)/debian.iso

# load the installer kernel from inside the ISO
linux (loop)/install.amd/vmlinuz

# load the installer initrd from inside the ISO
initrd (loop)/install.amd/initrd.gz

# boot the installer
boot
```

This can help when you have no USB stick but can still place an ISO on a local disk.

## Initramfs Learning

`initramfs` is the temporary Linux environment that runs before the real root filesystem starts.

Boot chain:

```text
UEFI/BIOS
-> GRUB
-> kernel
-> initramfs
-> real root filesystem
-> systemd
```

For Debian or Ubuntu systems, the best learning method is to pause boot inside initramfs from GRUB.

At the GRUB menu:

1. Highlight the normal boot entry.
2. Press `e`.
3. Find the line starting with `linux`.
4. Add `break=premount` or `break=mount` at the end.
5. Press `Ctrl+x` or `F10` to boot.

Example kernel line ending:

```text
ro quiet break=premount
```

You should land in an `(initramfs)` or BusyBox shell.

Useful commands inside initramfs:

```sh
# show the kernel command line
cat /proc/cmdline

# list detected block devices
ls /dev

# show filesystem and UUID information
blkid

# show kernel messages
dmesg

# inspect initramfs boot scripts
ls /scripts
```

To continue booting:

```sh
# leave the initramfs shell and continue boot
exit
```

On RHEL, Fedora, Rocky, or other dracut-based systems, use `rd.break` instead of Debian's `break=premount`.

## Initramfs Versus Rescue Shell

`break=premount` enters the real initramfs before the root filesystem mounts. This is best for learning boot internals.

`init=/bin/bash` is different. It skips normal userspace and starts Bash on the real root filesystem. That is useful for password resets or config repair, but it is not the same as learning initramfs.

## Learning Progression

A practical sysadmin learning path:

1. Install Debian from `netinst.iso` with no GUI.
2. Learn `tty`, SSH, `apt`, `systemctl`, `journalctl`, and networking.
3. Boot into initramfs with `break=premount`.
4. Practice GRUB command-line booting.
5. Try booting installer kernel/initrd from disk.
6. Learn `chroot` recovery.
7. Later try PXE, iPXE, preseed, `debootstrap`, and automated installs.

These skills map directly to server recovery, bare-metal provisioning, cloud VM bootstrap, Kubernetes node work, and infrastructure automation.

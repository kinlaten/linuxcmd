# Fix Slow Boot Caused by a Stale `RESUME` UUID on Debian

This note documents a real Debian boot issue where the system spent a long time in the early boot path because `initramfs-tools` was configured to resume from a swap UUID that no longer existed.

The machine used `zram` for swap, but `initramfs-tools` was still told to search for an old persistent swap device during boot.

## Symptom

The boot looked slow before normal userspace startup:

- `systemd-analyze` showed a very large `kernel` phase
- `systemd-analyze critical-chain` did not show a matching userspace bottleneck
- the root filesystem mounted only after a large delay

Example:

```text
Startup finished in 21.880s (firmware) + 1.209s (loader) + 35.641s (kernel) + 8.111s (userspace) = 1min 6.843s
```

In this case, the biggest number was the `kernel` phase, not userspace.

## Why This Happens

On Debian, `initramfs-tools` can be configured to look for a resume device during early boot.

If `/etc/initramfs-tools/conf.d/resume` points to a swap UUID that no longer exists, the system may waste time probing for a device that will never appear.

This is especially common after changes such as:

- removing a swap partition
- moving from a real swap partition to `zram`
- changing disk layout
- reinstalling while preserving some config files

## Evidence That Pointed to This Root Cause

### 1. Userspace was not the main problem

`systemd-analyze blame` showed some later boot delays, but nothing close to the `35s` kernel phase.

### 2. USB was a distraction, not the main blocker

USB devices appeared later in the boot log, after the root filesystem was already mounted.

That meant USB warnings were not the primary reason for the long early-boot delay.

### 3. The active resume config pointed to a missing UUID

The system had:

```bash
# Show the current initramfs resume configuration.

cat /etc/initramfs-tools/conf.d/resume
```

It originally contained:

```text
RESUME=UUID=65199b7a-9007-4e63-9978-745ecc242288
```

But the active swap setup was:

```bash
# Show active swap devices.

swapon --show

# Show disks, partitions, filesystems, and UUIDs.

lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINTS,UUID
```

And the only active swap device was `zram`, not a real block device with that UUID.

## How to Confirm the Problem

Run these checks:

```bash
# Show total boot timing.

systemd-analyze

# Show whether the delay is mainly in userspace or elsewhere.

systemd-analyze critical-chain

# Show the active kernel command line.

cat /proc/cmdline

# Show every RESUME setting that initramfs or GRUB may still read.

grep -R "^RESUME=" /etc/initramfs-tools/conf.d /etc/default/grub 2>/dev/null

# Show active swap devices.

swapon --show

# Show block devices and UUIDs so you can verify whether the RESUME UUID exists.

lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINTS,UUID
```

You likely have this problem if:

- `systemd-analyze` shows a large `kernel` phase
- `swapon --show` only shows `zram`
- `grep` shows `RESUME=UUID=...`
- that UUID does not exist in `lsblk`

## One-Boot Test

Before changing files permanently, you can test once with `noresume`.

1. Reboot.
2. In GRUB, press `e`.
3. Find the `linux` line.
4. Append `noresume`.
5. Boot once with that temporary edit.

If the next `systemd-analyze` output drops sharply in the `kernel` phase, the resume configuration was very likely the cause.

## Permanent Fix for a `zram`-Only System

If the machine uses `zram` instead of a real hibernation-capable swap device, disable resume probing.

```bash
# Back up the existing resume config.

sudo cp /etc/initramfs-tools/conf.d/resume /etc/initramfs-tools/conf.d/resume.bak

# Disable resume probing for initramfs-tools.

echo 'RESUME=none' | sudo tee /etc/initramfs-tools/conf.d/resume
```

Important:

- `initramfs-tools` may read every file in `/etc/initramfs-tools/conf.d/`
- if `resume.bak` stays in that directory, the old stale UUID may still be read

Move the backup out of the config directory:

```bash
# Move the backup somewhere initramfs-tools will not parse.

sudo mv /etc/initramfs-tools/conf.d/resume.bak /root/resume.bak

# Confirm the active configuration now only contains RESUME=none.

grep -R "^RESUME=" /etc/initramfs-tools/conf.d /etc/default/grub 2>/dev/null
```

Rebuild the initramfs:

```bash
# Rebuild the initramfs for the current kernel.

sudo update-initramfs -u
```

Expected result:

```text
/etc/initramfs-tools/conf.d/resume:RESUME=none
```

And `update-initramfs -u` should no longer warn about a missing swap UUID.

## Optional Extra Guardrail

If you want to force the same behavior at the kernel command line, add `noresume` to GRUB.

```bash
# Edit GRUB defaults.

sudoedit /etc/default/grub
```

Example:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet noresume"
```

Then apply it:

```bash
# Regenerate the GRUB config.

sudo update-grub
```

This is optional if `RESUME=none` is already working, but it can be a useful extra safeguard.

## Reboot and Verify

After applying the permanent fix:

```bash
# Reboot the machine.

sudo reboot
```

After the next login:

```bash
# Check whether the kernel phase dropped.

systemd-analyze

# Check the slowest userspace units after the early-boot problem is fixed.

systemd-analyze blame | head -n 20

# Confirm whether any remaining boot bottleneck is now in userspace.

systemd-analyze critical-chain
```

What you want to see:

- a much smaller `kernel` time
- the boot bottleneck moves from early boot to later userspace services
- any remaining delays become normal tuning targets such as networking or optional daemons

## If You Need Hibernation Later

`RESUME=none` is correct for a system that only uses `zram` swap and does not hibernate.

If you want hibernation in the future, you need:

- a real persistent swap partition or swap file
- a correct `RESUME=` target that points to it
- a rebuilt initramfs after changing the configuration

`zram` alone is not a valid hibernation resume target.

## Summary

This boot issue was not mainly caused by `systemd`, USB, or storage wait services.

The root cause was a stale `initramfs-tools` resume configuration:

- old `RESUME=UUID=...`
- no matching swap device
- only `zram` active

The fix was:

1. set `RESUME=none`
2. move any stale backup file out of `/etc/initramfs-tools/conf.d/`
3. rebuild initramfs
4. reboot and verify with `systemd-analyze`

## References

- Debian `initramfs-tools` documentation: <https://manpages.debian.org/bookworm/initramfs-tools-core/initramfs-tools.7.en.html>
- Debian initramfs overview: <https://wiki.debian.org/initramfs>

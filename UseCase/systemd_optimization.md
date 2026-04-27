# systemd Boot Optimization

This is a modernized Markdown rewrite of the historical upstream `systemd` optimization page, adjusted for a Debian-style system.

The original page is still useful as a source of ideas, but parts of it are old, Fedora-specific, or too aggressive for a modern general-purpose machine. This version keeps the good parts, adds safer command-driven guidance, and calls out where the original advice no longer fits current Debian setups.

## Profile First

Do not optimize boot blindly. Measure first, then change one thing at a time.

```bash
# Show total firmware, loader, kernel, and userspace boot time.

systemd-analyze

# Show the units that consumed the most startup time.

systemd-analyze blame | head -n 20

# Show the critical dependency chain that delayed boot completion.

systemd-analyze critical-chain

# Show unit relationships when you need to understand why something started.

systemctl show -p Wants,Requires,After,Before multi-user.target
```

## Practical Optimizations

### 1. Avoid boot-critical LVM or other slow storage layers when you do not need them

This is still one of the highest-impact ideas in the original document.

Why it matters:

- Storage stacks such as LVM, mdraid, iSCSI, FCoE, and multipath can pull extra services into the early boot path.
- Those services may indirectly require device-settle behavior or add storage probing work that slows boot.
- On Debian, the exact unit names differ from the old Fedora examples, so use inspection commands first instead of copying Fedora service names.

Do not do this if your root filesystem or `/boot` depends on LVM, mdraid, iSCSI, multipath, or similar storage.

#### Check whether boot-critical storage uses LVM or device-mapper

```bash
# Show the block device that backs the root filesystem.

findmnt -no SOURCE /

# Show the block device that backs /boot, if it exists as a separate mount.

findmnt -no SOURCE /boot 2>/dev/null

# Show disks, partitions, LVM logical volumes, and mountpoints in one view.

lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINTS

# Show physical volumes known to LVM.

sudo pvs

# Show volume groups known to LVM.

sudo vgs

# Show logical volumes and the devices behind them.

sudo lvs -o vg_name,lv_name,lv_path,devices
```

What to look for:

- If `/` or `/boot` resolves to `/dev/mapper/...`, `/dev/dm-*`, or an LVM path like `/dev/<vg>/<lv>`, your system is using device-mapper or LVM for boot-critical storage.
- If that is true, stop here and do not remove or mask storage-related components just to speed up boot.

#### Check whether `systemd-udev-settle` or storage helper units are part of the boot path

```bash
# Show whether the legacy settle unit exists and whether it is active.

systemctl status systemd-udev-settle.service --no-pager

# Show which units pull in systemd-udev-settle.service, if any.

systemctl list-dependencies --reverse systemd-udev-settle.service --no-pager

# List storage-related units that commonly affect early boot on Debian systems.

systemctl list-unit-files | grep -E 'systemd-udev-settle|lvm2|mdadm|iscsi|multipath'

# Show whether storage-related packages are installed.

dpkg -l | grep -E '^(ii)\s+(lvm2|mdadm|open-iscsi|multipath-tools)\b'
```

Typical outcomes:

- `systemd-udev-settle.service` may be absent, disabled, or never used. That is good.
- A reverse dependency list that mentions `lvm2`, `mdadm`, `open-iscsi`, or `multipath` tells you where the boot delay may be coming from.

#### If your system does not use those storage layers for boot, remove unused packages instead of copying old Fedora masks

```bash
# Remove LVM support if your system does not use LVM anywhere you care about.

sudo apt purge lvm2

# Remove mdraid support if your system does not use software RAID.

sudo apt purge mdadm

# Remove iSCSI support if your system does not boot from or use iSCSI.

sudo apt purge open-iscsi

# Remove multipath support if your system does not use SAN or multipath storage.

sudo apt purge multipath-tools
```

Safer than direct masking:

- On Debian, package removal is usually clearer than masking a random mix of units left behind by a storage stack you do not use.
- Masking only makes sense when you know exactly which unit is unnecessary and which package still needs to remain installed.

If you need the package installed for non-boot use, inspect the specific units first:

```bash
# Inspect the unit file for a storage-related service before disabling or masking it.

systemctl cat lvm2-monitor.service

# Check whether a unit is enabled before changing it.

systemctl is-enabled lvm2-monitor.service
```

Do not mask `systemd-udev-settle.service` blindly. If something still depends on it for your storage setup, masking it directly can break boot or leave devices unavailable.

#### Reboot and measure again

```bash
# Re-check overall boot time after making one change set.

systemd-analyze

# Re-check the slowest units after the reboot.

systemd-analyze blame | head -n 20

# Re-check the critical chain to confirm the bottleneck moved or disappeared.

systemd-analyze critical-chain
```

### 2. Be very cautious about bypassing the initramfs

The original page recommends bypassing the initrd entirely. That is now a niche optimization, not general advice.

On modern Debian systems, the initramfs often handles:

- early microcode loading
- encrypted root
- LVM and RAID activation
- filesystem and resume hooks
- hardware discovery needed before the real root mounts

Recommendation:

- Keep the initramfs unless you are building a tightly controlled appliance and fully understand the boot chain.
- If you suspect initramfs work is slow, profile it first instead of removing it blindly.

### 3. Keep security tradeoffs explicit

The original page suggests disabling SELinux and auditing. On Debian, SELinux is usually not the default MAC layer, and AppArmor is more common.

Recommendation:

- Do not disable security features just to chase small boot gains unless you are building a dedicated appliance and have accepted the risk.
- Treat security-off boot tuning as a last resort, not a default optimization.

### 4. Consider whether Plymouth is worth it

If your system boots quickly, a splash screen may not add much value.

Recommendation:

- If you want the fastest visible path to login, test with Plymouth disabled.
- If you care about polished user-facing boot, keep it.

This is a policy choice, not a universal win.

### 5. Prefer journald unless you have a reason to keep a classic syslog daemon

This part of the original advice still holds up well.

Recommendation:

- If you do not specifically need `rsyslog` or another syslog daemon, consider removing it.
- If you want logs to survive reboot, make the journal persistent.

```bash
# Create the persistent journal directory if it does not already exist.

sudo mkdir -p /var/log/journal

# Restart journald so it picks up persistent storage immediately.

sudo systemctl restart systemd-journald
```

### 6. Disable services you do not actually need

This is still one of the best general optimizations.

Recommendation:

- Review the output of `systemd-analyze blame`.
- Review enabled services.
- Disable user-visible or optional daemons that do not belong on your system.

```bash
# List enabled unit files.

systemctl list-unit-files --state=enabled

# List currently running service units.

systemctl --type=service --state=running
```

Examples that are sometimes removable, depending on the machine:

- printing services on systems without printers
- Bluetooth on systems that never use it
- local MTA packages on desktops
- remote storage packages on machines with only local disks

### 7. Keep console noise low while measuring

Console output slows boot and makes timing less consistent.

Recommendation:

- Use `quiet` on the kernel command line for production-style measurements.
- Avoid enabling verbose `systemd` debug logging unless you are actively debugging a boot issue.

### 8. Use systemd timers where they make operational sense

The original document is outdated here. Modern `systemd` timers do support calendar schedules through `OnCalendar=`.

Recommendation:

- Use timers when they fit your service management model.
- Keep cron if it is already simple and reliable for your use case.

This is no longer a feature-gap argument. It is mostly a consistency and operational-style choice.

### 9. Appliance-specific optimizations still matter

These are still valid, but mostly for controlled images, not general desktops:

- build required drivers directly into the kernel when you control the hardware
- remove optional `systemd` components you genuinely do not need
- keep the default boot target minimal
- avoid loading modules and services you know the appliance will never use

### 10. Hardware-specific tuning should stay hardware-specific

The original page mentions `libahci.ignore_sss=1`.

Recommendation:

- Treat kernel command-line storage tweaks as hardware-specific experiments.
- Test thoroughly before documenting them as standard practice.

### 11. Avoid unnecessary local mail stacks

If the machine is a desktop, kiosk, or appliance, a local MTA is often unnecessary.

Recommendation:

- Remove mail server packages you do not need.
- Keep them only when local delivery, relay, or cron mail actually matters.

### 12. Fewer boot units usually means a faster boot

This remains true regardless of distro.

Recommendation:

- Audit what starts at boot.
- Remove packages you do not need.
- Disable services you do not need.
- Re-measure after each change.

### 13. Do not benchmark with debug kernels

The original warning still makes sense.

Recommendation:

- Compare boot times on production-style kernels and production-style configs.
- Do not treat development or debug-heavy builds as representative performance numbers.

## Notes on Outdated Parts of the Original Page

These points in the historical source should be treated carefully on modern Debian systems:

- Fedora-specific service names such as `fedora-wait-storage.service` do not apply directly to Debian.
- The timer limitation is outdated. Modern `systemd` timers support `OnCalendar=`.
- `ConsoleKit` and references such as GNOME 3.4 are historical.
- Readahead-specific suggestions are largely historical and not a primary modern tuning lever on SSD-based systems.
- Bypassing the initramfs is now a specialized optimization for controlled systems, not normal advice.

## Upstream Engineering Ideas from the Original Page

The original document also listed ideas for improving `systemd` itself rather than tuning one machine. Those ideas are still useful as design directions, even if some details are dated:

- reduce synchronous work in early unit loading and dependency graph construction
- replace shell-heavy boot logic with native units
- prefer `Type=simple` or `Type=notify` over `Type=forking` when practical
- use socket activation or on-demand activation where appropriate
- reduce boot-time process churn
- improve I/O behavior for boot-critical paths
- push optional work later or activate it only when needed

For an end user or Debian administrator, the practical value is simple:

- profile first
- remove unneeded storage layers
- keep the boot path small
- avoid unnecessary services
- change one variable at a time and measure again

## References

- Historical upstream optimization notes: <https://github.com/systemd/systemd/blob/main/docs/OPTIMIZATIONS.md>
- Dependency inspection tips: <https://github.com/systemd/systemd/blob/main/docs/TIPS_AND_TRICKS.md>
- Current `systemd` timer capabilities: `OnCalendar=` is supported in modern `systemd`

# Files
## Prepare PDF for A4 printing with `ghostscript`

`gs` (`Ghostscript`) is very good at rewriting PDFs for printing, resizing pages, rasterizing pages, and doing batch document conversions.

Fit an existing PDF to A4 and write a new PDF:

```sh
# Rewrite a PDF so pages are fitted to A4

/usr/bin/gs \
  -o "Cloud-FinOps-A4.pdf" \
  -sDEVICE=pdfwrite \
  -sPAPERSIZE=a4 \
  -dFIXEDMEDIA \
  -dPDFFitPage \
  "Cloud FinOps{Fuller, Mike}(2023){105237891} libgen.li.pdf"
```

What the important flags do:

- `-sDEVICE=pdfwrite` writes a new PDF
- `-sPAPERSIZE=a4` sets the target paper size to A4
- `-dFIXEDMEDIA` forces the output page size to the selected paper size
- `-dPDFFitPage` scales each page to fit the target size
- `-o output.pdf` is short for setting the output file and running in batch mode

This is useful when:

- a PDF was made for `letter` paper and you want cleaner A4 printing
- a scanned or oddly-sized PDF prints cropped on A4 printers
- you want a separate print-friendly copy without editing the original

Merge multiple PDFs into one:

```sh
# Combine several PDFs into one file

gs -dBATCH -dNOPAUSE -sDEVICE=pdfwrite \
  -sOutputFile=merged.pdf \
  part1.pdf part2.pdf part3.pdf
```

Render a page to an image preview:

```sh
# Render only the first page to JPEG at 72 DPI

gs -dBATCH -dNOPAUSE -dFirstPage=1 -dLastPage=1 \
  -sDEVICE=jpeg -dJPEGQ=85 -r72 \
  -sOutputFile=preview.jpg \
  input.pdf
```

Other things `gs` does very well:

- rewrite or normalize broken or awkward PDFs for printing
- convert PDF or PostScript to raster images such as `png`, `jpeg`, or `tiff`
- merge many PDFs into one output file
- make grayscale print copies
- generate lower-resolution previews or thumbnails
- convert PostScript to PDF and PDF to PostScript for old print workflows

Common strong use cases:

- printer-friendly copy of a PDF for A4 or letter paper
- preview image generation for documents
- preparing PDFs for office printers that dislike unusual page boxes or embedded content
- batch conversion of PostScript or PDF documents

Niche but useful cases:

- producing `tiffg4` output for fax-like workflows
- generating separations or print-oriented intermediate files for legacy prepress tools
- rendering pages to exact raster sizes for OCR or downstream image pipelines

Check available devices and general usage:

```sh
# Show supported output devices and common switches

gs -h
```

## Set file immutable
```sh
# add immutable attribute
chattr +i <file>

# To change that file
# make it mutable
chattr -i <file>

# Make file only appendable
chattr +a <file>
```

# Packages
## Inspection
```sh
sudo dpkg -i --simulate <NAME.deb>
sudo apt install <NAME.deb> --dry-run

# Check what it needs
apt-cache depends <NAME.deb>
# Check if versions match your system
apt-cache policy lib-required-package

# Control file (Depends/Conflicts)
dpkg-deb -I <NAME.deb>          

# File list (check overwrites)
dpkg-deb -c <NAME.deb>          
```

# Coreutils
## `dd` use cases
Use `dd` for low-level byte copying between files and block devices.

- Clone a disk or partition into an image file
- Write an ISO image to a USB drive
- Create a file with a fixed size
- Fill a file with zeroes for testing
- Copy raw data with block-size control

Examples:

```sh
# Create a 1 GiB file filled with zeroes
dd if=/dev/zero of=test.img bs=1M count=1024 status=progress
```

```sh
# Write an ISO image to a USB drive
sudo dd if=debian.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

```sh
# Back up a full disk to an image file
sudo dd if=/dev/sdX of=disk.img bs=64K status=progress
```

```sh
# Restore a disk image back to a drive
sudo dd if=disk.img of=/dev/sdX bs=64K status=progress conv=fsync
```

```sh
# Create a zero-filled file until the filesystem is full
dd if=/dev/zero of=zero.fill bs=1M status=progress
rm zero.fill
```

Important options:

- `if=` input file or device
- `of=` output file or device
- `bs=` block size
- `count=` number of blocks to copy
- `status=progress` show progress while copying
- `conv=fsync` flush output before exit
- `conv=noerror,sync` continue past read errors and pad bad blocks
- `skip=` skip input blocks
- `seek=` skip output blocks

Warning: `dd` will overwrite exactly the target you give it. Double-check `of=` before running it. For damaged disks, prefer `ddrescue` instead of `dd`.

# Windows Troubleshooting

## InitRamFs cannot detect Windows partition after reinstall linux partition
1. USB boot to Windows iso
2. Open terminal Shift + F10
3. 
```sh
diskpart
list vol

# (EFI)
sel vol X
assign letter=S

# (Windows NTFS)
sel vol Y
assign letter=C
# If there is other volume currently use letter C, use : sel vol Z -> remove letter=C
bcdboot C:\Windows /s S: /f UEFI
```

Exit, reboot to boot menu

# Kubernetes
Test traffic in cluster
```sh
# Create a curl pod
kubectl run -it --rm curltest -n monitoring --image=curlimages/curl:8.8.0 --restart=Never -- sh

curl -vI http://<svc>.<namespace>.svc.cluster.local:<port>/<path>
```

# System

## Check laptop battery

Use `/sys/class/power_supply` first. It is simple and already available on a normal Linux system.

```sh
# List power devices to find the battery name
ls /sys/class/power_supply

# Show battery percent
cat /sys/class/power_supply/BAT0/capacity

# Show charging status
cat /sys/class/power_supply/BAT0/status
```

Example output:

```text
ACAD  BAT0
84
Discharging
```

Note:

- common battery names are `BAT0` or `BAT1`
- if `BAT0` does not exist, check the name from `ls /sys/class/power_supply`

If `upower` is already installed on the desktop image, it gives a fuller summary:

```sh
# List power devices
upower -e

# Show battery details
upower -i /org/freedesktop/UPower/devices/battery_BAT0
```

Example output:

```text
  native-path:          BAT0
  state:                discharging
  percentage:           84%
  time to empty:        3.8 hours
```

## Backup system with Timeshift on `ext4`

For a desktop on `ext4`, use `Timeshift` in `rsync` mode. This is the practical choice when you want to back up the Linux system for rollback without restructuring disks for `LVM` snapshots.

What this is good for:

- backs up the OS, packages, boot files, and system configuration
- works well on `ext4`
- keeps restore simple on a desktop

What this is not for:

- not your main backup for personal files
- not the lowest-overhead snapshot method

Install it:

```sh
# Install Timeshift
sudo apt install timeshift
```

Use `rsync` mode for `ext4`:

```sh
# Create an on-demand snapshot in rsync mode
sudo timeshift --create --rsync --comments "before risky change" --scripted
```

List snapshots:

```sh
# List available snapshots
sudo timeshift --list --rsync
```

Example output:

```text
Device : /dev/sdb1
UUID   : 1234abcd-56ef-7890-abcd-1234567890ef
Path   : /timeshift

Num     Name                 Tags  Description
------------------------------------------------------------------------------
0    >  2026-04-29_20-15-42  O     before risky change
```

Keep it system-focused, not user-space:

- in the GUI or config, keep home directories excluded unless you intentionally want hidden config files included
- do not treat `Timeshift` as the only backup for `/home`
- store snapshots on a separate disk or partition when possible

The main config file is `/etc/timeshift/timeshift.json`.

Example exclude/include idea:

```
{
  "exclude": [
    "/home/*/**",
    "+ /root/**"
  ]
}
```

Restore a snapshot:

```sh
# Interactive restore of a selected snapshot
sudo timeshift --restore --rsync
```

Restore a specific snapshot non-interactively:

```sh
# Replace the snapshot name and target with your values
sudo timeshift --restore --rsync --snapshot '2026-04-29_20-15-42' --target /dev/sda2 --yes
```

If the system will not boot:

1. Boot a live USB.
2. Install or start `Timeshift` in the live environment.
3. Mount the snapshot disk if needed.
4. Run `sudo timeshift --restore` and select the target root partition.
5. Reinstall `GRUB` during restore if the bootloader is damaged.

Useful note: `Timeshift` snapshots are for system rollback. Use a separate backup tool for documents, media, and other personal data.

### Timer recurring backup
Use a oneshot service plus a timer.

`/etc/systemd/system/timeshift-weekly.service`
```ini
[Unit]
Description=Create weekly Timeshift snapshot

[Service]
Type=oneshot
ExecStart=/usr/bin/timeshift --create --rsync --scripted --tags W --comments "Weekly Saturday snapshot"
```

`/etc/systemd/system/timeshift-weekly.timer`
```ini
[Unit]
Description=Run Timeshift every Saturday at 02:00

[Timer]
OnCalendar=Sat 02:00
Persistent=true

[Install]
WantedBy=timers.target
```

Then enable it:

```sh
# Reload unit files
sudo systemctl daemon-reload

# Enable and start the timer
sudo systemctl enable --now timeshift-weekly.timer
```

Check it:

```sh
# Show when it will run next
systemctl list-timers timeshift-weekly.timer

# Show timer status
systemctl status timeshift-weekly.timer
```

## No grub menu time
Modify file /etc/default/grub

GRUB_TIMEOUT=0
GRUB_TIMEOUT_STYLE=hidden

## Change `sudo` password timeout

On Debian, change the `sudo` credential cache timeout in `sudoers`.

Edit it safely:

```sh
# Open sudoers in visudo
sudo visudo
```

Add or update this line:

```text
Defaults timestamp_timeout=15
```

Useful values:

- `15` keeps the default-style cache for 15 minutes
- `0` asks for the password every time you run `sudo`
- `-1` disables timeout expiration for the cache

Example:

```text
Defaults timestamp_timeout=0
```

Note: on Debian, `sudo` typically uses a per-terminal timestamp, so commands from the same terminal reuse the cached authentication until the timeout expires.

## Change default editor to `vim`

On Debian, the usual system-wide way is to point the `editor` alternative to `vim`.

Install `vim` if needed:

```sh
# Install vim
sudo apt install vim
```

Select it as the default editor:

```sh
# Choose the command used by /usr/bin/editor
sudo update-alternatives --config editor
```

Example output:

```text
There are 3 choices for the alternative editor (providing /usr/bin/editor).

  Selection    Path            Priority   Status
------------------------------------------------------------
* 0            /usr/bin/nano    40        auto mode
  1            /usr/bin/nano    40        manual mode
  2            /usr/bin/vim.tiny 15       manual mode
  3            /usr/bin/vim.basic 30      manual mode

Press Enter to keep the current choice[*], or type selection number: 3
```

Optional: make your shell prefer `vim` too:

```sh
# Add to ~/.bashrc for user shell sessions
export EDITOR=/usr/bin/vim
export VISUAL=/usr/bin/vim
```

## Remap keys with `keyd`

On Debian, `keyd` is a system-wide remapping daemon. It works across X11, Wayland, and TTY because it remaps keys below the desktop session layer.

Install and enable it:

```sh
# Install keyd
sudo apt install keyd

# Enable and start the daemon
sudo systemctl enable --now keyd
```

Put your config in `/etc/keyd/default.conf`.

Example: swap left `Ctrl`, left `Alt`, and left `Super`, and make `CapsLock` act as `Esc` when tapped and `Control` when held. `Shift+CapsLock` stays normal `CapsLock`.

```ini
[ids]
*

[main]
leftcontrol = leftalt
leftalt = leftmeta
leftmeta = leftcontrol
capslock = overload(control, esc)

[shift]
capslock = capslock
```

Check and reload it:

```sh
# Validate the config
sudo keyd.rvaiya check /etc/keyd/default.conf

# Reload config without rebooting
sudo keyd.rvaiya reload
```

Monitor keys to discover names and debug mappings:

```sh
# Show key presses and device IDs
keyd.rvaiya monitor
```

Example output:

```text
DEVICE: 1452:832
leftalt down
leftalt up
capslock down
capslock up
```

Check logs if a config does not apply:

```sh
# Show recent keyd service logs
sudo journalctl -eu keyd
```

Notes:

- Debian provides the command as `keyd.rvaiya`
- the main config path is `/etc/keyd/default.conf`
- because `keyd` is system-wide, it is better than `xmodmap` for i3 and Wayland sessions
- if you use `chezmoi`, track the source in your home directory and copy or symlink it into `/etc/keyd/default.conf`

## Rotate logs with `logrotate`

`logrotate` keeps log files from growing forever. It can rotate logs by time or size, compress old logs, keep only a fixed number of old files, and run a command after rotation.

Common use cases:

- rotate app logs such as `/var/log/myapp.log`
- compress old logs to save disk space
- keep only the last `N` rotated logs
- reload a daemon after its log file is rotated

On Debian, the main config is `/etc/logrotate.conf` and package or app snippets usually live in `/etc/logrotate.d/`.

Basic example:

```conf
/var/log/myapp.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 0640 root adm
}
```

Useful directives:

- `daily`, `weekly`, `monthly`
- `rotate 7` keeps 7 old logs
- `compress` gzip-compresses old logs
- `missingok` skips missing files without error
- `notifempty` skips empty logs
- `create 0640 root adm` creates a new empty log file after rotation
- `size 100M` rotates when the file grows past `100 MB`
- `copytruncate` is useful for apps that keep the file open and cannot reopen logs cleanly

Example with service reload:

```conf
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    sharedscripts
    postrotate
        systemctl reload nginx
    endscript
}
```

Test it safely:

```sh
# Dry run without making changes
sudo logrotate --debug /etc/logrotate.conf

# Show what a real run does
sudo logrotate --verbose /etc/logrotate.conf

# Force rotation now
sudo logrotate --force /etc/logrotate.conf
```

Notes:

- `logrotate` is usually run automatically by cron or `systemd`
- use it for plain log files on disk
- if a service already logs only to `journald`, you may not need `logrotate` for that service

## Mount a remote filesystem with `sshfs`

Use `sshfs` when you want to mount a remote directory over SSH and browse it like a local filesystem.

Basic mount:

```sh
# Create a mountpoint and make it owned by your user

sudo mkdir -p /mnt/shared_data
sudo chown "$USER:$USER" /mnt/shared_data

# Mount a remote directory (hp-home is configured in ~/.ssh)
sshfs hp-home:/mnt/zen_data /mnt/shared_data
```

If you get `fusermount3: user has no write access to mountpoint`, the mountpoint is usually owned by `root` and must be owned by the user running `sshfs`.

Unmount it:

```sh
# Unmount an sshfs mount

fusermount3 -u /mnt/shared_data
```

Useful options:

- `-o reconnect` reconnect automatically if the SSH session drops
- `-o IdentityFile=/home/ed/.ssh/zenkin-cluster` uses a specific SSH key
- `-o allow_other` lets other local users access the mount
- `-o default_permissions` enables local permission checks, useful with `allow_other`

Example with common options:

```sh
# Mount with reconnect and a specific SSH key

sshfs hp-home:/mnt/zen_data /mnt/shared_data \
  -o reconnect \
  -o IdentityFile=/home/ed/.ssh/zenkin-cluster
```

To keep this persistent after boot, use system `systemd` `.mount` and `.automount` units.

Note:

- `After=network-online.target` and `Wants=network-online.target` belong on the `.mount` unit, not the `.automount` unit
- `.automount` is useful so boot does not block waiting for the remote host
- for `/mnt/shared_data`, the unit names must match the path:
  `mnt-shared_data.mount` and `mnt-shared_data.automount`

Create `/etc/systemd/system/mnt-shared_data.mount`:
```ini
[Unit]
Description=SSHFS mount for /mnt/zen_data from hp-home
Wants=network-online.target
After=network-online.target

[Mount]
What=hp-home:/mnt/zen_data
Where=/mnt/shared_data
Type=fuse.sshfs
Options=IdentityFile=/home/ed/.ssh/zenkin-cluster,reconnect,_netdev

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/mnt-shared_data.automount`:
```ini
[Unit]
Description=Automount for /mnt/shared_data

[Automount]
Where=/mnt/shared_data
TimeoutIdleSec=10min

[Install]
WantedBy=multi-user.target
```

Enable it:

```sh
# Reload systemd after adding the unit files

sudo systemctl daemon-reload

# Enable and start the automount unit

sudo systemctl enable --now mnt-shared_data.automount
```

Test it:

```sh
# Accessing the directory should trigger the mount

ls /mnt/shared_data

# Check automount and mount status

systemctl status mnt-shared_data.automount
systemctl status mnt-shared_data.mount
```

Notes:

- using a mountpoint under your home directory is simpler, but system mount units are clearer when you want a system-managed automount
- if you use `allow_other`, you will usually also want `default_permissions`
- if the remote host is not reachable during boot, `.automount` is safer than mounting immediately
- if your SSH key requires a passphrase, the system mount may need a non-interactive key setup to succeed automatically

## Target for suspend

Change the power button action to `suspend-then-hibernate` in `/etc/systemd/logind.conf`.

```ini
[Login]
HandlePowerKey=suspend-then-hibernate
```

Change the hibernate delay in `/etc/systemd/sleep.conf`.

```ini
[Sleep]
HibernateDelaySec=30m
```

Apply the change:

```sh
# Restart logind after changing the power button action
sudo systemctl restart systemd-logind

# Optional: verify the configured values in drop-ins and main config
sudo systemd-analyze cat-config systemd/logind.conf
sudo systemd-analyze cat-config systemd/sleep.conf
```

Example output:

```text
# /etc/systemd/logind.conf
[Login]
HandlePowerKey=suspend-then-hibernate

# /etc/systemd/sleep.conf
[Sleep]
HibernateDelaySec=30m
```

Note: `HibernateDelaySec=` is only used by `suspend-then-hibernate`.

## Create kvm
Dependency: qemu-kvm libvirt-daemon-system virt-manager python3

```sh
# Add user to libvirt group
sudo usermod -aG libvirt $USER

# 
sudo virt-manager
```

## Set swap to 16GB
### Swap file (disk)
```sh
sudo swapoff --all
sudo rm -rf /swapfile
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# sudo echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
sudo swapon --show
```

### ZRAM (compressed RAM)

On Debian/Ubuntu:

```sh
# Install zram generator
sudo apt install systemd-zram-generator
```

Create `/etc/systemd/zram-generator.conf`:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
```

For a system with `30GB` RAM, `zram-size = ram / 2` gives about `15GB` of compressed swap.

Start it:

```sh
# Reload systemd generators after adding the zram config
sudo systemctl daemon-reexec

# Start the zram swap device
sudo systemctl start systemd-zram-setup@zram0

# Verify zram swap is active
swapon --show
```

Example output:

```text
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition  15G   0B  100
```

Note: `compression-algorithm = zstd` requires kernel support for `zstd` in zram.

## Config input device

```sh
# /etc/X11/xorg.conf/00-input.conf
Section "InputClass"
  Identifier "system-keyboard"
  MatchIsKeyboard "true"
  Option "XkbOptions" "caps:escape_shifted_capslock"
EndSection

Section "InputClass"
        Identifier "Pointer"
        MatchIsPointer "true"
        #MatchIsTouchpad "true" #for laptop
        Driver "libinput"
        Option "LeftHanded" "true"
        Option "Tapping" "true"
        Option "NaturalScrolling" "true"
EndSection
```

# Systemd
## Boot to BIOS
```sh
# Reboot once directly into UEFI/BIOS setup.
sudo systemctl reboot --firmware-setup
```

## Keep journald logs across boots

Set `Storage=persistent` in `/etc/systemd/journald.conf` so `journald` keeps logs from previous boots.

```ini
[Journal]
Storage=persistent
```

`journald` stores logs in:

- persistent storage: `/var/log/journal/`
- volatile storage: `/run/log/journal/`

Default behavior:

- if `/var/log/journal/` exists at boot, logs are kept persistently
- otherwise `journald` uses `/run/log/journal/`, which is lost after reboot

Create the persistent journal directory if needed:

```sh
# Create the persistent journal directory with correct ownership and modes
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

Check where logs are stored and how much space they use:

```sh
# Show journal file locations
ls -lah /var/log/journal /run/log/journal

# Show total journal disk usage
journalctl --disk-usage
```

List prior boots:

```sh
# Show boot indexes and boot IDs stored in the journal
journalctl --list-boots
```

Example output:

```text
-1 a73415fade0e4e7f4bea60913883d180dc Fri 2016-02-26 15:01:25 UTC Fri 2016-02-26 15:05:16 UTC
 0 0c563fa3830047ecaa2d2b053d4e661d Fri 2016-02-26 15:11:03 UTC Fri 2016-02-26 15:12:28 UTC
```

Read logs from a prior boot by index or boot ID:

```sh
# Show logs from the previous boot
journalctl -b -1

# Show logs from a specific boot ID
journalctl -b a73415fade0e4e7f4bea60913883d180dc
```

Filter logs to a specific `systemd` unit:

```sh
# Show logs only for the ntp service
journalctl -u ntp
```

Example output:

```text
-- Logs begin at Fri 2016-02-26 15:11:03 UTC, end at Fri 2016-02-26 15:26:07 UTC. --
Feb 26 15:11:07 ub-test-1 systemd[1]: Stopped LSB: Start NTP daemon.
Feb 26 15:11:08 ub-test-1 systemd[1]: Starting LSB: Start NTP daemon...
Feb 26 15:11:08 ub-test-1 ntp[761]: * Starting NTP server ntpd
```

## AppArmor audit log search

For AppArmor or audit troubleshooting:

- use `ausearch` to search the audit log
- use `auditctl` to check `auditd` status or audit rules, not as the main log search tool
- use `journalctl` to search the journal
- even if some audit events appear in the journal, `ausearch` is usually better for audit log work

```sh
# Search AppArmor-related audit events
sudo ausearch -m AVC,USER_AVC,APPARMOR -ts recent

# Check auditd status
sudo auditctl -s

# Search the journal for AppArmor messages
sudo journalctl -g apparmor -b
```

Example output:

```text
----
time->Fri May  1 10:14:21 2026
type=APPARMOR msg=audit(1777605261.123:412): apparmor="DENIED" operation="open" class="file" profile="/usr/bin/foo" name="/etc/bar.conf" pid=1234 comm="foo" requested_mask="r" denied_mask="r" fsuid=1000 ouid=0

enabled 1
failure 0
pid 742
rate_limit 0
backlog_limit 8192
lost 0
backlog 0

May 01 10:14:21 host kernel: audit: type=1400 audit(1777605261.123:412): apparmor="DENIED" operation="open" class="file" profile="/usr/bin/foo" name="/etc/bar.conf" pid=1234 comm="foo" requested_mask="r" denied_mask="r" fsuid=1000 ouid=0
```

Note:

- `ausearch` output is the better source when you want audit event details
- `journalctl` is useful for quick correlation with the rest of the boot or service logs

## Autoboot into a user, no login require
For `lightdm`:

```bash
sudoedit /etc/lightdm/lightdm.conf
```

Add:

```ini
[Seat:*]
autologin-user=ed
autologin-user-timeout=0
```

Then:

```bash
sudo systemctl restart lightdm
```

## Host container with podman

Use Podman Quadlet by creating a `.container` file. `systemd` will turn it into a user service.

1. Install Podman

```sh
# Update apt metadata
sudo apt update

# Install Podman
sudo apt install podman
```

2. Create the data and Quadlet directories

```sh
# Create the Uptime Kuma data directory
mkdir -p ~/.local/share/uptime-kuma

# Create the user Quadlet directory
mkdir -p ~/.config/containers/systemd
```

3. Create `~/.config/containers/systemd/uptime-kuma.container`

```ini
[Unit]
Description=Uptime Kuma container

[Container]
Image=docker.io/louislam/uptime-kuma:latest
ContainerName=uptime-kuma
PublishPort=3001:3001
Volume=%h/.local/share/uptime-kuma:/app/data:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

4. Reload the user `systemd` daemon

```sh
# Reload user services so Quadlet picks up the new container definition
systemctl --user daemon-reload
```

5. Enable and start the service

```sh
# Start the container as a user service and enable it on login
systemctl --user start uptime-kuma.service
# enable keywork not work
```


Example output:

```text
Created symlink /home/ed/.config/systemd/user/default.target.wants/uptime-kuma.service -> /home/ed/.config/containers/systemd/uptime-kuma.container.
```

6. Check service and container status

```sh
# Show the user service status
systemctl --user status uptime-kuma.service

# List running Podman containers
podman ps
```

Example output:

```text
● uptime-kuma.service - Uptime Kuma container
     Loaded: loaded (/home/ed/.config/containers/systemd/uptime-kuma.container; generated)
     Active: active (running)

CONTAINER ID  IMAGE                                  COMMAND     CREATED         STATUS         PORTS                   NAMES
4f3c9a2d5c61  docker.io/louislam/uptime-kuma:latest  /usr/bin/..  12 seconds ago  Up 12 seconds  0.0.0.0:3001->3001/tcp  uptime-kuma
```

Open `http://localhost:3001`.

Important: keep it running after logout.

```sh
# Allow user services to keep running after you log out
sudo loginctl enable-linger $USER
```

Useful commands:

```sh
# Follow the service logs
journalctl --user -u uptime-kuma.service -f

# Restart the service
systemctl --user restart uptime-kuma.service

# Stop the service
systemctl --user stop uptime-kuma.service
```

For a server, I would usually place it behind a reverse proxy later, but this is the clean basic Quadlet setup.

# Android debug bridge - adb, gradle
## Security

```sh
# Install the Android app in debug mode
./gradlew :androidApp:installDebug
```

Example output:

```text
BUILD SUCCESSFUL in 1m 12s
```

```sh
# Show package metadata, including version and install timestamps
adb shell dumpsys package com.kebsoft.vpn | grep -E "versionCode|versionName|firstInstallTime|lastUpdateTime|installerPackageName"
```

Example output:

```text
versionCode=42
versionName=1.4.0
firstInstallTime=2026-04-15 10:12:03
lastUpdateTime=2026-04-15 10:14:11
installerPackageName=com.android.vending
```

## Force Stop and Relaunch App

```sh
# Force-stop the app process
adb shell am force-stop com.kebsoft.vpn
```

Example output:

```text
# no output
```

```sh
# Launch the app via Android's monkey tool
adb shell monkey -p com.kebsoft.vpn 1
```

Example output:

```text
Events injected: 1
```

## Check Connected Device

```sh
# List connected devices with detail
adb devices -l
```

Example output:

```text
List of devices attached
R52N30ABCDX    device usb:1-1 product:panther model:Pixel_7 device:panther transport_id:1
```

## Tap a Button by Coordinates or UI Bounds

Use this when a button is off-screen, hard to reach, or easier to locate from the Android UI hierarchy.

```sh
# Confirm the device screen size
adb shell wm size
```

Example output:

```text
Physical size: 1080x2400
```

The first number is the width and the second number is the height. On a `1080x2400` device, the horizontal center is around `540`.

```sh
# Swipe upward to reveal lower content
adb shell input swipe 500 1800 500 500 500
```

Example output:

```text
# no output
```

The coordinates are `start-x start-y end-x end-y duration-ms`. This example starts near the lower part of the screen and swipes upward.

If the button is visible, tap it by coordinates:

```sh
# Tap a visible button by screen coordinates
adb shell input tap X Y
```

Example for a bottom button on a `1080x2400` device:

```sh
# Tap near the lower center of the screen
adb shell input tap 540 2200
```

Example output:

```text
# no output
```

For a more reliable method, dump the UI hierarchy and search for likely button text:

```sh
# Dump the current UI hierarchy to the device
adb shell uiautomator dump /sdcard/window.xml
```

Example output:

```text
UI hierarchy dumped to: /sdcard/window.xml
```

```sh
# Pull the UI hierarchy file to the local machine
adb pull /sdcard/window.xml /tmp/window.xml
```

Example output:

```text
/sdcard/window.xml: 1 file pulled, 0 skipped. 0.1 MB/s (15624 bytes in 0.102s)
```

```sh
# Search for common button text in the UI hierarchy
grep -i -E "accept|terms|agree|continue" /tmp/window.xml
```

Example output:

```xml
<node index="0" text="Accept" resource-id="" class="android.widget.Button" bounds="[80,2030][1000,2140]" />
```

The `bounds` value is `[left,top][right,bottom]`. Tap the center point of the bounds. For `[80,2030][1000,2140]`, the center is:

```sh
# Tap the center of the detected button bounds
adb shell input tap 540 2085
```

Example output:

```text
# no output
```

If the screen supports keyboard or DPAD focus, try moving focus and pressing enter:

```sh
# Move focus forward and press enter
adb shell input keyevent KEYCODE_TAB
adb shell input keyevent KEYCODE_TAB
adb shell input keyevent KEYCODE_ENTER
```

Example output:

```text
# no output
```

Alternative DPAD navigation:

```sh
# Move focus down and select the focused item
adb shell input keyevent KEYCODE_DPAD_DOWN
adb shell input keyevent KEYCODE_DPAD_DOWN
adb shell input keyevent KEYCODE_DPAD_CENTER
```

Example output:

```text
# no output
```

Fast sequence:

```sh
# Swipe up, dump the UI tree, pull it locally, and search for the target button
adb shell input swipe 500 1800 500 500 500
adb shell uiautomator dump /sdcard/window.xml
adb pull /sdcard/window.xml /tmp/window.xml
grep -i -E "accept|agree|terms|continue" /tmp/window.xml
```

## Check Android SDK Version

```sh
# Print the Android SDK API level on the device
adb shell getprop ro.build.version.sdk
```

Example output:

```text
34
```

## Check USB Debugging Setting

```sh
# Read the USB debugging setting from secure settings
adb shell settings get secure adb_enabled
```

Expected values:

```sh
# 1 = USB debugging enabled
# 0 = USB debugging disabled
```

Example output:

```text
1
```

## Check Trust / Lock Runtime State

```sh
# Dump trust-agent and device trust state
adb shell dumpsys trust
```

Example output:

```text
deviceLocked=0
TrustAgents:
  com.google.android.gms/.trustagent.GoogleTrustAgent
```

`deviceLocked=0` means the phone is currently unlocked. It does not prove a PIN/password is configured.

## Check Lock Credential Configuration

```sh
# Dump lock settings service state and configuration
adb shell dumpsys lock_settings
```

Important fields for “no secure lock”:

```sh
# Secure Mode: 0
# Quality: 0
# CredentialType: None
```

Example output:

```text
Secure Mode: 0
Quality: 0
CredentialType: None
```

## Clear Logcat Before Manual Test

```sh
# Clear logcat before running a manual test
adb logcat -c
```

Example output:

```text
# no output
```

## Get App Process ID

```sh
# Print the PID for the app process
adb shell pidof com.kebsoft.vpn
```

Example output:

```text
29535
```

## Capture Focused App Logs by PID

Replace `29535` with the PID from the previous command:

```sh
# Replace 29535 with the PID from the previous command
adb logcat -v time --pid=29535
```

Example output:

```text
04-15 10:18:22.103 D VpnManager(29535): Tunnel state changed: CONNECTING
04-15 10:18:22.487 I WireGuard/GoBackend/wg0(29535): Device started
```

## Capture Focused WireGuard / App Logs for 20 Seconds

```sh
# Capture logs for 20 seconds with the selected tags
timeout 20s adb logcat -v time WireGuard/GoBackend/wg0:D VpnManager:D AndroidRuntime:E System.err:W System.out:I '*:S'
```

During this window, press Connect in the app.

If WireGuard starts, logs look like:

```text
WireGuard/GoBackend/wg0: Attaching to interface tun0
WireGuard/GoBackend/wg0: Device started
VpnManager: Tunnel state changed: UP
```

Example output:

```text
04-15 10:20:01.001 D WireGuard/GoBackend/wg0: Attaching to interface tun0
04-15 10:20:01.204 I WireGuard/GoBackend/wg0: Device started
04-15 10:20:01.322 D VpnManager: Tunnel state changed: UP
```

If the device-access policy blocks correctly, WireGuard should not start.

## Stop ADB Server if Logcat Gets Stuck

```sh
# Stop the adb server if logcat is stuck
adb kill-server
```

Example output:

```text
# no output
```

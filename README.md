# Files
## Set file immutable
```sh
chattr +i <file> # add immutable attribute

# To change that file
chattr -i <file> # make it mutable

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

sel vol X #(EFI)
assign letter=S

sel vol Y #(Windows NTFS)
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

## No grub menu time
Modify file /etc/default/grub

GRUB_TIMEOUT=0
GRUB_TIMEOUT_STYLE=hidden

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

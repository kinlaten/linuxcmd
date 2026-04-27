```sh
ed@ed-pc:~/gitRepo/linuxcmd$ systemd-analyze

# Show the units that consumed the most startup time.

systemd-analyze blame | head -n 20

# Show the critical dependency chain that delayed boot completion.

systemd-analyze critical-chain

# Show unit relationships when you need to understand why something started.

systemctl show -p Wants,Requires,After,Before multi-user.target
Startup finished in 22.043s (firmware) + 1.229s (loader) + 5.912s (kernel) + 8.096s (userspace) = 37.280s
graphical.target reached after 7.919s in userspace.
5.745s NetworkManager-wait-online.service
 572ms tailscaled.service
 550ms NetworkManager.service
 212ms dev-nvme0n1p2.device
 207ms user@1000.service
 205ms containerd.service
 187ms cups.service
 160ms systemd-udev-trigger.service
 130ms lvm2-monitor.service
 128ms udisks2.service
 114ms wtmpdb-update-boot.service
 103ms dev-zram0.swap
  98ms virtlockd.service
  97ms virtlogd.service
  92ms systemd-journal-flush.service
  82ms plymouth-start.service
  79ms ModemManager.service
  78ms accounts-daemon.service
  74ms ssh.service
  70ms systemd-journald.service
The time when unit became active or started is printed after the "@" character.
The time the unit took to start is printed after the "+" character.

graphical.target @7.919s
└─multi-user.target @7.919s
  └─getty.target @7.919s
    └─getty@tty1.service @7.919s
      └─plymouth-quit-wait.service @7.885s +32ms
        └─systemd-user-sessions.service @7.877s +5ms
          └─remote-fs.target @7.841s
            └─remote-fs-pre.target @7.841s
Requires=basic.target
Wants=libvirt-guests.service systemd-homed.service cups-browsed.service lm-sensors.service plymouth-quit.service libvirtd.service cron.ser>
Before=shutdown.target graphical.target
After=rustdesk.service basic.target nix-daemon.socket lm-sensors.service plymouth-quit-wait.service systemd-logind.service cups-browsed.se>
ed@ed-pc:~/gitRepo/linuxcmd$
```

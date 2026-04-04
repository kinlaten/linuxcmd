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

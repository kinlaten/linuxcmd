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

## Create kvm
Dependency: qemu-kvm libvirt-daemon-system virt-manager

```sh
# Add user to libvirt group
sudo usermod -aG libvirt $USER

# 
sudo virt-manager
```

## Set swap to 16GB

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

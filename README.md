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

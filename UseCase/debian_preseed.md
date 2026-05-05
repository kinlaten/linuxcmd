# Debian preseed

[Preseed example](https://www.debian.org/releases/stable/example-preseed.txt)

Use `debconf-utils` to discover real debconf keys and validate syntax.

```sh
# Install helper tools
sudo apt install debconf-utils

# Dump Debian Installer answers after a manual install
sudo debconf-get-selections --installer > installer.answers

# Dump installed package debconf answers
sudo debconf-get-selections > package.answers

# Search for attributes you want to preseed
grep -E 'time/zone|passwd/|grub-installer/' installer.answers

# Validate preseed syntax
debconf-set-selections -c preseed.cfg
```

Note:

- use the dump as reference, not as the final preseed file
- copy only the needed keys into your curated `preseed.cfg`
- start from the official example for the same Debian release
- `--installer` is for Debian Installer questions
- plain `debconf-get-selections` is for installed package questions

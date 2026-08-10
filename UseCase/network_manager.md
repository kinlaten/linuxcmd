```sh
#show
nmcli connection

nmcli connection show <SSID>
```
# Wifi
```sh
#disable wifi power save
nmcli connection modify <SSID> 802-11-wireless.powersave 2
```
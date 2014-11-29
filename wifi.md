## WiFi setup mini guide

```shell
wpa_passphrase network_name password > wifi.conf
ip link list # remember name of the interface
wpa_supplicant -i ifname -cwifi.conf &
dhcpcd
## Hopefully the internet connection now works.
```

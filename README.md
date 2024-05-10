[OpenWRT on Ubiquiti UniFi UAP-AC-PRO (Updated)](./Flashing-Ubiquiti-UniFI-AP-AC-with-OpenWRT.md)

```bash
#Determine Linux install time
fs=$(df /boot | tail -1 | cut -f1 -d' ') && tune2fs -l $fs | grep 'Filesystem created'
```


```bash
#HTTP Request / Response
HTTP_PORT=8000
tcpdump -A -s 10240 'tcp port $HTTP_PORT and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' | egrep --line-buffered "^........(GET |HTTP\/|POST |HEAD )|^[A-Za-z0-9-]+: " | sed -r 's/^........(GET |HTTP\/|POST |HEAD )/\n\1/g'
```

[OpenWRT on Ubiquiti UniFi UAP-AC-PRO (Updated)](./Flashing-Ubiquiti-UniFI-AP-AC-with-OpenWRT.md)

**Determine Linux install time** 
```bash
fs=$(df /boot | tail -1 | cut -f1 -d' ') && tune2fs -l $fs | grep 'Filesystem created'
```

**View REQUEST / RESPONSE headers using TCPDUMP** 
```bash
#HTTP Request / Response
HTTP_PORT=8000
tcpdump -A -s 10240 'tcp port '"$HTTP_PORT"' and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' | egrep --line-buffered "^........(GET |HTTP\/|POST |HEAD )|^[A-Za-z0-9-]+: " | sed -r 's/^........(GET |HTTP\/|POST |HEAD )/\n\1/g'
```

**Transfer files from `A` (`172.21.1.2`) to `B` (`172.21.1.3`) using netcat and gzip compression on port `7000`.** 
Optionally uses pipeviewer (pv), which'll probably need installed if you're on Ubuntu (`apt-get install pv`)

```bash
## On machine A: 
`cd src-dir/`

`tar -zc * | pv | nc 172.21.1.3 7000`

## On machine B: 
`cd dst-dir/`

`nc -l 7000 | pv | tar -xz`
```


Lower case

a="HI ALL"

tr
```bash
echo "$a" | tr '[:upper:]' '[:lower:]'
hi all
```
awk
```bash
echo "$a" | awk '{print tolower($0)}'
hi all
```

[OpenWRT on Ubiquiti UniFi UAP-AC-PRO (Updated)](./Flashing-Ubiquiti-UniFI-AP-AC-with-OpenWRT.md)

```bash
#Determine Linux install time
fs=$(df /boot | tail -1 | cut -f1 -d' ') && tune2fs -l $fs | grep 'Filesystem created'
```

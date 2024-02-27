# Installing OpenWRT on Ubiquiti UniFi AP AC

UniFi is cool... I guess

This guide walks you through the process of flashing your Ubiquiti UniFi AP AC with [OpenWRT](https://openwrt.org/)

## Prerequisites

- A Ubiquiti UniFi AP AC device.
- A computer with OpenSSH installed.
- Internet access to download necessary files.

## Overview

In the past, installing OpenWRT on your Ubiquiti UniFi AP AC required downgrading the device's firmware to version ≤ 3.7.58 which included an `mtd` utility. This is no longer possible and newer firmware restricts downgrading.
Now - the downgrade step has been replaced with `mtd` installation so we can install OpenWRT directly, just needs some dependencies. 

## Flashing

DEVICE_IP is `192.168.1.20`\
USER: `ubnt`\
PASS: `ubnt`

### Step 1: Start with factory defaults

If you didn't just pull the device out of the factory sealed box, Follow [UniFi's official reset guide](https://help.ui.com/hc/en-us/articles/205143490-UniFi-How-to-Reset-Devices-to-Factory-Defaults).\
**TLDR;** Using a pin / paper clip - hold reset for 10 seconds

### Step 2: Confirm SSH reachability

If you've plugged the device into your DHCP enabled network, it will have picked up an IP from the DHCP server, so find it's IP and use that instead.\
***Note:*** Not ideal - as OpenWRT will boot with a DHCP server enabled on the LAN port by default, and that could be screwy for your network.

Verify SSH access to your device:
```bash
ssh -o HostKeyAlgorithms=+ssh-rsa ubnt@DEVICE_IP
```
If you get a password prompt, `<ctrl c>` to return to your local machine, and continue to step 3.


### Step 3: Download Required Files into a local tmp directory

Download the OpenWRT `sysupgrade` image, `mtd` utility, and dependencies on your computer:\
(versions and support may have changed since this was written)
```bash
mkdir -p /tmp/openwrt && cd /tmp/openwrt
wget https://downloads.openwrt.org/releases/21.02.0/targets/ath79/generic/packages/mtd_26_mips_24kc.ipk
wget https://downloads.openwrt.org/releases/21.02.0/targets/ath79/generic/packages/libc_1.1.24-3_mips_24kc.ipk
wget https://downloads.openwrt.org/releases/21.02.0/packages/mips_24kc/base/libubox20210516_2021-05-16-b14c4688-2_mips_24kc.ipk
```
**Note:** The following `sysupgrade` image is for OpenWRT 23.05.2 and the UAP-AC-PRO model. Find yours [firmware selector](https://firmware-selector.openwrt.org/)
```bash
wget https://downloads.openwrt.org/releases/23.05.2/targets/ath79/generic/openwrt-23.05.2-ath79-generic-ubnt_unifiac-pro-squashfs-sysupgrade.bin
```

### Step 4: Transfer Files to the Device

Use `scp` to copy the downloaded files to the device.\
Or use `scp -O` if you encounter an `ash: /usr/libexec/sftp-server: not found` error ([newer ssh clients](https://www.openssh.com/txt/release-9.0))

```bash
scp /tmp/openwrt/* ubnt@DEVICE_IP:/tmp/
```

### Step 5: Install OpenWRT

Finally, SSH into the device and issue the following commands to install OpenWRT (copy and paste):
```bash
mkdir /tmp/flash
cd /tmp/flash
tar -xzOf /tmp/libc_1.1.24-3_mips_24kc.ipk ./data.tar.gz | tar -xz
tar -xzOf /tmp/mtd_26_mips_24kc.ipk ./data.tar.gz | tar -xz
tar -xzOf /tmp/libubox20210516_2021-05-16-b14c4688-2_mips_24kc.ipk ./data.tar.gz | tar -xz
firmwarefile=$(ls /tmp/openwrt*squashfs-sysupgrade.bin)
LD_LIBRARY_PATH=${PWD}/lib lib/ld-musl-mips-sf.so.1 ./sbin/mtd write $firmwarefile kernel0
LD_LIBRARY_PATH=${PWD}/lib lib/ld-musl-mips-sf.so.1 ./sbin/mtd erase kernel1
dd if=/dev/zero bs=1 count=1 of=/dev/mtd4
reboot
```

After rebooting, your device will boot into OpenWRT, accessible at http://192.168.1.1, with a DHCP server running on the LAN port. WiFi is not configured by default.

## Enjoy!

OpenWRT installed (hopefully!)
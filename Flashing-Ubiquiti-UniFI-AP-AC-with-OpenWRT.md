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

**This does not work on every device / firmware combination.** Ubiquiti has progressively locked flash partitions across the 5.x and 6.x firmware lines, and some hardware revisions (notably `UAP-AC-Pro-Gen2`) use a different partition layout than the one below. Step 5 tells you how to find out *before* you touch the flash. See [Troubleshooting](#troubleshooting) if `mtd` refuses to write.

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
(versions and support may have changed since this was written — see [Choosing package versions](#choosing-package-versions))
```bash
mkdir -p /tmp/openwrt && cd /tmp/openwrt
B=https://downloads.openwrt.org/releases/24.10.8
wget $B/targets/ath79/generic/packages/mtd_26_mips_24kc.ipk
wget $B/targets/ath79/generic/packages/libc_1.2.5-r4_mips_24kc.ipk
wget $B/targets/ath79/generic/packages/libgcc1_13.3.0-r4_mips_24kc.ipk
wget $B/packages/mips_24kc/base/libubox20240329_2025.07.23~49056d17-r1_mips_24kc.ipk
```
`mtd` is dynamically linked against **three** libraries — `libubox.so.<date>`, `libgcc_s.so.1` and `libc.so`. Older versions of this guide omitted `libgcc1` and got away with it because the stock UniFi firmware happens to ship `libgcc_s.so.1` in `/lib`; bundling it removes that assumption.

> ⚠️ **Do not use OpenWRT 23.05.0–23.05.2 or 24.10.0–24.10.2 on Ubiquiti hardware.**
> Both ranges shipped a bug that leaves the flash **read-only** on many UniFi devices, so you can flash it once and then never sysupgrade again. Fixed in 23.05.3 and 24.10.3 respectively — see the [OpenWRT Ubiquiti page](https://openwrt.org/toh/ubiquiti/common). Pick a current release instead.

**Note:** The following `sysupgrade` image is for the UAP-AC-PRO model. Find yours with the [firmware selector](https://firmware-selector.openwrt.org/) and substitute the URL.
```bash
wget https://downloads.openwrt.org/releases/24.10.8/targets/ath79/generic/openwrt-24.10.8-ath79-generic-ubnt_unifiac-pro-squashfs-sysupgrade.bin
```

### Step 4: Transfer Files to the Device

Use `scp` to copy the downloaded files to the device.\
Or use `scp -O` if you encounter an `ash: /usr/libexec/sftp-server: not found` error ([newer ssh clients](https://www.openssh.com/txt/release-9.0))

```bash
scp -O -o HostKeyAlgorithms=+ssh-rsa /tmp/openwrt/* ubnt@DEVICE_IP:/tmp/
```

### Step 5: Check the partition layout **before** flashing

SSH back into the device and look at what your board actually has:
```bash
cat /proc/mtd
```
A UAP-AC-Pro (Gen1) looks like this:
```
dev:    size   erasesize  name
mtd0: 00060000 00010000 "u-boot"
mtd1: 00010000 00010000 "u-boot-env"
mtd2: 00790000 00010000 "kernel0"
mtd3: 00790000 00010000 "kernel1"
mtd4: 00020000 00010000 "bs"
mtd5: 00040000 00010000 "cfg"
mtd6: 00010000 00010000 "EEPROM"
```
You need `kernel0`, `kernel1` and `bs` to be present. **If they are not, stop** — the commands in step 6 do not apply to your board, and blindly writing to a partition index is how you brick the device. Go to [Troubleshooting](#troubleshooting).

### Step 6: Install OpenWRT

Issue the following commands to install OpenWRT (copy and paste):
```bash
mkdir /tmp/flash
cd /tmp/flash
for f in /tmp/*.ipk; do tar -xzOf "$f" ./data.tar.gz | tar -xz; done

# resolve paths / partition indexes rather than hardcoding them
firmwarefile=$(ls /tmp/openwrt-*-squashfs-sysupgrade.bin)
bsdev=$(awk -F: '/"bs"/{print $1}' /proc/mtd)
echo "image=$firmwarefile  bootselect=/dev/$bsdev"

mtd() { LD_LIBRARY_PATH=/tmp/flash/lib /tmp/flash/lib/ld-musl-mips-sf.so.1 /tmp/flash/sbin/mtd "$@"; }

mtd write "$firmwarefile" kernel0
mtd erase kernel1
dd if=/dev/zero bs=1 count=1 of=/dev/$bsdev
reboot
```

Do not reboot if `mtd write` failed — see below. A failed write leaves the stock firmware intact, so you are safe as long as you stop there.

After rebooting, your device will boot into OpenWRT, accessible at http://192.168.1.1, with a DHCP server running on the LAN port. WiFi is not configured by default.

## Choosing package versions

Rules, in order of importance:

1. **Take `mtd` and `libubox` from the same OpenWRT release.** `mtd` links against a *dated* soname — `libubox.so.20210516`, `libubox.so.20220515`, `libubox.so.20230523`, `libubox.so.20240329` — which changes every release. Mixing releases gives you a loader failure, not a helpful error. This is the one way to break the procedure by "just using newer".
2. **Do not use 25.12 or later for the tooling.** OpenWRT switched from `.ipk` to **apk v3** (`ADBd` magic — not `ar`, not gzip). `tar -xzOf pkg.apk ./data.tar.gz` fails outright, so the extraction idiom above does not work at all. Packages there are named `mtd-27.apk` / `libc-1.2.5-r5.apk`.
3. **There is no functional reason to prefer a newer `mtd`.** It is package version **26** in 21.02, 22.03, 23.05 *and* 24.10. Diffing `package/system/mtd/src/mtd.c` between the 21.02 and 24.10 branches turns up exactly one change: an added `-M <magic>` option for `fixtrx`, irrelevant to UniFi. The command set (`write`/`erase`/`verify`/`dump`/`unlock`/…) is byte-for-byte the same set. The reason to move off 21.02 is simply that it is **end of life** — the URLs still resolve today but will eventually move to `archive.openwrt.org`.

The `sysupgrade` image version is independent of the tooling version — just keep it out of the known-bad ranges in step 3.

## Troubleshooting

### `Could not open mtd device: kernel0` / `Can't open device for writing!`

**Changing the OpenWRT package version will not fix this.** `mtd_open()` is unchanged from 21.02 through 24.10: it builds the search string with `snprintf(name, sizeof(name), "\"%s\"", mtd)` — *quotes included* — then `strstr()`s it against each line of `/proc/mtd`, and opens the match with `O_RDWR | O_SYNC`. The same message is printed whether the name wasn't found or the open failed, which is why step 5 exists.

(The unreleased `main` branch does change this: `mtd_open()` gained a `write_mode` flag and now defaults to `O_RDONLY`, so `mtd dump` and `mtd verify` will work against locked partitions. The write path still needs `O_RDWR`, so it will not help you flash.)

`mtd` prints this from `mtd_open()`, which greps `/proc/mtd` for the partition name and then opens `/dev/mtdN` with `O_RDWR`. The same message covers both failure modes, so check `cat /proc/mtd` to tell them apart:

- **`kernel0` is absent** — your hardware revision uses a different layout. `UAP-AC-Pro-Gen2` and the U6 line are known to differ; the U6 Pro (IPQ5018) has no kernel partitions at all and its bootloader only accepts signed images.
- **`kernel0` is present** — the partition is write-locked by the stock kernel. Ubiquiti locked these progressively through 5.x/6.x. Nothing was written, so the device is unharmed.

Options from here, roughly in order of preference:

1. **Downgrade to 3.7.58 first** and flash from there (the original method). Modern firmware validates against an allow-list and may reject it with `Invalid version 'BZ.qca956x.v3.7.58...'` — if so, step down gradually (e.g. 6.5.x → 6.2.x → 5.43.x) rather than jumping.
2. **TFTP recovery mode** — hold reset while powering on, then push an OpenWRT factory image to `192.168.1.20`. This goes through U-Boot and bypasses the running firmware's locks entirely.
3. **Serial console (UART)** — requires opening the case, but always works.

### Known-bad OpenWRT releases

If you already flashed 23.05.0–23.05.2 or 24.10.0–24.10.2 and now find the flash read-only, that is the bug noted in step 3, not your mistake. Recovery procedure is on the [OpenWRT Ubiquiti page](https://openwrt.org/toh/ubiquiti/common).

## Enjoy!

OpenWRT installed (hopefully!)

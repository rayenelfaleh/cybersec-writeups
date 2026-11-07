# Dumping Router Firmware — TryHackMe — Easy

> **Date:** 2026-06-06\
> **Platform:** TryHackMe\
> **Difficulty:** Easy\
> **Tags:** `#firmware` `#iot` `#embedded-linux` `#binwalk` `#jffs2` `#hardware`

---

## Summary

> This room covers the analysis and extraction of router firmware from a Linksys WRT1900ACS v2 image. The main focus is on static firmware analysis using `strings` and `binwalk`, followed by mounting and exploring the extracted JFFS2 filesystem to uncover device internals such as the OS, architecture, SSH server, and network configuration.

---

## Environment Setup

### Tools Required

- `strings` — extract readable text from binary files
- `binwalk` — firmware analysis and extraction
- `7z` — unzip multipart archive
- `jefferson` — JFFS2 filesystem extraction support for binwalk
- `modprobe`, `mknod`, `dd`, `mount` — kernel module and filesystem mounting

### Setup Commands

```bash
# Install JFFS2 support for binwalk
sudo pip install cstruct
git clone https://github.com/sviehb/jefferson
cd jefferson && sudo python setup.py install

# Clone and extract firmware
git clone https://github.com/Sq00ky/Dumping-Router-Firmware-Image/ /opt/Dumping-Router-Firmware
cd /opt/Dumping-Router-Firmware
7z x ./FW_WRT1900ACSV2_2.0.3.201002_prod.zip

# Verify integrity
sha256sum FW_WRT1900ACSV2_2.0.3.201002_prod.img
# Expected: dbbc9e8673149e79b7fd39482ea95db78bdb585c3fa3613e4f84ca0abcea68a4
```

---

## Firmware Analysis

### Strings Analysis

```bash
strings FW_WRT1900ACSV2_2.0.3.201002_prod.img | head
```

**Notable findings:**

|Finding|Value|
|---|---|
|First clear text line|`Linksys WRT1900ACS Router`|
|Operating System|`Linux` (`Uncompressing Linux... done, booting the kernel.`)|

### Binwalk — Initial Scan & Extraction

```bash
binwalk -e --run-as=root FW_WRT1900ACSV2_2.0.3.201002_prod.img
```

**Extraction results:**

|Offset (Dec)|Offset (Hex)|Description|
|---|---|---|
|0|0x0|**uImage header** ← first extracted item|
|64|0x40|Linux kernel ARM boot executable zImage|
|26736|0x6870|gzip compressed data (misidentified)|
|6291456|0x600000|JFFS2 filesystem, little endian|

**uImage Header Details:**

|Field|Value|
|---|---|
|Creation Date|`2020-04-22 11:07:26`|
|Image Size|`4229755 bytes`|
|Data CRC|`0xABEBC439`|
|OS|Linux|
|CPU Architecture|ARM|
|Compression|none|
|Image Name|`Linksys WRT1900ACS Router`|

> **Note:** The `6870` file is flagged as gzip compressed data, but binwalk misidentified it. Extraction will not succeed — however, strings and a second binwalk pass are still useful.

### Binwalk — Second Pass on `6870`

```bash
cd _FW_WRT1900ACSV2_2.0.3.201002_prod.img.extracted/
binwalk -e --run-as=root 6870
```

**Notable finding:**

|Offset|Description|
|---|---|
|0x59B0A0|Linux kernel version **3.10.39**|

---

## Filesystem Mounting

The extracted `600000.jffs2` is a **little endian** JFFS2 image and can be mounted directly (no endian conversion needed).

```bash
# Step 1 — Create block device
rm -rf /dev/mtdblock0
mknod /dev/mtdblock0 b 31 0

# Step 2 — Create mount point
mkdir /mnt/jffs2_file/

# Step 3 — Load kernel modules
modprobe jffs2
modprobe mtdram
modprobe mtdblock

# Step 4 — Write image to block device
dd if=/opt/Dumping-Router-Firmware/_FW_WRT1900ACSV2_2.0.3.201002_prod.img.extracted/600000.jffs2 of=/dev/mtdblock0

# Step 5 — Mount
mount -t jffs2 /dev/mtdblock0 /mnt/jffs2_file/

# Step 6 — Navigate
cd /mnt/jffs2_file/
```

---

## Filesystem Exploration

### Root Directory (`ls -la`)

**Symbolic links of interest:**

|Name|Links To|Notes|
|---|---|---|
|`linuxrc`|`/bin/busybox`|Standard embedded Linux init helper|
|`mnt`|`/tmp/mnt`|Writable at runtime via tmpfs|
|`opt`|`/tmp/opt`|Writable at runtime via tmpfs|
|`var`|`/tmp/var`|Writable at runtime via tmpfs|

> `mnt`, `opt`, and `var` redirect runtime data into `/tmp`. These symlinks alone do not establish whether the JFFS2 filesystem is mounted read-only.

### `/bin/`

```bash
ls -la /mnt/jffs2_file/bin/
```

- Most binaries are **symbolic links to `/bin/busybox`** — a single binary providing a stripped-down suite of Unix utilities, essential for keeping firmware size minimal.
- `sqlite3` is present — the router runs **SQLite** for configuration storage when online.

### `/etc/`

Key files found:

|File|Content|
|---|---|
|`builddate`|`2020-04-22 11:44` — firmware build date|
|`version`|`2.0.3.201002` — firmware version|
|`services`|Standard Unix network services and port mappings|
|`system_defaults`|Default system configuration settings|
|`dropbear` config|SSH server: **Dropbear**|
|`mediaserver.ini`|Media server developed by **Cisco** (former Linksys owner)|

> Country-specific Access Point power level files are also present (FCC, CE, AU, CA, etc.).

### `/JNAP/modules/`

Three network folders found:

- `guest_lan`
- `lan`
- `wan`

---

## Key Findings Summary

|Category|Finding|
|---|---|
|Device|Linksys WRT1900ACS v2|
|OS|Linux|
|CPU|ARM (1.6 GHz dual-core Marvell Armada 385)|
|Kernel Version|3.10.39|
|Firmware Version|2.0.3.201002|
|Build Date|2020-04-22 11:44|
|Init System|BusyBox|
|SSH Server|Dropbear|
|Database|SQLite|
|Media Server|Cisco|
|Filesystem|JFFS2 (little endian)|

---

## Lessons Learned

### 🔴 Côté Attaquant (Offensive)

- strings comme premier réflexe → avant tout outil complexe, strings sur un binaire firmware donne immédiatement le nom du device, l'OS, les messages de boot. Coût : zéro. Valeur : énorme.
- binwalk récursif → une extraction ratée n'est pas une impasse. Toujours relancer binwalk -e sur les blobs extraits — le fichier 6870 semblait inutilisable mais a quand même livré la version du kernel.
- Monter JFFS2 manuellement → workflow non-trivial (mknod, modprobe, dd, mount -t jffs2) à mémoriser. S'applique à n'importe quel firmware IoT/router extrait physiquement via UART, JTAG ou lecture de chip flash.
- /etc en priorité → Dropbear config, version firmware, build date, credentials par défaut — tout ce qui est exploitable en engagement réel est là.


### 🔵 Côté Défenseur (Defensive / Blue Team)

- Un firmware public = surface d'analyse complète pour un attaquant. Les versions, services exposés et configurations par défaut ne devraient jamais être trivialement extractibles.
- BusyBox = environnement minimal → sur un device réel compromis, adapter les techniques (pas de wget, pas de gcc, parfois pas de bash).
- Désactiver Dropbear ou changer les credentials par défaut est critique — ils sont lisibles statiquement dans le firmware.

---

## Attack Chain Summary

```
Firmware .img file
    └── strings → clear text OS/device identification
    └── binwalk -e → extracts JFFS2 filesystem + kernel blob
            └── binwalk on 6870 → Linux kernel version
            └── Mount JFFS2 → full filesystem access
                    └── /bin → BusyBox + SQLite
                    └── /etc → credentials, config, build info
                    └── /JNAP → network module structure
```

---

## References

- [Binwalk GitHub](https://github.com/ReFirmLabs/binwalk)
- [Jefferson JFFS2 tool](https://github.com/sviehb/jefferson)
- [BusyBox](https://busybox.net/)
- [JFFS2 Wikipedia](https://en.wikipedia.org/wiki/JFFS2)
- [Memory Technology Device](https://en.wikipedia.org/wiki/Memory_Technology_Device)
- [Linksys WRT1900ACS specs](https://www.linksys.com/)
- [TryHackMe — Dumping Router Firmware](https://tryhackme.com/room/rfirmware)

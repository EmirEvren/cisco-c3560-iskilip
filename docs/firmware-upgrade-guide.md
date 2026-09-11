# IOS Firmware Upgrade Guide — Cisco WS-C3560-24PS-E

**Device:** Cisco WS-C3560-24PS-E  
**IOS:** 12.2(37)SE1 ADVIPSERVICESK9  
**TFTP Server:** 10.6.2.2  
**IOS Image:** `c3560-advipservicesk9-mz.122-37.SE1.bin`

---

## Table of Contents

- [Overview](#overview)
- [Pre-Upgrade Checklist](#pre-upgrade-checklist)
- [Step-by-Step TFTP IOS Transfer](#step-by-step-tftp-ios-transfer)
- [Boot System Commands](#boot-system-commands)
- [Verification After Upgrade](#verification-after-upgrade)
- [Rollback Procedure](#rollback-procedure)
- [Common Errors and Fixes](#common-errors-and-fixes)

---

## Overview

IOS firmware upgrades on the WS-C3560-24PS-E are performed by:

1. Downloading the new IOS image from a TFTP server to the switch flash memory
2. Configuring the `boot system` command to point to the new image
3. Saving the configuration and reloading the switch
4. Verifying the correct image loaded after the reload

> **Important:** This procedure requires a full switch reload. Plan for a maintenance window and notify users before proceeding.

---

## Pre-Upgrade Checklist

Verify each item before starting the upgrade:

- [ ] **Network reachability:** `ping 10.6.2.2` returns 100% success
- [ ] **IOS image on TFTP server:** Confirm `c3560-advipservicesk9-mz.122-37.SE1.bin` exists on the TFTP server
- [ ] **Flash space:** `show flash:` — minimum 15 MB free required (typically 11-12 MB for the image)
- [ ] **Running config backed up:** `copy running-config tftp:` to 10.6.2.2
- [ ] **Startup config backed up:** `copy startup-config tftp:` to 10.6.2.2
- [ ] **Current IOS version noted:** Run `show version` and record the output
- [ ] **Maintenance window scheduled:** Reload will disrupt all connected hosts
- [ ] **Console access available:** In case network management fails after reload

---

## Step-by-Step TFTP IOS Transfer

### Step 1 — Verify Current IOS Version

```ios
Iskilip# show version
Cisco IOS Software, C3560 Software (C3560-ADVIPSERVICESK9-M), Version 12.2(37)SE1, RELEASE SOFTWARE (fc1)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2007 by Cisco Systems, Inc.
...
Cisco WS-C3560-24PS-E (PowerPC405) processor (revision J0) with 122880K/10240K bytes of memory.
...
Model number              : WS-C3560-24PS-E
System serial number      : CAT1037RJF7
```

### Step 2 — Check Flash Space

```ios
Iskilip# show flash:
Directory of flash:/

    1  -rwx  11607040  Mar 01 2026 00:00:00 +00:00  c3560-advipservicesk9-mz.122-37.SE1.bin
    2  -rwx      1531  Mar 01 2026 00:01:00 +00:00  Iskilip

64016384 bytes total (52409344 bytes free)
```

Ensure you have at least 15 MB free. If needed, delete old images first:

```ios
Iskilip# delete flash:<old-image>.bin
Delete filename [<old-image>.bin]? [Enter]
Delete flash:/<old-image>.bin? [confirm]
```

### Step 3 — Back Up Current Configuration

```ios
Iskilip# copy running-config tftp:
Address or name of remote host []? 10.6.2.2
Destination filename [iskilip-confg]? Iskilip_preupgrade_20260912
!
[OK - 1531 bytes]
```

### Step 4 — Download New IOS Image from TFTP

```ios
Iskilip# copy tftp: flash:
Address or name of remote host []? 10.6.2.2
Source filename []? c3560-advipservicesk9-mz.122-37.SE1.bin
Destination filename [c3560-advipservicesk9-mz.122-37.SE1.bin]? [Enter]
Accessing tftp://10.6.2.2/c3560-advipservicesk9-mz.122-37.SE1.bin...
Loading c3560-advipservicesk9-mz.122-37.SE1.bin from 10.6.2.2:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
[OK - 11607040 bytes]
```

Each `!` represents one successful UDP TFTP block transfer. The transfer may take 1-5 minutes depending on network speed.

### Step 5 — Verify the Image Was Written Correctly

```ios
Iskilip# verify flash:c3560-advipservicesk9-mz.122-37.SE1.bin
Starting image verification of flash:c3560-advipservicesk9-mz.122-37.SE1.bin
...
Verified flash:c3560-advipservicesk9-mz.122-37.SE1.bin
```

### Step 6 — Configure Boot System Command

```ios
Iskilip# configure terminal

! Clear existing boot statements
Iskilip(config)# no boot system

! Set new boot image
Iskilip(config)# boot system flash:c3560-advipservicesk9-mz.122-37.SE1.bin

Iskilip(config)# exit
```

### Step 7 — Save Configuration

```ios
Iskilip# write memory
Building configuration...
[OK]
```

### Step 8 — Reload the Switch

```ios
Iskilip# reload
System configuration has been modified. Save? [yes/no]: no
Proceed with reload? [confirm] [Enter]

*Sep 12 01:29:00.000: %SYS-5-RELOAD: Reload requested by console.
```

The switch will reboot. Console connection will be lost temporarily.

---

## Boot System Commands

| Command | Purpose |
| :--- | :--- |
| `show boot` | Display configured boot system statement and BOOT variable |
| `boot system flash:<image>` | Configure primary boot image from flash |
| `no boot system` | Remove all boot system statements (resets to default) |
| `show flash:` | List all files in flash and available space |
| `verify flash:<image>` | MD5/SHA verify image integrity before booting |
| `delete flash:<image>` | Remove an image from flash (cannot be undone) |

---

## Verification After Upgrade

After the switch reloads, verify the correct image is running:

```ios
! Verify IOS version
Iskilip# show version
Cisco IOS Software, C3560 Software (C3560-ADVIPSERVICESK9-M), Version 12.2(37)SE1, RELEASE SOFTWARE (fc1)

! Verify boot image
Iskilip# show boot
BOOT path-list      : flash:c3560-advipservicesk9-mz.122-37.SE1.bin
Config file         : flash:/config.text
Private Config file : flash:/private-config.text
Enable Break        : no
Manual Boot         : no

! Verify management IP is still configured
Iskilip# show interface vlan 1
Vlan1 is up, line protocol is up
  Internet address is 10.19.2.254/24

! Verify connectivity
Iskilip# ping 10.19.2.1
!!!!!
```

---

## Rollback Procedure

If the upgrade fails or the switch behaves unexpectedly after the upgrade, use one of these rollback methods:

### Method 1 — Switch to Old Image (Flash Available)

If the old image is still in flash:

```ios
Iskilip# configure terminal
Iskilip(config)# no boot system
Iskilip(config)# boot system flash:<old-image-name>.bin
Iskilip(config)# exit
Iskilip# write memory
Iskilip# reload
```

### Method 2 — ROMMON Recovery (IOS Won't Load)

Access the switch via console cable. Interrupt the boot sequence with Ctrl+Break during the ROMMON countdown (15 seconds):

```
rommon 1> dir flash:
rommon 2> set BOOT=flash:c3560-advipservicesk9-mz.122-37.SE1.bin
rommon 3> boot
```

### Method 3 — TFTP Boot from ROMMON

If flash is corrupted and no local image is available:

```
rommon 1> set IP_ADDRESS=10.19.2.254
rommon 2> set IP_SUBNET_MASK=255.255.255.0
rommon 3> set DEFAULT_GATEWAY=10.19.2.1
rommon 4> set TFTP_SERVER=10.6.2.2
rommon 5> set TFTP_FILE=c3560-advipservicesk9-mz.122-37.SE1.bin
rommon 6> tftpdnld
rommon 7> boot
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `%Error opening tftp://10.6.2.2/<image> (Timed out)` | TFTP server unreachable or file not found | Verify ping to 10.6.2.2, confirm filename is exact (case-sensitive) |
| `insufficient flash space` | Not enough free flash memory | Delete old images with `delete flash:<old>.bin` |
| `%Error verifying flash` | Flash write error or corrupted image | Delete image and retry transfer; check cable integrity |
| Switch boots to ROMMON after reload | Boot system command points to missing file | Use ROMMON `dir flash:` to list files and set correct BOOT variable |
| Management IP lost after reload | Startup-config was not saved before reload | Reconnect via console, re-apply IP config, save |
| TFTP transfer stalls at `!!...` | Network congestion or TFTP timeout | Retry; TFTP uses UDP — ensure no firewall blocking port 69 |
| `%SYS-3-BADBLOCK` errors during boot | Flash corruption | Copy new image to flash via console Xmodem or TFTP boot |

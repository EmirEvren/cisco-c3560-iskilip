# TFTP Configuration Management Guide — Cisco WS-C3560-24PS-E

**Device:** Cisco WS-C3560-24PS-E (Iskilip)  
**Management IP:** 10.19.2.254/24  
**TFTP Server:** 10.6.2.2 (Cankaya LAN, Ankara)  
**Default Gateway:** 10.19.2.1  

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Backup Procedure](#backup-procedure)
  - [Backup Running Configuration](#backup-running-configuration)
  - [Backup Startup Configuration](#backup-startup-configuration)
  - [Naming Convention](#naming-convention)
- [Restore Procedure](#restore-procedure)
  - [Restore to Running Configuration](#restore-to-running-configuration)
  - [Restore to Startup Configuration](#restore-to-startup-configuration)
- [Scheduling Regular Backups](#scheduling-regular-backups)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)

---

## Overview

TFTP (Trivial File Transfer Protocol) provides a simple, lightweight mechanism for transferring configuration files and IOS images between Cisco devices and a TFTP server. It operates over UDP port 69 and requires no authentication.

In this lab setup:
- The switch Iskilip (10.19.2.254) reaches the TFTP server (10.6.2.2) through the WAN backbone via the Corum router (10.19.2.1)
- TFTP is used for both **config backup** (switch to server) and **config restore** (server to switch)
- The TFTP file `Iskilip` (1531 bytes) was used to restore the switch hostname and IP configuration

---

## Prerequisites

Before performing any TFTP operation, verify these conditions:

```ios
! 1. Verify VLAN 1 SVI is up with the correct IP
Iskilip# show interface vlan 1
Vlan1 is up, line protocol is up
  Internet address is 10.19.2.254/24

! 2. Verify default gateway is configured
Iskilip# show ip default-gateway
10.19.2.1

! 3. Verify reachability to the TFTP server
Iskilip# ping 10.6.2.2
Sending 5, 100-byte ICMP Echos to 10.6.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5)
```

All three checks must pass before proceeding.

---

## Backup Procedure

### Backup Running Configuration

The running configuration is the active in-memory configuration. Backing it up preserves the current state of the switch:

```ios
! Step 1: Initiate backup
Iskilip# copy running-config tftp:

! Step 2: Enter TFTP server IP
Address or name of remote host []? 10.6.2.2

! Step 3: Enter destination filename (use a descriptive name with date)
Destination filename [iskilip-confg]? Iskilip_running_20260912

! Step 4: Confirm transfer
!
[OK - 1531 bytes]
```

### Backup Startup Configuration

The startup configuration is saved in NVRAM and loaded at boot time. Back it up separately:

```ios
Iskilip# copy startup-config tftp:
Address or name of remote host []? 10.6.2.2
Destination filename [iskilip-confg]? Iskilip_startup_20260912
!
[OK - 1531 bytes]
```

### Naming Convention

Use consistent naming for easy identification and rollback:

| File Name Format | Example | Contents |
| :--- | :--- | :--- |
| `<hostname>_running_<YYYYMMDD>` | `Iskilip_running_20260912` | Running config snapshot |
| `<hostname>_startup_<YYYYMMDD>` | `Iskilip_startup_20260912` | Startup config snapshot |
| `<hostname>_preupgrade_<YYYYMMDD>` | `Iskilip_preupgrade_20260912` | Backup before firmware upgrade |
| `<hostname>_baseline` | `Iskilip_baseline` | Known-good baseline config |

---

## Restore Procedure

### Restore to Running Configuration

`copy tftp: running-config` **merges** the TFTP file into the existing running configuration. Lines in the TFTP file are processed as if typed at the CLI:

```ios
! Step 1: Verify TFTP server is reachable
Iskilip# ping 10.6.2.2

! Step 2: Copy config from TFTP
Iskilip# copy tftp: running-config

! Step 3: Enter source details
Address or name of remote host []? 10.6.2.2
Source filename []? Iskilip
Destination filename [running-config]? [Enter]

! Step 4: Monitor transfer
Accessing tftp://10.6.2.2/Iskilip...
Loading Iskilip from 10.6.2.2:
!
[OK - 1531 bytes]

! Step 5: Verify changes took effect (in this lab, hostname changed)
Iskilip#

! Step 6: Save to NVRAM
Iskilip# copy running-config startup-config
[OK]
```

> **Behavior note:** Unlike `copy tftp: startup-config` (which replaces the startup config file entirely), `copy tftp: running-config` is a **merge** operation. Existing configuration lines that are NOT in the TFTP file remain in place. Only lines present in the TFTP file are applied.

### Restore to Startup Configuration

To replace the startup config entirely with a TFTP file (effective after next reload):

```ios
Iskilip# copy tftp: startup-config
Address or name of remote host []? 10.6.2.2
Source filename []? Iskilip_baseline
Destination filename [startup-config]? [Enter]
!
[OK - 1531 bytes]

! Then reload to apply the startup config as the new running config
Iskilip# reload
```

---

## Scheduling Regular Backups

Cisco IOS does not have a built-in cron scheduler for automatic TFTP backups. In production environments, regular backups are achieved through:

### Option 1 — Network Management System (NMS)

Use a NMS platform (Cisco Prime, SolarWinds, PRTG) to schedule and pull configs from the switch via TFTP or SCP at regular intervals.

### Option 2 — Linux/Python Script on TFTP Server

Create a scheduled script on the TFTP server (10.6.2.2) that triggers a TFTP pull via SNMP or SSH:

```bash
#!/bin/bash
# Daily config backup for Iskilip switch
DATE=$(date +%Y%m%d)
SWITCH_IP="10.19.2.254"
TFTP_DIR="/var/lib/tftpboot"

# Via SSH (requires SSHv2 configured on switch)
ssh admin@$SWITCH_IP "copy running-config tftp://10.6.2.2/Iskilip_$DATE"
```

### Option 3 — EEM (Embedded Event Manager)

On Cisco IOS, use EEM applets to trigger backups on specific events (e.g., config changes):

```ios
Iskilip(config)# event manager applet BACKUP_ON_CHANGE
Iskilip(config-applet)# event syslog pattern "SYS-5-CONFIG_I"
Iskilip(config-applet)# action 1.0 cli command "enable"
Iskilip(config-applet)# action 2.0 cli command "copy running-config tftp://10.6.2.2/Iskilip_autobackup"
```

---

## Security Considerations

TFTP transmits data in **plaintext over UDP** with **no authentication**. This is acceptable for isolated lab environments but presents risks in production:

| Risk | Impact | Mitigation |
| :--- | :--- | :--- |
| **No authentication** | Any host can read/write files on the TFTP server | Restrict TFTP server to trusted IPs via ACL |
| **Plaintext transfer** | Config files visible to network sniffers | Use SCP/SFTP in production; limit TFTP to out-of-band management |
| **No integrity check** | Files may be corrupted or tampered in transit | Use `verify` command after transfer; use SCP in production |
| **UDP-based** | Susceptible to packet loss on congested links | Use reliable transfers (SCP) over WAN links |

### ACL to Restrict TFTP Access

Apply an ACL on the switch to only permit TFTP to/from the authorized server:

```ios
! Permit TFTP only from the authorized server
Iskilip(config)# access-list 10 permit 10.6.2.2
Iskilip(config)# access-list 10 deny any log

! Apply to VTY lines for Telnet/SSH management
Iskilip(config)# line vty 0 15
Iskilip(config-line)# access-class 10 in
```

### Use SCP for Production

In production environments, replace TFTP with SCP (Secure Copy Protocol):

```ios
! Enable SSH and SCP server on the switch
Iskilip(config)# ip domain-name iskilip.corum.local
Iskilip(config)# crypto key generate rsa modulus 2048
Iskilip(config)# ip ssh version 2
Iskilip(config)# ip scp server enable

! Copy config using SCP from a management workstation
! (run on the workstation, not the switch)
scp admin@10.19.2.254:running-config ./Iskilip_backup_20260912.txt
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
| :--- | :--- | :--- |
| `%Error opening tftp://...` | TFTP server unreachable or file missing | Verify `ping 10.6.2.2`; confirm filename; check TFTP service is running |
| Hostname does not change after restore | TFTP file does not contain `hostname` line | Check file contents on TFTP server |
| `[OK - 0 bytes]` | Empty file on TFTP server | Verify the file has content; check TFTP server permissions |
| Transfer starts then times out | Network congestion or firewall blocking UDP 69 | Ensure UDP port 69 is not filtered on the WAN path |
| Config partially applied | TFTP file has syntax errors | Check file line by line; test on a console session first |
| `%TFTP: unknown host` | DNS resolution failure | Use IP address directly instead of hostname |

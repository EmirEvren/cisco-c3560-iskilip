# Cisco WS-C3560-24PS — VLAN, TFTP Config Restore & IOS Firmware Upgrade

[![Cisco Packet Tracer](https://img.shields.io/badge/Cisco%20Packet%20Tracer-v8.0%2B-049cdb?style=flat-square&logo=cisco&logoColor=white)](https://www.netacad.com/courses/packet-tracer)
[![IOS Version](https://img.shields.io/badge/Cisco%20IOS-12.2(37)SE1-1BA0D7?style=flat-square&logo=cisco&logoColor=white)](https://www.cisco.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Device](https://img.shields.io/badge/Device-WS--C3560--24PS--E-brightgreen?style=flat-square)](.)
[![TFTP](https://img.shields.io/badge/Management-TFTP-orange?style=flat-square)](.)

Packet Tracer lab on a Cisco WS-C3560-24PS-E switch. This is the Iskilip district access switch in the Corum city network. Things covered: assigning a management IP via VLAN 1 SVI, restoring a config from a TFTP server, and the IOS firmware upgrade process.

This switch is part of the [4-City WAN Routing Project](https://github.com/EmirEvren/cisco-packet-tracer-4-city-wan-routing) — it handles the `10.19.2.0/24` (Iskilip) subnet and connects upstream to the Corum border router (`10.19.2.1`).

---

## Device Info

| Parameter | Value |
| :--- | :--- |
| **Model** | Cisco WS-C3560-24PS-E |
| **IOS** | 12.2(37)SE1 ADVIPSERVICESK9 |
| **Serial** | CAT1037RJF7 |
| **MAC** | 0030.F2D4.CE7C |
| **RAM** | 122880K |
| **Flash** | 64 MB |
| **Ports** | 24x FastEthernet + 2x GigabitEthernet |
| **Management IP** | 10.19.2.254 / 24 (VLAN 1 SVI) |
| **Default Gateway** | 10.19.2.1 (Corum Router) |
| **TFTP Server** | 10.6.2.2 (Ankara — Cankaya LAN) |
| **TFTP Config File** | `Iskilip` (1531 bytes) |
| **Hostname (after restore)** | Iskilip |

---

## Switch Management Summary

All 4 district switches are configured with both **Telnet and SSH** access, local user authentication, and VLAN 1 SVI management IPs.

| Switch | Region | Management IP | Gateway | Config File |
| :--- | :--- | :--- | :--- | :--- |
| **Cankaya** | Ankara | 10.6.2.254 / 24 | 10.6.2.1 | [Cankaya-switch-config.txt](configs/Cankaya-switch-config.txt) |
| **Iskilip** | Corum | 10.19.2.254 / 24 | 10.19.2.1 | [switch-running-config.txt](configs/switch-running-config.txt) |
| **Ikizdere** | Rize | 10.53.2.254 / 24 | 10.53.2.1 | [Ikizdere-switch-config.txt](configs/Ikizdere-switch-config.txt) |
| **Nilufer** | Bursa | 10.16.2.254 / 24 | 10.16.2.1 | [Nilufer-switch-config.txt](configs/Nilufer-switch-config.txt) |

**Credentials:** `admin` (privilege 15) / `user` (privilege 1) — Enable: `cisco`

**SSH:** version 2, 1024-bit RSA, 4 retries, 30s timeout — **Transport:** `all` (Telnet + SSH)

---

## Table of Contents

- [Topology](#topology)
- [Boot & POST](#boot--post)
- [VLAN 1 Configuration](#vlan-1-configuration)
- [Default Gateway](#default-gateway)
- [Ping Test](#ping-test)
- [TFTP Config Restore](#tftp-config-restore)
- [IOS Firmware Upgrade](#ios-firmware-upgrade)
- [Command Reference](#command-reference)
- [Repo Structure](#repo-structure)
- [How to Run](#how-to-run)

---

## Topology

### Local

```text
===================================================================================
         ISKILIP — WS-C3560-24PS-E | 10.19.2.0/24
===================================================================================

  [TFTP Server 10.6.2.2]                    [Corum Router]
   Ankara - Cankaya LAN                      GigabitEthernet0/2
        |                                         |
        |    WAN backbone (1.1.1.0/24)            |
        |--------------------------------------->  |
        |                                         |
        |        GigabitEthernet0/2               |
        +-------------------------------------> [WS-C3560-24PS-E]
                  (TFTP transfer path)           Hostname  : Iskilip
                                                 VLAN 1 IP : 10.19.2.254/24
                                                 Gateway   : 10.19.2.1
                                                      |
                               +----------------------+-------------------+
                               | Fa0/1 ~ Fa0/24                          |
                               v                         v
                      [LAN Host 10.19.2.x]     [LAN Host 10.19.2.x]  ...
===================================================================================
```

### 4-City WAN Context

```text
=============================================================================================================
                                    CENTRAL MPLS WAN BACKBONE — 1.1.1.0/24
=============================================================================================================
       |                          |                          |                          |
       | 1.1.1.6/24               | 1.1.1.16/24              | 1.1.1.19/24              | 1.1.1.53/24
       v                          v                          v                          v
+-------------+          +-------------+          +-------------+          +-------------+
| ANKARA (06) |          |  BURSA (16) |          |  CORUM (19) |          |  RIZE  (53) |
+-------------+          +-------------+          +-------------+          +-------------+
  |        |               |        |               |        |               |        |
10.6.1.1 10.6.2.1                               10.19.1.1 10.19.2.1
  v        v               v        v               v        v               v        v
[SINCAN][CANKAYA]     [OSMANGAZI][NILUFER]      [ALACA] [ISKILIP***]    [PAZAR][IKIZDERE]
             |                                               |
        TFTP Server                               *** THIS LAB — WS-C3560-24PS-E
        10.6.2.2                                     Management: 10.19.2.254/24
=============================================================================================================
```

---

## Boot & POST

When the switch powers on, the boot loader loads the IOS image from flash, then runs POST:

```
C3560 Boot Loader Version 12.2(25r)SEC
Loading "flash:/c3560-advipservicesk9-mz.122-37.SE1.bin"... [OK]
```

| # | Test | Result |
| :---: | :--- | :---: |
| 1 | CPU MIC Register Tests | ✅ PASS |
| 2 | PortASIC Memory Tests | ✅ PASS |
| 3 | CPU MIC Interface Loopback | ✅ PASS |
| 4 | PortASIC RingLoopback | ✅ PASS |
| 5 | Inline Power Controller | ✅ PASS |
| 6 | PortASIC CAM Subsystem | ✅ PASS |
| 7 | PortASIC Port Loopback | ✅ PASS |

All 7 tests passed, switch booted normally.

---

## VLAN 1 Configuration

On the C3560, physical ports are L2 switchports by default — you can't assign an IP directly to them. Management IP goes on the VLAN 1 SVI instead, which makes it reachable from any port on the switch.

```ios
Switch> enable
Switch# configure terminal

Switch(config)# interface vlan 1
Switch(config-if)# no shutdown
Switch(config-if)# ip address 10.19.2.254 255.255.255.0
Switch(config-if)# exit
```

Verify:
```ios
Switch# show interface vlan 1
Vlan1 is up, line protocol is up
  Internet address is 10.19.2.254/24
```

---

## Default Gateway

Without a default gateway the switch can't reach anything outside `10.19.2.0/24` — including the TFTP server at `10.6.2.2`. Fix:

```ios
Switch(config)# ip default-gateway 10.19.2.1
```

> `ip default-gateway` is used here because `ip routing` is off on this switch. If routing were enabled, you'd use `ip route 0.0.0.0 0.0.0.0 10.19.2.1` instead.

---

## Ping Test

```ios
Switch# ping 10.6.2.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.6.2.2, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 0/0/0 ms
```

First packet drops (`.`) — that's ARP resolving the gateway's MAC. Completely normal. Packets 2-5 go through fine.

---

## TFTP Config Restore

Pulled the `Iskilip` config file from the TFTP server and merged it into running-config:

```ios
Switch# copy tftp: running-config
Address or name of remote host []? 10.6.2.2
Source filename []? Iskilip
Destination filename [running-config]? 

Accessing tftp://10.6.2.2/Iskilip...
Loading Iskilip from 10.6.2.2: !
[OK - 1531 bytes]

1531 bytes copied in 0 secs
Iskilip#
```

Hostname flipped from `Switch` to `Iskilip` — the config from TFTP was applied successfully.

> `copy tftp: running-config` **merges** the file into the existing config, it doesn't wipe and replace. Commands run line by line.

### Backup

```ios
Iskilip# copy running-config tftp:
Address or name of remote host []? 10.6.2.2
Destination filename [iskilip-confg]? Iskilip_backup_20260912
[OK - 1531 bytes]
```

### Transfer Methods

| Method | Speed | Security | Merge? | When to use |
| :--- | :---: | :---: | :---: | :--- |
| TFTP | Fast | None (UDP plaintext) | Yes | Lab / trusted network |
| SCP | Medium | Encrypted (SSH) | No | Production |
| USB | Fast | Physical | No | Offline / air-gapped |
| Console Xmodem | Very slow | Physical | No | Emergency recovery |

---

## IOS Firmware Upgrade

### Pre-check

```ios
! Check current version
Iskilip# show version

! Check free flash space (need at least 15 MB free)
Iskilip# show flash:
64016384 bytes total (52409344 bytes free)
```

### Steps

```ios
! 1. Download new IOS from TFTP to flash
Iskilip# copy tftp: flash:
Address or name of remote host []? 10.6.2.2
Source filename []? c3560-advipservicesk9-mz.122-55.SE.bin
Destination filename []? 
[OK - ~11 MB]

! 2. Set boot image
Iskilip# configure terminal
Iskilip(config)# no boot system
Iskilip(config)# boot system flash:c3560-advipservicesk9-mz.122-55.SE.bin
Iskilip(config)# exit

! 3. Save config
Iskilip# write memory

! 4. Reload
Iskilip# reload
Proceed with reload? [confirm]

! 5. Verify after boot
Iskilip# show version
```

### Rollback

```ios
! If old image is still on flash:
Iskilip(config)# no boot system
Iskilip(config)# boot system flash:c3560-advipservicesk9-mz.122-37.SE1.bin
Iskilip# write memory
Iskilip# reload

! From ROMMON (if IOS won't load):
rommon 1> set BOOT=flash:c3560-advipservicesk9-mz.122-37.SE1.bin
rommon 2> boot

! TFTP boot from ROMMON:
rommon 1> set IP_ADDRESS=10.19.2.254
rommon 2> set IP_SUBNET_MASK=255.255.255.0
rommon 3> set DEFAULT_GATEWAY=10.19.2.1
rommon 4> set TFTP_SERVER=10.6.2.2
rommon 5> set TFTP_FILE=c3560-advipservicesk9-mz.122-37.SE1.bin
rommon 6> tftpdnld
```

> **Heads up:** `reload` takes the switch offline for 2–5 minutes. Everything connected to it loses network during that time.

---

## Command Reference

| Command | Mode | What it does |
| :--- | :--- | :--- |
| `enable` | User EXEC | Enter privileged mode |
| `configure terminal` | Privileged | Enter global config mode |
| `interface vlan 1` | Global Config | Open VLAN 1 SVI |
| `ip address <ip> <mask>` | Interface | Assign IP to SVI |
| `no shutdown` | Interface | Bring interface up |
| `ip default-gateway <ip>` | Global Config | Set default gateway |
| `write memory` | Privileged | Save config to NVRAM |
| `ping <ip>` | Privileged | Connectivity test |
| `copy tftp: running-config` | Privileged | Restore config from TFTP |
| `copy running-config tftp:` | Privileged | Back up config to TFTP |
| `copy tftp: flash:` | Privileged | Download IOS image from TFTP |
| `boot system flash:<image>` | Global Config | Set boot image |
| `no boot system` | Global Config | Clear boot statements |
| `show version` | Privileged | IOS version + hardware info |
| `show flash:` | Privileged | Flash contents + free space |
| `show interface vlan 1` | Privileged | VLAN 1 SVI status |
| `show ip default-gateway` | Privileged | Check gateway setting |
| `show arp` | Privileged | ARP table |
| `show running-config` | Privileged | Current active config |
| `reload` | Privileged | Reboot the switch |

---

## Repo Structure

```
cisco-c3560-vlan-tftp-firmware/
├── Iskilip-C3560-Switch-Lab.pkt      ← Packet Tracer file
├── README.md
├── configs/
│   ├── switch-running-config.txt     ← Full config after TFTP restore
│   └── tftp-backup-config.txt        ← TFTP backup example
├── topologies/
│   └── network_topology.txt          ← ASCII topology diagram
├── docs/
│   ├── firmware-upgrade-guide.md     ← Detailed firmware upgrade guide
│   └── tftp-management-guide.md      ← TFTP management guide
├── LICENSE
└── .gitignore
```

---

## How to Run

1. Open [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (v8.0+)
2. Open `Iskilip-C3560-Switch-Lab.pkt`
3. Add the `Iskilip` config file to the TFTP server (contents from `configs/switch-running-config.txt`)
4. Open the switch CLI and follow the steps in order

---

*EmirEvren — MIT License 2026*

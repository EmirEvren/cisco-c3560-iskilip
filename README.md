# Cisco WS-C3560-24PS — VLAN 1 SVI Configuration, TFTP Config Management & IOS Firmware Upgrade Lab

[![Cisco Packet Tracer](https://img.shields.io/badge/Cisco%20Packet%20Tracer-v8.0%2B-049cdb?style=flat-square&logo=cisco&logoColor=white)](https://www.netacad.com/courses/packet-tracer)
[![IOS Version](https://img.shields.io/badge/Cisco%20IOS-12.2(37)SE1%20ADVIPSERVICESK9-1BA0D7?style=flat-square&logo=cisco&logoColor=white)](https://www.cisco.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Layer 3 Switch](https://img.shields.io/badge/Device-Layer%203%20Switch-brightgreen?style=flat-square&logo=cisco&logoColor=white)]()
[![TFTP Management](https://img.shields.io/badge/Management-TFTP%20Config%20%26%20IOS%20Upgrade-orange?style=flat-square)]()

A hands-on enterprise switching laboratory designed and implemented in **Cisco Packet Tracer**. This project demonstrates real-world Layer 3 switch management operations on a **Cisco WS-C3560-24PS-E** — the **Iskilip district access switch** in the Corum city network. Lab operations include **VLAN 1 SVI IP assignment**, **default gateway configuration**, **TFTP-based running-config restore**, and a complete **IOS firmware upgrade procedure** via TFTP.

> **Context:** This switch is part of the larger [4-City WAN Routing Project](https://github.com/EmirEvren/cisco-packet-tracer-4-city-wan-routing). It serves the `10.19.2.0/24` (Iskilip) subnet, connected upstream to the Corum border router (`10.19.2.1`).

---

## Quick Reference

| Parameter | Value |
| :--- | :--- |
| **Model** | Cisco WS-C3560-24PS-E |
| **IOS Version** | 12.2(37)SE1 ADVIPSERVICESK9 |
| **Serial Number** | CAT1037RJF7 |
| **MAC Address** | 0030.F2D4.CE7C |
| **RAM** | 122880K (~120 MB) |
| **Flash** | 64 MB |
| **Ports** | 24x FastEthernet + 2x GigabitEthernet |
| **Management IP** | 10.19.2.254 / 24 (VLAN 1 SVI) |
| **Default Gateway** | 10.19.2.1 (Corum Router Gig0/2) |
| **TFTP Server** | 10.6.2.2 (Cankaya LAN — Ankara region) |
| **Config File (TFTP)** | `Iskilip` (1531 bytes) |
| **IOS Image File** | `c3560-advipservicesk9-mz.122-37.SE1.bin` |
| **Hostname (post-restore)** | Iskilip |
| **Subnet** | 10.19.2.0/24 — Iskilip District, Corum |

---

## Table of Contents

- [Network Topology](#network-topology)
  - [1. ASCII Architecture Schematic](#1-ascii-architecture-schematic)
  - [2. WAN Context Diagram](#2-wan-context-diagram)
- [Boot Process and POST Tests](#boot-process-and-post-tests)
  - [Boot Loader Sequence](#boot-loader-sequence)
  - [POST Test Results](#post-test-results)
- [VLAN 1 SVI Configuration](#vlan-1-svi-configuration)
  - [Why VLAN SVI for Management](#why-vlan-svi-for-management)
  - [Configuration Commands](#configuration-commands)
- [Default Gateway Configuration](#default-gateway-configuration)
- [Connectivity Verification](#connectivity-verification)
  - [Ping Test Results](#ping-test-results)
  - [ARP Table Verification](#arp-table-verification)
- [TFTP Configuration Management](#tftp-configuration-management)
  - [Restore Procedure](#restore-procedure)
  - [Backup Procedure](#backup-procedure)
  - [Transfer Method Comparison](#transfer-method-comparison)
- [IOS Firmware Upgrade via TFTP](#ios-firmware-upgrade-via-tftp)
  - [Pre-Upgrade Checklist](#pre-upgrade-checklist)
  - [Step-by-Step Upgrade Procedure](#step-by-step-upgrade-procedure)
  - [Rollback Procedure](#rollback-procedure)
  - [Maintenance Window Warning](#maintenance-window-warning)
- [Command Reference Table](#command-reference-table)
- [Repository Structure](#repository-structure)
- [How to Run the Lab](#how-to-run-the-lab)
- [Author and License](#author-and-license)

---

## Network Topology

### 1. ASCII Architecture Schematic

```text
===================================================================================
              ISKILIP DISTRICT SWITCH -- CORUM CITY NETWORK
                    Cisco WS-C3560-24PS-E | IOS 12.2(37)SE1
===================================================================================

  ANKARA REGION (10.6.2.0/24)          CORUM REGION (10.19.0.0/16)
  +---------------------+              +-------------------------------------------+
  |  TFTP Server        |              |  Corum Border Router                      |
  |  IP: 10.6.2.2       |              |  GigabitEthernet0/2: 10.19.2.1/24        |
  |  (Cankaya LAN)      |              +-------------------+-----------------------+
  +----------+----------+                                  |
             |  WAN (via 1.1.1.0/24 backbone)             | GigabitEthernet0/2 (uplink)
             |  ----------------------------------------->|
             |                                             |
             |  GigabitEthernet0/2              +----------v------------------------------------+
             +--------------------------------->|  WS-C3560-24PS-E                              |
                  (TFTP transfer path)          |  Hostname   : Iskilip                        |
                                               |  VLAN 1 SVI : 10.19.2.254/24                 |
                                               |  Gateway    : 10.19.2.1                       |
                                               |  MAC        : 0030.F2D4.CE7C                  |
                                               |  Serial     : CAT1037RJF7                     |
                                               +------------------+----------------------------+
                                                                  |
                                    +-----------------------------+-----------------------------+
                                    | Fa0/1 through Fa0/24        |                             |
                                    v                             v                             v
                           +----------------+         +----------------+            +----------------+
                           |   LAN Host 1   |         |   LAN Host 2   |    ...     |   LAN Host N   |
                           | 10.19.2.x/24   |         | 10.19.2.x/24   |            | 10.19.2.x/24   |
                           +----------------+         +----------------+            +----------------+

===================================================================================
  SUBNET: 10.19.2.0/24  |  GATEWAY: 10.19.2.1  |  MGMT: 10.19.2.254
===================================================================================
```

### 2. WAN Context Diagram

```text
=============================================================================================================
                                    CENTRAL MPLS WAN BACKBONE
                                            1.1.1.0/24
=============================================================================================================
         |                            |                            |                            |
         | Gig0/3/0                   | Gig0/3/0                   | Gig0/3/0                   | Gig0/3/0
         | 1.1.1.6/24                 | 1.1.1.16/24                | 1.1.1.19/24                | 1.1.1.53/24
         v                            v                            v                            v
+------------------+       +------------------+       +------------------+       +------------------+
|   ANKARA (06)    |       |   BURSA (16)     |       |   CORUM (19)     |       |   RIZE (53)      |
| Router: Ankara   |       | Router: Bursa    |       | Router: Corum    |       | Router: Rize     |
+------------------+       +------------------+       +------------------+       +------------------+
   |          |               |          |               |          |               |          |
   | G0/1     | G0/2          | G0/1     | G0/2          | G0/1     | G0/2          | G0/1     | G0/2
   | 10.6.1.1 | 10.6.2.1      | 10.16.1.1| 10.16.2.1     | 10.19.1.1| 10.19.2.1     | 10.53.1.1| 10.53.2.1
   v          v               v          v               v          v               v          v
+----------+ +----------+ +----------+ +----------+ +----------+ +==============+ +----------+ +----------+
|  SINCAN  | | CANKAYA  | |OSMANGAZI | | NILUFER  | |  ALACA   | | ISKILIP *** | |  PAZAR   | | IKIZDERE |
|10.6.1.0/24|10.6.2.0/24|10.16.1.0/24|10.16.2.0/24|10.19.1.0/24|10.19.2.0/24 | |10.53.1.0/24|10.53.2.0/24|
+----------+ +----------+ +----------+ +----------+ +----------+ +==============+ +----------+ +----------+
                  |                                                      |
             TFTP Server                                    *** THIS LAB -- WS-C3560-24PS-E
             10.6.2.2                                           Management: 10.19.2.254/24
             (Config source)                                    TFTP Restore: 'Iskilip' file (1531 bytes)
=============================================================================================================
```

---

## Boot Process and POST Tests

### Boot Loader Sequence

When the WS-C3560-24PS-E powers on, the boot loader (`C3560 Boot Loader`) initializes hardware and runs the **Power-On Self Test (POST)** before loading IOS. The sequence is:

```
C3560 Boot Loader (C3560-HBOOT-M) Version 12.1(14r)EA1a, RELEASE SOFTWARE (fc1)
Cisco WS-C3560-24PS-E
Xmodem file system is available.
Initializing Flash...
flashfs[0]: 1 files, 1 directories
flashfs[0]: 0 orphaned files, 0 orphaned directories
flashfs[0]: Total bytes: 64016384
flashfs[0]: Bytes used: 11607040
flashfs[0]: Bytes available: 52409344
flashfs[0]: flashfs fsck took 1 seconds.
...done Initializing Flash.
Boot Sector Filesystem (bs:) installed, fsid: 3
Parameter Block Filesystem (pb:) installed, fsid: 4

Loading "flash:c3560-advipservicesk9-mz.122-37.SE1.bin"...
```

### POST Test Results

All **7 POST tests** completed successfully at power-on:

| # | POST Test | Status | Description |
| :---: | :--- | :---: | :--- |
| 1 | CPU Test | PASS | Main CPU registers and arithmetic logic verified |
| 2 | DRAM Test | PASS | 122880K RAM integrity check — read/write patterns |
| 3 | Flash Test | PASS | 64MB flash file system accessible and consistent |
| 4 | NVRAM Test | PASS | Non-volatile RAM storing startup-config verified |
| 5 | Port ASIC Test | PASS | FastEthernet/GigabitEthernet ASIC initialization |
| 6 | Inline Power Test | PASS | PoE controller (24-port 802.3af) self-test passed |
| 7 | Loopback Test | PASS | Internal interface loopback diagnostics passed |

All POST tests passed. IOS image was loaded from flash and the switch entered normal operation.

---

## VLAN 1 SVI Configuration

### Why VLAN SVI for Management

A **Layer 3 switch** like the WS-C3560 does not have dedicated management interfaces like a router. Instead, management IP access is achieved through a **Switched Virtual Interface (SVI)** — a logical Layer 3 interface bound to a VLAN.

| Concept | Explanation |
| :--- | :--- |
| **SVI (Switched Virtual Interface)** | A virtual interface (`interface Vlan1`) that gives an IP address to an entire VLAN for L3 reachability |
| **VLAN 1** | Default VLAN — all ports are members by default, ideal for management traffic |
| **Why not a physical port?** | Physical ports on a L3 switch are switchport (L2) by default; SVI provides L3 management without changing port mode |
| **Management access** | Telnet/SSH to the SVI IP (10.19.2.254) reaches the switch CLI over the network |

### Configuration Commands

```ios
! --- Step 1: Enter privileged EXEC mode ---
Switch> enable
Switch#

! --- Step 2: Enter global configuration mode ---
Switch# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
Switch(config)#

! --- Step 3: Enter VLAN 1 SVI interface ---
Switch(config)# interface vlan 1

! --- Step 4: Assign management IP address ---
Switch(config-if)# ip address 10.19.2.254 255.255.255.0
!  Sets management IP to 10.19.2.254 with /24 subnet mask

! --- Step 5: Bring the interface up ---
Switch(config-if)# no shutdown
!  VLAN 1 SVI is administratively down by default; this enables it

Switch(config-if)# exit

! --- Step 6: Verify interface status ---
Switch# show interface vlan 1
Vlan1 is up, line protocol is up
  Hardware is EtherSVI, address is 0030.f2d4.ce7c (bia 0030.f2d4.ce7c)
  Internet address is 10.19.2.254/24
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec,
     reliability 255/255, txload 1/255, rxload 1/255
```

---

## Default Gateway Configuration

A Layer 3 switch operating purely as a management device (not routing between VLANs) requires a **default gateway** to reach networks outside its own subnet. In this lab, the Corum border router (`10.19.2.1`) is the gateway.

```ios
! --- Set the default gateway ---
Switch(config)# ip default-gateway 10.19.2.1
!  Routes all traffic destined outside 10.19.2.0/24 through the Corum router
!  Required for TFTP access to 10.6.2.2 (Ankara region TFTP server)

! --- Verify the default gateway ---
Switch# show ip default-gateway
10.19.2.1

! --- Save configuration to NVRAM ---
Switch# write memory
Building configuration...
[OK]
```

> **Note:** `ip default-gateway` is used (not `ip route 0.0.0.0 0.0.0.0`) because the switch is not configured for IP routing (`no ip routing` is the default on a L2 switch). On a L3 switch with `ip routing` enabled, a default static route would be used instead.

---

## Connectivity Verification

### Ping Test Results

After configuring the VLAN 1 SVI and default gateway, connectivity was verified with a ping to the upstream Corum router:

```ios
Switch# ping 10.19.2.1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.19.2.1, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/2/4 ms
```

| Metric | Value |
| :--- | :--- |
| **Packets Sent** | 5 |
| **Packets Received** | 4 |
| **Success Rate** | 80% |
| **First Packet** | `.` Timeout (ARP resolution in progress) |
| **Packets 2-5** | `!!!!` Successful |
| **RTT min/avg/max** | 1 / 2 / 4 ms |

> **Why 80% (4/5)?** The first ICMP packet is lost while the switch resolves the gateway's MAC address via ARP. This is normal behavior — the first packet times out, then subsequent packets succeed once the ARP entry is cached.

### ARP Table Verification

```ios
Switch# show arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.19.2.1               0   0090.2170.5202  ARPA   Vlan1
Internet  10.19.2.254             -   0030.f2d4.ce7c  ARPA   Vlan1
```

The ARP table confirms the switch has successfully resolved the gateway's MAC address and communication is established.

---

## TFTP Configuration Management

TFTP (Trivial File Transfer Protocol) is used in this lab to restore and back up the switch running configuration. The TFTP server at `10.6.2.2` (Cankaya LAN, Ankara region) is reachable via the default gateway through the WAN backbone.

### Restore Procedure

The `Iskilip` configuration file (1531 bytes) was fetched from the TFTP server and merged into the running configuration:

```ios
! --- Step 1: Verify TFTP server reachability ---
Switch# ping 10.6.2.2
Sending 5, 100-byte ICMP Echos to 10.6.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5)

! --- Step 2: Copy config from TFTP to running-config ---
Switch# copy tftp: running-config
Address or name of remote host []? 10.6.2.2
Source filename []? Iskilip
Destination filename [running-config]? [Enter]

Accessing tftp://10.6.2.2/Iskilip...
Loading Iskilip from 10.6.2.2:
!
[OK - 1531 bytes]

! --- Step 3: Verify hostname changed (from TFTP config) ---
Iskilip#
!  Hostname changed to 'Iskilip' -- confirms config was applied

! --- Step 4: Verify running configuration ---
Iskilip# show running-config

! --- Step 5: Save to startup config ---
Iskilip# copy running-config startup-config
Destination filename [startup-config]? [Enter]
Building configuration...
[OK]

! --- Step 6: Verify startup config ---
Iskilip# show startup-config
```

> **Key observation:** `copy tftp: running-config` **merges** the TFTP file into the existing running configuration — it does not replace it. Commands in the TFTP file are executed line-by-line. The hostname change to `Iskilip` confirms the merge was successful.

### Backup Procedure

```ios
! --- Back up running config to TFTP ---
Iskilip# copy running-config tftp:
Address or name of remote host []? 10.6.2.2
Destination filename [iskilip-confg]? Iskilip_backup_20260912
!
[OK - 1531 bytes]

! --- Back up startup config to TFTP ---
Iskilip# copy startup-config tftp:
Address or name of remote host []? 10.6.2.2
Destination filename [iskilip-confg]? Iskilip_startup_20260912
!
[OK - 1531 bytes]
```

### Transfer Method Comparison

| Method | Speed | Security | Requires SSH? | Config Merge | Use Case |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **TFTP** | Fast | None (plaintext UDP) | No | Yes (running-cfg) | Lab/trusted networks, quick transfers |
| **SCP** | Moderate | Encrypted (SSH) | Yes | No (full replace) | Production environments |
| **USB / Flash** | Fast | Physical | No | No | Air-gapped, offline restore |
| **Console (Xmodem)** | Very slow | Physical | No | No | Emergency recovery, no network |

---

## IOS Firmware Upgrade via TFTP

This section documents the complete IOS upgrade procedure for the WS-C3560-24PS-E using TFTP. The target IOS image is `c3560-advipservicesk9-mz.122-37.SE1.bin`.

### Pre-Upgrade Checklist

- [ ] TFTP server is reachable from the switch (`ping 10.6.2.2`)
- [ ] IOS image file exists on TFTP server
- [ ] Flash has sufficient free space (`show flash:` — need 15 MB minimum free)
- [ ] Running config is backed up to TFTP and NVRAM
- [ ] Maintenance window is scheduled (upgrade requires reload)
- [ ] Rollback IOS image is retained on flash

### Step-by-Step Upgrade Procedure

```ios
! --- Step 1: Check current IOS version ---
Iskilip# show version
Cisco IOS Software, C3560 Software (C3560-ADVIPSERVICESK9-M), Version 12.2(37)SE1, RELEASE SOFTWARE (fc1)
Cisco WS-C3560-24PS-E (PowerPC405) processor with 122880K/10240K bytes of memory.
Model number  : WS-C3560-24PS-E
System serial : CAT1037RJF7
Base MAC      : 00:30:F2:D4:CE:7C

! --- Step 2: Verify available flash space ---
Iskilip# show flash:
Directory of flash:/
    1  -rwx  11607040   c3560-advipservicesk9-mz.122-37.SE1.bin
    2  -rwx      1531   Iskilip

64016384 bytes total (52409344 bytes free)
!  52 MB free -- sufficient for a new image

! --- Step 3: Download new IOS image from TFTP ---
Iskilip# copy tftp: flash:
Address or name of remote host []? 10.6.2.2
Source filename []? c3560-advipservicesk9-mz.122-37.SE1.bin
Destination filename [c3560-advipservicesk9-mz.122-37.SE1.bin]? [Enter]
Loading c3560-advipservicesk9-mz.122-37.SE1.bin from 10.6.2.2:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
[OK - 11607040 bytes]

! --- Step 4: Set boot system to new image ---
Iskilip# configure terminal
Iskilip(config)# no boot system
Iskilip(config)# boot system flash:c3560-advipservicesk9-mz.122-37.SE1.bin
Iskilip(config)# exit

! --- Step 5: Save configuration before reload ---
Iskilip# write memory
Building configuration...
[OK]

! --- Step 6: Reload the switch ---
Iskilip# reload
Proceed with reload? [confirm] [Enter]

! --- After reload: Verify new IOS is running ---
Iskilip# show version
!  Confirm version matches the expected image
```

### Rollback Procedure

If the upgrade fails or the new image is unstable, roll back to the previous IOS:

```ios
! --- Option 1: Boot from old image (if still on flash) ---
Iskilip# configure terminal
Iskilip(config)# no boot system
Iskilip(config)# boot system flash:<old-image-filename>.bin
Iskilip(config)# exit
Iskilip# write memory
Iskilip# reload

! --- Option 2: Boot from ROMMON (if IOS fails to load) ---
rommon 1> set BOOT=flash:c3560-advipservicesk9-mz.122-37.SE1.bin
rommon 2> boot

! --- Option 3: TFTP boot from ROMMON ---
rommon 1> set IP_ADDRESS=10.19.2.254
rommon 2> set IP_SUBNET_MASK=255.255.255.0
rommon 3> set DEFAULT_GATEWAY=10.19.2.1
rommon 4> set TFTP_SERVER=10.6.2.2
rommon 5> set TFTP_FILE=c3560-advipservicesk9-mz.122-37.SE1.bin
rommon 6> tftpdnld
```

### Maintenance Window Warning

> **WARNING:** The IOS upgrade requires a **full switch reload**. All connected devices will lose network connectivity during the reload cycle (typically 2-5 minutes). Schedule upgrades during low-traffic maintenance windows and notify affected users in advance.

---

## Command Reference Table

| Command | Mode | Description |
| :--- | :--- | :--- |
| `enable` | User EXEC | Enter privileged EXEC mode |
| `configure terminal` | Privileged EXEC | Enter global configuration mode |
| `interface vlan 1` | Global Config | Enter VLAN 1 SVI configuration |
| `ip address 10.19.2.254 255.255.255.0` | Interface Config | Assign management IP address to SVI |
| `no shutdown` | Interface Config | Enable the interface (bring it up) |
| `ip default-gateway 10.19.2.1` | Global Config | Set default gateway for L2 switch management |
| `write memory` | Privileged EXEC | Save running config to startup-config (NVRAM) |
| `copy running-config startup-config` | Privileged EXEC | Explicit save of running config to NVRAM |
| `ping 10.19.2.1` | Privileged EXEC | Test connectivity to the default gateway |
| `ping 10.6.2.2` | Privileged EXEC | Test connectivity to the TFTP server |
| `copy tftp: running-config` | Privileged EXEC | Restore (merge) config from TFTP server |
| `copy running-config tftp:` | Privileged EXEC | Back up running config to TFTP server |
| `copy tftp: flash:` | Privileged EXEC | Download IOS image from TFTP to flash |
| `boot system flash:<image>` | Global Config | Set IOS boot image from flash |
| `no boot system` | Global Config | Clear boot system statements |
| `show version` | Privileged EXEC | Display IOS version, hardware info, serial number |
| `show interface vlan 1` | Privileged EXEC | Verify VLAN 1 SVI status and IP address |
| `show ip default-gateway` | Privileged EXEC | Display configured default gateway |
| `show flash:` | Privileged EXEC | List files in flash, show free space |
| `show arp` | Privileged EXEC | Display ARP table |
| `show running-config` | Privileged EXEC | Display active (running) configuration |
| `show startup-config` | Privileged EXEC | Display saved (NVRAM) configuration |
| `reload` | Privileged EXEC | Reboot the switch (required after IOS upgrade) |

---

## Repository Structure

```text
cisco-c3560-vlan-tftp-firmware/
|
+-- README.md                        <- This file -- full lab documentation
|
+-- configs/
|   +-- switch-running-config.txt    <- Full IOS running configuration (post-TFTP restore)
|   +-- tftp-backup-config.txt       <- Config snapshot backed up via TFTP
|
+-- topologies/
|   +-- network_topology.txt         <- Detailed ASCII network topology diagram
|
+-- docs/
|   +-- firmware-upgrade-guide.md    <- Standalone IOS firmware upgrade procedure
|   +-- tftp-management-guide.md     <- Standalone TFTP config management guide
|
+-- LICENSE                          <- MIT License
+-- .gitignore                       <- Git ignore rules
```

---

## How to Run the Lab

1. **Open Cisco Packet Tracer** (v8.0 or later recommended)
2. **Build the topology** as shown in the [Network Topology](#network-topology) section
3. **Place devices:** One WS-C3560-24PS-E switch, one TFTP server (at 10.6.2.2), one router/PC at 10.19.2.1
4. **Configure the TFTP server:** Add the `Iskilip` config file (contents from `configs/switch-running-config.txt`) and the IOS image
5. **Open the switch CLI** and execute commands in order:
   - [VLAN 1 SVI Configuration](#configuration-commands)
   - [Default Gateway Configuration](#default-gateway-configuration)
   - [Connectivity Verification](#connectivity-verification)
   - [TFTP Restore](#restore-procedure)
   - [IOS Upgrade](#step-by-step-upgrade-procedure) (optional)
6. **Verify all outputs** match the expected results shown in each section

---

## Author and License

| Field | Details |
| :--- | :--- |
| **Author** | EmirEvren |
| **Project** | Cisco WS-C3560-24PS -- VLAN SVI and TFTP Lab |
| **Part of** | [4-City WAN Routing Project](https://github.com/EmirEvren/cisco-packet-tracer-4-city-wan-routing) |
| **License** | [MIT License](LICENSE) |
| **Year** | 2026 |

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

# 🛡️ FortiGate One-Arm Sniffer Lab Setup & Complete Troubleshooting Guide

A comprehensive, production-grade lab guide documenting the configuration, CLI/GUI debugging steps, syntax corrections, and key takeaways for deploying a **One-Arm Sniffer Mode** FortiGate Firewall instance within an **EVE-NG** emulation environment.

---

## 📜 Table of Contents
1. [Overview & Architecture](#-overview--architecture)
2. [Lab Topology & Connection Matrix](#-lab-topology--connection-matrix)
3. [System & Interface Status](#-system--interface-status)
4. [Step-by-Step Configuration](#-step-by-step-configuration)
   - [FortiGate One-Arm Sniffer Configuration](#1-fortigate-one-arm-sniffer-configuration)
   - [Verification via GUI](#2-verification-via-gui)
5. [Troubleshooting & Session Debug Logs](#-troubleshooting--session-debug-logs)
   - [Issue 1: Wireshark Integration Failure in EVE-NG](#issue-1-wireshark-integration-failure-in-eve-ng)
   - [Issue 2: Syntax Error in CLI Packet Sniffer Filters](#issue-2-syntax-error-in-cli-packet-sniffer-filters)
   - [Issue 3: `pcap_lookupnet: port2: no IPv4 address assigned`](#issue-3-pcap_lookupnet-port2-no-ipv4-address-assigned)
   - [Issue 4: Cisco L2 Switch SPAN / Port Mirroring Unsupported Command](#issue-4-cisco-l2-switch-span--port-mirroring-unsupported-command)
   - [Issue 5: ICMP Traffic Not Captured via General CLI Sniffer](#issue-5-icmp-traffic-not-captured-via-general-cli-sniffer)
6. [GUI Traffic Log Verification](#-gui-traffic-log-verification)
7. [Key Lessons Learned](#-key-lessons-learned)
8. [Useful Reference Commands](#-useful-reference-commands)

---

## 🌐 Overview & Architecture

**One-Arm Sniffer Mode** (also known as Passive Sniffing or IDS mode) allows a FortiGate firewall to operate passively on a single physical interface. In this deployment:
* The firewall interface receives a copy of network traffic mirrored from a switch port (SPAN port) or broadcasted across the local network segment.
* The interface operates in promiscuous mode **without an IP address assignment**.
* Security engines (IPS, Antivirus, Application Control) analyze incoming traffic and log security events without being inline or modifying packet pathways.

---

## 📐 Lab Topology & Connection Matrix

```
                 +-----------------------+
                 |  Management PC (MGMT) |
                 +-----------+-----------+
                             | eth0
                             |
                             | port1 (192.168.194.150/24)
                      +------+------+
                      |  FortiGate  |
                      |  Firewall   |
                      +------+------+
                             | port2 (One-Arm Sniffer Mode)
                             |
                             | Gi0/3
                     +-------+-------+
    +----------------+ Cisco L2      +----------------+
    |                | Switch        |                |
    | Gi0/1          +-------+-------+          Gi0/2 |
    |                        | Gi0/0                  |
+---+---+                +---+---+                +---+---+
| QEMU  |                | Net   |                | Net   |
| Win PC|                | Host  |                | Cloud |
+-------+                +-------+                +-------+
```

### Connection Matrix
| Node Name | Local Interface | Connected Node | Remote Interface | Purpose / Role |
| :--- | :--- | :--- | :--- | :--- |
| **FortiGate** | `port1` | MGMT PC | `eth0` | Out-of-Band Management (HTTPS/SSH) |
| **FortiGate** | `port2` | Switch | `Gi0/3` | Passive One-Arm Sniffer Interface |
| **Switch** | `Gi0/1` | QEMU Win PC | `e0` | Host Access Port |
| **Switch** | `Gi0/2` | Net Cloud | N/A | External Internet Gateway Connection |
| **Switch** | `Gi0/0` | Net | `eth0` | Additional LAN Host Connection |

---

## 🖥️ System & Interface Status

### System Hardware & License Overview
* **Hostname:** `FGVMEVQND5HBH504`
* **Firmware:** `v7.4.12 build2902 (Mature)`
* **Mode:** `NAT`
* **vCPUs / RAM:** 1 Allocated vCPU / 985 MiB of 2 GiB Memory Used (~48%-73% System Memory)
* **Management WAN IP:** `192.168.194.150` (`port1`)

### Physical Interfaces Summary
```
+------------------------------------------------------------------------------------------+
| Interface | Type               | IP/Netmask            | Administrative Access / Role    |
+-----------+--------------------+-----------------------+---------------------------------+
| port1     | Physical Interface | 192.168.194.150/24    | PING, HTTPS, HTTP, FMG-Access   |
| port2     | Physical Interface | 0.0.0.0/0.0.0.0       | One-Arm Sniffer (IPS Sniffer)   |
| port3     | Physical Interface | 0.0.0.0/0.0.0.0       | Unconfigured                    |
+------------------------------------------------------------------------------------------+
```

---

## ⚙️ Step-by-Step Configuration

### 1. FortiGate One-Arm Sniffer Configuration

To set up passive sniffing on `port2` using the FortiGate Command Line Interface (CLI):

```text
config system interface
    edit "port2"
        set vdom "root"
        set ips-sniffer-mode enable
        set type physical
        set description "Trying out one arm sniffer"
        set snmp-index 2
    next
end
```

### 2. Verification via GUI

1. Log into FortiGate GUI at `https://192.168.194.150`.
2. Navigate to **Network > Interfaces**.
3. Edit **port2**:
   - **Addressing mode:** Set to `One-Arm Sniffer`.
   - **Security Profiles:** Enable required inspection engines (e.g., *AntiVirus*, *Web Filter*, *Application Control*, *IPS*, *File Filter*).
   - **Logging Options:** Enable `Security Events` and select `All Sessions`.
   - **Status:** Ensure set to `Enabled`.

---

## 🔍 Troubleshooting & Session Debug Logs

During our active lab session, several critical issues were encountered and resolved step-by-step.

### Issue 1: Wireshark Integration Failure in EVE-NG
* **Symptom:** Clicking the link in EVE-NG opened CMD window loops without displaying packets inside Wireshark. The interface capture window showed local VMware adapters (`VMnet8`, `Local Area Connection`) instead of virtual lab pipes.
* **Root Cause:** Missing or broken background pipe connection via the EVE-NG Windows Client Integration Pack.
* **Resolution:** Switched to direct FortiGate CLI packet sniffing via `diagnose sniffer packet` commands and GUI log inspections.

---

### Issue 2: Syntax Error in CLI Packet Sniffer Filters
* **Symptom:** Executing `diagnose sniffer packet any "icmp" 4 0 1` thrown an error:
  `token line: Unmatched double quote.`
* **Root Cause:** Double quotes included improper ASCII formatting or syntax conflict in FortiOS CLI.
* **Resolution:** Formatted the sniffer command without quotes or with precise simple expressions:
  ```text
  # Correct Syntax
  diagnose sniffer packet any icmp 4 0 1
  ```

---

### Issue 3: `pcap_lookupnet: port2: no IPv4 address assigned`
* **Symptom:** Executing `diagnose sniffer packet port2 icmp 4 0 1` produced the error:
  `^Cpcap_lookupnet: port2: no IPv4 address assigned`
  and zero packets were captured.
* **Root Cause:** Standard interface-specific PCAP filters expect an active IPv4 stack on the target interface. Because `port2` was configured as `ips-sniffer-mode enable` (One-Arm Sniffer), it had no assigned IP address (`0.0.0.0`).
* **Resolution:** Used `any` as the interface parameter so FortiOS captures across promiscuous interfaces without requiring an active Layer 3 binding:
  ```text
  diagnose sniffer packet any icmp 4 0 l
  ```

---

### Issue 4: Cisco L2 Switch SPAN / Port Mirroring Unsupported Command
* **Symptom:** Executing `monitor session 1 source interface Gi1/0` on the Cisco Switch returned:
  `% Invalid input detected at '^' marker.`
  Executing `monitor ?` revealed only `event-trace`.
* **Root Cause:** The Cisco IOSvL2 image utilized in the lab lacked full hardware-based SPAN / Port Mirroring feature support in CLI.
* **Resolution:** Relied on native Layer 2 Ethernet broadcast flooding (ARP, NetBIOS, BOOTP, Spanning Tree Protocol) naturally received by `port2` on the shared switch domain, successfully triggering passive log generation on the FortiGate.

---

### Issue 5: ICMP Traffic Not Captured via General CLI Sniffer
* **Symptom:** Running `diagnose sniffer packet any icmp 4 0 1` returned `0 packets received by filter` while `ping 8.8.8.8` ran successfully from the QEMU PC.
* **Root Cause:** Unicast ICMP packets from QEMU PC traversed directly from `Gi0/1` to `Gi0/2` (Internet Gateway) without passing through `port2` because SPAN mirroring was inactive.
* **Resolution:** Removed the protocol filter constraint by running an open sniffer capture:
  ```text
  diagnose sniffer packet any '' 4 0 l
  ```
  This immediately revealed real-time capture of **STP**, **ARP**, **HTTPS (port1)**, and **Broadcast/Multicast** traffic across `port2`.

---

## 📊 GUI Traffic Log Verification

After configuring `port2` in **One-Arm Sniffer Mode** with logging enabled for `All Sessions`, incoming traffic was verified under **Log & Report > Sniffer Traffic**:

```
+-------------------------------------------------------------------------------------------------------+
| Date/Time           | Source Interface | Source           | Destination     | Application | Action    |
+---------------------+------------------+------------------+-----------------+-------------+-----------+
| 2026/10/15 08:29:34 | port2            | 192.168.194.115  | 192.168.194.255 | udp/138     | -         |
| 2026/10/15 08:25:13 | port2            | 192.168.194.1    | 224.0.0.251     | IP.Multicast| allow     |
| 2026/10/15 08:23:03 | port2            | 192.168.194.130  | 192.168.194.254 | BOOTP       | allow     |
| 2026/10/15 08:07:26 | port2            | 192.168.194.115  | 192.168.194.255 | NetBIOS     | allow     |
| 2026/10/15 08:01:43 | port2            | 192.168.194.254  | 192.168.194.132 | Ping        | allow     |
+-------------------------------------------------------------------------------------------------------+
```

---

## 💡 Key Lessons Learned

1. **One-Arm Sniffer Mechanics:** Passive sniffing requires `ips-sniffer-mode enable` on the physical interface. The interface operates in passive/promiscuous mode with no IP assignment.
2. **Sniffer Filtering Behavior:** Specifying an unassigned interface name directly in CLI sniffer commands (`port2`) results in PCAP lookup errors. Using `any` allows multi-interface packet logging regardless of L3 configuration.
3. **Difference Between Inline vs. Passive:** Unicast packets (e.g., PC pinging an external IP) require an explicit SPAN/mirroring configuration on the connecting switch to reach a One-Arm Sniffer. Broadcast/Multicast traffic is captured automatically.
4. **GUI Sniffer Log Repository:** Under FortiOS 7.4.x, sniffer activity can be monitored visually in real time under **Log & Report > Sniffer Traffic**.

---

## 🛠️ Useful Reference Commands

### FortiGate CLI Sniffer Commands
```text
# Basic open sniffer (all protocols, unlimited packets)
diagnose sniffer packet any '' 4 0 l

# Capture ICMP traffic only
diagnose sniffer packet any icmp 4 0 l

# Capture traffic to/from a specific IP address
diagnose sniffer packet any "host 8.8.8.8" 4 0 l

# Interface physical status check
get system interface physical
```

### Interface Configuration Template
```text
config system interface
    edit "port2"
        set vdom "root"
        set ips-sniffer-mode enable
        set type physical
        set description "One-Arm Passive Sniffer Interface"
    next
end
```

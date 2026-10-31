# FortiGate Firewall: Destination NAT (DNAT), Virtual IPs (VIP), and Port Forwarding Deep Dive

A comprehensive, production-grade lab and technical guide covering **Destination Network Address Translation (DNAT)**, **Virtual IP (VIP) mapping**, and **Port Forwarding** on FortiGate firewalls running FortiOS 7.x. This document provides theoretical concepts, network topology design, step-by-step graphical and CLI configuration steps, real-time troubleshooting workflows, and verification logs.

## Table of Contents

* [1. Overview & Theoretical Concepts](#1-overview--theoretical-concepts)

  * [What is Destination NAT (DNAT)?](#what-is-destination-nat-dnat)

  * [Why is DNAT Required in Modern Networks?](#why-is-dnat-required-in-modern-networks)

  * [Static DNAT vs. Port Forwarding (PAT)](#static-dnat-vs-port-forwarding-pat)

  * [Real-World Enterprise Use-Cases](#real-world-enterprise-use-cases)

* [2. Lab Topology & Environment Specifications](#2-lab-topology--environment-specifications)

  * [Network Topology Diagram](#network-topology-diagram)

  * [Addressing & Interface Baseline](#addressing--interface-baseline)

* [3. Step-by-Step Configuration Guide](#3-step-by-step-configuration-guide)

  * [Step 1: Baseline Interface & Routing Setup (CLI)](#step-1-baseline-interface--routing-setup-cli)

  * [Step 2: Configuring Virtual IP (VIP) on FortiGate (GUI)](#step-2-configuring-virtual-ip-vip-on-fortigate-gui)

  * [Step 3: Creating the DNAT Security Policy (GUI)](#step-3-creating-the-dnat-security-policy-gui)

  * [Step 4: DMZ Server Listener Setup (Tiny Core Linux)](#step-4-dmz-server-listener-setup-tiny-core-linux)

* [4. Verification & Testing Workflow](#4-verification--testing-workflow)

* [5. Lab Challenges & Troubleshooting Logs](#5-lab-challenges--troubleshooting-logs)

* [6. Repository Structure & Next Steps](#6-repository-structure--next-steps)

## 1. Overview & Theoretical Concepts

### What is Destination NAT (DNAT)?

Destination Network Address Translation (**DNAT**) is a technique where the **destination IP address** (and optionally the destination port) of an incoming IP packet is altered by an intermediate device (such as a FortiGate Firewall) before forwarding the packet to its final internal destination.

### Why is DNAT Required in Modern Networks?

1. **Public IP Conservation:** Organizations possess limited public IPv4 blocks. DNAT maps published public IPs directly to private RFC-1918 backend servers (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).

2. **Internal Infrastructure Obfuscation:** Prevents external threat actors from discovering internal topology addresses. Public users interact solely with the Firewall's Virtual IP.

3. **DMZ Protection:** Places sensitive application servers behind full Stateful Firewall inspection before traffic reaches the backend operating system.

### Static DNAT vs. Port Forwarding (PAT)

| Feature | Static Destination NAT (1-to-1) | Port Forwarding (1-to-Many / PAT) | 
| ----- | ----- | ----- | 
| **Translation Type** | Maps 1 External IP $\rightarrow$ 1 Internal IP (All Ports) | Maps 1 External IP + Specific Port $\rightarrow$ Internal IP + Port | 
| **Port Translation** | Keeps original destination port unchanged | Can translate ports (e.g., Ext `6000` $\rightarrow$ Int `80`) | 
| **Primary Use Case** | Dedicated enterprise web servers, Mail servers | Multi-service hosting, custom port obfuscation | 

### Real-World Enterprise Use-Cases

* **Web & Application Hosting:** Publishing internal DMZ web servers (`10.1.2.1:80`) to an external public IP (`172.29.129.150`).

* **Port Translation Security:** Hiding standard service ports behind non-standard high ports (e.g., `6000`) to mitigate automated scanning bots.

## 2. Lab Topology & Environment Specifications

### Network Topology Diagram

The lab environment is built using EVE-NG containing a FortiGate VM node (`FortiOS 7.0.9`), Management PC, Internal Linux node, DMZ Tiny Core Linux server, and an Outside Client node (`Linux3`).


*Figure 2.1: Complete EVE-NG network topology displaying FortiGate interfaces (MGMT, Inside, Outside, DMZ) connected to respective client and server nodes.*

### Addressing & Interface Baseline

| Node Name | Node Type / OS | Interface | IP Address / Subnet | Default Gateway | Function / Role | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| **FortiGate-VM** | FortiOS 7.0.9 | `port1` | `192.168.188.99/24` | N/A | Out-of-band Management (`MGMT`) | 
| **FortiGate-VM** | FortiOS 7.0.9 | `port2` | `10.1.1.254/24` | N/A | LAN / Inside Gateway | 
| **FortiGate-VM** | FortiOS 7.0.9 | `port3` | `172.29.129.1/24` | N/A | WAN / Outside Gateway | 
| **FortiGate-VM** | FortiOS 7.0.9 | `port4` | `10.1.2.254/24` | N/A | DMZ Gateway | 
| **Linux (DMZ Server)** | Tiny Core Linux | `eth0` | `10.1.2.1/24` | `10.1.2.254` | Web Server (HTTP Listener) | 
| **Linux3 (Client)** | Tiny Core GUI | `eth0` | `172.29.129.189/24` | `172.29.129.1` | External Internet Client | 

## 3. Step-by-Step Configuration Guide

### Step 1: Baseline Interface & Routing Setup (CLI)

Access the FortiGate CLI and configure interface IP addresses for MGMT, INSIDE, OUTSIDE, and DMZ.


*Figure 3.1: Initializing interface configurations and handling subnet overlap correction.*


*Figure 3.2: Completing port3 WAN IP allocation (`172.29.129.1/24`) and DMZ port4 setup.*

```
config system interface
    edit "port1"
        set alias "MGMT"
        set mode static
        set ip 192.168.188.99 255.255.255.0
        set allowaccess ping https http ssh fgfm
    next
    edit "port2"
        set alias "INSIDE"
        set mode static
        set ip 10.1.1.254 255.255.255.0
        set allowaccess ping https http ssh fgfm
    next
    edit "port3"
        set alias "OUTSIDE"
        set mode static
        set ip 172.29.129.1 255.255.255.0
        set allowaccess ping https http ssh fgfm
    next
    edit "port4"
        set alias "DMZ"
        set mode static
        set ip 10.1.2.254 255.255.255.0
        set allowaccess ping https http ssh fgfm
    next
end

```

### Step 2: Configuring Virtual IP (VIP) on FortiGate (GUI)

Before creating a firewall rule, a **Virtual IP (VIP)** must be created to define the translation object between external public IP and internal private server IP.


*Figure 3.3: Creating the Virtual IP object mapping external address `172.29.129.150` to internal DMZ server `10.1.2.1`.*

1. Navigate to **Policy & Objects > Virtual IPs**.

2. Click **Create New > VIP**.

3. Configure parameters: Name (`Outside-to-DMZ`), Interface (`OUTSIDE (port3)`), External IP (`172.29.129.150`), and Mapped IP (`10.1.2.1`).

### Step 3: Creating the DNAT Security Policy (GUI)

Create a stateful firewall policy allowing external traffic to hit the Virtual IP object.


*Figure 3.4: Defining the incoming OUTSIDE to DMZ security policy with Source NAT disabled.*

1. Navigate to **Policy & Objects > Firewall Policy** and click **Create New**.

2. Set Incoming Interface (`port3`) and Outgoing Interface (`port4`).

3. Set Source to `all` and Destination to the created Virtual IP object (`Outside-to-DMZ`).

4. Disable **NAT** switch and enable **All Sessions** logging.

## 4. Verification & Testing Workflow

1. **Verify Connectivity from Outside Client (`Linux3`):**
   Configure IP & Gateway on `Linux3` and test ping reachability to the WAN gateway (`172.29.129.1`).


*Figure 4.1: Successful ping verification between outside client `Linux3` and FortiGate outside gateway.*

2. **Access VIP via Web Browser:**
   Open Firefox on `Linux3` and browse to `http://172.29.129.150`.


*Figure 4.2: Tiny Core Linux DMZ server recording live HTTP GET requests incoming from the external client through the FortiGate DNAT mapping.*

3. **Check Dashboard Session Metrics:**
   Verify active sessions and traffic counters on the FortiGate dashboard.


*Figure 4.3: FortiGate Dashboard displaying active stateful sessions, CPU utilization, and system status.*

## 5. Lab Challenges & Troubleshooting Logs

During implementation, several operational challenges were encountered and resolved:

* **Subnet Overlap Error:** Assigned `port3` to a separate WAN subnet (`172.29.129.1/24`) after overlap error withMGMT subnet (`192.168.188.0/24`).

* **Missing cURL Binary:** Utilized pre-installed Tiny Core Linux Firefox browser instead of command-line cURL.

* **Firewall Unable to Connect:** Resolved by ensuring Source NAT was explicitly **Disabled** inside the DNAT security policy.

## 6. Repository Structure & Next Steps

```
salman-network-security-labs/
├── Fortigate-labs/
│   ├── Lab 01 — FortiGate Firewall Deployment...
│   ├── Lab 02 — Basic Fortigate Config...
│   ├── Lab 03 — FortiGate One-Arm SNAT...
│   ├── Lab 04 — FortiGate Virtual Wire...
│   ├── Lab 05 — FortiGate Interface Types...
│   ├── Lab 06 — FortiGate L3 Interfaces...
│   ├── Lab 07 — FortiGate Firewall (Source NAT, Dynamic IP Pool, PAT)...
│   └── Lab 08 — FortiGate Destination NAT & Port Forwarding/  <-- (NEW LAB FOLDER)
│       ├── README.md
│       ├── Topology.png
│       ├── Firewall_IP_Assign.png
│       ├── Firewall_IP_Assign2.png
│       ├── Firewall_Policy.png
│       ├── Fortigate_Dashboard.png
│       ├── Linux_monitoring.png
│       ├── Ping_dmz.png
│       └── Virtual_IP.png
├── palo-alto-labs/
├── LICENSE
└── README.md

```

### How to Create & Push the Next Folder to GitHub

```
cd salman-network-security-labs/Fortigate-labs/
mkdir "Lab 08 — FortiGate Destination NAT & Port Forwarding"
cd "Lab 08 — FortiGate Destination NAT & Port Forwarding"
# Paste all lab images and README.md here
git add .
git commit -m "Add Lab 08 - FortiGate Destination NAT (DNAT) & Port Forwarding"
git push origin main

```

*Documentation maintained by [Salman](https://github.com/Salman-1210)*
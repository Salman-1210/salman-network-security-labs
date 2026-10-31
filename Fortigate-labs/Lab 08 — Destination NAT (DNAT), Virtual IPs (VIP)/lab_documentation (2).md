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

* [6. What I Learned & Repository Structure](#6-what-i-learned--repository-structure)

## 1. Overview & Theoretical Concepts

### What is Destination NAT (DNAT)?

Destination Network Address Translation (**DNAT**) is a technique where the **destination IP address** (and optionally the destination port) of an incoming IP packet is altered by an intermediate device (such as a FortiGate Firewall) before forwarding the packet to its final internal destination.

### Why is DNAT Required in Modern Networks?

1. **Public IP Conservation:** Organizations possess limited public IPv4 blocks. DNAT maps published public IPs directly to private RFC-1918 backend servers (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).

2. **Internal Infrastructure Obfuscation:** Prevents external threat actors from discovering internal topology addresses. Public users interact solely with the Firewall's Virtual IP.

3. **DMZ Protection:** Places sensitive application servers behind full Stateful Firewall inspection before traffic reaches the backend operating system.

### Static DNAT vs. Port Forwarding (PAT)

* **Static DNAT (1-to-1):** Translates an entire external public IP address to an internal private server IP address across all ports.

* **Port Forwarding (PAT):** Translates specific incoming external ports to different internal ports or servers (e.g., mapping external port `6000` to internal port `80`).

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


*Figure 3.1: Initializing interface configurations and handling subnet overlap correction when attempting to assign conflicting subnets.*


*Figure 3.2: Successfully completing port3 WAN IP allocation (`172.29.129.1/24`) and DMZ port4 setup.*

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

### Step 3: Creating the DNAT Security Policy (GUI)

Create a stateful firewall policy allowing external traffic to hit the Virtual IP object.


*Figure 3.4: Defining the incoming OUTSIDE to DMZ security policy with Source NAT disabled and All Sessions logging enabled.*

## 4. Verification & Testing Workflow

1. **Verify Connectivity from Outside Client (`Linux3`):**
   Test ping reachability from `Linux3` to the WAN gateway (`172.29.129.1`).


*Figure 4.1: Successful ping verification showing 0% packet loss between outside client `Linux3` and FortiGate outside gateway.*

2. **Access VIP via Web Browser:**
   Open Firefox on `Linux3` and browse to `http://172.29.129.150`.


*Figure 4.2: Tiny Core Linux DMZ server recording live HTTP GET requests incoming from the external client through the FortiGate DNAT mapping.*

3. **Check Dashboard Session Metrics:**
   Verify active sessions and traffic counters on the FortiGate dashboard.


*Figure 4.3: FortiGate Dashboard displaying active stateful sessions, CPU utilization, and system status during traffic traversal.*

## 5. Lab Challenges & Troubleshooting Logs

During implementation, several operational challenges were encountered and resolved:

* **Subnet Overlap Error:** Encountered a `node_check_object fail!` error when attempting to assign an overlapping subnet on `port3`. Resolved by assigning a dedicated WAN subnet (`172.29.129.1/24`).

* **Missing cURL Binary:** The lightweight Tiny Core Linux image lacked pre-installed `curl`. Bypassed this by utilizing the pre-installed Firefox browser and Netcat (`nc`) listener.

* **Firewall Unable to Connect:** Initially faced "Unable to connect" errors on browser testing. Resolved by ensuring Source NAT switch was explicitly **Disabled** in the firewall policy because DNAT handles destination translation, not source translation.

## 6. What I Learned & Repository Structure

### What I Learned Today:

* How to configure Destination NAT (DNAT) using Virtual IPs (VIP) in FortiOS.

* The difference between Source NAT and Destination NAT policies.

* How to troubleshoot routing and policy hit counts using FortiGate logs and Netcat listeners.

* Implementing port forwarding for service obfuscation and security.

### Repository Structure & Next Folder Steps:

To add this lab to your GitHub repository, create a new folder named `Lab 08 — FortiGate Destination NAT & Port Forwarding` inside your `Fortigate-labs` directory and push your files:

```
cd salman-network-security-labs/Fortigate-labs/
mkdir "Lab 08 — FortiGate Destination NAT & Port Forwarding"
cd "Lab 08 — FortiGate Destination NAT & Port Forwarding"
# Place this README.md and all 8 PNG images here
git add .
git commit -m "Add Lab 08 - FortiGate Destination NAT (DNAT) & Port Forwarding"
git push origin main

```

*Documentation maintained by [Salman](https://github.com/Salman-1210)*
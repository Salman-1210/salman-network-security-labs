# FortiGate Firewall: Source NAT, NAT Pool, and PAT Implementation Guide

A detailed, technical guide covering the fundamentals of Network Address Translation (NAT), its variations, operational requirements, CLI session analysis, and step-by-step configuration on FortiGate Firewalls (FortiOS).

## Table of Contents

* [1. Overview and Core Concepts](#1-overview-and-core-concepts)
  * [What is NAT and Why is it Required?](#what-is-nat-and-why-is-it-required)
  * [Types of IP Addresses](#types-of-ip-addresses)
  * [Types of NAT](#types-of-nat)
* [2. Types of Source NAT](#2-types-of-source-nat)
* [3. System and Interface Baseline Setup](#3-system-and-interface-baseline-setup)
* [4. Step-by-Step Configuration Guide](#4-step-by-step-configuration-guide)
  * [Task 1: System Interfaces Verification](#task-1-system-interfaces-verification)
  * [Task 2: Firewall Policy and PAT Overload Configuration](#task-2-firewall-policy-and-pat-overload-configuration)
  * [Task 3: Dynamic IP Pool and One-to-One Static SNAT Configuration](#task-3-dynamic-ip-pool-and-one-to-one-static-snat-configuration)
* [5. Traffic Testing and Verification](#5-traffic-testing-and-verification)
  * [Client Outbound Traffic Testing](#client-outbound-traffic-testing)
  * [CLI Session Table Inspection](#cli-session-table-inspection)
  * [Forward Traffic Log Analysis](#forward-traffic-log-analysis)
* [6. Key Takeaways and Practical Lessons](#6-key-takeaways-and-practical-lessons)

---

## 1. Overview and Core Concepts

### What is NAT and Why is it Required?

Network Address Translation (NAT) is a Layer 3 method that changes IP address information in packet headers as traffic passes through a router or firewall.

* **Primary Purpose:** Translates non-routable private IP addresses from local networks into routable public IP addresses so packets can travel across the internet.
* **Key Reasons for Using NAT:**
  1. **Conserves IPv4 Addresses:** Allows hundreds of internal devices to share a single public IP address or a small pool of public addresses.
  2. **Improves Network Security:** Hides internal IP addressing schemes from external networks, blocking unsolicited inbound traffic.
  3. **Simplifies Routing:** Allows internal networks to use standard RFC 1918 private IP address ranges without acquiring public IP assignments for individual endpoints.

### Types of IP Addresses

| Address Type | Scope and Routability | Description |
| :--- | :--- | :--- |
| **Private IP** | Local Area Network (LAN) / Non-Routable | Addresses defined by RFC 1918 (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`). Public internet routers automatically drop these packets. |
| **Public IP** | Worldwide / Routable | Globally unique IP addresses assigned by Internet Service Providers (ISPs). These can route across the public internet. |

### Types of NAT

1. **Source NAT (SNAT):** Replaces the **Source IP** address in packet headers. Used when internal traffic (LAN) travels outward to external networks (WAN/Internet).
2. **Destination NAT (DNAT):** Replaces the **Destination IP** address in packet headers. Used to forward incoming external traffic to internal servers (such as Web servers in a DMZ).

---

## 2. Types of Source NAT

```text
                   +----------------------------------+
                   |         Source NAT Types         |
                   +----------------------------------+
                                    |
     +-----------------+------------+------------+------------------+
     |                 |                         |                  |
+----+-----+     +-----+-----+             +-----+-----+      +-----+-----+
|  Static  |     |  NAT Pool |             | Policy NAT|      |    PAT    |
| (1-to-1) |     | (Many-to- |             | (Conditional)|   | (1-to-Many|
+----------+     |   Many)   |             +-----------+      | /Overload)|
                 +-----------+                                +-----------+
```

1. **Static SNAT (1-to-1):** Maps a single internal private IP address directly to a single public IP address.
2. **NAT Pool (Many-to-Many):** Assigns a specified range of public IP addresses (for example, `192.168.188.129` to `192.168.188.135`) to a group of internal devices.
3. **Policy NAT:** Applies NAT rules only when traffic matches specific conditions, such as source address, destination address, and destination port.
4. **PAT (Port Address Translation / Overload):** Maps multiple private IP addresses to a single public IP address by assigning unique source port numbers to each connection.

---

## 3. System and Interface Baseline Setup

### Network Addressing Plan

* **Port 1 (Management):** `192.168.188.129/24` - Out-of-band management interface used to access the FortiGate GUI and CLI.
* **Port 2 (Inside / LAN Zone):** `10.1.1.10/24` - Gateway interface servicing internal workstations (`10.1.1.2`).
* **Port 3 (Outside / WAN Zone):** `192.168.188.131/24` - External interface connecting to the Internet gateway router.

---

## 4. Step-by-Step Configuration Guide

### Task 1: System Interfaces Verification

Before setting up translation rules, verify that physical interfaces, IP configurations, and administrative access settings are active.

![FortiGate Interfaces Overview](./Interfaces.png)
*Figure 1: `Interfaces.png` - FortiGate Interface list displaying Port 1 (Management), Port 2 (Inside/LAN), and Port 3 (Outside/WAN).*

#### Purpose:
Confirms that Layer 3 interface configurations are functioning and assigned to their respective network zones (`inside` and `wan`).

---

### Task 2: Firewall Policy and PAT Overload Configuration

#### Why I Configured It:
Configuring Port Address Translation (PAT) allows all internal LAN hosts (`10.1.1.0/24`) to access the internet through `port3` using a single outgoing IP address, minimizing public IP requirements.

![Firewall Policy Configuration](./Firewall_Policy.png)
*Figure 2: `Firewall_Policy.png` - Standard Firewall Policy table displaying active rule "Inside-Out" enabling outgoing interface NAT.*

#### GUI Step-by-Step Instructions:
1. Go to **Policy & Objects > Firewall Policy**.
2. Click **Create New** and enter:
   * **Name:** `Inside-Out`
   * **Incoming Interface:** `inside` (`port2`)
   * **Outgoing Interface:** `wan` (`port3`)
   * **Source:** `all`
   * **Destination:** `all`
   * **Schedule:** `always`
   * **Service:** `ALL`
   * **Action:** `ACCEPT`
3. Under **Firewall / Network Options**, enable **NAT**.
4. Select **Use Outgoing Interface Address**.
5. Click **OK**.

---

### Task 3: Dynamic IP Pool and One-to-One Static SNAT Configuration

#### Why I Configured It:
When specific servers or workstations require dedicated public IP identification instead of sharing the interface IP, a Dynamic IP Pool or Static 1-to-1 translation is configured.

![Dynamic IP Pool Configuration](./Dynamic_IP_POOL.png)
*Figure 3: `Dynamic_IP_POOL.png` - Configuring Dynamic IP Pool settings inside the Firewall Policy menu.*

![Static 1-to-1 Firewall Policy](./ONE-ONE_NAT_POLICY.png)
*Figure 4: `ONE-ONE_NAT_POLICY.png` - Firewall Policy overview showing the active rule set to Static 1-to-1 NAT mode.*

#### GUI Step-by-Step Instructions:
1. Go to **Policy & Objects > IP Pools** and click **Create New**.
2. Define the IP Pool parameters:
   * **Name:** `Static_Pool`
   * **Type:** `One-to-One`
   * **External IP Range:** `192.168.188.129` - `192.168.188.129`
   * **ARP Reply:** Enabled
3. Open **Policy & Objects > Firewall Policy** and edit policy `Inside-Out`.
4. Change **IP Pool Configuration** from *Use Outgoing Interface Address* to **Use Dynamic IP Pool**.
5. Select `Static_Pool` and click **OK**.

---

## 5. Traffic Testing and Verification

### Client Outbound Traffic Testing

To confirm internet connectivity and translation execution, outbound web sessions were generated from internal test client `10.1.1.2`.

![Traffic Generation Google](./Traffic_Generation.png)
*Figure 5: `Traffic_Generation.png` - Internal test machine accessing google.com during PAT overload testing.*

![Traffic Generation YouTube](./ONE-ONE_TRAFFIC_GENERATION.png)
*Figure 6: `ONE-ONE_TRAFFIC_GENERATION.png` - Internal test machine accessing youtube.com during Static 1-to-1 NAT testing.*

---

### CLI Session Table Inspection

To verify header translation at the FortiOS kernel level, run:

```bash
get system session list
```

#### 1. PAT Session Output Analysis

![NAT Session Translation Output](./NAT_Translation.png)
*Figure 7: `NAT_Translation.png` - CLI session output for PAT showing internal IP 10.1.1.2 translated to interface IP 192.168.188.131.*

* **Source (Internal):** `10.1.1.2`
* **Source-NAT (Translated):** `192.168.188.131` (`port3` IP)
* **Port Translation:** Port numbers (e.g., `52880`, `52512`) are dynamically modified per connection.

#### 2. Static 1-to-1 NAT Session Output Analysis

![Static One-to-One NAT Translation Output](./ONE-ONE_NAT_TRANSLATION.png)
*Figure 8: `ONE-ONE_NAT_TRANSLATION.png` - CLI session output for Static 1-to-1 NAT showing internal IP 10.1.1.2 translated to IP Pool 192.168.188.129.*

* **Source (Internal):** `10.1.1.2`
* **Source-NAT (Translated):** `192.168.188.129` (Assigned Dynamic IP Pool)
* **Port Mapping:** Original source ports are maintained where possible due to dedicated 1-to-1 mapping.

---

### Forward Traffic Log Analysis

![FortiGate Forward Traffic Log](./Traffic_Log.png)
*Figure 9: `Traffic_Log.png` - Forward Traffic logs showing allowed sessions, matching Policy ID 1, and active NAT status.*

#### Log Details Explained:
* **Source:** Private client address `10.1.1.2`.
* **Destination:** External target IP addresses (such as `8.8.8.8` or web server endpoints).
* **Policy ID:** Matches Policy ID `1` (`Inside-Out`).
* **NAT State:** Confirms translation is `Enabled`.

---

## 6. Key Takeaways and Practical Lessons

1. **Enterprise Boundary Requirement:** Source NAT is required on edge firewalls to connect RFC 1918 internal networks to the internet.
2. **Resource Efficiency of PAT:** Port Address Translation allows multiple internal endpoints to share a single public IP address concurrently using unique source ports.
3. **Kernel Session Diagnostics:** CLI diagnostics via `get system session list` provide real-time verification of source IP translation and port mappings beyond GUI logs.
4. **Proxy ARP Necessity:** When configuring IP Pools outside the primary physical interface subnet, enabling Proxy ARP allows upstream network routers to discover the firewall's translation pool MAC address.
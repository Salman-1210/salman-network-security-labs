# 🌐 FortiGate Firewall: Source NAT, NAT Pool & PAT Deep Dive (Lab 12)

A comprehensive, production-grade documentation covering the fundamentals of **Network Address Translation (NAT)**, its types, architectural requirements, CLI session verification, and hands-on configuration on **FortiGate Firewalls (FortiOS)**.

---

## 📜 Table of Contents
- [1. Overview & Core Concepts](#1-overview--core-concepts)
  - [What is NAT & Why is it Required?](#what-is-nat--why-is-it-required)
  - [Types of IP Addresses](#types-of-ip-addresses)
  - [Types of NAT](#types-of-nat)
- [2. Deep Dive: Source NAT Types](#2-deep-dive-source-nat-types)
- [3. Lab Topology & Baseline Setup](#3-lab-topology--baseline-setup)
- [4. What I Learned & What I Configured](#4-what-i-learned--what-i-configured)
  - [Task 1: Port Address Translation (PAT / Overload)](#task-1-port-address-translation-pat--overload)
  - [Task 2: One-to-One Static Source NAT (IP Pool)](#task-2-one-to-one-static-source-nat-ip-pool)
- [5. Verification & CLI Command Reference](#5-verification--cli-command-reference)
- [6. Key Takeaways](#6-key-takeaways)

---

## 1. Overview & Core Concepts

### What is NAT & Why is it Required?
**Network Address Translation (NAT)** is a crucial Layer 3 mechanism used to modify IP address information in packet headers while in transit across a routing device.

* **Primary Purpose:** Translates non-routable private IP addresses to routable public IP addresses so local network devices can access the internet.
* **Why it's essential:** 
  1. Preserves the limited global IPv4 address space.
  2. Hides internal network topology from external actors (Security boundary).
  3. Enables seamless outbound connectivity without re-addressing local networks.

---

### Types of IP Addresses
| Address Type | Scope / Routability | Description |
| :--- | :--- | :--- |
| **Private IP** | Local Area Network (LAN) / Non-Routable | Used inside home/office networks (e.g., `10.x.x.x`, `192.168.x.x`). Cannot traverse the public internet. |
| **Public IP** | Globally Unique / Routable | Assigned by ISPs. Directly accessible over the global internet. |

---

### Types of NAT
1. **Source NAT (SNAT):** Modifies the **Source IP** address in the IP header. Primarily used for outbound traffic (LAN to WAN/Internet).
2. **Destination NAT (DNAT):** Modifies the **Destination IP** address in the IP header. Used for inbound traffic (e.g., exposing an internal web server in a DMZ to external users).

---

## 2. Deep Dive: Source NAT Types

```
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

1. **Static SNAT (1-to-1):** Maps a single internal private IP address to a dedicated single public IP address.
2. **NAT Pool (Many-to-Many):** Translates internal hosts using a pool/range of external IP addresses (e.g., `200.1.1.1` to `200.1.1.10`).
3. **Policy NAT:** Translates traffic conditionally based on specific policy criteria (matching specific source, destination, and service parameters).
4. **PAT (Port Address Translation / Overload):** Maps multiple private IP addresses to a single public IP or outgoing interface IP address by appending dynamic port numbers to distinguish sessions.

---

## 3. Lab Topology & Baseline Setup

### Network Diagram
```
  +-----------------------+
  | Management PC (Local) |
  +-----------+-----------+
              | port1 (Management UI/CLI)
       +------+------+
       |  FortiGate  |
       |  Firewall   |
       +---+-----+---+
  port2    |     |    port3
 (Inside)  |     |   (Outside)
   +-------+     +-------+
   |                     |
+--+-------+         +---+-------+
| Local PC |         | Internet  |
| (Client) |         |  Cloud    |
+----------+         +-----------+
```

---

## 4. What I Learned & What I Configured

### Task 1: Port Address Translation (PAT / Overload)

#### **Why I Configured It:**
To allow all LAN clients to share a single public interface IP (`Port 3`) for outbound internet access, conserving public IP addresses while protecting internal addresses.

#### **GUI Configuration Steps:**
1. Navigated to **Policy & Objects > Firewall Policy**.
2. Created policy: `Inside-Out`.
   * **Incoming Interface:** `port2` (Inside)
   * **Outgoing Interface:** `port3` (Outside / WAN)
   * **Source / Destination:** `all`
   * **Service:** `ALL`
   * **Action:** `ACCEPT`
3. Enabled **NAT** and selected **"Use Outgoing Interface Address"**.

![Firewall Policy Configuration](./Firewall_Policy.png)

#### **CLI Configuration:**
```haproxy
config firewall policy
    edit 1
        set name "Inside-Out"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

---

### Task 2: One-to-One Static Source NAT (IP Pool)

#### **Why I Configured It:**
To demonstrate explicit 1-to-1 translation where a client traffic requires a deterministic translation address from a configured Dynamic IP Pool instead of the default interface IP.

#### **GUI Configuration Steps:**
1. Navigated to **Policy & Objects > IP Pools**.
2. Created dynamic IP pool `Static_Pool` with type **One-to-One** (or overload).
3. Updated Firewall Policy `Inside-Out`:
   * Enabled **NAT**.
   * Selected **"Use Dynamic IP Pool"** and bound the created IP Pool.

![Dynamic IP Pool Configuration](./Dynamic_IP_POOL.png)

![One-to-One Policy Binding](./ONE-ONE_NAT_POLICY.png)

---

## 5. Verification & CLI Command Reference

### Traffic Testing
Accessed external resources (`google.com`, `youtube.com`) from the internal test machine to generate real-time firewall session entries.

![Traffic Generation Test](./Traffic_Generation.png)

### FortiGate CLI Verification

To view the live active NAT session table and verify IP translation:

```bash
# Display active system session table
get system session list
```

#### **CLI Session Output Analysis:**

![PAT Session List](./NAT_Translation.png)

* **PAT Output Analysis:** The source `10.1.1.2` (Private IP) is translated to `192.168.188.131` (Interface IP) with changing dynamic source port numbers.

![Static NAT Translation](./ONE-ONE_NAT_TRANSLATION.png)

* **Static Pool Output Analysis:** The source `10.1.1.2` translates directly to the designated pool address (`192.168.188.129`).

### Log & Report Traffic Monitor
Checked **Log & Report > Forward Traffic** to confirm policy match ID and successful translation status.

![Traffic Logs](./Traffic_Log.png)

---

## 6. Key Takeaways

1. **Every Organization Uses NAT:** Whether small office home networks or enterprise data centers, NAT is mandatory for security and internet connectivity.
2. **PAT is Default for Outbound:** Port Address Translation allows hundreds of devices to surf the internet using a single public IP by changing L4 source ports.
3. **Session Table is Truth:** GUI gives status, but CLI `get system session list` reveals exact real-time translation state ($1:1$ vs PAT).
4. **Proxy ARP:** When using an IP Pool on a subnet different from the interface address, Proxy ARP must be enabled so upstream routers can route traffic back to the firewall.

---
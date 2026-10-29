# FortiGate Firewall: Source NAT, NAT Pool & PAT Deep Dive (Lab 12)

A comprehensive, production-grade documentation covering the fundamentals of **Network Address Translation (NAT)**, its types, architectural requirements, CLI session verification, and hands-on configuration on **FortiGate Firewalls (FortiOS)**.

## Table of Contents

* [1. Overview & Core Concepts](#1-overview--core-concepts)

  * [What is NAT & Why is it Required?](#what-is-nat--why-is-it-required)

  * [Types of IP Addresses](#types-of-ip-addresses)

  * [Types of NAT](#types-of-nat)

* [2. Deep Dive: Source NAT Types](#2-deep-dive-source-nat-types)

* [3. Lab Topology & Baseline Setup](#3-lab-topology--baseline-setup)

* [4. What I Learned & What I Configured](#4-what-i-learned--what-i-configured)

  * [Task 1: Interface & Zone Verification](#task-1-interface--zone-verification)

  * [Task 2: Port Address Translation (PAT / Overload)](#task-2-port-address-translation-pat--overload)

  * [Task 3: Dynamic IP Pool & Static One-to-One SNAT](#task-3-dynamic-ip-pool--static-one-to-one-snat)

* [5. Traffic Generation & Session Table Verification](#5-traffic-generation--session-table-verification)

  * [Client Outbound Traffic Testing](#client-outbound-traffic-testing)

  * [FortiGate CLI Session Verification (PAT vs Static NAT)](#fortigate-cli-session-verification-pat-vs-static-nat)

  * [Forward Traffic Log Monitoring](#forward-traffic-log-monitoring)

* [6. Key Takeaways & Enterprise Lessons Learned](#6-key-takeaways--enterprise-lessons-learned)

## 1. Overview & Core Concepts

### What is NAT & Why is it Required?

**Network Address Translation (NAT)** is a Layer 3 network mechanism that modifies IP address headers while packets traverse a routing/security boundary device.

* **Primary Purpose:** Translates internal non-routable private IP addresses into public routable IP addresses for internet connectivity.

* **Why it's essential:**

  1. **IPv4 Address Preservation:** Conserves global IPv4 address allocation.

  2. **Security & Anonymity:** Conceals internal IP topology from external actors.

  3. **Seamless Routing:** Enables internal hosts to access public endpoints without requiring globally unique IP addresses on every host.

### Types of IP Addresses

| Address Type | Scope / Routability | Description | 
| ----- | ----- | ----- | 
| **Private IP** | Local Area Network (LAN) / Non-Routable | RFC 1918 addresses (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`). Dropped at the internet edge. | 
| **Public IP** | Globally Unique / Routable | Assigned by RIRs/ISPs. Routable across the global internet. | 

### Types of NAT

1. **Source NAT (SNAT):** Modifies the **Source IP** address in the packet header. Used primarily for outbound internet access from LAN/DMZ to WAN.

2. **Destination NAT (DNAT):** Modifies the **Destination IP** address in the packet header. Used primarily to expose internal servers (e.g., Web/Mail in DMZ) to external users on WAN.

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

1. **Static SNAT (1-to-1):** Direct mapping between a single internal private IP and a single dedicated public IP address.

2. **NAT Pool (Many-to-Many):** Allocates a pool/range of public IP addresses (e.g., `200.1.1.1` to `200.1.1.10`) dynamically to internal clients.

3. **Policy NAT:** Conditional translation enforced only when traffic matches specific source, destination, and service parameters.

4. **PAT (Port Address Translation / Overload):** Maps multiple private IP addresses to a single public IP by changing Layer 4 port numbers dynamically.

## 3. Lab Topology & Baseline Setup


*Figure 1: Lab Topology showing Management PC on Port 1, LAN Client PC on Port 2 (Inside), and Internet Cloud Gateway on Port 3 (Outside/WAN).*

### Architecture Overview

* **FortiGate Firewall:** Acts as the central gateway performing security policy enforcement and NAT.

* **Port 1 (Management):** `192.168.188.129/24` — Dedicated Out-of-Band Management access for GUI & CLI.

* **Port 2 (Inside LAN):** `10.1.1.10/24` — Connects client hosts in the internal network.

* **Port 3 (Outside WAN):** `192.168.188.131/24` — Outbound interface facing upstream ISP/Internet.

## 4. What I Learned & What I Configured

### Task 1: Interface & Zone Verification

Before defining firewall policies, I verified interface IP assignments, administrative access permissions, and zone bindings on the FortiGate firewall.


*Figure 2: FortiGate Network Interfaces status showing `port1` (Management), `port2` (Inside/LAN), and `port3` (Outside/WAN).*

* **Configuration Purpose:** Ensures interface Layer 3 connectivity is operational and correct zones are mapped before configuring NAT policies.

### Task 2: Port Address Translation (PAT / Overload)

#### Why I Configured It:

To allow all internal LAN users (`10.1.1.0/24`) to access the internet using a single external interface IP (`port3`), reducing public IP address requirements to a single address.


*Figure 3: Active Firewall Policy `Inside-Out` enforcing Port Address Translation (PAT) from `inside` zone to `wan` zone.*

#### GUI Step-by-step:

1. Navigate to **Policy & Objects > Firewall Policy**.

2. Create policy **Inside-Out**:

   * **Incoming Interface:** `inside` (`port2`)

   * **Outgoing Interface:** `wan` (`port3`)

   * **Source / Destination:** `all`

   * **Service:** `ALL`

   * **Action:** `ACCEPT`

3. Enable **NAT** -> Select **"Use Outgoing Interface Address"**.

#### Equivalent CLI Command:

```
config firewall policy
    edit 1
        set name "Inside-Out"
        set srcintf "inside"
        set dstintf "wan"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end

```

### Task 3: Dynamic IP Pool & Static One-to-One SNAT

#### Why I Configured It:

When specific enterprise systems require deterministic outbound translation using a dedicated IP address range instead of sharing the interface IP, an **IP Pool** is configured.


*Figure 4: Configuring Dynamic IP Pool type Static (One-to-One) in Firewall Policy NAT options.*


*Figure 5: Policy Table displaying `Inside-Out` rule updated to use Static NAT IP Pool translation.*

#### GUI Step-by-step:

1. Navigate to **Policy & Objects > IP Pools** and click **Create New**.

2. Set Pool Type to **One-to-One** or **Overload**.

3. Edit Policy `Inside-Out` -> Under **IP Pool Configuration**, select **Use Dynamic IP Pool** and assign the created Pool.

## 5. Traffic Generation & Session Table Verification

### Client Outbound Traffic Testing

To verify end-to-end routing and NAT translation, outbound HTTPS/HTTP sessions were generated from the LAN client PC to public websites (`google.com`, `youtube.com`).


*Figure 6: LAN Client PC successfully loading Google web portal through FortiGate NAT Policy.*


*Figure 7: LAN Client PC accessing YouTube while Static 1-to-1 SNAT translation is active.*

### FortiGate CLI Session Verification (PAT vs Static NAT)

To inspect real-time packet translation in the kernel, I utilized FortiGate CLI session diagnostic tools.

```
# FortiGate CLI command to inspect session table
get system session list

```

#### Session Table Output Analysis (PAT):


*Figure 8: FortiGate CLI session table showing PAT translation where private source `10.1.1.2` translates to interface IP `192.168.188.131` with changing dynamic port numbers.*

#### Session Table Output Analysis (Static 1-to-1 NAT):


*Figure 9: FortiGate CLI session table showing 1-to-1 translation where private source `10.1.1.2` translates directly to designated IP Pool address `192.168.188.129`.*

### Forward Traffic Log Monitoring

Finally, I verified log generation to ensure session hits, policy matching, and bytes transferred are properly monitored under the administrative dashboard.


*Figure 10: FortiGate Log & Report > Forward Traffic table recording outbound client sessions, matched Policy ID (1), and active NAT translation status.*

## 6. Key Takeaways & Enterprise Lessons Learned

1. **NAT is Universal in Enterprise Security:** Every modern enterprise architecture requires NAT to decouple internal addressing schemes from external providers.

2. **PAT Efficiency:** Port Address Translation allows thousands of concurrent internal sessions to share a single public IP address using Layer 4 ephemeral source port tracking.

3. **CLI Session Inspection is Essential:** While GUI confirms rule hits, `get system session list` via CLI is the definitive tool to verify real-time Layer 3/Layer 4 IP and port translation headers.

4. **Proxy ARP Requirement:** When configuring IP Pools outside the outgoing interface IP subnet, Proxy ARP must be enabled so upstream routers can seamlessly resolve ARP requests for pool addresses.

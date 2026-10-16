# Lab 04 — FortiGate Virtual Wire Pair (V-Wire) Deployment & Troubleshooting Guide

This technical documentation covers the deployment, architectural design, step-by-step configuration, and comprehensive troubleshooting for integrating a **FortiGate Firewall in Virtual Wire Pair (Transparent / Stealth) Mode** within an EVE-NG environment.

---

## 📌 Executive Summary & Key Learnings

### What is Virtual Wire Pair (V-Wire)?
A Virtual Wire Pair (V-Wire) consists of two interfaces bound together logically without assigned IP addresses or MAC layer modifications. Traffic entering one port is directly forwarded out the other port, bridging the path at Layer 2 while allowing the FortiGate to process UTM inspections, security policies, application control, and full traffic logging in a complete stealth mode.

### Key Conceptual Takeaways:
* **Stealth Security Monitoring**: Hides the firewall from Layer 3 trace routes (`traceroute` / `tracert`) so attackers cannot detect the security gateway in the path.
* **Perimeter Security Deployment**: Ideal for inline placement between core routers and internal hosts, or directly in front of DMZ web servers without re-addressing existing subnets.
* **Bidirectional Policy Mapping**: Traffic passing through a Virtual Wire Pair requires explicit firewall policy pairs (Directional: Port A to Port B and Port B to Port A) to permit full TCP/UDP sessions.
* **Upstream/Downstream Offloading**: Layer 3 routing, default gateways, and NAT functions remain on the upstream router while FortiGate focuses strictly on deep packet inspection.

---

## 🏗️ Network Topology & Infrastructure Architecture

The network infrastructure connects a Windows client endpoint (`Win`) to an upstream Cisco vIOS Core Router through a transparent FortiGate Virtual Wire Pair (`port2` ↔ `port3`). FortiGate management is isolated on `port1`.

* **Topology Diagram**:

```
                       +-------------------------------+
                       |      Local Management PC      |
                       +---------------+---------------+
                                       | (Management Network)
                                       v
                                [ FortiGate port1 ]
                                (192.168.194.150)
                                       
+------------------+         +-------------------+         +-------------------+         +-------------------+
|  Windows Client  | (e0)    |  FortiGate port2  | (V-Wire)|  FortiGate port3  | (Gi0/0) |  Cisco vIOS Core  | (Gi0/1)  +----------+
|  (192.168.1.10)  +-------->+  (Inside V-Wire)  +========>+ (Outside V-Wire)  +-------->+  (192.168.1.254)  +--------->| Internet |
+------------------+         +-------------------+         +-------------------+         +---------+---------+          +----------+
                                                                                                   | (NAT / Default Route)
                                                                                                   v
                                                                                           [ Internet Cloud ]
```

---

## ⚙️ Step-by-Step Configuration Workflow

### 1. FortiGate Management & Global DNS Configuration

Management access was established via `port1` with static IP configuration, static default gateway routing, and public DNS servers to facilitate FortiCare cloud licensing.

```text
config system interface
    edit "port1"
        set mode static
        set ip 192.168.194.150 255.255.255.0
        set allowaccess http https ping fgfm
    next
end

config router static
    edit 1
        set gateway 192.168.194.2
        set device "port1"
    next
end

config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

### 2. Cisco vIOS Core Gateway & NAT Setup

The Cisco router hosts the Default Gateway IP (`192.168.1.254`) for the internal network and performs Dynamic Overload NAT (PAT) toward the internet.

```text
configure terminal

! Inside Interface (Facing FortiGate V-Wire Port 3)
interface GigabitEthernet0/0
 ip address 192.168.1.254 255.255.255.0
 ip nat inside
 no shutdown

! Outside Interface (Facing Internet Gateway)
interface GigabitEthernet0/1
 ip address dhcp
 ip nat outside
 no shutdown

! NAT ACL and Overload Translation
ip access-list standard NAT_ACL
 permit 192.168.1.0 0.0.0.255

ip nat inside source list NAT_ACL interface GigabitEthernet0/1 overload

! Static Gateway Route
ip route 0.0.0.0 0.0.0.0 192.168.194.2
end
write memory
```

### 3. FortiGate Virtual Wire Pair & Security Policy Configuration

1. **Create V-Wire Pair**:
   * **Name**: `V-wire-pair`
   * **Members**: `port2` and `port3`
2. **Create V-Wire Policy**:
   * **Name**: `Internet-Access`
   * **Virtual Wire Pair**: `V-wire-pair` (`port2` ↔ `port3`)
   * **Source**: `all`
   * **Destination**: `all`
   * **Service**: `ALL`
   * **Action**: `ACCEPT`
   * **NAT**: `Disabled` (Handled by Cisco Router)
   * **Logging**: `All Sessions`

---

## 🛠️ Debugging & Issue Resolution Log

### 1. DNS Resolution & Outbound Connectivity Failure
* **Problem**: ICMP ping tests to raw IP addresses (`8.8.8.8`) passed, but domain name resolution (`direct.fortinet.net`, `google.com`) timed out with `Unable to resolve hostname`.
* **Root Cause**: FortiOS VM instance lacked system-level DNS server configurations during initial provisioning.
* **Fix**: Added public DNS resolvers under `config system dns` and verified resolution using `execute ping google.com`.

### 2. FortiCare Trial License Authorization Mismatch
* **Problem**: The GUI displayed `VM is not licensed or license is invalid`, and CLI initialization logs reported `License invalid due to exceeding allowed 0 CPUs and 0 MB RAM`.
* **Root Cause**: Stale hardware UUID binding on the FortiCloud portal from a previous VM instance.
* **Fix**:
  1. Logged into [FortiCloud Support Portal](https://support.fortinet.com).
  2. Navigated to **Asset Management > Products**.
  3. Selected the active VM serial number (`FGVMEV8CNLPT4S6B`) and executed **Decommission This Unit**.
  4. Re-triggered license activation from FortiGate CLI (`execute fortiguard-log-login` / trial refresh reboot).

### 3. Client Packet Blackholing Across V-Wire
* **Problem**: Endpoints (`192.168.1.10`) could ping local interfaces but failed to reach external internet targets.
* **Root Cause**: Mismatched interface definitions for NAT inside/outside on the vIOS router alongside missing bidirectional forwarding rules in V-Wire mode.
* **Fix**: Re-applied `ip nat inside` on `Gi0/0` and `ip nat outside` on `Gi0/1`, ensured default route pointed to the upstream cloud gateway (`192.168.194.2`), and configured bidirectional policy execution on FortiGate V-Wire.

---

## 📊 Lab Verification & Results

* **Client IP Configuration**: IP `192.168.1.10` / Gateway `192.168.1.254` / DNS `8.8.8.8`.
* **Web Browsing Verification**: Successfully accessed `google.com` and `youtube.com` via client browser through stealth firewall path.
* **Session Logging**: Verified real-time HTTP/HTTPS/DNS sessions under **Log & Report > Forward Traffic** matching policy `Internet-Access`.

---

## 🔗 Repository Resources & Documentation Links

* **GitHub Repository Workspace**: [Salman Network Security Labs Repository](https://github.com/Salman-1210/salman-network-security-labs/tree/main/Fortigate-labs)
* **Related Documentation**:
  * [Firewall Types & Use Cases Guide](https://github.com/Salman-1210/salman-network-security-labs/blob/main/Fortigate-labs/Firewall-Types-and-Usecases.md)
  * [FortiGate Interface Types Guide](https://github.com/Salman-1210/salman-network-security-labs/blob/main/Fortigate-labs/Fortigate_interface_types_guide.md)
  * [Network Security Fundamentals](https://github.com/Salman-1210/salman-network-security-labs/blob/main/Fortigate-labs/Network-Security-Fundamentals.md)
# Lab 02: FortiGate Firewall Deployment & Initial Configuration on EVE-NG

**Time Required:** ~2 Hours
**Tools:** EVE-NG (Community), FortiGate VM64-KVM (v7.4.12), Windows PC, L2 Switch
**Platform:** EVE-NG + FortiGate VM-Series + Windows

---

## 1. Lab Objectives

By the end of this lab you will be able to:

- Understand the two FortiGate deployment modes: **NAT mode** and **Transparent mode**
- Perform **first-time setup** of a FortiGate (console access, admin password, management IP)
- Configure **LAN (inside)** and **WAN (outside)** interfaces
- Configure **DHCP on the WAN side** and a **dynamic default route**
- Configure **DNS servers** on the FortiGate
- Build an **outbound firewall policy with NAT (PAT)**
- Verify traffic with **logs, sessions, and pings**
- Understand **EVE-NG network types (Cloud0 vs NAT)** and connectivity to your local machine

---

## 2. Lab Topology

![Topology](screenshots/Topology.png)

```javascript
                    [MGMT PC]
                        |
                      port3            (Management / Out-of-Band)
                        |
[Win-PC] e0 ---- [Switch] Gi0/1 ---- port1 [FortiGate] port2 ---- [Internet]
  10.1.1.1        Gi0/0                      |   10.1.1.10       (DHCP / Cloud0)
```

### Device Details

| Device | Interface | IP Address | Role |
| --- | --- | --- | --- |
| Win-PC | e0 | 10.1.1.1/24 | Client PC |
| FortiGate | port1 | 10.1.1.10/255.255.255.0 | LAN / Inside (Gateway for PC) |
| FortiGate | port2 | DHCP (192.168.194.129/24) | WAN / Internet (Cloud0) |
| FortiGate | port3 | 192.168.194.150/24 | Management (GUI access from local PC) |
| Win-PC | Gateway | 10.1.1.10 | Points to FortiGate port1 |

---

## 3. FortiGate Deployment Modes (Theory First)

Before touching the CLI, know the two modes:

### 3.1 NAT Mode (used in this lab)

- All firewall interfaces become **Layer 3** — you assign IP addresses and do routing
- This is the **most common deployment**; you get the maximum benefit of FortiGate features
- The firewall performs **routing + policy + NAT + UTM**

### 3.2 Transparent Mode

- All interfaces become **Layer 2** — no IP addresses needed on data interfaces
- Only a **management IP** is configured (so you can manage the box)
- The firewall sits **invisibly (stealth mode)** between two segments — no routing changes required
- Policies are still enforced for inbound/outbound traffic

> 💡 Use transparent mode when you want to insert a firewall into an existing network **without re-IP'ing** anything; use NAT mode everywhere else.

---

## 4. Step-by-Step Configuration

### Step 4.1 — First Boot (Console Access)

Connect to the FortiGate console in EVE-NG:

- Default username: `admin`
- Default password: *(blank)* — the firewall will **force you to set a new password** on first login

### Step 4.2 — Configure the Management Interface (port3) via CLI

We'll give port3 a static IP in the same subnet as the EVE-NG **Cloud0 / NAT** network so we can open the GUI from our local browser.

```bash
config system interface
    edit "port3"
        set mode static
        set ip 192.168.194.150 255.255.255.0
        set allowaccess https http ssh ping fmgm
    next
end
```

![port3 CLI Config](screenshots/port3-cli-config.png)

> **Why `allowaccess`?** Without it, the firewall won't respond to GUI/SSH/ping on that interface. `https http ssh ping` = manage + test reachability, `fmgm` = FortiManager access.

Now from your local PC open the GUI:

```javascript
https://192.168.194.150
```

![Dashboard](screenshots/Dashboard.png)

### Step 4.3 — Configure the LAN Interface (port1)

**GUI:** `Network > Interfaces > port1 > Edit`

| Setting | Value |
| --- | --- |
| Mode | Manual / Static |
| IP/Netmask | 10.1.1.10/255.255.255.0 |
| Administrative Access | PING, HTTPS, SSH, HTTP (as needed) |

The Windows PC uses **10.1.1.10 as its default gateway**.

### Step 4.4 — Configure the WAN Interface (port2) as DHCP

**GUI:** `Network > Interfaces > port2 > Edit`

- Set **Mode = DHCP**
- Enable **"Retrieve default gateway from server"** — this lets the firewall learn the ISP gateway automatically from DHCP

In this lab, port2 got **192.168.194.129/24** from the Cloud0 network (simulating an ISP broadband connection).

### Step 4.5 — Configure DNS

**GUI:** `Network > DNS`

| Setting | Value |
| --- | --- |
| Primary DNS | 8.8.8.8 |
| Secondary DNS | 1.1.1.1 |

![DNS Config](screenshots/DNS-config.png)

### Step 4.6 — Default Route (Dynamic Gateway)

**GUI:** `Network > Static Routes > Create New`

| Setting | Value |
| --- | --- |
| Destination | 0.0.0.0/0.0.0.0 |
| Gateway Address | **Dynamic** (learned from DHCP on port2) |
| Interface | port2 |

Because the gateway is retrieved from the DHCP server, you don't need to hardcode it — the route updates automatically if the ISP gateway changes.

**Verify internet reachability from the FortiGate CLI:**

```bash
execute ping 8.8.8.8
```

### Step 4.7 — Outbound Firewall Policy with NAT (PAT)

**GUI:** `Policy & Objects > Firewall Policy > Create New`

| Field | Value |
| --- | --- |
| Name | `Outbound` |
| Incoming Interface | port1 (LAN) |
| Outgoing Interface | port2 (WAN) |
| Source | all |
| Destination | all |
| Schedule | always |
| Service | ALL |
| Action | ACCEPT |
| NAT | ✅ **Enabled (PAT on outgoing interface)** |
| Log | ✅ All Sessions / UTM (so we can see traffic) |

![Firewall Policy](screenshots/Firewall-policy-config.png)

> **NAT (PAT)** translates the inside private IPs (10.1.1.x) to the WAN IP of port2 — this is how the single PC reaches the internet.

### Step 4.8 — Verify from the Windows PC

On Win-PC (IP: 10.1.1.1, GW: 10.1.1.10):

```javascript
ping 10.1.1.10        → gateway reachable ✅
ping 8.8.8.8          → internet reachable ✅
open google.com       → browsing works ✅
```

Then check **Log & Report > Forward Traffic**:

![FortiGate Logs](screenshots/Fortigate-logs.png)

You should see sessions from `10.1.1.1` going out, matched to policy **Outbound (1)** with `Accept (Start)` — DNS to 8.8.8.8 and HTTPS to google.com etc.

### Step 4.9 — Interface Overview (Final State)

![Interfaces](screenshots/Interfaces-config.png)

| Interface | IP | Role |
| --- | --- | --- |
| port1 | 10.1.1.10/24 | LAN (with admin access) |
| port2 | 192.168.194.129/24 (DHCP) | WAN (internet) |
| port3 | 192.168.194.150/24 | Management |

---

## 5. EVE-NG Connectivity: Cloud0 vs NAT (Important!)

How do EVE-NG nodes talk to your **local machine** and the **internet**?

### Cloud0 (Management / bridged)

- Cloud0 is **bridged directly to your local machine's network**
- A node connected to Cloud0 gets an IP **in the same subnet as your PC** (DHCP or static)
- **You can access the node's GUI/SSH directly from your browser** (this is how we reach the FortiGate GUI on 192.168.194.150)
- The node also gets **direct internet access** through your PC's connection

### NAT (vmnet8 style)

- The node connects to an **internal EVE-NG DHCP server** and gets a 172.x.x.x address
- Traffic goes through the EVE-NG DHCP server → vmnet adapter → your local machine
- Nodes get **internet access**, but you **cannot directly access node GUIs** from your PC

### ⚠️ Why not Cloud0 on BOTH port2 and port3?

If port2 (WAN) and port3 (mgmt) both used Cloud0, both would land in the **same subnet on two different L3 interfaces** → **overlapping network**, routing breaks. That's why this lab uses:

- **port3 → Cloud0** (management access from local PC)
- **port2 → NAT/Internet** (simulated WAN with internet access)

---

## 6. Complete CLI Configuration Summary

```bash
# --- Management interface ---
config system interface
    edit "port3"
        set mode static
        set ip 192.168.194.150 255.255.255.0
        set allowaccess https http ssh ping fmgm
    next
end

# --- LAN interface ---
config system interface
    edit "port1"
        set mode static
        set ip 10.1.1.10 255.255.255.0
        set allowaccess ping https ssh http
    next
end

# --- WAN interface (DHCP, retrieve gateway from server) ---
config system interface
    edit "port2"
        set mode dhcp
        set allowaccess ping
    next
end

# --- DNS ---
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end

# --- Outbound firewall policy with NAT ---
config firewall policy
    edit 1
        set name "Outbound"
        set srcintf "port1"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set action accept
        set nat enable
        set logtraffic all
    next
end
```

---

## 7. Verification Commands

**From the FortiGate CLI:**

```bash
get system interface physical      # interface status & IPs
get router info routing-table all  # routing table (default route via DHCP gateway)
execute ping 8.8.8.8               # internet reachability
diagnose sniffer packet any 'host 10.1.1.1' 4   # live packet capture
diagnose firewall proute list      # policy routes
```

**GUI:** `Log & Report > Forward Traffic` — confirm traffic matches policy **Outbound** and is accepted.

---

## 8. Troubleshooting Cheatsheet

| Symptom | Check | Fix |
| --- | --- | --- |
| Can't open GUI on 192.168.194.150 | `allowaccess` on port3? Same subnet as your PC? | `set allowaccess https http ssh ping` on port3 |
| PC can't ping gateway 10.1.1.10 | Switch passing traffic? PC GW = 10.1.1.10? port1 IP correct? | Restart switch node in EVE-NG, verify links |
| Gateway pings OK, but no internet | Default route missing? DHCP gateway retrieved? Policy exists? | Check Static Routes + enable NAT on policy |
| Policy hit but no browsing | DNS misconfigured | Set DNS to 8.8.8.8 / 1.1.1.1 and test `execute ping 8.8.8.8` first |
| No logs visible | Log setting on policy | Set `set logtraffic all` on the policy |
| Works in lab, breaks in real branch | Overlapping subnets on two L3 interfaces | Never put two interfaces in the same subnet |

---

## 9. Key Lessons Learned

1. **NAT mode = L3 interfaces + routing**; Transparent mode = L2 stealth, only mgmt IP. NAT mode is the standard deployment.
2. **First-time setup is always console → management IP → GUI.** CLI first, GUI after.
3. **WAN on DHCP** with *"retrieve gateway from server"* = zero-touch default route for branch/broadband setups.
4. **A firewall policy without NAT won't get private IPs to the internet** — PAT on the outgoing interface is mandatory.
5. **Always enable logging on the policy** — otherwise you're flying blind.
6. **Cloud0 vs NAT in EVE-NG** decides whether you can manage devices from your PC and how internet egress works. Never put two interfaces in the same subnet.
7. Check **Layer 2 (switch/links) before Layer 3** — EVE-NG virtual links and switch nodes are frequent culprits.

---

## 10. Lab Completion Checklist

- [ ] Topology created in EVE-NG (PC ↔ Switch ↔ FortiGate ↔ Internet, MGMT PC on port3)
- [ ] Admin password changed on first login
- [ ] port3 configured (static mgmt IP + allowaccess)
- [ ] GUI accessible from local PC (`https://192.168.194.150`)
- [ ] port1 (LAN) = 10.1.1.10/24
- [ ] port2 (WAN) = DHCP with dynamic default route
- [ ] DNS configured (8.8.8.8 / 1.1.1.1)
- [ ] Outbound firewall policy created with NAT enabled
- [ ] PC pings gateway (10.1.1.10) ✅
- [ ] PC reaches internet (google.com) ✅
- [ ] Forward Traffic logs show sessions matching "Outbound" policy ✅

**Lab Completed Successfully! ✅**

---

*Documented by: Salman*
*Date: October 2026*
*Lab ID: 001-FORTIGATE-INITIAL*
*Platform: EVE-NG + FortiGate VM64-KVM v7.4.12 + Windows*
*Reference: 48-Day Firewall Training Series — Day 6 (FortiGate Initial Setup)*

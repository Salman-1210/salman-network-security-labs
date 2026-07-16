# Lab 011: Palo Alto Virtual Wire (VWIRE) Configuration

> **Difficulty:** Intermediate  
> **Time Required:** 4 HOURS
> **Tools:** EVE-NG, Palo Alto VM, Windows XP, Cisco vIOS Router, Switch

---

## 1. Lab Topology

```
                    [Win-PC2]
                       | e0
                       | (Management/Out-of-Band)
                       |
                       v
                    [ mgmt ]
                       |
    [Win-PC1]          |          [PaloAlto]          [vIOS-R1]          [Internet/Net]
        |              |              |                   |                   |
       e0          Gi0/0          eth1/1              Gi0/1               Gi0/0
        |              |              |                   |                   |
        +-------> [Switch] ------> [VWIRE] ----------> [Router] ----------> [Cloud]
                 Gi0/0   Gi0/1    eth1/1   eth1/2

    IP: 10.1.1.1         Virtual Wire (Transparent)      IP: 10.1.1.10
    GW: 10.1.1.10                                    GW: ISP/Next Hop
```

### Device Details:

| Device | Interface | IP Address | Role |
|--------|-----------|------------|------|
| Win-PC1 | e0 | 10.1.1.1/24 | Client PC |
| PaloAlto | eth1/1 | Layer 2 (VWIRE) | Trust Side |
| PaloAlto | eth1/2 | Layer 2 (VWIRE) | Untrust Side |
| vIOS-R1 | Gi0/1 | 10.1.1.10/24 | Gateway for PC1 |
| vIOS-R1 | Gi0/0 | DHCP/ISP | Internet |
| Win-PC2 | e0 | DHCP/Static | Management PC |

---

## 2. What is Virtual Wire (VWIRE)?

**Virtual Wire** is a Palo Alto Networks feature that allows the firewall to operate in **transparent Layer 2 mode** without changing IP addresses or network topology.

### Key Points:
- **No IP configuration** needed on firewall interfaces
- **No routing** performed by firewall
- **Invisibly sits** between two network segments
- **Inspects traffic** while forwarding at Layer 2
- **Security policies** still apply (zones, App-ID, etc.)

### Use Cases:
- Inserting firewall into existing network without re-IP
- Temporary security deployment
- Data center segmentation
- ISP peering points

---

## 3. Initial Configuration Steps

### Step 3.1: Configure Palo Alto Interfaces as Virtual Wire

**GUI Path:** `Network > Interfaces > Ethernet`

**For ethernet1/1 (Trust Side):**
```
Interface Type: Virtual Wire
Virtual Wire: vwire (or create new)
Zone: virtual-wire (or trust-vwire)
```

**For ethernet1/2 (Untrust Side):**
```
Interface Type: Virtual Wire
Virtual Wire: vwire (same as above)
Zone: virtual-wire (or untrust-vwire)
```

**CLI Commands:**
```bash
configure
set network interface ethernet ethernet1/1 virtual-wire vwire
set network interface ethernet ethernet1/2 virtual-wire vwire
set zone virtual-wire network layer2 ethernet1/1
set zone virtual-wire network layer2 ethernet1/2
commit
```

### Step 3.2: Verify Virtual Wire Binding

```bash
show vwire all
```

**Expected Output:**
```
Name: vwire
Interface1: ethernet1/1
Interface2: ethernet1/2
```

### Step 3.3: Configure Security Policy

**GUI Path:** `Policies > Security > Add`

| Field | Value |
|-------|-------|
| Name | Allow-VWIRE-Traffic |
| Source Zone | virtual-wire |
| Destination Zone | virtual-wire |
| Source Address | any |
| Destination Address | any |
| Application | any |
| Service | any (or application-default) |
| Action | Allow |

**CLI Commands:**
```bash
configure
set rulebase security rules Allow-VWIRE-Traffic from virtual-wire to virtual-wire source any destination any application any service any action allow
commit
```

---

## 4. The Problem: PC Cannot Ping Gateway (10.1.1.10)

### 4.1 Symptoms
- Win-PC1 (10.1.1.1) cannot ping 10.1.1.10 (Router)
- Ping to 10.1.1.10 **fails**
- Firewall shows **no active sessions**
- Firewall ARP table is **empty (0 entries)**

### 4.2 Initial Checks

**Check PC IP Configuration:**
```cmd
ipconfig
```
**Expected:**
```
IP Address: 10.1.1.1
Subnet Mask: 255.255.255.0
Default Gateway: 10.1.1.10
```

**Check Firewall Interfaces:**
```bash
show interface all
```
**Expected:**
```
ethernet1/1: up, zone: virtual-wire, forwarding: vwire:ethernet1/2
ethernet1/2: up, zone: virtual-wire, forwarding: vwire:ethernet1/1
```

**Check Security Policy:**
```bash
show running security-policy
```
**Expected:**
```
Name: Allow-VWIRE-Traffic
From: virtual-wire, To: virtual-wire
Action: allow
```

---

## 5. DEBUGGING PHASE (Step-by-Step)

### 5.1 Step 1: Verify Traffic Reaches Firewall

**On PC1, start continuous ping:**
```cmd
ping -t 10.1.1.10
```

**On Firewall, check sessions:**
```bash
show session all
```
**Result:** No sessions found ❌

**On Firewall, check ARP:**
```bash
show arp all
```
**Result:** 0 entries ❌

**Diagnosis:** Traffic is NOT reaching the firewall at all!

---

### 5.2 Step 2: Check Switch (The Real Culprit)

**Problem Identified:** The switch between PC and firewall is NOT passing traffic!

**On Switch, check interfaces:**
```bash
show interfaces status
```

**On Switch, check VLANs:**
```bash
show vlan brief
```

**Common Switch Issues in EVE-NG:**
1. Switch not properly booted
2. Ports in different VLANs
3. Virtual link stale/disconnected
4. Switch node needs restart

---

### 5.3 Step 3: Verify Switch Port Counters

**Clear counters first:**
```bash
clear counters
```

**Check specific interfaces during ping:**
```bash
show interfaces Gi0/0
show interfaces Gi0/1
```

**Look for:**
```
packets input: 0  (if still 0, traffic not arriving)
```

---

### 5.4 Step 4: EVE-NG Link Verification

**Check topology connections:**
1. Right-click on links between PC ↔ Switch ↔ Firewall
2. Verify link status (should be green/active)
3. Check interface mapping matches topology

**Quick Fix - Restart Switch Node:**
1. Right-click Switch node in EVE-NG
2. Select **Stop**
3. Wait 10 seconds
4. Select **Start**
5. Verify all links reconnect

---

## 6. The Solution

### Root Cause
**Layer 2 connectivity issue** between PC and Firewall via Switch. The Palo Alto configuration (VWIRE + Security Policy) was **correct all along**.

### Fix Applied
1. **Restarted Switch node** in EVE-NG
2. **Verified** all virtual links reconnected properly
3. **Confirmed** PC could now reach firewall

### Verification After Fix

**On PC1:**
```cmd
ping 10.1.1.10
```
**Result:** Reply from 10.1.1.10 ✅

**On Firewall:**
```bash
show session all
```
**Result:** Sessions visible ✅

```bash
show arp all
```
**Result:** 10.1.1.1 entry visible ✅

---

## 7. Key Lessons Learned

### 7.1 VWIRE Troubleshooting Checklist

| Step | Command | What to Check |
|------|---------|---------------|
| 1 | `ipconfig` (PC) | PC IP, Gateway correct? |
| 2 | `show interface all` (FW) | Interfaces up in VWIRE mode? |
| 3 | `show vwire all` (FW) | Both interfaces bound to same vwire? |
| 4 | `show running security-policy` (FW) | Allow rule exists for vwire zone? |
| 5 | `show session all` (FW) | Traffic reaching firewall? |
| 6 | `show arp all` (FW) | ARP entries populated? |
| 7 | `show interfaces` (Switch) | Switch passing traffic? |
| 8 | Topology check (EVE-NG) | Links properly connected? |

### 7.2 Important Notes

1. **VWIRE does NOT need IP addresses** on firewall interfaces
2. **Security policies ARE required** even in VWIRE mode
3. **Default interzone policy is DENY** - you MUST create explicit allow rules
4. **Switch issues** can appear as firewall problems
5. **Always check Layer 2** before Layer 3 in VWIRE setups

---

## 8. Complete Configuration Summary

### 8.1 Palo Alto Firewall (Full Config)

```bash
# Enter configuration mode
configure

# Set interfaces to Virtual Wire
set network interface ethernet ethernet1/1 virtual-wire vwire
set network interface ethernet ethernet1/2 virtual-wire vwire

# Assign zones (both in same zone for intra-zone traffic)
set zone virtual-wire network layer2 ethernet1/1
set zone virtual-wire network layer2 ethernet1/2

# Create security policy
set rulebase security rules Allow-VWIRE-Traffic from virtual-wire to virtual-wire source any destination any application any service any action allow

# Commit changes
commit
```

### 8.2 Router (vIOS-R1) Configuration

```bash
interface GigabitEthernet0/1
 ip address 10.1.1.10 255.255.255.0
 no shutdown

interface GigabitEthernet0/0
 ip address dhcp  (or static ISP IP)
 no shutdown

# Enable routing if needed
ip routing
```

### 8.3 PC1 Configuration

```
IP Address: 10.1.1.1
Subnet Mask: 255.255.255.0
Default Gateway: 10.1.1.10
DNS: 8.8.8.8 (optional)
```

---

## 9. Verification Commands

### 9.1 From PC1
```cmd
ping 10.1.1.10          # Test gateway reachability
ping 8.8.8.8            # Test internet (if router configured)
traceroute 10.1.1.10    # See path
```

### 9.2 From Firewall
```bash
show interface all              # Interface status
show vwire all                  # Virtual wire binding
show running security-policy    # Security rules
show session all                # Active sessions
show arp all                    # ARP table
show counter global filter packet_filter yes  # Packet counters
```

### 9.3 From Router
```bash
show ip interface brief         # Interface status
show ip route                   # Routing table
ping 10.1.1.1 source 10.1.1.10 # Test back to PC
```

---

## 10. Common Mistakes to Avoid

| Mistake | Why It Happens | Solution |
|---------|---------------|----------|
| Forgetting security policy | Think VWIRE = auto-allow | Always create explicit allow rule |
| Wrong zone assignment | eth1/1 and eth1/2 in different zones | Put both in same zone or create inter-zone rule |
| Switch not passing traffic | EVE-NG virtual link issues | Restart switch node, verify topology |
| Checking NAT first | NAT not needed for local traffic | Check Layer 2 first in VWIRE |
| Not verifying ARP | Assuming Layer 3 issue | `show arp all` tells if traffic arrives |

---

## 11. Lab Completion Checklist

- [ ] Topology created in EVE-NG
- [ ] Palo Alto interfaces configured as VWIRE
- [ ] Virtual Wire object created and bound
- [ ] Security policy created (Allow any)
- [ ] Router configured with 10.1.1.10
- [ ] PC configured with 10.1.1.1, GW 10.1.1.10
- [ ] Switch passing traffic (verify with `show interfaces`)
- [ ] Ping from PC to 10.1.1.10 **successful**
- [ ] Firewall shows active sessions
- [ ] Firewall ARP table shows 10.1.1.1

---

**Lab Completed Successfully! ✅**

> **Documented by:** Salman  
> **Date:** July 2026  
> **Lab ID:** 011-VWIRE  
> **Platform:** EVE-NG + Palo Alto VM-Series + vIOS

# Lab 03 — TAP Interface and Zone Configuration
**Platform:** Palo Alto Firewall (PA-VM) | **Environment:** EVE-NG  
**Reference:** Senior Network Engineer Training 2023  
**Author:** Salman | **Date:** 2023  

---

## Objective

Configure a TAP (Test Access Point) interface and zone on a Palo Alto firewall to passively monitor network traffic without being inline. Verify traffic visibility through the Monitor > Session Browser.

---

## What is TAP Mode?

TAP (Test Access Point) mode allows a Palo Alto firewall to **passively monitor** traffic by connecting to a SPAN/mirror port on a switch. The firewall receives a copy of traffic but does **not** interfere with it — no blocking, no routing, just visibility.

| Feature | TAP Mode | Layer 3 Mode |
|---|---|---|
| Inline traffic | ❌ No | ✅ Yes |
| Blocks threats | ❌ No | ✅ Yes |
| Traffic visibility | ✅ Yes | ✅ Yes |
| IP address on interface | ❌ Not required | ✅ Required |
| Use case | Monitoring / Audit | Production firewall |

### Devices Used

| Device | Role | Interface |
|---|---|---|
| PaloAlto PA-VM | Firewall (TAP mode monitoring) | mgmt, eth1/1 |
| Switch | Layer 2 switching | Gi0/1, Gi0/0 |
| Win PC (Management) | Admin access to PaloAlto GUI | e0 |
| vSRX | Router | ge-0/0/0, ge-0/0/1, ge-0/0/2 |
| Win PC (Inside) | Traffic generator | e0 |
| Net (Cloud) | Internet/Outside | — |

---

## Lab Tasks

### Task 1 — Create TAP Zone

1. Login to PaloAlto GUI: `https://192.168.1.1`
2. Navigate to: **Network → Zones → Add**
3. Configure:

| Field | Value |
|---|---|
| Name | `TAP-Zone` |
| Type | `TAP` |

4. Click **OK**

---

### Task 2 — Configure TAP Interface

1. Navigate to: **Network → Interfaces → Ethernet**
2. Click on **ethernet1/1**
3. Configure:

| Field | Value |
|---|---|
| Interface Type | `TAP` |
| Zone | `TAP-Zone` |
| IP Address | *(not required in TAP mode)* |

4. Click **OK**
5. Click **Commit**

---

### Task 3 — Create Security Policy

1. Navigate to: **Policies → Security → Add**
2. Configure:

| Field | Value |
|---|---|
| Name | `TAP-Monitor-Policy` |
| Source Zone | `TAP-Zone` |
| Destination Zone | `TAP-Zone` |
| Application | `any` |
| Action | `Allow` |

3. Click **OK**
4. Click **Commit**

---

### Task 4 — Create Security Profile & Monitor Session Browser

1. Navigate to: **Objects → Security Profiles**
2. Attach profiles to TAP policy (Antivirus, Vulnerability, URL Filtering)
3. To verify traffic:
   - Go to: **Monitor → Session Browser**
   - Traffic from Inside Win PC (browsing) will appear captured by TAP interface
   - Columns visible: Start Time, From Zone, To Zone, Source, Destination, Application, Rule

---

## Key Concepts Learned

### TAP vs Inline Mode
- TAP mode = **passive monitoring** — firewall sees traffic but does not block
- Used in **proof of concept** or **audit** scenarios before going inline
- No IP needed on TAP interface — it just listens

### Zone-Based Policy (vs Cisco ASA Interface-Based)
- Palo Alto uses **zones** — multiple interfaces can belong to one zone
- One rule covers entire zone — **less rules, more manageable**
- Cisco ASA uses per-interface ACLs — more rules, harder to scale

### EVE-NG Lab Notes
- PaloAlto default credentials: `admin / admin` (forced password change on first login)
- Management IP configured via CLI: `set deviceconfig system ip-address`
- TAP interface does **not** require IP address assignment
- Commit required after every configuration change

---

## CLI Reference

```bash
# Enter configuration mode
configure

# Set management IP
set deviceconfig system ip-address 192.168.1.1 netmask 255.255.255.0 default-gateway 192.168.1.254

# Configure TAP interface (CLI method)
set network interface ethernet ethernet1/1 tap

# Create TAP Zone and assign interface
set zone TAP-Zone network tap ethernet1/1

# Commit all changes
commit

# Verify management interface
show interface management
```

---

## Troubleshooting Encountered

| Issue | Cause | Fix |
|---|---|---|
| `admin/admin` login failed | PA-VM forces password change on first boot | Enter old password → set new password |
| Management IP showing `unknown` | Wrong gateway typed (`192.169` instead of `192.168`) | Re-enter correct IP and commit |
| Ping from Win PC failed (100% loss) | mgmt interface IP not applied | Recommit with correct gateway |
| Win PC VNC console not opening | Console type mismatch in EVE-NG | Change console type or use HTML5 console |
| "Net" node not found in templates | Net/Cloud is not a node in EVE-NG | Use Add Network → Cloud object |

---

## Screenshots

> *(Add screenshots here: TAP Zone config, Interface config, Security Policy, Monitor Session Browser)*

---

## References

- Palo Alto Networks Official Documentation: [docs.paloaltonetworks.com](https://docs.paloaltonetworks.com)
- EVE-NG Documentation: [eve-ng.net](https://www.eve-ng.net)
- Training: TAP Interface and Zone — Senior Network Engineer 2023

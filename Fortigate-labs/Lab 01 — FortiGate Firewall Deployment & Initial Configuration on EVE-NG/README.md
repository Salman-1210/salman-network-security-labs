# Lab 01 — FortiGate Firewall Deployment & Initial Configuration on EVE-NG

**Platform:** FortiGate-VM64 (KVM) | **Environment:** EVE-NG
**Reference:** FortiGate Firewall Training Series — Day 1 (Hands-On)
**Author:** Salman | **Date:** 2026-10-11

---

## Objective

Deploy a FortiGate-VM firewall inside EVE-NG, perform the first-boot setup, request the FortiCare evaluation license, and verify reachability (CLI ping to the Internet, workstation ping to the firewall) before accessing the GUI. This lab is the foundation for all upcoming FortiGate hands-on labs in this series.

## Lab Overview

This was **Day 1 of hands-on work on FortiGate firewall**. The firewall image was already imported into EVE-NG (import process documented in the Day 5 video of the training series — see References). Today's work covered:

1. First login & password change
2. FortiCare trial license request
3. Management/network configuration via CLI (DNS, default route, port1 IP)
4. Connectivity verification (Internet + workstation)
5. GUI access & dashboard overview

## Devices Used

| Device | Role | Interface | IP Address |
| --- | --- | --- | --- |
| FortiGate-VM64 | Firewall (evaluation) | port1 | 192.168.194.158 /24 |
| Win PC (Workstation) | Management / verification | — | 192.168.194.0/24 subnet |
| Net (Cloud) | Internet / Outside | — | Gateway: 192.168.194.1 |

> The lab runs entirely inside EVE-NG. The firewall's `port1` is bridged toward the management network so both CLI and GUI are reachable from the workstation.

## Prerequisites

- FortiGate KVM image imported in EVE-NG at `/opt/unetlab/addons/qemu/` with correct naming convention (see *EVE-NG Notes* below)
- EVE-NG lab created with one FortiGate node (1 vCPU / 1 GB RAM — image is only ~72 MB)
- Console access to the node (native or HTML5)

---

## Lab Tasks

### Task 1 — First Login & Password Change

Open the FortiGate console. Default credentials on first boot:

```javascript
Login: admin
Password: (blank)
```

The system forces a password change on first login:

```javascript
New password: ********
```

### Task 2 — Request FortiCare Evaluation License

Request the built-in 15-day evaluation license directly from CLI:

```javascript
execute FortiCare trial-license proxy {host:port}
```

Sample output:

```javascript
Requesting FortiCare Trial license, proxy(null)...
License status: no update error
```

> **Note:** The first request returned `no update error` — this is expected behavior when the box has no DNS/DNS resolution yet. Continue with Task 3 (network config), then re-request or verify the license status.

### Task 3 — Initial Network Configuration (CLI)

Configure DNS, default route, and the management interface `port1`:

```javascript
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end

config router static
    edit 1
        set gateway 192.168.194.1
    next
end

config system interface
    edit "port1"
        set ip 192.168.194.158 255.255.255.0
        set allowaccess ping https ssh http fgfm
    next
end
```

**Key point:** `allowaccess` controls which protocols are permitted on the interface. At minimum enable `ping https ssh`; add `http` as a fallback if HTTPS does not load in the browser.

### Task 4 — Verify Connectivity

Verify upstream/Internet reachability from the firewall itself:

```javascript
execute ping 8.8.8.8
execute ping google.com
```

Sample output:

```javascript
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=21.0 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=22.0 ms
...
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 21.0/21.6/22.2 ms
```

Then verify from the workstation (Windows CMD):

```javascript
C:\> ping 192.168.194.158

Reply from 192.168.194.158: bytes=32 time=1ms TTL=255
Reply from 192.168.194.158: bytes=32 time=1ms TTL=255
Reply from 192.168.194.158: bytes=32 time=1ms TTL=255
Reply from 192.168.194.158: bytes=32 time=1ms TTL=255

Ping statistics for 192.168.194.158:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
```

### Task 5 — Access the GUI Dashboard

Browse to the management IP:

```javascript
https://192.168.194.158
```

Login with `admin` and the password set in Task 1. The dashboard shows:

- System Information (model FortiGate-VM64, serial number)
- License Status — **Evaluation, 15 days remaining**
- CPU / Memory utilization graphs
- Interface & session statistics

---

## Key Concepts Learned

| Concept | Detail |
| --- | --- |
| Evaluation license | FortiGate-VM ships with a **15-day evaluation license** — no FortiCare account needed for lab work |
| License renewal trick | If the evaluation expires, simply **delete the node and re-add it** in EVE-NG — the 15-day counter restarts |
| `allowaccess` | FortiGate interfaces allow **no management protocols by default** — you must explicitly permit ping/https/ssh |
| DNS before license | The trial license request needs working DNS (`no update error` = DNS not configured yet) |
| CLI-first workflow | Day-1 config is always done via CLI console before GUI is reachable |

## EVE-NG Notes

- **Image naming convention:** folder must be named `fortinet-FGT-v7.0-buildXXXX-FORTINET.out` — anything after the hyphen is free text, but the prefix must match exactly or the node won't appear in the template list.
- Path: `/opt/unetlab/addons/qemu/<image-folder>/`
- After uploading, run: `cd /opt/unetlab/addons/qemu/<folder> && /opt/unetlab/wrappers/unl_wrapper -a fixpermissions`
- Resource footprint: **1 vCPU / 1 GB RAM** — boots in under a minute.
- FortiGate default credentials: `admin` / *(blank)* — forced password change on first login.

## CLI Reference (Full Day-1 Command Set)

```javascript
# First login
admin / (blank) → forced password change

# Request evaluation license
execute FortiCare trial-license proxy {host:port}

# DNS
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end

# Default route
config router static
    edit 1
        set gateway 192.168.194.1
    next
end

# Management interface
config system interface
    edit "port1"
        set ip 192.168.194.158 255.255.255.0
        set allowaccess ping https ssh http fgfm
    next
end

# Verification
execute ping 8.8.8.8
execute ping google.com
```

## Troubleshooting Encountered

| Issue | Cause | Fix |
| --- | --- | --- |
| `License status: no update error` | No DNS configured yet — FortiCare cannot be reached | Configure DNS (Task 3) first, then re-request/verify license |
| GUI not opening in browser | `allowaccess` missing `https` (or `http`) | Add protocol under `config system interface → edit port1 → set allowaccess ...` |
| Workstation cannot ping firewall | Interface IP not applied or wrong subnet | Re-check `set ip` and gateway in the same /24 as workstation |
| HTTPS page refused but ping works | Only `ping` allowed in `allowaccess` | Add `https` and `http` to `allowaccess` |

## Screenshots

All screenshots live in the [`screenshots/`](./screenshots) folder:

| File | Description |
| --- | --- |
| `01-fortigate-cli-login-license.png` | CLI login, trial license request & `no update error` status |
| `02-fortigate-cli-day1-config.png` | Full Day-1 CLI session: DNS, static route, port1 IP, `allowaccess`, Internet pings |
| `03-cmd-ping-firewall.png` | Workstation ping to 192.168.194.158 — 0% loss |
| `04-gui-dashboard.png` | FortiGate GUI dashboard — system info, license status, CPU/Memory |
| `05-github-repo-structure.png` | Lab repository directory structure on GitHub |

## References

- FortiGate Firewall Training Series — Day 5: *EVE-NG Lab Setup* (YouTube transcript) — image download from `support.fortinet.com`, WinSCP import, image naming convention
- FortiGate Firewall Training Series — Day 1: *Hands-On First Configuration* (this lab)
- Fortinet Documentation: https://docs.fortinet.com
- EVE-NG Documentation: https://www.eve-ng.net
- Companion labs: Palo Alto Firewall series (Lab 03 — TAP Interface & Zone Configuration) — same documentation format

---

*Part of the Network Security Engineer lab series — FortiGate track.*

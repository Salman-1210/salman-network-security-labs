# Network Security Labs — Salman

Hands-on network security labs built in **EVE-NG**, covering **Palo Alto**, **Cisco ISE**, and **Fortinet FortiGate**.
Each lab includes topology diagrams, device configurations, verification steps, and troubleshooting notes from real lab sessions.

> 🎯 Currently studying toward: **PCNSA** (Palo Alto) · **NSE 4** (Fortinet) · **Cisco ISE Specialist**

---

## 📂 Repository Structure

```javascript
├── palo-alto-labs/     # Palo Alto firewall labs (PAN-OS)
├── fortigate-labs/     # Fortinet FortiGate labs
├── cisco-ise-labs/     # Cisco ISE labs (planned in enar future)
├── *.md                # Shared notes (fundamentals, firewall theory)
└── README.md           # You are here
```

Each vendor folder contains numbered labs:

```javascript
palo-alto-labs/
├── lab-01-initial-configuration/
│   ├── README.md        ← full write-up: objective, topology, steps, fixes
│   ├── screenshots/     ← configs, logs, verification output
│   └── eve-ng-topology.unl
├── lab-02-security-zones/
└── ...
```

**Lab numbers follow order of complexity — start from `lab-01` and work forward.**

---

## 🧪 What Each Lab Contains

| Item | Description |
| --- | --- |
| `README.md` | Lab objective, topology, step-by-step config, verification, and *what broke & how I fixed it* |
| `topology.png` | Network diagram |
| `screenshots/` | Key configs, dashboards, and log output proving the lab works |
| `eve-ng-topology.unl` | Importable EVE-NG topology file |
| `configs/` | Sanitised device configurations (CLI) |

> Every lab README follows the same format: **Topology → Theory → Step-by-Step Config → Verification → Troubleshooting → Lessons Learned.**

---

## 🗺️ Topic Coverage

**Palo Alto (PAN-OS):** initial setup, interfaces, security zones, TAP & Virtual Wire, security policies, NAT, GlobalProtect VPN, SSL decryption

**FortiGate:** initial setup & deployment modes, interfaces & zones, firewall policies with NAT, DNS/routing, IPSec VPN, security profiles (web filter, app control)

**Cisco ISE:** initial setup, RADIUS, 802.1X (planned in enar future)

**Multi-vendor:** identity integration, full office network simulations

*(New topics are added as labs are completed — check the vendor folders for the latest.)*

---

## 🖥️ Lab Environment

| Component | Details |
| --- | --- |
| Platform | EVE-NG Community Edition |
| Palo Alto | PAN-OS VM-Series |
| Fortinet | FortiGate VM (v7.4.x) |
| Cisco | ISE / CML |
| Clients | Windows VMs, vIOS switches/routers |
| Diagrams | draw.io |

---

## 🚀 How to Use These Labs

1. Pick a vendor folder, start from the lowest-numbered lab
2. Read the lab README first — it explains the theory before the config
3. Rebuild the topology in EVE-NG (or import the `.unl` file)
4. Follow the steps, then compare your output against the screenshots
5. Stuck? Check the **Troubleshooting** section — it covers the mistakes I actually hit

---

## 📜 Notes

- All configurations are **sanitised** — no real IPs, credentials, or sensitive data
- Labs are written for learning purposes; validate everything before using in production
- Progress timeline: started **May 2026**, certification target **November 2026**

---

*Documented by Salman — built lab by lab, mistake by mistake.*

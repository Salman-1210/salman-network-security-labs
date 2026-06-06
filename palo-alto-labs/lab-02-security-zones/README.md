# Lab 02 — Security Zones & Interfaces

> **Series:** Palo Alto Firewall Training (EVE-NG Lab Environment)
> **Level:** Beginner → Intermediate
> **Platform:** PAN-OS | EVE-NG
> **Prerequisite:** [Lab 01 — Initial Configuration](../lab-01-initial-configuration/README.md)

---

## 🎯 Topics Covered

1. Administrative Access & Login (2:49 - 7:00)
2. Operational Mode & Basic Configuration (7:00 - 13:40)
3. Configuration Management (13:46 - 22:30)
4. Licensing & Software Updates (22:34 - 28:50)
5. URL Filtering & Categorization (29:42 - 31:26)
6. Account Administration (31:29 - 35:01)
7. Log Monitoring (35:06 - 36:35)

---

## 🔐 1. Administrative Access & Login (2:49 - 7:00)

Two methods to access the firewall:

| Method | When to Use |
|--------|-------------|
| **GUI (HTTPS)** | After management IP is assigned |
| **CLI / Console** | When no IP is pre-configured |

**Default Credentials:**

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin` |

> ⚠️ Change the default password immediately after first login.

---

## ⚙️ 2. Operational Mode & Basic Configuration (7:00 - 13:40)

| Mode | Prompt | Purpose |
|------|--------|---------|
| **Operational** | `>` | Show commands, ping, diagnostics |
| **Configuration** | `#` | Make and stage changes |

Assigning a static IP to the management interface:

```bash
configure

set deviceconfig system ip-address 192.168.1.1 netmask 255.255.255.0 default-gateway 192.168.1.254
set deviceconfig system dns-setting servers primary 8.8.8.8
set deviceconfig system hostname PA-LAB-01

commit
```

> 💡 Changes are NOT live until `commit` is run — this is different from Cisco IOS.

---

## 💾 3. Configuration Management (13:46 - 22:30)

### Candidate vs Running Config

```
[Changes Made] → Candidate Config → [commit] → Running Config
```

### Snapshots & Backups

**Device → Setup → Operations**

| Action | Purpose |
|--------|---------|
| Save named configuration snapshot | Save current state with a custom name |
| Load named configuration | Revert to a previous snapshot |
| Export device state | Full backup including certificates and licenses |
| Import configuration | Restore from an exported file |

> ✅ Best Practice: Take a snapshot before every major change.

---

## 📜 4. Licensing & Software Updates (22:34 - 28:50)

**Device → Licenses → Activate feature using authorization code**

Devices must be registered and licenses activated to enable advanced features.

**Device → Dynamic Updates**

| Signature Type | Purpose |
|---------------|---------|
| Antivirus | Blocks known malware and viruses |
| Threat Prevention | IPS/IDS signature updates |
| Applications & Threats | App identification and threat detection |

> ⚠️ The firewall must have internet access and DNS configured to download updates automatically.

---

## 🌐 5. URL Filtering & Categorization (29:42 - 31:26)

- PAN-OS categorizes websites into predefined categories (e.g., social media, malware, news)
- Security policies can allow or block traffic based on these categories
- Miscategorized URLs can be submitted to Palo Alto Networks for review and correction

---

## 👥 6. Account Administration (31:29 - 35:01)

**Device → Administrators → Add**

| Role | Access Level | Use Case |
|------|-------------|----------|
| superuser | Full access | Senior Network Engineer |
| devicereader | Read-only | Level 1 / NOC Engineer |
| Custom Role | Granular permissions | Specific team responsibilities |

Role-Based Access Control (RBAC) allows different access levels per engineer, restricting sensitive configurations from lower-level staff.

---

## 📊 7. Log Monitoring (35:06 - 36:35)

**Monitor Tab → Logs**

| Log Type | What It Shows |
|----------|--------------|
| Traffic | All flows allowed or denied through the firewall |
| Threat | IPS alerts and malware detections |
| System | Device events, admin logins, commits |

---

## ✅ Lab Completion Checklist

- [ ] Logged in via console with default credentials (admin/admin)
- [ ] Changed default admin password
- [ ] Assigned static IP to management interface
- [ ] Committed changes successfully
- [ ] Accessed GUI via HTTPS
- [ ] Created a named configuration snapshot
- [ ] Exported device state for backup
- [ ] Verified license status
- [ ] Checked Dynamic Updates section
- [ ] Reviewed URL filtering categories
- [ ] Created a read-only admin account
- [ ] Viewed traffic and threat logs in Monitor tab

---

## 🔗 Navigation

⬅️ **[Lab 01 — Initial Configuration](../lab-01-initial-configuration/README.md)**

➡️ **[Lab 02 — Security Policies](../lab-02-security-policies/README.md)**

---

*Lab documented by Salman | Cybersecurity Portfolio | 2026*

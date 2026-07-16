# Network Security Fundamentals — Common Terms & Malware Types

> **Series:** Cybersecurity Theory Notes
> **Level:** Beginner
> **Relevant for:** PCNSA · NSE 4 · Cisco ISE Specialist

---

## 🎯 Part 1 — Core Network Security Terms

The 7 key terms every security professional must know:
**Asset · Vulnerability · Exploit · Threat · Attack · Risk · Countermeasure**

### 1. Asset
Anything the organization is invested in and that is valuable to it.
**Examples:** buildings, vehicles, equipment, employees, computers, data, intellectual property.

> Protecting assets is the **prime function** of security — physical or network.

### 2. Vulnerability
A **weakness** in a system or its design. Every system is human-created, so errors and mistakes always exist — in applications, network protocols, and operating systems.

### 3. Exploit
The **method, way, or tool** an attacker uses *on a vulnerability* to cause damage.
**Examples:** software that triggers a buffer overflow, or social engineering to steal a password.

### 4. Threat
Anything that is a **danger to an asset**. Threats can be **accidentally triggered** or **intentionally exploited**.

### 5. Attack
The **action** taken by an attacker to harm an asset.

### 6. Risk
The **potential for loss, compromise, damage, or destruction** of an asset. Risk arises when a threat exploits a vulnerability.

```
Risk = Asset + Threat + Vulnerability
```

### 7. Countermeasure
An **action taken by the organization** (typically security professionals) to **mitigate a threat**.

### 🔗 How They Connect (one line to remember)

> A **Threat** uses an **Exploit** on a **Vulnerability** to launch an **Attack** against an **Asset** — creating **Risk**, which we reduce with a **Countermeasure**.

---

## 🦠 Part 2 — Malware

**Malware = Malicious Software** — any file, program, or code harmful to a computer user.

- Typically **delivered over a network** — infects, explores, and steals
- Can perform **virtually any behavior** the attacker wants
- It is an **inclusive term** covering: Viruses, Worms, Trojans, Rootkits, Spyware, Adware, Scareware, Botnets, Logic Bombs, Keyloggers

### 🛠️ Tools to Identify Malware
| Tool | Purpose |
|------|---------|
| Packet Captures | Analyze suspicious traffic |
| Snort | IDS/IPS rule-based detection |
| NetFlow | Network traffic pattern analysis |
| IPS | Inline prevention |
| AMP (Advanced Malware Protection) | File reputation & sandboxing |
| Cisco FirePOWER | NGFW/NGIPS inspection |

---

## 🐛 Part 3 — Types of Malware

| Malware | What It Does | Key Trait |
|---------|--------------|-----------|
| **Virus** | Malicious code attached to executable files (often regular applications) | ⚠️ **Requires end-user activation** to damage the system |
| **Worm** | Replicates itself and spreads to infect other systems; targets OS files, consumes bandwidth | 🔁 **Self-replicating** — no user action needed |
| **Trojan** | Appears as a regular application or media file but carries a malicious payload | 🚪 Payload opens a **backdoor** for unauthorized access |
| **Ransomware** | Encrypts the hard drive and all files, then demands payment for the decryption key | 💰 Examples: Reveton, CryptoLocker, CryptoWall, **WannaCry (2017)** |
| **Spyware** | Monitors user activity on the PC to collect private information | 👁️ Often arrives bundled with **free software downloads** |
| **Adware** | Advertising-supported malware — plays, displays, or downloads ads automatically | 📢 Generates **ad revenue** for attackers |
| **Rootkit** | Collection of software that works silently in the background and permits other malware into the system | 🫥 User notices **nothing suspicious** |
| **Keylogger** | Records everything typed on the keyboard and sends it to the attacker | ⌨️ Steals **passwords & credit card details** |
| **Scareware** | Fake pop-ups resembling Windows/AV messages claiming "infected files," tricking users into buying useless software | 😱 The "fix" itself **contains malware** |
| **Logic Bomb** | Dormant malware triggered by an **event** — launching an app, or reaching a specific date/time | ⏰ Often embedded inside **fake apps or Trojans** |

---

## ⚡ Quick Revision — One-Liners

- **Virus vs Worm:** Virus needs a user to run it; a worm spreads by itself
- **Trojan vs Virus:** Trojan pretends to be legit software; virus attaches itself to real files
- **Spyware vs Keylogger:** Spyware monitors everything; a keylogger specifically records keystrokes
- **Scareware vs Ransomware:** Scareware *fakes* a problem to sell a fix; ransomware *creates* a real problem (encryption) and demands payment
- **Logic Bomb:** The only malware that waits for a **trigger event** (date/time/action)

---

## 🧠 What I Learned

- Security exists to protect **assets**; everything else (threats, exploits, attacks) is the chain that puts assets at **risk**
- **Risk = Asset + Threat + Vulnerability** — remove any one element and risk drops
- **Countermeasures** are the defender's side of the equation — this is exactly where firewalls like Palo Alto and FortiGate fit in
- Malware is an **umbrella term** — each type differs in *how it enters*, *how it spreads*, and *what it wants*

---

*Notes compiled from training material (by Ahmad Ali). Written in my own structure for revision & portfolio documentation.*

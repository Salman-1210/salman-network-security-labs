# Lab 06 — FortiGate Application Control & Troubleshooting

A comprehensive, real-world laboratory guide on understanding, configuring, and troubleshooting **Application Control Profiles** on FortiGate Next-Generation Firewalls (NGFW) in an EVE-NG environment.

## 📋 Table of Contents

* [Overview & Objectives](#-overview--objectives)

* [Why Application Control? (Theory)](#-why-application-control-theory)

* [Network Topology & Addressing](#-network-topology--addressing)

* [Step-by-Step Lab Execution](#-step-by-step-lab-execution)

  * [Step 1: Network & Routing Fixes](#step-1-network--routing-fixes)

  * [Step 2: Tiny Core Linux IP Configuration](#step-2-tiny-core-linux-ip-configuration)

  * [Step 3: Creating Application Control Profile](#step-3-creating-application-control-profile)

  * [Step 4: Applying Security Profile to IPv4 Policy](#step-4-applying-security-profile-to-ipv4-policy)

  * [Step 5: Verification & Log Analysis](#step-5-verification--log-analysis)

* [Troubleshooting Log & Issues Solved](#-troubleshooting-log--issues-solved)

* [Lab Screenshots & Visual Evidence](#-lab-screenshots--visual-evidence)

* [GitHub Repository Structure Guide](#-github-repository-structure-guide)

* [Conclusion](#-conclusion)

## 🎯 Overview & Objectives

The goal of this lab is to demonstrate how modern application layer (Layer 7) filtering operates on a FortiGate Firewall. Traditional firewalls only restrict traffic by Port and Protocol (Layer 3/4), whereas FortiGate Application Control identifies specific network patterns/signatures to restrict granular sub-functions (e.g., allowing base Facebook access while blocking Facebook Chat/Games).

### Learning Objectives:

1. Understand the necessity of Layer 7 Application Control in modern enterprise networks.

2. Differentiate between Category-Based blocking and Specific Application Overrides.

3. Configure Application Control profiles via FortiGate CLI and Web GUI.

4. Integrate Application Control sensors into active Firewall Policies with NAT enabled.

5. Troubleshoot Linux endpoint IP binding, routing table mismatches, and interface mappings.

## 🧠 Why Application Control? (Theory)

In modern corporate environments, user traffic heavily relies on web applications operating over standard ports like HTTP (`80`) and HTTPS (`443`).

* **The Problem:** Blocking port 443 completely disables all secure web browsing. However, allowing port 443 permits access to productivity risks (YouTube, Social Media) and potential threat vectors (Cloud Drives, Peer-to-Peer sharing).

* **The Solution:** FortiGate Application Control inspects the packet flow pattern (Application Signatures) between the client machine and destination servers. It allows network administrators to:

  * Block entire categories (e.g., Social Media, Video/Audio streaming).

  * Apply granular sub-application blocks (e.g., Block YouTube videos while allowing Google Search).

  * Enforce bandwidth and compliance controls.

## 🌐 Network Topology & Addressing

| Device | Interface | IP Address / Subnet | Role / Description | 
| ----- | ----- | ----- | ----- | 
| **FortiGate NGFW** | `port1` | `192.168.188.132/24` | WAN Interface (Connected to EVE-NG Net Cloud) | 
| **FortiGate NGFW** | `port2` | `10.1.1.1/24` | LAN / Internal Gateway | 
| **FortiGate NGFW** | `port3` | `172.29.129.50/24` | Alternate WAN / Secondary Link | 
| **Tiny Core Linux** | `eth0` | `10.1.1.2/24` | Internal Test PC (`Gateway: 10.1.1.1`) | 
| **EVE Net Cloud** | Gateway | `192.168.188.2` | Upstream Internet Gateway | 

## 🛠️ Step-by-Step Lab Execution

### Step 1: Network & Routing Fixes

During initial testing, traffic failed because FortiGate's static route was pointing to an unreachable gateway via `port3`.

1. **Configure WAN Interface (DHCP/Static):**

   ```
   config system interface
       edit "port1"
           set mode dhcp
       next
   end
   
   ```

2. **Configure Correct Static Default Route:**

   ```
   config router static
       purge
       edit 1
           set gateway 192.168.188.2
           set device "port1"
       next
   end
   
   ```

3. **Verify FortiGate Internet Reachability:**

   ```
   execute ping 8.8.8.8
   
   ```

### Step 2: Tiny Core Linux IP Configuration

Due to GUI execution delays on Tiny Core Linux, configure network properties directly using Linux terminal commands:

```
# Assign static IP to LAN card
sudo ip addr add 10.1.1.2/24 dev eth0
sudo ip link set eth0 up

# Set Default Gateway to FortiGate LAN Port
sudo ip route add default via 10.1.1.1 dev eth0

# Configure Google Public DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

```

Verify reachability from Linux node:

```
ping -c 3 8.8.8.8

```

### Step 3: Creating Application Control Profile

1. Log in to **FortiGate GUI**.

2. Navigate to **Security Profiles** > **Application Control**.

3. Click `+` (Create New) and name the profile **`test`**.

4. **Category Blocking:** Set categories like **Social Media** to `Block`.

5. **Application Overrides:** Add specific application signatures (e.g., `YouTube`) and set the Action to **Block**.

6. Save the Application Profile sensor.

### Step 4: Applying Security Profile to IPv4 Policy

Map the created sensor to the outgoing internet firewall policy:

```
config firewall policy
    edit 1
        set name "Test"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set utm-status enable
        set application-list "test"
        set ssl-ssh-profile "certificate-inspection"
        set logtraffic all
        set nat enable
    next
end

```

### Step 5: Verification & Log Analysis

1. Open Mozilla Firefox inside **Tiny Core Linux**.

2. Navigate to `https://www.google.com` $\rightarrow$ **Status: Allowed / Working**.

3. Navigate to `https://www.twitter.com` or `https://www.youtube.com` $\rightarrow$ **Status: Blocked / Connection Reset**.

4. In FortiGate GUI, check **Log & Report** > **Application Control** to verify real-time blocked session logs.

## 🔍 Troubleshooting Log & Issues Solved

| \# | Error / Problem | Root Cause | Resolution | 
| ----- | ----- | ----- | ----- | 
| **1** | Linux Node 100% Packet Loss (`8.8.8.8`) | Default route pointing to incorrect WAN port (`port3` instead of `port1`). | Updated static route `0.0.0.0/0` via `192.168.188.2` on `port1`. | 
| **2** | GUI "Apply" button unresponsive in Tiny Core | Background GUI script hanging on Tiny Core desktop. | Executed `ip route` and `resolv.conf` manual terminal commands. | 
| **3** | `Attribute 'vdom' MUST be set` on FortiGate CLI | Typo in interface command (`edit por1` created a custom interface). | Executed exact `edit "port1"` syntax. | 
| **4** | `command parse error before 'dstintf'` | Missing double quotes around interface string in CLI. | Used `set dstintf "port1"` with explicit syntax. | 
| **5** | Website DNS resolution failed (`Server Not Found`) | Destination interface on Policy 1 was mismatched. | Re-aligned firewall policy source (`port2`) to destination (`port1`). | 

## 📸 Lab Screenshots & Visual Evidence

Here are the step-by-step diagnostic and execution screenshots captured during this laboratory exercise:

### 1. Tiny Core Linux Routing & IP Setup

### 2. Initial Ping Failure Diagnostic

### 3. FortiGate Policy Verification

### 4. FortiGate Static Route Corrections

### 5. Successful FortiGate Upstream Ping

### 6. Topology & GUI Troubleshooting

### 7. Terminal Fix & Final Ping Test

## 📁 GitHub Repository Structure Guide

Aapki repository ko clean aur professional rakhne ke liye naya folder aise banayein:

### Folder Hierarchy:

```
salman-network-security-labs/
│
├── Fortigate-labs/
│   ├── Lab 05 — FortiGate Interface Types (/ vs Redundant)
│   │   └── README.md
│   │
│   └── Lab 06 — FortiGate Application Control & Troubleshooting/  <-- (NEW FOLDER)
│       ├── README.md                                               <-- (THIS FILE)
│       ├── image_e2ede7.png
│       ├── image_e28584.png
│       ├── image_e284ed.png
│       ├── image_e284e7.png
│       ├── image_e281c2.png
│       ├── image_e2814a.png
│       ├── image_e27dc4.png
│       ├── image_e27a02.png
│       ├── image_e27627.png
│       ├── image_d6987e.png
│       ├── image_d694c3.png
│       ├── image_d62fe2.png
│       ├── image_d62c22.png
│       ├── image_d6287e.png
│       ├── image_d6285e.png
│       └── image_d62483.png

```

### Steps to Add Next Lab to GitHub:

1. Open local cloned repository or visit [Salman Network Security Labs](https://github.com/Salman-1210/salman-network-security-labs/tree/main/Fortigate-labs).

2. Go into `Fortigate-labs/` directory.

3. Create a new folder named exactly:
   `Lab 06 — FortiGate Application Control & Troubleshooting`

4. Upload all 16 screenshot `.png` files inside this new folder (make sure filenames match exact image names listed above).

5. Paste this generated `README.md` file inside the same folder.

6. Commit changes with message: `"Add Lab 06 - Application Control and Troubleshooting Documentation"`.

## 📌 Conclusion

Application Control on FortiGate is a fundamental Next-Generation Firewall feature that protects enterprise environments from unauthorized web application usage and modern L7 vulnerabilities. Through this lab, we successfully built an Application Profile, mapped it to NAT policies, resolved real-world routing and endpoint bugs, and verified packet flow inspection end-to-end.
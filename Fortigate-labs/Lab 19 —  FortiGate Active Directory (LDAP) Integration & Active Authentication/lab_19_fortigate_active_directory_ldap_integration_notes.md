# 📝 Lab 19: FortiGate Active Directory (LDAP) Integration & Active Authentication

## 1. Core Concepts

### 🔹 What is Active Directory (AD)?
* **Simple Definition:** Active Directory is a centralized digital database (similar to a phonebook) that stores and manages records of all users and devices within an organization's network.
* **What Does It Store?**
  * **Users:** Usernames, passwords, email IDs, phone numbers, departments, etc.
  * **Devices:** Domain-joined computers, laptops, servers, and network printers.

### 🔹 Why Integrate FortiGate with Active Directory?
* **The Problem:** Active Directory only manages user identities and local permissions; it does not sit inline with network traffic. Therefore, AD alone cannot filter or block a user's access to the internet or other restricted network zones.
* **The Solution:** Integrating AD with the FortiGate firewall allows FortiGate to query and fetch user and group information from the AD database. This enables administrators to create **user- and group-based firewall policies** instead of relying solely on static IP addresses (e.g., *Allow the IT group to access the internet, block the Guest group*).

---

## 2. Authentication Modes in FortiGate

When traffic passes through the FortiGate firewall, there are two primary methods to identify and authenticate AD users:

1. **Active Authentication (Focus of this Lab):**
   * The user is explicitly required to enter their AD username and password.
   * When a user opens a web browser to access the internet, FortiGate intercepts the request and presents a **Captive Portal (Web Login Page)**. Traffic is only permitted after valid credentials are submitted.
2. **Passive Authentication (FSSO - Fortinet Single Sign-On):**
   * The user is never prompted for credentials. The firewall silently identifies the user in the background by monitoring Domain Controller login events or using an agent.

---

## 3. General Configuration Summary

### Step 1: Active Directory (Windows Server) Preparation
1. **Create a Group:** Open **Active Directory Users and Computers (ADUC)** and create a Security Group (e.g., `IT` or `Sales`).
2. **Create a User:** Create a test user account and add it as a member of that security group.
3. **Bind Account:** FortiGate requires an AD account to query the directory. *(Best Practice: In production environments, create a dedicated read-only service account rather than using the built-in Administrator account).*

### Step 2: Configure LDAP Server on FortiGate
* **Navigation:** `User & Authentication > LDAP Servers > Create New`
* **Key Parameters:**
  * **Server IP/Name:** IP address of the Domain Controller (Windows AD Server).
  * **Server Port:** `389` (Standard LDAP) or `636` (LDAPS / Secure).
  * **Common Name Identifier:** `cn` or `sAMAccountName` (The attribute matched against the username entered at login).
  * **Distinguished Name (DN):** The base domain path where FortiGate searches for objects (e.g., `dc=domain,dc=com`).
  * **Bind Type:** `Regular` (Allows FortiGate to authenticate itself to AD before searching for user records).
  * **Username & Password:** Credentials for the AD bind account.
* **Verification:** Use **Test Connectivity** and **Test User Credentials** to verify both network reachability and credential validation.

### Step 3: Create a Firewall User Group
* **Navigation:** `User & Authentication > User Groups > Create New`
* **Key Rule:** FortiGate firewall policies cannot reference an LDAP server directly; the AD group must first be mapped inside a Firewall User Group.
* **Settings:** Set the Type to **Firewall**, click **Add** under **Remote Groups**, select your configured LDAP server, and choose the desired AD security group.

### Step 4: Configure the Firewall Policy
* **Navigation:** `Policy & Objects > Firewall Policy > Create New`
* **Key Rule:** In the **Source** field, you must include **both** the **Source IP/Subnet** and the **User Group**. A policy containing only a User Group without a source address object will not function.
* **Settings:** Set Action to `ACCEPT`, enable **NAT** (for outbound internet traffic), and enable session logging for visibility.

### Step 5: Client Workstation Setup
* Configure the client PC's **Preferred DNS Server** to point to the **AD Server's IP address** so it can resolve the domain name.
* Join the workstation to the Active Directory domain, restart the machine, and log in using a domain user account.

---

## 4. End-to-End Traffic & Authentication Flow

When a domain user opens a browser to access the internet, the following 5-step process occurs behind the scenes:

```text
[Client PC] ---> (1) Web Request ---> [FortiGate Firewall]
     ^                                        |
     |--- (2) Captive Portal Login Prompt <---|
     |                                        |
     |--- (3) Enters AD User & Password ----->|---> (4) LDAP Query (Port 389) ---> [Windows AD Server]
                                              |<-- (5) Auth OK + Group Info <----|
                                              |
                                     [Policy & Group Match]
                                              |
                                              v
                                         [Internet]
```

1. **Traffic Initiation:** The user opens a web browser, sending traffic toward their default gateway (the FortiGate firewall).
2. **Traffic Interception:** FortiGate checks its firewall policies and matches a rule requiring **User Group** authentication. Since the client's IP is not yet associated with an authenticated user, FortiGate holds the traffic and redirects the browser to the **Captive Portal login page**.
3. **LDAP Verification:** The user submits their AD username and password. FortiGate forwards these credentials via **LDAP (TCP Port 389)** to the Windows AD Server.
4. **Group Verification:** The AD Server validates the password and responds to FortiGate confirming authentication success along with the user's group memberships.
5. **Session Creation & Access:** FortiGate verifies that the user's AD group matches the allowed User Group in the firewall policy, registers the active session, applies NAT, and permits traffic to the internet.

---

## 5. Verification, Troubleshooting & Key Takeaways

### 📊 Monitoring Active Users (GUI)
* Navigate to **Dashboard > Users & Devices** (Firewall User Monitor) to view actively authenticated users, their IP addresses, session duration, and data usage.
* Administrators can also manually **De-authenticate** users from this monitor, immediately revoking their access and forcing them to log in again.

### 🛠️ Essential CLI Troubleshooting Commands
```bash
diagnose debug application fnbamd -1   # Real-time debugging for LDAP/AD authentication processes
diagnose debug enable                  # Enable debug output on the console
diagnose firewall auth list            # Display all currently authenticated firewall users
```

### ⚠️ Common Pitfalls to Remember
1. **Incorrect DNS on Client:** If the client PC does not use the AD Server as its primary DNS, domain joining and name resolution will fail.
2. **Missing Source Address in Policy:** Always pair the **User Group** with a **Source IP/Subnet** object in the firewall policy.
3. **Improper DN Syntax:** Ensure the Distinguished Name follows proper LDAP syntax (e.g., `dc=company,dc=local`).
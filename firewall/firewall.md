# FIREWALL

Perfeito. Aqui está a versão **completa, estruturada e sem emojis**, escrita em inglês formal, ideal para o teu GitBook:

***

## FIREWALL — Complete Guide

### What Is a Firewall

A **firewall** is a security mechanism, implemented in software, hardware, or both, that monitors and controls network traffic based on predefined security rules. Its main purpose is to act as a **barrier between trusted and untrusted networks**, such as between a company’s internal LAN and the public Internet.

In simple terms, a firewall **determines which network packets can enter or leave** a system or network. It analyzes packets according to established rules that define which types of communication are allowed or denied.

Firewalls can operate at various layers of the **OSI model**, from the network to the application layer. They are a fundamental part of cybersecurity, preventing unauthorized access, blocking malicious traffic, and logging suspicious activities for further analysis.

***

### How Firewalls Work in Networks

When data travels through a network, it is divided into **packets**. Each packet contains source and destination addresses, protocols, and other metadata. A firewall inspects these packets and applies filtering rules to determine whether they should be allowed or blocked.

* **Inbound filtering** manages traffic entering the network.
* **Outbound filtering** manages traffic leaving the network.

A firewall can block or allow traffic based on attributes such as the **source IP address**, **destination IP address**, **port number**, or **protocol** (for example, TCP, UDP, ICMP).\
Advanced firewalls can perform **Deep Packet Inspection (DPI)**, analyzing the content of the packets and inspecting higher-level protocols such as HTTP, HTTPS, and DNS.

***

### Relationship Between Firewall and NAT

**NAT (Network Address Translation)** and firewalls are closely related because they often operate on the same network edge device, such as a router or security gateway.

* **NAT** hides private IP addresses by mapping them to one or more public IP addresses. This allows multiple internal devices to share a single Internet connection.
* **The firewall** enforces security rules, deciding which of these translated connections are allowed to pass through.

In practice, they complement each other:

* NAT provides privacy by masking internal network details.
* The firewall provides control by deciding what traffic is permitted.

Example:\
When an internal host with IP `192.168.1.10` sends a request to the Internet, NAT replaces that private IP with a public one, such as `203.0.113.5`. The firewall then decides whether that connection should be allowed based on its rules.

In short: **NAT hides, Firewall decides.**

***

### Types of Firewalls

#### 1. Packet-Filtering Firewall

Operates mainly at the network layer (Layer 3). It inspects packet headers and enforces simple “allow or deny” decisions based on IP addresses, ports, and protocols.\
**Examples:** Cisco ACLs, Linux `iptables`, BSD `pf`.

#### 2. Stateful Inspection Firewall

Tracks the state of active connections and can determine whether packets belong to an existing, legitimate session. More secure than simple packet filtering.\
**Examples:** pfSense, Check Point, Windows Defender Firewall.

#### 3. Proxy Firewall (Application-Level Gateway)

Acts as an intermediary between clients and servers, inspecting data at the application layer (Layer 7). It can understand and filter traffic for specific applications such as HTTP or FTP.\
**Examples:** Squid Proxy, Blue Coat ProxySG.

#### 4. Next-Generation Firewall (NGFW)

Combines traditional firewall capabilities with additional features such as deep packet inspection, intrusion prevention, malware detection, and application awareness.\
**Examples:** Palo Alto Networks, Fortinet FortiGate, Cisco Firepower.

#### 5. Cloud-Based Firewall (Firewall-as-a-Service, FWaaS)

Deployed in cloud environments to protect distributed infrastructures and hybrid networks.\
**Examples:** Cloudflare Magic Firewall, Zscaler, Azure Firewall.

***

### Real-World Cases Where Firewalls Made an Impact

1. **WannaCry Ransomware (2017):**\
   Many organizations prevented internal spread of the ransomware by applying firewall rules that blocked SMB traffic (port 445).
2. **SolarWinds Supply Chain Attack (2020):**\
   Proper internal segmentation with firewalls could have reduced lateral movement between compromised systems, mitigating damage.
3. **Small Business Network:**\
   A small retail company used pfSense to block unauthorized remote desktop (RDP) attempts, preventing brute-force access to point-of-sale systems.
4. **University Campus Network:**\
   Internal firewalls separated student labs from administrative networks, preventing students from accessing confidential staff or financial data.

***

### How to Configure a Firewall in a Lab Environment

#### Example 1: Basic Linux Firewall with iptables

**Goal:** Allow only SSH (port 22) and HTTP (port 80) connections; block all others.

```bash
# Check existing rules
sudo iptables -L -v -n

# Set default policies
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow loopback and established connections
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH and HTTP
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
```

**Test:**\
Attempt to connect via SSH or a web browser; other ports should be inaccessible.

***

#### Example 2: Simplified Firewall Using UFW

**Goal:** The same rule set, managed with a simpler interface.

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw enable
sudo ufw status verbose
```

***

### Example 3: Firewall and BIND9 (DNS Server)

**Scenario:**\
You have a DNS server running BIND9 on port 53. You want to allow DNS queries only from your internal LAN (`192.168.1.0/24`).

#### Step 1: Install BIND9

```bash
sudo apt install bind9 bind9-utils
```

#### Step 2: Configure `/etc/bind/named.conf.options`

```bash
options {
    directory "/var/cache/bind";
    allow-query { 192.168.1.0/24; };
    recursion yes;
};
```

#### Step 3: Apply Firewall Rules

```bash
# Allow DNS queries only from LAN
sudo iptables -A INPUT -p udp --dport 53 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 53 -s 192.168.1.0/24 -j ACCEPT

# Block all other DNS traffic
sudo iptables -A INPUT -p udp --dport 53 -j DROP
sudo iptables -A INPUT -p tcp --dport 53 -j DROP
```

#### Step 4: Restart Services

```bash
sudo systemctl restart bind9
sudo systemctl restart netfilter-persistent
```

#### Step 5: Test

From a host within the LAN:

```bash
dig @192.168.1.10 example.com
```

This should return a valid DNS response.\
From outside the LAN, the request should time out or fail, as the firewall blocks it.

***

### In conclusion

A **firewall** is one of the most essential components in modern network security. It protects networks by filtering traffic, enforcing security policies, and cooperating with NAT to ensure safe communication between internal and external environments. Whether implemented in a simple home router, a corporate gateway, or a cloud-based infrastructure, firewalls remain the first and most critical line of defense against unauthorized access and cyberattacks.

***


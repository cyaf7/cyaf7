# Phishing & Email Defense Toolkit

### **1. Phishing Investigation Tools**

**1.1 PhishTool**

What it is: A dedicated email forensics tool that helps investigators analyze suspicious emails.

How it works: Upload the raw email (.eml/.msg), and it automatically parses headers, checks SPF/DKIM/DMARC, extracts links/attachments, and correlates with threat feeds.

Example: A phishing email pretending to be Microsoft 365 is uploaded → PhishTool highlights that the Reply-To address is a free Gmail account, SPF fails, and the embedded link points to login-office365-security\[.]com.

Usefulness: Saves time by automating header analysis and giving quick risk scores, helping SOC analysts handle many alerts efficiently.

### 1.2 Visualization Tools

What they are: Tools that map relationships between artifacts (IP → domain → email → file hashes).

Example: Maltego or MISP visualizations showing how a phishing domain (phish-paypal.com) connects to an IP, which also hosts malware downloads.

Usefulness: Helps analysts see the “bigger picture” — linking phishing campaigns to infrastructure and understanding attacker methods.

### 2. URL Analysis & Reputation

**2.1 URL2PNG**

What it is: A tool that generates screenshots of a webpage.

Example: A suspicious URL is submitted → tool captures the rendered phishing login page for investigation.

Usefulness: Safe way to preview phishing sites without visiting them directly in a browser.

**2.2 URLScan**

What it is: Online sandbox for URLs.

How it works: Submits the link, fetches the page in an isolated browser, and records network requests, redirects, and scripts.

Example: Analyst uploads bit.ly/secure-login → URLScan expands it, showing redirects to malicious-login.xyz and calls to suspicious IPs.

Usefulness: Detects phishing websites, malicious redirects, and hidden scripts used in phishing campaigns.

### 2.3 URL Reputation Tools

Used to check if a link/domain is known malicious.

VirusTotal: Aggregates URL/domain/file reputation across many AV vendors. Example: login-paypa1.com flagged as phishing by 20 vendors.

**URLScan**: Sandbox URL analysis (network, scripts, redirects).

**Threat Feeds**: Commercial or open-source lists (e.g., Abuse.ch, Spamhaus).

**PhishTank**: Community-driven database of phishing URLs.

**URLHaus**: Specialized in malware and phishing URLs.

Usefulness: Prevents users from accessing dangerous links and helps SOCs quickly block malicious domains.

### 3. File Reputation Tools

**VirusTotal:** Upload a file, and it’s scanned by dozens of antivirus engines. Example: suspicious invoice.pdf is flagged by 15 engines as Trojan.

**Cisco Talos File Reputation**: Database of known file hashes with reputation scores. Example: hash of invoice.exe is already known as “high risk.”

Usefulness: Quickly determine if a file is widely recognized as malicious or unknown.

### 4. Malware Sandboxing

**Hybrid Analysis**

What it is: A sandbox that executes suspicious files in a safe, isolated environment.

How it works: Submits file or URL → system runs it → captures behavior (registry changes, network connections, dropped files).

Example: Malicious Word doc downloads payload.exe and tries to connect to malware-c2\[.]net.

Usefulness: Provides behavior-based evidence when signature-based detection fails.

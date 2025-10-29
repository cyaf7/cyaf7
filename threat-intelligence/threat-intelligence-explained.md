# Threat Intelligence explained

### 2. Threat Intelligence Analysis

Threat intelligence connects **artifacts (IOCs)**, **actors**, and **activities** into a coherent picture.

**Example Process:**

1. Security logs show multiple failed logins from a Russian IP.
2. TI feeds reveal this IP belongs to a botnet used for credential stuffing.
3. Analysts conclude an ongoing brute-force attempt.
4. SOC applies **rate-limiting**, blocks the IP range, and alerts the Incident Response (IR) team.

**Key value:** TI helps analysts **contextualize data** and respond appropriately, instead of treating every alert equally.

***

### &#x20;3. Types of Intelligence

Threat intelligence borrows classification from military and national intelligence domains. Each type has cyber applications:

#### 3.1 COMINT – _Communications Intelligence_

* **Definition:** Intelligence from intercepted communications (voice, text, email).
* **Cyber Example:** Capturing attacker **Command & Control (C2)** traffic.

#### 3.2 SIGINT – _Signals Intelligence_

* **Definition:** Intelligence from electronic signals, not necessarily content.
* **Cyber Example:** Monitoring Wi-Fi or radio communications attackers use for persistence.

#### 3.3 ELINT – _Electronic Intelligence_

* **Definition:** Information from electronic devices or radar emissions.
* **Cyber Example:** Side-channel analysis, RF monitoring of compromised IoT devices.

#### 3.4 OSINT – _Open Source Intelligence_

* **Definition:** Intelligence from publicly available information.
* **Cyber Example:** Collecting leaked credentials from Twitter, forums, or the dark web.

#### 3.5 HUMINT – _Human Intelligence_

* **Definition:** Information gathered from people.
* **Cyber Example:** Insider reporting activity seen on underground forums.

#### 3.6 GEOINT – _Geospatial Intelligence_

* **Definition:** Location-based intelligence.
* **Cyber Example:** Mapping botnet infrastructure by geolocating attacker IPs.

***

### 4. Types of Threat Intelligence

Each “type” is tailored to its **audience and use case**:

#### 4.1 Strategic Threat Intelligence

* **Audience:** Executives & business leaders.
* **Content:** High-level trends, geopolitical risks, industry-specific targeting.
* **Example:** “Ransomware groups are increasingly targeting healthcare providers in Europe.”

#### 4.2 Operational Threat Intelligence

* **Audience:** Incident Response (IR) teams.
* **Content:** Attacker TTPs (tactics, techniques, procedures), active campaigns.
* **Example:** “Group X uses phishing emails with OneDrive links to deliver malware.”

#### 4.3 Tactical Threat Intelligence

* **Audience:** SOC & blue teams.
* **Content:** Technical indicators (IOCs) such as IP addresses, domains, hashes.
* **Example:** “SHA-256 hash of the latest ransomware sample used for detection rules.”

***

### &#x20;5. Why Threat Intelligence is Useful

Threat Intelligence transforms **raw data** into **actionable security knowledge**.

#### 5.1 Cyber Threat Context

* TI answers **who, why, and how** attackers operate.
* **Example:** Linking a phishing email to a known APT group.

#### 5.2 Incident Prioritization

* TI helps analysts **triage alerts** based on relevance.
* **Example:** A C2 IP linked to ransomware gets priority over generic scans.

#### 5.3 Investigation Enrichment

* TI adds context to incidents.
* **Example:** Matching a suspicious domain to PhishTank confirms it belongs to a phishing campaign.

#### 5.4 Information Sharing

* TI is stronger when shared.
* **Example:** Organizations share IOCs through ISACs (Information Sharing and Analysis Centers).

***

### &#x20;6. The Future of Threat Intelligence

Threat intelligence is evolving with automation, big data, and AI.

#### 6.1 CVEs and CVSS Scores

* **CVE:** Unique identifiers for vulnerabilities.
* **CVSS:** Standardized severity scoring (0–10).
* **Example:** _CVE-2025-12345_ with CVSS 9.8 (critical) → prioritized for patching.

#### 6.2 Vulnerability Context

* Raw scores aren’t enough.
* TI adds **real-world exploitability** info: is the vulnerability actively being exploited?
* **Example:** Prioritizing a CVE with a 7.5 score if it’s under active attack, instead of patching a theoretical 9.8 with no exploits.

#### 6.3 Predictive Prioritization

* AI/ML models forecast which vulnerabilities attackers will weaponize next.
* **Example:** Prioritizing VPN appliance patches after TI shows widespread scanning by ransomware groups.

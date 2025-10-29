# Threat Actors, APTs, and Operational Threat Intelligence

Understanding **who the attackers are**, how they operate, and what their campaigns look like is essential for building strong defenses. Threat actors range from nation-states to financially motivated groups, and their tactics are studied through frameworks like **MITRE ATT\&CK** and enriched by **Operational Threat Intelligence**.

### 1. Actor Naming Conventions

Different vendors use **unique naming systems** for threat actors. This can cause confusion, but it also allows analysts to correlate reports across vendors.

* **CrowdStrike:** Uses animal names tied to regions.
  * 🐻 Bear = Russia (e.g., _Fancy Bear / APT28_).
  * 🐼 Panda = China (e.g., _OceanLotus / APT32_).
  * 🐱 Kitten = Iran.
  * 🐎 Chollima = North Korea.
* **Mandiant / FireEye:** Uses sequential **APT numbers**.
  * APT28 (Russia), APT29 (Russia), APT32 (Vietnam), APT33 (Iran).
* **Microsoft:** Uses weather and element-based names.
  * _Nobelium_ (Russia), _Strontium_ (Russia), _Phosphorus_ (Iran), _Hafnium_ (China).

**Why it matters:** Analysts need to understand these conventions to **map reports across vendors** and avoid misinterpreting actor activity.

***

### 🌍 2. Global Threat Landscape

#### Nation-State Adversaries

Backed by governments, they aim for **espionage, disruption, or geopolitical advantage**.

* **APT29 (Cozy Bear):** Russian group behind the SolarWinds supply-chain attack.
* **APT41:** Chinese group mixing espionage with cybercrime.
* **Lazarus Group:** North Korean actor known for _WannaCry_ and financial heists.

#### Non-Nation-State Adversaries

Independent groups motivated by **money, hacktivism, or personal gain**.

* **FIN7 (Carbanak):** Targets financial institutions and POS systems.
* **Anonymous:** Hacktivist collective conducting political and social operations.

***

### &#x20;3. Financially Motivated Cybercrime Groups (FIN Groups)

These groups operate **like businesses**, with structured teams, infrastructure, and monetization strategies.

* **FIN4:** Specializes in stock market manipulation using insider info.
* **FIN6:** Steals payment card data from retail and hospitality.
* **FIN7 (Cobalt Group):** Famous for ATM cash-outs and POS malware.
* **FIN8:** Targets financial services with spear phishing and card theft.

&#x20;**Why important:** Although not nation-states, these groups can be as **sophisticated and dangerous as APTs**.

***

### &#x20;4. What are APTs?

**Advanced Persistent Threats (APTs):** Long-term, stealthy campaigns conducted by nation-states or highly organized groups.

* **Advanced:** Custom malware, zero-days, targeted phishing.
* **Persistent:** Maintain access for months or years.
* **Threat:** Clear intent and resources to cause damage.

**Real-World Examples:**

* **APT28 (Fancy Bear):** Linked to Russian election interference.
* **APT29 (Cozy Bear):** Responsible for SolarWinds compromise.
* **APT10 (Stone Panda):** Chinese espionage group targeting intellectual property.

***

### &#x20;5. Why APTs Are Special

* Long-term, resource-backed campaigns.
* Supported by governments.
* Use advanced methods: supply-chain attacks, zero-days, _living off the land_ (using legitimate tools like PowerShell).
* Focus more on **espionage and disruption** than immediate financial gain.

***

### &#x20;6. Case Study: FIN7 (a.k.a. Cobalt Group)

* **Background:** Financially motivated group but with APT-level sophistication.
* **Tactics:**
  * Phishing emails with malicious Word docs.
  * Remote malware controlling POS systems.
  * ATM jackpotting (forcing ATMs to eject cash).
* **Impact:** Stole over **$1 billion worldwide** from banks and retailers.
* **Lesson:** Not only nation-states — **cybercriminal groups can be equally advanced.**

***

### &#x20;7. Tools, Techniques, and Procedures (TTPs)

Defined in the **MITRE ATT\&CK Framework**, TTPs describe how attackers operate.

**12 Categories:**

1. Initial Access
2. Execution
3. Persistence
4. Privilege Escalation
5. Defense Evasion
6. Credential Access
7. Discovery
8. Lateral Movement
9. Collection
10. Command & Control
11. Exfiltration
12. Impact

**Example Flow:**

* **Initial Access:** Phishing email with a malicious link.
* **Execution:** Victim opens Excel with a macro.
* **Persistence:** Attacker creates a scheduled task.
* **Lateral Movement:** Compromises additional hosts.

&#x20;**MITRE ATT\&CK** helps defenders **map activity → understand threats → build detection rules.**

***

### &#x20;8. Proactive Defense

* **Threat Hunting:** Actively searching for attacker presence.
* **Purple Teaming:** Collaboration between offensive (red) and defensive (blue) teams.
* **Deception:** Honeypots, fake accounts, and decoy systems.
* **Threat Intelligence:** Tracking adversaries and anticipating attacks.

***

### 9. Operational Threat Intelligence & Precursors

#### Operational Threat Intelligence (OTI)

**Actionable insights** about attacker infrastructure, campaigns, and active threats.

#### Precursors

Early signs that an attack might be coming.

**Types:**

* **Port Scanning & Fingerprinting**
  * Example: Scan on port 3389 → possible RDP brute force.
* **Social Engineering & Reconnaissance**
  * Example: Fake LinkedIn profiles gathering job details.
* **OSINT Sources & Bulletin Boards**
  * Example: Credentials leaked on forums like _Pastebin_ or _BreachForums_.

⚠ **Challenges with Precursors:**

* Not all precursors are malicious (false positives).
* Require **context + correlation** to distinguish noise from real threats.

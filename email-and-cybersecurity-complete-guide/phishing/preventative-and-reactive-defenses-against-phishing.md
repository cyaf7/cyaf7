# Preventative & Reactive Defenses Against Phishing

Phishing remains one of the most common entry points for attackers. A strong defense requires both **preventative controls** (to stop threats before reaching users) and **reactive controls** (to quickly contain and mitigate when something slips through).



### Preventative Measures

### 1. Attachment Sandboxing

**What it is:**\
Attachments are automatically opened in a secure, isolated environment (sandbox) before being delivered to the user. This prevents malicious files from reaching inboxes.

**How it works:**

* Sandbox executes the file in a controlled VM.
* Monitors behavior (e.g., script execution, network calls, registry modifications).
* If malicious activity is detected → file is quarantined.

**Example:**

* A Word document with an embedded macro is opened in the sandbox.
* The macro attempts to launch PowerShell and download malware.
* The system detects this, blocks delivery, and alerts security teams.

**Why it matters:**\
Stops **zero-day malware** and **evasive attachments** that bypass signature-based detection.

***

### 2. Security Awareness Training

**What it is:**\
Educating employees to recognize, avoid, and report phishing attempts.

**Best practices:**

* Regular phishing simulations (controlled, fake phishing emails).
* Ongoing micro-trainings (2–5 minutes, scenario-based).
* Clear reporting channels (e.g., “Report Phish” button in Outlook).

**Example:**\
Employees receive a fake email offering a “company bonus.” Those who click are redirected to a learning page explaining red flags they missed.

**Why it matters:**\
Technology cannot stop 100% of threats. **Humans are the last line of defense.** Well-trained employees drastically reduce successful phishing attacks.

***

### Reactive Measures

When prevention fails, **speed and precision** in response are critical.

***

### 1. Immediate Response Process

**Standard steps for analysts once a phishing email is reported:**

1. Retrieve the original email (with full headers).
2. Collect artifacts: sender, subject, URLs, attachments.
3. Notify potentially affected users.
4. Investigate Indicators of Compromise (IOCs).
5. Apply defensive measures (quarantine, block).
6. Document the case for tracking and lessons learned.

**Example:**

* A user reports a suspicious message.
* The SOC retrieves the `.eml` file, extracts a malicious domain, and blocks it at the proxy.
* All copies of the phishing email are quarantined across inboxes.

***

### 2. Blocking Email Artifacts

Attackers often reuse the same infrastructure. Blocking known **email-based indicators** reduces exposure.

* Block sender addresses/domains.
* Block reply-to addresses (commonly different from sender).
* Configure security gateways to quarantine similar messages.

**Example:**\
Block all future emails from `@secure-microsoft-support.com`.

***

### 3. Blocking Web Artifacts

Phishing often involves malicious links. Web defenses stop users from reaching attacker-controlled sites.

**Techniques:**

* **Web proxy filtering** (monitors outgoing traffic).
* **URL blocking** (specific phishing links).
* **Domain blocking** (entire malicious domain).
* **DNS blackholing** (redirect malicious domains to a sinkhole).
* **Firewall blocking** (prevent outbound traffic to attacker IPs).

**Decision-making:**

* If it’s a dedicated phishing domain → block URL + domain.
* If attacker uses shared infrastructure (e.g., Google Drive, Dropbox) → block only the malicious path to avoid disrupting legitimate services.

***

### 4. Blocking File Artifacts

Malicious attachments can be fingerprinted and blocked globally.

**By Hash:**

* **MD5** → fast but outdated, prone to collisions.
* **SHA-1** → stronger, but deprecated.
* **SHA-256** → current standard.

**By Name:**

* Block suspicious filenames (e.g., `invoice.exe`, `update.scr`).
* Less effective, as attackers can rename easily.

**Example:**\
SOC identifies a malware sample. Its SHA-256 hash is added to the blocklist. Any attempt to send/receive that file in the future is automatically blocked.

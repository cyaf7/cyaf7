# Phishing Email Investigation — End-to-End Examples

### 1. Email Header, Artifacts, and Body Content

#### Example Email

```
From: Microsoft Security <support@micr0soft-security.com>
Reply-To: recovery@mailsafe.pro
To: user@company.com
Subject: Urgent - Your Office365 Account Has Been Suspended
Date: Tue, 02 Sep 2025 14:15:00 -0500
Received: from mail.evilhost.ru (203.0.113.25)
Authentication-Results: spf=fail; dkim=none; dmarc=fail
Body: 
Dear User,
Your account has been suspended. 
Login immediately to restore access: https://bit.ly/3FakeLink
```

#### Artifacts Retrieved

* **Sender:** `support@micr0soft-security.com` (typosquatting: “0” instead of “o”).
* **Reply-To:** `recovery@mailsafe.pro`.
* **Subject:** _Urgent - Your Office365 Account Has Been Suspended_.
* **Recipient:** `user@company.com`.
* **Sending Server IP:** `203.0.113.25`.
* **Reverse DNS:** `mail.evilhost.ru`.
* **Authentication Results:** SPF/DKIM/DMARC all failed.
* **Body Content:** Urgency, phishing link (`bit.ly/3FakeLink`).



### 2. Analysis Process, Tools, and Results

#### Step 1. Header Analysis

* **Tool:** PhishTool
* **Result:** Highlights SPF/DKIM/DMARC failures, mismatched From and Reply-To.

#### Step 2. URL Analysis

* **Tool:** URLScan.io
* **Result:** Expands bit.ly → `https://office365-login-secure.evilhost.ru`. Fake login form detected, hosted in Russia.
* **Tool:** VirusTotal (URL)
* **Result:** Domain flagged by 15 vendors as phishing.

#### Step 3. File/Attachment Check

(No attachments in this case — skip.)

#### Step 4. Visualization

* **Tool:** Maltego
* **Result:** Links `micr0soft-security.com` domain → IP → other phishing domains (`paypal-login.evilhost.ru`).

### 3. Defensive Measures Taken

#### Report Section (Example Extract)

**Incident Description:**\
User received phishing email claiming to be Microsoft. Email spoofed sender domain with typosquatting, contained malicious shortened URL, and SPF/DKIM/DMARC failed.

**Artifacts Blocked:**

* Domain: `micr0soft-security.com`.
* IP: `203.0.113.25`.
* URL: `office365-login-secure.evilhost.ru`.
* Short link: `bit.ly/3FakeLink`.

**Actions Taken:**

1. Quarantined all similar emails in user inboxes.
2. Blocked malicious domain/IP at firewall and DNS filters.
3. Added artifacts to email gateway blocklist.
4. Informed targeted users; reinforced phishing awareness.

**Follow-Up:**

* Submitted phishing URL to PhishTank/URLHaus.
* Shared IOCs with internal threat intel team.

***

### 4. Artifact Sanitization with CyberChef

* **Purpose:** Extract malicious parts for analysis without risk.
* **Process Example:**
  * Load email in CyberChef.
  * Use **“From Base64”** recipe to decode obfuscated URL.
  * Use **“Extract URLs”** to pull all links from the body.
  * Hash attachments with **“SHA-256”** recipe before uploading anywhere.

**Result:** Safe, sanitized IOCs ready for sharing and blocking.

***

### 5. Phishing Response Brief

**Title:** Phishing Campaign — Microsoft 365 Suspension Scam\
**Date/Time:** 02 Sep 2025, 14:15 CST\
**Target:** Employees at `company.com` (finance + IT)\
**Summary:**\
Phishing emails imitating Microsoft targeted employees. Emails failed SPF/DKIM/DMARC and contained shortened URLs leading to fake login pages hosted in Russia. No users reported credential compromise.

**Key Artifacts:**

* Domain: `micr0soft-security.com`
* IP: `203.0.113.25`
* URL: `bit.ly/3FakeLink` → `office365-login-secure.evilhost.ru`

**Actions Taken:**

* Quarantined and blocked emails.
* Blocked domains/IPs at firewall and DNS.
* Updated spam filter rules.
* User awareness reminder sent.

**Lessons Learned:**

* Strengthen external email tagging.
* Consider stricter DMARC enforcement.
* Continue phishing simulations to reduce user risk.

***

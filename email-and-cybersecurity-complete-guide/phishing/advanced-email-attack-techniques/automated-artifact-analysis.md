# Automated Artifact Analysis

### **5.1 File Artifact Analysis**

When investigating suspicious attachments, analysts look at:

**Hashes:** MD5, SHA-1, SHA-256 → identify if file is known malware.

**Metadata:** Creation date, author, software used. Suspicious if inconsistent.

**Embedded macros/scripts**: Office documents often contain VBA macros.

Behavior: When executed in a sandbox, what processes, network calls, or registry edits occur?

Example:

{% hint style="info" %}
A PDF attachment contains embedded JavaScript that, when opened, spawns powershell.exe to download malware from an attacker server.
{% endhint %}

Usefulness: Helps determine if a file is malicious without fully executing it on production machines.

### 5.2 Web Artifact Analysis

URLs and domains linked in phishing emails are **analyzed** for:

**Redirect chains:** Does the link forward multiple times before landing on final page?

**SSL certificates:** New domains often use free/short-lived certs.

**WHOIS info:** Recently registered or hidden registrant details.

**Hosting provider:** Bulletproof hosting or VPS providers often abused by attackers.

Example:

{% hint style="info" %}
A phishing link secure-login-paypaI\[.]com (note the “I” instead of “l”) was registered 2 days ago, uses a free SSL certificate, and hides ownership in WHOIS.
{% endhint %}

&#x20;Usefulness: Allows analysts to confirm phishing or malicious infrastructure quickly, without needing to click risky links.

### 6. Taking Defensive Actions

6.1 Preventative: Marking External Emails

What it is: Add a visible banner to any email from outside the company.

{% hint style="info" %}
Example: \[EXTERNAL] This email originated outside your organization. Do not click links unless trusted.
{% endhint %}

Why useful: Warns employees to apply extra caution when links or attachments are present.

### 6.2 Preventative: Email Security Technology

SPF (Sender Policy Framework)

Defines which servers can send mail for your domain.

{% hint style="info" %}
Example: v=spf1 include:\_spf.google.com -all → only Google servers allowed.
{% endhint %}

DKIM (DomainKeys Identified Mail)

Cryptographic signature added to emails to verify authenticity.

{% hint style="info" %}
Example: DKIM-Signature: d=example.com; s=selector1; ...
{% endhint %}

DMARC (Domain-based Message Authentication, Reporting & Conformance)

Aligns SPF/DKIM and gives policy on how to handle failures.

{% hint style="info" %}
Example: v=DMARC1; p=reject; rua=mailto:dmarc@example.com
{% endhint %}

Usefulness: Prevents attackers from spoofing your company domain in phishing attacks.

### 6.3 Preventative: Email Filters

Why important?

Stop spam/phishing before they ever reach inboxes.

Deployment Types:

Gateway spam filters: Installed on corporate email gateways.

Hosted spam filters: Cloud services (Proofpoint, Microsoft Defender).

Desktop spam filters: Local filtering on user’s machine.

### 6.4 Types of Spam Filters

Content filters: Match suspicious words/phrases.

{% hint style="info" %}
Example: “Lottery winner” + attachment = block.
{% endhint %}

Rule-based filters: Admin-defined policies.

{% hint style="info" %}
Example: Block all emails from @freemail.biz.
{% endhint %}

Bayesian filters: Learn patterns of spam vs. legit emails using probability.

{% hint style="info" %}
Example: Improves over time with user feedback.
{% endhint %}

### 6.5 Preventative: Attachment Filtering

Block high-risk file types at the gateway:

**Executables:** .exe, .bat, .vbs, .js, .ps1

**Disk images & HTML tricks:** .iso, .htm, .html

**Business files with risks:** .zip, .doc, .pdf, .xls

Example:

{% hint style="info" %}
Attacker sends Invoice-August.zip. Inside: Invoice-August.pdf.exe.
{% endhint %}

{% hint style="info" %}
Attachment filtering blocks it before it reaches the user.
{% endhint %}

Usefulness: Cuts off one of the most common malware delivery methods.

### Quick IOC Collection Checklist

When analyzing a suspicious email, collect:

**Headers:** From, Reply-To, Received chain, SPF/DKIM/DMARC results.

**Subject line:** Unusual urgency or misspellings.

**Body content:** Links, brand impersonation, language style.

**URLs**: Extract all links, expand shortened ones, check encoding tricks.

**Attachments:** Hash before uploading to sandboxes; note file type.

**Metadata:** Timestamps, sender infrastructure.

**Recipients:** Helps spot targeted campaigns.

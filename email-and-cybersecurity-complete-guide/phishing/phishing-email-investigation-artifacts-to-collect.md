---
description: >-
  When investigating a suspected phishing email, always preserve the full
  original message with headers intact. Then extract the following artifacts:
---

# Phishing Email Investigation — Artifacts to Collect

### 1. Email Artifacts (General)

* **Full email headers** (raw format, not just what the client shows).
* **Message body** (text and HTML versions).
* **Embedded URLs** (visible and hidden).
* **Attachments** (hash them before opening in sandbox).

**Example:**

* Extract both the plain text and HTML parts, because sometimes attackers hide malicious links in HTML styling.

***

### 2. Sending Email Address (From:)

* What the user sees in the “From” field.
* Compare with **Return-Path** and **Received** headers to detect spoofing.

**Example:**

```
From: PayPal Support <support@paypaI.com>
Return-Path: attacker@malicious.ru
```

(The “I” in PayPal is actually a capital letter i.)

***

### 3. Subject Line

* Can reveal urgency, lures, or campaigns.
* Useful for clustering similar phishing attempts.

**Example:**\
`Subject: URGENT - Your Account Will Be Suspended in 24 Hours`

***

### 4. Recipient Email Address

* Confirms who was targeted.
* Can show whether attackers used role-based targets (e.g., `finance@`, `hr@`) or specific individuals.

**Example:**\
Targeted: `ap@company.com` → clearly finance-related phishing attempt.

***

### 5. Sending Server IP & Reverse DNS

* Extract from the **first Received header** (closest to the source).
* Perform reverse DNS lookup to see if the IP matches the claimed domain.
* Check against threat intelligence feeds or blacklists.

**Example:**

```
Received: from mail.hacker.ru (203.0.113.55)
Reverse DNS: mail.hacker.ru
```

But the email _claimed_ to be from `paypal.com`.

***

### 6. Reply-To Address

* Sometimes different from the “From:” field.
* Attackers use this trick to redirect responses to their own inbox.

**Example:**

```
From: support@paypal.com
Reply-To: attacker@gmail.com
```

***

### 7. Date & Time

* Compare with known phishing waves (campaign correlation).
* Check if multiple users received the same email around the same time.

**Example:**\
`Date: Thu, 21 Aug 2025 09:15:00 -0500`

***

## &#x20;Best Practices During Collection

* **Do NOT click** links or open attachments directly. Always analyze in a sandbox.
* **Hash artifacts** (SHA256) before uploading to online scanners (VirusTotal, Hybrid Analysis).
* **Correlate across incidents:** Similar subject lines, sending IPs, or reply-to addresses often indicate campaign reuse.
* **Preserve chain of custody** if this could lead to legal action.

***

&#x20;These artifacts form the **baseline evidence set** in any phishing analysis. They help you pivot into:

* IOC (Indicators of Compromise) feeds
* Threat intel enrichment (WHOIS, IP reputation, URL sandboxing)
* Defensive actions (blocking domains, IPs, sender addresses)

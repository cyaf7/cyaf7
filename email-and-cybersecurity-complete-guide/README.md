# Email & Cybersecurity — Complete Guide

### 1. Email Fundamentals

#### 1.1 How Electronic Mail Works

Email follows a client–server process:

1. You write a message in an email client (Outlook, Thunderbird, Gmail app).
2. It’s sent to your mail server via **SMTP**.
3. That server looks up the recipient’s server using DNS **MX records**.
4. The message is delivered to the recipient’s mail server.
5. The recipient reads it via **IMAP** (synchronized) or **POP3** (downloaded).

**Example:**\
Alice sends `alice@example.com` → `bob@company.org`. DNS shows that `company.org`’s MX record points to `mx.company.org`. Alice’s server sends the message to that server using SMTP. Bob’s client uses IMAP to fetch the mail.

**Defense Tip:** Require TLS for SMTP and IMAP/POP3, and enable spam/malware filtering.

***

#### 1.2 Email Addresses

Format: `local-part@domain`.

* Local part = username (e.g., `alice`)
* Domain = server that handles the email (e.g., `example.com`)

**Examples:**

* `support@shop.com` → company support mailbox
* `john.smith@university.edu` → individual mailbox

***

#### 1.3 Email Protocols

* **SMTP:** Sends messages between servers.
* **IMAP:** Synchronizes mailboxes across devices.
* **POP3:** Downloads and usually deletes messages from the server.

**Example:** A user checks Gmail on their laptop and phone: IMAP keeps both synchronized.

***

#### 1.4 Webmail

Browser-based access to email (Gmail, Outlook.com, Yahoo Mail).

* **Pros:** Accessible anywhere, no client setup.
* **Cons:** Requires internet, session hijacking risks.

**Example:** Logging into Gmail from a café computer. Always log out after.

***

#### 1.5 Anatomy of an Email

Two main parts:

* **Header:** Metadata like sender, recipient, and routing info.
* **Body:** The actual message text, HTML, and attachments.

**Example:** A newsletter with a text-only fallback (for accessibility) and an HTML body with images and links.

***

#### 1.6 Email Header

Key fields:

* `From`, `To`, `Cc`, `Subject`, `Date`.
* `Received:` → server hops (useful for forensics).
* `Authentication-Results:` → shows SPF/DKIM/DMARC status.

**Example:**

```
From: Billing <billing@vendor.com>
To: ap@company.com
Subject: Invoice 2025-08
Received: from mx.vendor.com by mx.company.com
Authentication-Results: spf=pass; dkim=pass; dmarc=pass
```

***

#### 1.7 Email Body

* **Plain text:** Simple and safe.
* **HTML:** Can hide malicious links.
* **Multipart:** Includes both plain text and HTML.

**Example:**\
Displayed: `https://bank.com`\
Hidden: `<a href="http://evil.com">https://bank.com</a>`

**Defense Tip:** Hover over links to see the true destination.

***

#### 1.8 Domain Glossary

* **MX Record:** DNS entry for mail servers. Example: `example.com MX 10 mail.example.com`.
* **SPF:** Authorizes sending servers. Example: `v=spf1 include:_spf.example.com -all`.
* **DKIM:** Cryptographic signature for authenticity.
* **DMARC:** Policy on how to handle failed SPF/DKIM. Example: `v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com`.

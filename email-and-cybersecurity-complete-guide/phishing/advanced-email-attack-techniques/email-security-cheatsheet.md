# Email Security Cheatsheet

### Red Flags in Emails

* **Sender mismatch:** “From” looks right, but the return-path or domain is different.
* **Urgency:** “Your account will be blocked in 24h!”
* **Too good to be true:** Lottery winnings, miracle products.
* **Suspicious links:** Displayed text ≠ real destination (hover to check).
* **Unexpected attachments:** Especially `.zip`, `.exe`, `.docm`, `.xlsm`.
* **Typos / misspellings:** `micorsoft.com`, `paypaI.com`.
* **Requests for sensitive info:** Passwords, MFA codes, bank details.
* **Generic greetings:** “Dear User” instead of your real name.

***

### &#x20; Defenses (For Users)

* **Hover before clicking:** Always check the true URL.
* **Never share MFA codes:** Banks and IT will never ask.
* **Verify requests out-of-band:** Call your boss/vendor directly.
* **Be cautious with attachments:** If unexpected, confirm with sender.
* **Use bookmarks:** Access banking or corporate portals via saved links, not emails.
* **Enable MFA everywhere:** Prevents password-only compromise.
* **Report suspicious emails:** Don’t delete—help security teams improve filters.

***

### &#x20;Defenses (For Organizations)

* **Authentication:** Enforce SPF, DKIM, DMARC (`p=quarantine` or `reject`).
* **Attachment policies:** Block risky file types, sandbox unknown senders.
* **Link protection:** Rewrite and scan URLs, expand shortened links.
* **Impersonation protection:** Alert when external users impersonate executives.
* **MFA enforced:** Disable legacy IMAP/POP protocols.
* **Awareness training:** Regular phishing simulations + employee education.
* **Dual approval:** Required for payment/bank detail changes.

| **Red Flag**                         | **Example**                                                            | **Defense**                                                  |
| ------------------------------------ | ---------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Sender mismatch**                  | From: `security@paypal.com` but real sender is `attacker@malicious.ru` | Check full headers; enforce SPF/DKIM/DMARC                   |
| **Urgency / Fear**                   | “Your account will be deleted in 24 hours!”                            | Pause, verify with official support before acting            |
| **Too good to be true**              | “You won the lottery!”                                                 | Ignore & report; real services don’t notify via random email |
| **Suspicious link**                  | Display shows `bank.com` but points to `evil.com`                      | Hover before clicking; use bookmarks                         |
| **Unexpected attachment**            | Invoice.zip containing `malware.exe`                                   | Don’t open; confirm with sender; block risky file types      |
| **Typos / Homograph domains**        | `micorsoft.com`, `paypaI.com` (capital i)                              | Carefully read domains; train users on lookalikes            |
| **Generic greeting**                 | “Dear User”                                                            | Legitimate businesses usually use your real name             |
| **Requests for sensitive info**      | “Send your MFA code to IT”                                             | Never share codes/passwords via email/phone                  |
| **Fake invoices / payment requests** | “Please wire funds to this updated account”                            | Verify out-of-band (call vendor); dual approval required     |

### Admin Quick Wins

* Enforce **MFA** everywhere (prefer FIDO2/WebAuthn).
* Deploy **SPF, DKIM, DMARC** (reject/quarantine).
* Block dangerous attachment types (`.exe`, `.js`, `.docm`).
* Expand & scan shortened URLs.
* Tag **external senders** in email headers.
* Train employees with phishing simulations.
* Require **two-person verification** for money transfers.

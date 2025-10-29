# TLP & PAP — Intelligence Sharing Controls

When sharing threat intelligence, organizations must control **who can access information** and **what actions are allowed**. Two protocols are widely used for this purpose:

* **Traffic Light Protocol (TLP):** Controls _distribution_.
* **Permissible Actions Protocol (PAP):** Controls _usage and actions_.

Together, they ensure sensitive intelligence is shared responsibly while still being operationally useful.

***

### &#x20;1. Traffic Light Protocol (TLP)

#### What It Is

The **Traffic Light Protocol (TLP)** is a simple classification system created by **FIRST** (Forum of Incident Response and Security Teams).

* **Purpose:** Ensure sensitive information is distributed appropriately.
* **Format:** Labels written in **all caps** (e.g., `TLP:GREEN`).
* **Usage:** Found in reports, MISP instances, ISAC alerts, and CERT bulletins.

***

#### TLP Classifications

&#x20;**TLP:CLEAR (formerly TLP:WHITE)**

* **Meaning:** May be shared publicly, without restriction.
* **Example:** Blog posts, press releases, public IOCs.
* **Use case:** Widely useful intelligence meant for open distribution.

**🟢 TLP:GREEN**

* **Meaning:** Can be shared within the community/sector but not publicly.
* **Example:** An FS-ISAC alert shared across financial institutions.
* **Use case:** Encourages collaboration within industries while preventing public leaks.

**🟡 TLP:AMBER**

* **Meaning:** Shareable only _within the recipient’s organization_.
* **Example:** SOC team receives details of an active phishing campaign.
* **Use case:** Protects sensitive intelligence while enabling defense.

**🟡 TLP:AMBER+STRICT (new)**

* **Meaning:** Same as Amber, but restricted to **specific need-to-know roles** within the org.
* **Example:** Zero-day exploit details shared only with SOC analysts, not the entire company.
* **Use case:** Prevents leaks even inside large organizations.

**🔴 TLP:RED**

* **Meaning:** For direct recipients only, **no further sharing**.
* **Example:** Classified intel shared with a CISO under NDA.
* **Use case:** Prevents disclosure that could compromise investigations.

***

### &#x20;2. Permissible Actions Protocol (PAP)

#### What It Is

The **Permissible Actions Protocol (PAP)** complements TLP by defining **what actions can be taken** with the intelligence, not just _who can see it_.

* **Purpose:** Avoid misuse (e.g., premature attribution or PR exposure).
* **Usage:** Often in **ISACs, CERTs, and trusted intel-sharing communities**.

***

#### PAP Classifications

**🟢 PAP:GREEN**

* **Meaning:** Information can be shared and acted upon freely.
* **Example:** IOC of a phishing domain → block it in firewalls, publish in blogs.

**🟡 PAP:AMBER**

* **Meaning:** Information can be used for **internal defense only**, but not externally attributed.
* **Example:** Malware hash may be blocked internally, but cannot be announced as “APT28 malware” in public reports.

**🔴 PAP:RED**

* **Meaning:** Information is for **intelligence purposes only**, not for external defense or disclosure.
* **Example:** Government-provided TTPs → cannot be used for PR or public reports.

***

### 3. TLP vs PAP in Practice

* **TLP = Distribution control** → _Who can know?_
* **PAP = Action control** → _What can you do with it?_

**Example:**

* A report marked **TLP:AMBER + PAP:AMBER** →
  * Can only be shared _inside your organization_.
  * May be used for **defense**, but not for **public attribution**.

***

### Key Takeaways

* **TLP** ensures information reaches the _right audience_.
* **PAP** ensures information is _used appropriately_.
* Together, they prevent overexposure, misuse, and reputational risk while preserving operational value.

# Threat Intelligence

Threat Intelligence (TI) is the process of collecting, analyzing, and using information about adversaries, their tactics, techniques, and procedures (TTPs), to **anticipate, prevent, and respond** to cyber threats.



### 1. Threat Intelligence Lifecycle

The **Threat Intelligence Lifecycle** describes the structured process to produce actionable intelligence:

#### 1.1 Planning and Direction

* **What:** Define goals and requirements.
* **Example:** “Identify phishing campaigns targeting our finance team.”
* **Why:** Sets the scope so analysts don’t waste time on irrelevant data.

#### 1.2 Collection

* **What:** Gather raw data from internal and external sources.
* **Sources:** Logs, IDS alerts, honeypots, threat feeds, dark web forums.
* **Example:** Collect phishing domains from spam filters and URLHaus.

#### 1.3 Processing

* **What:** Normalize and prepare raw data for analysis.
* **Techniques:** Parsing logs, extracting IOCs (Indicators of Compromise), converting into structured formats (STIX/TAXII).
* **Example:** Converting firewall logs into a timeline of suspicious IPs.

#### 1.4 Analysis

* **What:** Transform processed data into meaningful intelligence.
* **Example:** Determining that repeated login attempts from `203.0.113.50` match a known brute-force campaign.

#### 1.5 Dissemination

* **What:** Deliver intelligence to stakeholders in the right format.
* **Example:** Security engineers receive IOCs in machine-readable feeds; executives get risk summaries in reports.

#### 1.6 Feedback

* **What:** Stakeholders give feedback → improve next cycle.
* **Example:** Analysts prioritize new phishing techniques after management reports higher risk.

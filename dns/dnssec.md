---
description: In this one I tried (with chat gpt's help) to understand dnssec usage.
---

# DNSSEC

Great — here is the **complete, integrated DNSSEC chapter** for your GitBook, now **including zones and reverse (inverted IP) zones**. It’s written in clear, academic English, structured with Markdown headings, and comes with step-by-step lab sections (a minimal “normal” validation lab and a full **BIND9** signing lab for forward and reverse zones).

***

## DNSSEC — Complete Guide

### 1) What Is DNSSEC

**DNSSEC (Domain Name System Security Extensions)** is a family of extensions that adds **authentication and integrity** to DNS. It allows DNS data to be **digitally signed** so that validating resolvers can verify that responses are authentic and unmodified. DNSSEC mitigates DNS spoofing and cache poisoning, and enables higher-level mechanisms such as **DANE**.

DNSSEC does **not** provide confidentiality; it authenticates data. (Encryption of queries is provided by other protocols such as DoT/DoH.)

***

### 2) Why DNSSEC Matters

* **Authenticity:** Confirms the data comes from the authoritative zone owner.
* **Integrity:** Detects tampering in transit.
* **Authenticated denial of existence:** Proves securely that a name or record does not exist (via **NSEC/NSEC3**).
* **Foundation for trust:** Supports DANE/TLSA, reduces reliance on third-party CAs in some workflows.

***

### 3) Core Concepts and Records

* **DNSKEY:** Public keys published by the zone. Two roles are typical:
  * **KSK (Key Signing Key):** Signs the DNSKEY RRset (i.e., signs keys).
  * **ZSK (Zone Signing Key):** Signs all other RRsets (A/AAAA/MX/TXT/etc.).
* **RRSIG:** The cryptographic signature over each RRset.
* **DS (Delegation Signer):** In parent zone; points to (hash of) child zone KSK to build the chain of trust.
* **NSEC / NSEC3:** Authenticated denial of existence (NSEC3 adds hashing to limit zone walking).
* **CD/AD flags (resolver side):** `AD` means “Authenticated Data” (successful validation).

***

### 4) Chain of Trust (High-Level)

Validation starts from a configured **trust anchor** (the **root** KSK), then follows DS/DNSKEY links down the hierarchy.

```
Root (.)  --signs-->  TLD (.com)  --signs-->  example.com  --signs-->  records (A/MX/TXT...)
```

If any step is missing or signatures fail, validation fails.

#### 4.1 Text Diagram

***

```
                 ┌──────────────────────────────────┐
                 │            ROOT ZONE (.)          │
                 │  Contains the root KSK and ZSK    │
                 │  Signs DS records for all TLDs    │
                 └───────────────┬───────────────────┘
                                 │
                                 ▼
                 ┌──────────────────────────────────┐
                 │          TLD ZONE (.com)          │
                 │  Has its own KSK and ZSK          │
                 │  DS record signed by root zone    │
                 │  Signs DS record for example.com  │
                 └───────────────┬───────────────────┘
                                 │
                                 ▼
                 ┌──────────────────────────────────┐
                 │     SECOND-LEVEL DOMAIN           │
                 │         example.com               │
                 │  Holds its KSK and ZSK            │
                 │  KSK signs ZSK                    │
                 │  ZSK signs DNS records (A, MX…)   │
                 │  DS record stored in .com zone    │
                 └───────────────┬───────────────────┘
                                 │
                                 ▼
                 ┌──────────────────────────────────┐
                 │     AUTHENTICATED DNS RECORDS     │
                 │  RRSIG attached to each record:   │
                 │   - A (IPv4 Address)              │
                 │   - AAAA (IPv6 Address)           │
                 │   - MX (Mail Server)              │
                 │   - TXT, CNAME, etc.              │
                 └──────────────────────────────────┘
```

### 5) What Is a DNS Zone

A **DNS zone** is an administrative portion of the DNS namespace with its own **SOA**, **NS** set, and resource records. Zones enable **delegation** and independent management.

* **Master (primary) zone:** Authoritative source of truth (edited here).
* **Slave (secondary) zone:** Read-only replica via AXFR/IXFR.
* **Stub zone:** Only NS (and glue) to locate authoritative servers.
* **Forward zone:** Forwards queries to upstream resolvers (not authoritative data).

Example delegation:

```
Root (.)
 └─ com
     └─ example.com  (zone managed by Example Corp)
        ├─ www.example.com
        ├─ mail.example.com
        └─ _443._tcp.www.example.com (TLSA for DANE)
```

***

### 6) Reverse (Inverted IP) Zones

Reverse DNS maps **IP → name** using **PTR** records.

* **IPv4:** `in-addr.arpa` (octets reversed by labels).
* **IPv6:** `ip6.arpa` (nibbles reversed).

Examples:

* IPv4 address `192.168.1.10` belongs in zone: `1.168.192.in-addr.arpa.` (the zone usually covers a subnet boundary).
* IPv6 address `2001:db8::1234` → reversed nibble zone under `ip6.arpa`.

Reverse DNS is widely used for:

* Mail server reputation and policy checks.
* Logging, audits, and troubleshooting.
* Some access controls or identity hints in legacy systems.

**DNSSEC applies equally to reverse zones:** PTR answers can be signed and validated.

***

### 7) How DNSSEC Works (Step-by-Step)

1. **Authoritative side (zone owner):**
   * Generate **ZSK** and **KSK**.
   * Publish **DNSKEY** records and sign all RRsets → produce **RRSIGs**.
   * Publish a **DS** record in the **parent** zone that references the child zone’s KSK (out-of-band step with registrar/registry).
2. **Resolver side (validator):**
   * Trusts the **root** KSK (preconfigured trust anchor).
   * Fetches responses, RRSIGs, DNSKEYs; verifies signatures and chain of DS→DNSKEY links down to the answer.

***

### 8) Operational Topics

* **Key sizes/algorithms:** Common modern choice is **ECDSA P-256 (algorithm 13)** or **Ed25519 (algorithm 15)** for speed and smaller responses; RSA still used widely.
* **Key rollover:** Periodically rotate ZSK; rotate KSK less frequently (requires DS update in parent). Follow pre-publish/double-sign best practices to avoid validation breaks.
* **NSEC vs NSEC3:** Use **NSEC3** if you want to limit trivial zone enumeration; choose appropriate iterations and opt-out if needed for large delegations.
* **Response size & fragmentation:** DNSSEC increases response sizes; prefer **EDNS(0)** and ensure PMTU/UDP handling is sane; TCP fallback should work.
* **Monitoring:** Use `dnsviz`, `delv`, `dig +dnssec`, and resolver logs to detect validation issues early.

***

### 9) Minimal “Normal” Lab: DNSSEC Validation Without Running Authoritative DNS

**Goal:** On a client or lab VM, confirm that your resolver validates DNSSEC.

1. Use a validating public resolver (e.g., 1.1.1.1 or 8.8.8.8) or install a local validator (e.g., **Unbound**).
2. Query a deliberately broken domain:

```bash
dig dnssec-failed.org +dnssec
```

* A validating resolver should **reject** the answer (SERVFAIL or no data).
* Then test a signed domain:

```bash
dig cloudflare.com +dnssec
```

* Look for the `ad` flag (Authenticated Data) in the header of the response.

This lab proves you understand the **validation side** (consumer of DNSSEC).

***

### 10) Full Authoritative Lab with BIND9 (Forward Zone)

**Goal:** Sign a forward zone `lab.local` with BIND9 and validate answers.

#### 10.1 Install and Base Zone

```bash
sudo apt update
sudo apt install bind9 bind9utils dnsutils
```

`/etc/bind/named.conf.local`:

```conf
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
    allow-transfer { none; };
};
```

`/etc/bind/db.lab.local`:

```dns
$TTL 86400
@   IN  SOA ns1.lab.local. admin.lab.local. (
        2025010101 ; Serial
        3600       ; Refresh
        1800       ; Retry
        1209600    ; Expire
        86400 )    ; Minimum
@       IN  NS      ns1.lab.local.
ns1     IN  A       192.168.1.10
www     IN  A       192.168.1.20
```

Reload to verify syntax:

```bash
sudo named-checkconf
sudo named-checkzone lab.local /etc/bind/db.lab.local
sudo systemctl restart bind9
```

#### 10.2 Generate Keys (KSK and ZSK)

From `/etc/bind`:

```bash
cd /etc/bind
# ZSK (example RSA; you may prefer ECDSA P-256 with -a ECDSAP256SHA256)
dnssec-keygen -a RSASHA256 -b 2048 -n ZONE lab.local
# KSK
dnssec-keygen -f KSK -a RSASHA256 -b 4096 -n ZONE lab.local
```

This creates `Klab.local.+008+<id>.key|private` files (DNSKEYs).

#### 10.3 Sign the Zone

```bash
dnssec-signzone -A -N increment -o lab.local -t /etc/bind/db.lab.local
```

This produces `/etc/bind/db.lab.local.signed` and a `dsset-lab.local.` file (contains DS candidates).

Update the zone stanza to use the signed file:

```conf
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local.signed";
    allow-transfer { none; };
};
```

Restart:

```bash
sudo systemctl restart bind9
```

#### 10.4 Test (from a client)

```bash
dig @192.168.1.10 www.lab.local +dnssec
```

You should receive RRSIGs. If your client’s **resolver** validates DNSSEC and you have an established chain (i.e., DS in parent for a real domain), you will see the `ad` flag.\
In a closed lab without a parent DS, you still see the signatures (`RRSIG`), but external validators will not mark them as `ad` unless you add the trust chain or configure a local trust anchor.

#### 10.5 Resolver-Side Local Trust Anchor (Optional for Lab)

If you want **full validation** without publishing DS to a real parent, configure your testing resolver with a **local trust anchor** (e.g., Unbound `trust-anchor-file` pointing to `Klab.local...key`). Then `dig +dnssec` will show `ad` for `lab.local`.

***

### 11) Reverse Zone Lab with BIND9 (IPv4)

**Goal:** Provide authenticated PTR records for `192.168.1.0/24`.

#### 11.1 Define the Reverse Zone

`/etc/bind/named.conf.local` (add this alongside `lab.local`):

```conf
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
    allow-transfer { none; };
};
```

Create `/etc/bind/db.192.168.1`:

```dns
$TTL 86400
@   IN  SOA ns1.lab.local. admin.lab.local. (
        2025010101 ; Serial
        3600
        1800
        1209600
        86400 )
@       IN  NS      ns1.lab.local.
10      IN  PTR     www.lab.local.
20      IN  PTR     mail.lab.local.
```

Check and reload:

```bash
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1
sudo systemctl restart bind9
```

#### 11.2 Sign the Reverse Zone

Generate keys (separate keyset is recommended per zone):

```bash
cd /etc/bind
dnssec-keygen -a RSASHA256 -b 2048 -n ZONE 1.168.192.in-addr.arpa
dnssec-keygen -f KSK -a RSASHA256 -b 4096 -n ZONE 1.168.192.in-addr.arpa
```

Sign:

```bash
dnssec-signzone -A -N increment -o 1.168.192.in-addr.arpa -t /etc/bind/db.192.168.1
```

Point BIND to the signed file:

```conf
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1.signed";
    allow-transfer { none; };
};
```

Reload:

```bash
sudo systemctl restart bind9
```

#### 11.3 Test PTR and Signatures

```bash
dig @192.168.1.10 -x 192.168.1.10 +dnssec
dig @192.168.1.10 1.168.192.in-addr.arpa DNSKEY +dnssec
```

You should see `RRSIG` on answers and on the DNSKEY RRset. As with the forward zone, full validation requires either (a) DS in the parent reverse delegation or (b) a local trust anchor on your validating resolver.

***

### 12) Side-by-Side Diagram: Forward vs Reverse with DNSSEC

```
FORWARD PATH (name → address):
example.com (zone)
  - DNSKEY (KSK/ZSK)
  - RRSIG over A/AAAA/MX
  - DS published at parent (.com)
Validation: Root → .com → example.com → A/AAAA RRSIG

REVERSE PATH (address → name):
1.168.192.in-addr.arpa (zone)
  - DNSKEY (KSK/ZSK)
  - RRSIG over PTR
  - DS published at parent (reverse delegation)
Validation: Root → in-addr.arpa → 192.in-addr.arpa → 168.192.in-addr.arpa → PTR RRSIG
```

***

### 13) Example: BIND9 Configuration Reference (Snippets)

#### 13.1 named.conf.options (EDNS/UDP hygiene suggested)

```conf
options {
    directory "/var/cache/bind";
    recursion no;             // authoritative servers often disable recursion
    dnssec-enable yes;
    dnssec-validation no;     // authoritative only; set 'auto' on resolvers
    max-udp-size 1232;        // help avoid fragmentation on some paths
    listen-on { any; };
    listen-on-v6 { any; };
};
```

#### 13.2 Resolver Mode (if you build a lab validator on BIND)

```conf
options {
    directory "/var/cache/bind";
    recursion yes;
    dnssec-validation auto;   // uses built-in root trust anchor
    listen-on { 127.0.0.1; 192.168.1.10; };
};
```

***

### 14) Testing and Troubleshooting

*   **Check zone syntax:**

    ```bash
    named-checkzone lab.local /etc/bind/db.lab.local
    named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1
    ```
*   **Inspect signatures and keys:**

    ```bash
    dig @192.168.1.10 lab.local DNSKEY +dnssec
    dig @192.168.1.10 www.lab.local A +dnssec
    dig @192.168.1.10 -x 192.168.1.10 +dnssec
    ```
*   **Validation probes (using an external validating resolver):**

    ```bash
    dig cloudflare.com +dnssec
    dig dnssec-failed.org +dnssec
    ```
* **Common problems:**
  * **Expired RRSIGs:** Re-sign or ensure automatic re-signing.
  * **Mismatched DS:** DS at parent does not match new KSK after rollover. Publish new DS **before** switching KSK signatures.
  * **EDNS/size issues:** Set conservative `max-udp-size` (e.g., 1232), allow TCP fallback.
  * **Clock skew:** Ensure NTP; signature validity windows rely on correct time.

***

### 15) Security and Operations Best Practices

* Prefer modern algorithms (e.g., **ECDSAP256SHA256** or **Ed25519**) for smaller responses and faster crypto.
* **Automate re-signing** (e.g., `inline-signing` + `auto-dnssec maintain` in BIND, or cron with `dnssec-signzone`).
* **Document key rollover** runbooks; test in staging.
* Monitor with `dnsviz.net`, `delv`, resolver logs, and alerts for SERVFAIL spikes.
* For large delegations, consider **NSEC3 with opt-out** to scale.

***

### 16) Summary

* **Zones** are administrative slices of the DNS namespace; they contain SOA/NS and the records they authoritatively serve.
* **DNSSEC** augments zones with **DNSKEY** and **RRSIG** records and establishes authenticity via a parent-to-child **DS** chain.
* **Reverse zones** (`in-addr.arpa`, `ip6.arpa`) map **IP → name** and can be signed exactly like forward zones, allowing authenticated **PTR** answers.
* In labs, you can validate with a public resolver (“normal” validation lab) and practice full authoritative signing with **BIND9** for both forward and reverse zones.

***


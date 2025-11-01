# IP TABL

## iptables — Complete Guide

### 1) Overview

**iptables** is a Linux command-line tool used to configure the **Netfilter** framework in the kernel.\
It allows administrators to create, list, and modify firewall rules that control **how packets are handled** — whether they are accepted, dropped, logged, or modified.

iptables uses **tables** that contain **chains** of **rules**.\
Each rule matches certain traffic and performs an action (known as a **target**).

***

### 2) Netfilter Framework

Linux processes packets through several hooks in the kernel network stack.\
When a packet enters or leaves a system, it passes through these hooks where **iptables** rules are evaluated.

#### Packet Flow

```
Inbound traffic to local machine:
  PREROUTING → INPUT → local process

Traffic routed through host:
  PREROUTING → FORWARD → POSTROUTING

Outbound traffic from host:
  OUTPUT → POSTROUTING
```

***

### 3) Tables and Chains

#### 3.1 Main Tables

| Table        | Purpose                     | Typical Use                      |
| ------------ | --------------------------- | -------------------------------- |
| **filter**   | Main firewall table         | Allow or deny traffic            |
| **nat**      | Network Address Translation | SNAT, DNAT, port forwarding      |
| **mangle**   | Packet modification         | Change TTL, TOS, or mark packets |
| **raw**      | Pre-connection tracking     | Exempt packets from conntrack    |
| **security** | SELinux controls            | Fine-grained labeling (optional) |

#### 3.2 Default Chains by Table

| Table      | Chains                                                    | Typical Use         |
| ---------- | --------------------------------------------------------- | ------------------- |
| **filter** | `INPUT`, `FORWARD`, `OUTPUT`                              | General filtering   |
| **nat**    | `PREROUTING`, `POSTROUTING`, `OUTPUT`                     | NAT and redirection |
| **mangle** | `PREROUTING`, `POSTROUTING`, `INPUT`, `OUTPUT`, `FORWARD` | Modify packets      |
| **raw**    | `PREROUTING`, `OUTPUT`                                    | Disable conntrack   |

***

### 4) Command Structure

Every iptables command follows this format:

```bash
iptables [OPTIONS] [CHAIN] [CONDITIONS] [ACTIONS]
```

Example:

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

This appends (`-A`) a rule to the `INPUT` chain that accepts TCP packets going to port 22 (SSH).

***

### 5) Command and Option Reference

| Command / Option        | Meaning              | Explanation                                                        |
| ----------------------- | -------------------- | ------------------------------------------------------------------ |
| `-t`                    | **table**            | Select which table to operate on (`filter`, `nat`, `mangle`, etc.) |
| `-A`                    | **Append**           | Add a new rule at the end of a chain                               |
| `-I`                    | **Insert**           | Insert a rule at a specific position (default: at top)             |
| `-D`                    | **Delete**           | Delete a rule (by number or full match)                            |
| `-R`                    | **Replace**          | Replace a rule at a specific position                              |
| `-L`                    | **List**             | Show current rules                                                 |
| `-F`                    | **Flush**            | Delete all rules in a chain                                        |
| `-Z`                    | **Zero**             | Reset packet and byte counters                                     |
| `-P`                    | **Policy**           | Set default policy for a chain (`ACCEPT` or `DROP`)                |
| `-N`                    | **New chain**        | Create a custom chain                                              |
| `-X`                    | **Delete chain**     | Remove a custom chain                                              |
| `-s`                    | **Source**           | Match packets from a specific source IP or subnet                  |
| `-d`                    | **Destination**      | Match packets to a specific destination                            |
| `-p`                    | **Protocol**         | Match protocol (`tcp`, `udp`, `icmp`, etc.)                        |
| `--dport`               | **Destination port** | Match a specific destination port (for TCP/UDP)                    |
| `--sport`               | **Source port**      | Match a specific source port                                       |
| `-i`                    | **Input interface**  | Match the incoming network interface                               |
| `-o`                    | **Output interface** | Match the outgoing interface                                       |
| `-m`                    | **Match module**     | Load an extension (e.g., `state`, `conntrack`, `limit`, `recent`)  |
| `--state` / `--ctstate` | **Connection state** | Match packets by connection state (e.g., NEW, ESTABLISHED)         |
| `-j`                    | **Jump to target**   | Defines what to do with matching packets (`ACCEPT`, `DROP`, etc.)  |
| `-v`                    | **Verbose**          | Show detailed output (packet counts, interfaces, etc.)             |
| `-n`                    | **Numeric**          | Display numeric IPs and ports (no DNS lookup)                      |

***

### 6) Common Targets (Actions)

| Target         | Description                                                              |
| -------------- | ------------------------------------------------------------------------ |
| **ACCEPT**     | Allow the packet to continue                                             |
| **DROP**       | Silently discard the packet                                              |
| **REJECT**     | Block the packet and send an ICMP “unreachable” message                  |
| **LOG**        | Record packet info in system logs (`/var/log/syslog`)                    |
| **DNAT**       | Change the destination address (used in `PREROUTING` of the `nat` table) |
| **SNAT**       | Change the source address (used in `POSTROUTING` of the `nat` table)     |
| **MASQUERADE** | Dynamic SNAT for systems with changing IP (used in routers)              |
| **REDIRECT**   | Redirect traffic to a local port (often for proxies)                     |
| **RETURN**     | Return to previous chain (used with custom chains)                       |

***

### 7) Practical Labs

#### Lab 1 — Simple Host Firewall

**Goal:** Only allow SSH (22) and HTTP (80) inbound, drop everything else.

```bash
# Flush old rules
sudo iptables -F

# Default policies
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH and HTTP
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# View rules
sudo iptables -L -v -n
```

**Explanation:**

* `-P INPUT DROP` → default drop.
* `--ctstate ESTABLISHED,RELATED` → allows replies for outgoing requests.
* `-A INPUT -p tcp --dport 22` → accepts SSH connections.

***

#### Lab 2 — NAT Gateway (with Internet Sharing)

**Goal:** Share Internet from `eth0` (external) to LAN `eth1` (internal).

```bash
# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# NAT: Masquerade outbound traffic
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Allow forwarding between interfaces
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

**Explanation:**

* `-t nat` selects the NAT table.
* `POSTROUTING` modifies packets _after_ routing.
* `MASQUERADE` dynamically hides internal IPs behind the external one.

***

#### Lab 3 — Port Forwarding (DNAT)

**Goal:** Forward external port 80 to an internal web server (192.168.1.10:80).

```bash
# Redirect incoming traffic to internal server
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 \
  -j DNAT --to-destination 192.168.1.10:80

# Allow forwarding to that destination
sudo iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT
```

**Explanation:**

* `PREROUTING` in `nat` handles packets before routing.
* `DNAT` rewrites destination IP to internal host.

***

#### Lab 4 — Rate Limiting SSH

**Goal:** Limit SSH connections to 3 attempts per minute per IP.

```bash
sudo iptables -A INPUT -p tcp --dport 22 -m limit --limit 3/minute --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
```

**Explanation:**

* `-m limit` loads the “limit” module.
* `--limit 3/minute` restricts rate of new connections.
* Once exceeded, further attempts are dropped.

***

#### Lab 5 — Integrating iptables with BIND9 (DNS)

**Goal:** Allow DNS queries only from internal LAN, block all others.

```bash
# Allow DNS (UDP/TCP port 53) from LAN only
sudo iptables -A INPUT -p udp --dport 53 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 53 -s 192.168.1.0/24 -j ACCEPT

# Drop all other DNS requests
sudo iptables -A INPUT -p udp --dport 53 -j DROP
sudo iptables -A INPUT -p tcp --dport 53 -j DROP
```

**Result:**\
Internal hosts can resolve domains using your BIND9 DNS server, but external sources cannot.

***

### 8) Logging and Monitoring

```bash
# Log dropped packets
sudo iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "Dropped packet: " --log-level 4
```

Check logs:

```bash
sudo tail -f /var/log/syslog
```

**Explanation:**

* `-m limit` prevents flooding logs.
* `--log-prefix` adds a readable tag.
* Level `4` = warning/info severity.

***

### 9) Saving and Restoring Rules

```bash
# Save current rules (Debian/Ubuntu)
sudo iptables-save > /etc/iptables/rules.v4

# Restore at boot
sudo iptables-restore < /etc/iptables/rules.v4
```

Alternative (systemd-based):

```bash
sudo apt install netfilter-persistent
sudo systemctl enable netfilter-persistent
```

***

### 10) Troubleshooting Commands

| Command                    | Description                                             |
| -------------------------- | ------------------------------------------------------- |
| `iptables -L -v -n`        | List all rules with packet counters and numeric output  |
| `iptables -S`              | Show raw rules as command syntax                        |
| `iptables -t nat -L -n -v` | Show NAT rules                                          |
| `iptables -xvnL`           | Extended counters, no name resolution                   |
| `iptables -Z`              | Reset counters                                          |
| `conntrack -L`             | Display active connections (requires `conntrack-tools`) |

***

### 11) Summary

* **iptables** controls Linux packet filtering and NAT through rules organized in **tables** and **chains**.
* Each rule has **conditions (matches)** and a **target (action)**.
* The **filter table** decides whether packets are allowed; the **nat table** modifies packet addresses.
* Use **stateful matching** (`--ctstate`) to safely allow return traffic.
* iptables remains a cornerstone of network security on Linux, despite the rise of **nftables**.

***


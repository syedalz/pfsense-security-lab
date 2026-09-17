# pfSense Firewalled Lab Network

A virtualized network built to demonstrate core firewall, routing, and network-segmentation fundamentals. A pfSense firewall sits between an untrusted "outside" network and a sealed "inside" network, controlling all traffic between them. A client machine lives behind the firewall and reaches the internet only by routing through it.

**Status:** Milestone 3 complete — an existing Active Directory environment was integrated as a third firewalled zone; a domain client in the user zone authenticates to a domain controller in the AD zone through explicit least-privilege rules, and can reach *only* the permitted AD services.

---

## Objective

Build the smallest complete version of an enterprise network topology: a firewall dual-homed between two networks, serving DHCP and routing traffic for an internal client. This is the foundation layer that segmentation, monitoring, and detection work is later built on top of.

---

## Architecture

```
   Internet
      │
      │ VMnet8 (NAT)
      │
 ┌────┴──────────────────────────────────────────────────────────┐
 │  WAN em0 192.168.150.128/24 (leased by NAT)                    │
 │                                                                │
 │                          pfSense                               │
 │                 firewall / router / DHCP                       │
 │                                                                │
 │  LAN em1          SERVERS em2         CORP em3                 │
 │  192.168.1.1/24   192.168.2.1/24      192.168.20.1/24         │
 └───┬──────────────────┬───────────────────┬───────────────────┘
     │ VMnet3           │ VMnet4            │ VMnet2
     │ user zone        │ server zone       │ AD zone
     │                  │                   │
 ┌───┴──────────┐  ┌────┴─────────┐   ┌─────┴──────────────────┐
 │ Client-Ubuntu│  │ Client-Servers│   │ DC01  192.168.20.10    │
 │ 192.168.1.100│  │ 192.168.2.100 │   │ (Server 2022, corp.lab)│
 └───┬──────────┘  └───────────────┘   │ DHCP + DNS for AD zone │
     │                                  └────────────────────────┘
 ┌───┴──────────────────┐
 │ CLIENT01 (domain PC)  │   ← moved into LAN zone; authenticates to
 │ 192.168.1.x           │     DC01 across zones via AD rules only
 │ DNS → 192.168.20.10   │
 └───────────────────────┘

 Policy: LAN → DC01 allowed on AD ports only; all other LAN → CORP blocked.
         SERVERS → internet allowed, SERVERS → LAN blocked.
```

---

## Components

| Component | Role | Key specs |
|---|---|---|
| **pfSense CE 2.9.0** | Firewall / router / DHCP server | FreeBSD-based; 1 GB RAM, 1 vCPU, 20 GB disk; **four** network adapters (WAN, LAN, SERVERS, CORP) |
| **Client-Ubuntu** | User-zone test machine (LAN) | Ubuntu Desktop; 4 GB RAM, 2 vCPU; one adapter on VMnet3 |
| **Client-Servers** | Server-zone test machine (SERVERS) | Linked clone of Client-Ubuntu; one adapter on VMnet4 |
| **DC01** | Active Directory domain controller (`corp.lab`) | Windows Server 2022; static `192.168.20.10`; runs DHCP + DNS for the AD zone; in CORP zone (VMnet2) |
| **CLIENT01** | Domain-joined Windows client | Windows 11; moved from CORP into the LAN zone (VMnet3) to test cross-zone authentication |
| **Host** | Virtualization platform | VMware Workstation Pro on Windows (Intel i7, 16 GB RAM) |

---

## Network configuration

| Setting | Value | Notes |
|---|---|---|
| WAN interface | `em0` on VMnet8 (NAT) | Address `192.168.150.128/24`, leased automatically by VMware NAT |
| LAN interface | `em1` on VMnet3 (Host-only) | Static `192.168.1.1/24`, set manually via pfSense console |
| SERVERS interface | `em2` on VMnet4 (Host-only) | Static `192.168.2.1/24`, assigned and configured manually via the GUI |
| CORP interface | `em3` on VMnet2 (Host-only) | Static `192.168.20.1/24`; pfSense acts as gateway for the pre-existing AD subnet |
| DHCP server (LAN) | Enabled | Pool `192.168.1.100 – 192.168.1.200`; DNS handed out = `192.168.20.10` (the DC) |
| DHCP server (SERVERS) | Enabled | Pool `192.168.2.100 – 192.168.2.200` |
| DHCP server (CORP) | **Disabled on pfSense** | The domain controller (`192.168.20.10`) owns DHCP/DNS for the AD zone |
| Domain controller | `192.168.20.10` static, gateway `192.168.20.1` | Server 2022, `corp.lab`; DNS points to itself (correct for a DC) |
| LAN client address | `192.168.1.100` | Leased from pfSense; gateway `192.168.1.1` |
| SERVERS client address | `192.168.2.100` | Leased from pfSense; gateway `192.168.2.1` |
| VMware DHCP on VMnet3 / VMnet4 | **Disabled** | Ensures pfSense is the only DHCP authority on each inside network |

---

## Design decisions

**Why pfSense has two network adapters.** A firewall's entire purpose is to sit *between* two networks and control what crosses. One adapter (WAN) faces the untrusted outside; the other (LAN) faces the trusted inside. With only one, there would be no boundary to police. pfSense holds an address on both networks simultaneously — it belongs to each side.

**Why WAN is NAT and LAN is Host-only.** VMnet8 (NAT) has a path to the internet, which is what an outside/WAN interface should have. VMnet3 (Host-only) is deliberately sealed — it has no internet route of its own. This forces every packet leaving the client to travel *through* pfSense to get out, which is exactly where a firewall does its work. If the inside network could reach the internet directly, the firewall would guard nothing.

**Why pfSense runs DHCP and VMware's is turned off.** The DHCP server must live on the inside, facing the devices that need addresses. VMware's built-in DHCP on VMnet3 was disabled so that pfSense is the single DHCP authority — otherwise two servers would compete and the client could bypass the firewall's addressing. As part of each lease, pfSense also tells the client its default gateway is `192.168.1.1`, which is how the client learns to route through the firewall.

**DHCP client vs. DHCP server — two opposite roles on one box.** On WAN, pfSense is a DHCP *client*: it asks VMware's NAT for an address and receives `192.168.150.128`. On LAN, pfSense is a DHCP *server*: it hands addresses out to internal clients. Receiving from upstream and giving to downstream are separate mechanisms pointing in opposite directions, which is why they never conflict.

---

## Segmentation and firewall rules (Milestone 2)

A second internal zone, **SERVERS** (`192.168.2.0/24`), was added alongside the user **LAN** to demonstrate multi-zone segmentation and least-privilege traffic control.

**Why segment at all — blast radius.** On a single flat network, an attacker who compromises one machine can move freely to every other machine. Splitting machines into zones and blocking traffic between them by default means a foothold in one zone is contained there unless a rule explicitly permits movement. "Default-deny, then allow only what's needed" is the core principle being applied.

**Default-deny is pfSense's built-in posture.** Every interface has an invisible implicit "block all" as its final rule. Traffic only passes if an explicit rule allows it. The original LAN reached the internet because the installer auto-created an "allow LAN to any" rule during setup; a manually added interface (SERVERS) starts with no rules, so it was completely sealed until rules were written — get an address via DHCP, but pass no traffic. This made default-deny directly observable.

**The policy written on the SERVERS interface**, in order:

| # | Action | Source | Destination | Purpose |
|---|---|---|---|---|
| 1 | Block | SERVERS net | LAN net | Isolate the server zone from the user zone |
| 2 | Pass | SERVERS net | any | Allow the server zone to reach the internet |

**Why the order matters.** pfSense evaluates rules top-down, first match wins. The block must sit **above** the allow, because the allow's destination ("any") technically includes LAN. With the block first, `SERVERS → LAN` is caught and dropped before the broad allow can match it, while `SERVERS → internet` falls through to the allow. The readable result: "block the one thing to isolate, allow everything else." Specific denials on top, broad allows below.

**Why rules live on the source interface.** A rule is applied on the interface where traffic *enters* pfSense — i.e. the zone it originates in — not where it is headed. Traffic from the server zone enters through the SERVERS interface, so both rules governing that traffic live on the SERVERS tab, regardless of destination.

---

## Active Directory integration (Milestone 3)

An existing, self-contained Active Directory lab (a Server 2022 domain controller `DC01` + a domain-joined Windows client `CLIENT01`, on their own `192.168.20.0/24` network) was integrated into the segmented network as a third firewalled zone (**CORP**), and cross-zone domain authentication was locked down to least privilege.

**Design choice — integrate without re-addressing the DC.** The AD lab already had its own subnet, DHCP, and DNS bound to `192.168.20.x`, with the DC at `192.168.20.10`. Rather than re-IP the domain controller onto an existing zone (finicky and risky — AD and DNS bind tightly to addresses), pfSense was given a new leg *on the AD's existing subnet* (`192.168.20.1`) and became the gateway that network never had. The DC was left completely unchanged except for adding that gateway. This mirrors real enterprise design — the domain controller keeps DHCP/DNS, the firewall provides routing and segmentation — and demonstrates integrating a working system into a segmented network without breaking it.

**The DC keeps DHCP/DNS; pfSense's DHCP stays off on CORP.** In an AD environment the domain controller is the correct DHCP/DNS authority (DNS is how clients locate the domain). pfSense's DHCP on the CORP interface was therefore left disabled to avoid a second DHCP authority on that subnet.

**The cross-zone scenario.** To make least privilege demonstrable, `CLIENT01` was moved out of the AD zone and into the **LAN** zone. It now gets its address from pfSense's LAN DHCP, but that DHCP scope was configured to hand out the **DC (`192.168.20.10`) as the DNS server** — because a domain client must use the DC for DNS to locate the domain. The client must now cross the firewall into CORP to authenticate, passing through explicit rules.

**Least-privilege rules on the LAN interface** (top to bottom):

| # | Action | Source | Destination | Port(s) | Purpose |
|---|---|---|---|---|---|
| 1 | Pass | LAN net | `192.168.20.10` (DC only) | 53 | DNS — locate the domain |
| 2 | Pass | LAN net | `192.168.20.10` | 88 | Kerberos — authentication |
| 3 | Pass | LAN net | `192.168.20.10` | 389 | LDAP — directory queries |
| 4 | Pass | LAN net | `192.168.20.10` | 445 | SMB — SYSVOL / group policy |
| 5 | Block | LAN net | `192.168.20.0/24` (CORP) | any | Deny all other LAN → AD-zone traffic |
| 6 | Pass | LAN net | any | any | Internet / everything else |

**Why this is least privilege.** The allows target a **single host** (the DC, `/32`) on **only the specific AD service ports** — not the whole CORP network, not all ports. Everything else from LAN into the AD zone is blocked by rule 5. So a user workstation can do exactly what AD requires and nothing more; if another machine were later added to CORP, LAN could not reach it, and even the DC is reachable only on those four ports.

**Stateful-firewall note (real troubleshooting).** After adding the block rule, ping to the DC still succeeded — because pfSense is stateful and an existing connection state from before the rule was still permitting the traffic. Clearing the state table (Diagnostics → States → Reset States) forced new traffic to be re-evaluated against the current rules, and the block then took effect. "Rule added but traffic still flows" is almost always a stale state, not a wrong rule.

---

## Verification

All three checks were run from the Ubuntu client with pfSense running.

| Check | Command / action | Result | What it proves |
|---|---|---|---|
| **1. Addressing** | `ip a` | Client received `192.168.1.100` | pfSense's DHCP server is working and the client is correctly on the inside network |
| **2. Routing** | `ping -c4 google.com` | Replies received | pfSense is routing traffic from LAN out to WAN and back |
| **3. Management** | Browse to `https://192.168.1.1` | pfSense login page reached | Client can reach the firewall's admin interface |

**Note on the certificate warning:** browsing to the pfSense GUI produces a browser warning because pfSense uses a **self-signed certificate** — no trusted Certificate Authority vouches for it, so the browser cannot independently verify its identity and warns rather than silently trusting it. On an isolated lab network where the firewall's identity is known, proceeding past this is safe. On the public internet the same warning would be a reason to stop. Recognizing which situation applies is the relevant judgment.

### Milestone 2 — segmentation policy

Run from **Client-Servers** (`192.168.2.100`), with pfSense and both clients running.

| Check | Command | Result | What it proves |
|---|---|---|---|
| **Internet reachable** | `ping -c4 google.com` | Replies received | The allow rule opened the path; SERVERS routes out to WAN |
| **Cross-zone blocked** | `ping -c4 192.168.1.100` | Timed out / no replies | The block rule holds; the server zone is isolated from the user zone |

The contrast is the point: the same zone can reach the internet yet is walled off from another internal zone — by explicit design, not by default.

### Milestone 3 — cross-zone AD authentication under least privilege

Run from **CLIENT01** (now in the LAN zone, `192.168.1.x`), after clearing the pfSense state table.

| Check | Command | Result | What it proves |
|---|---|---|---|
| **Client config** | `ipconfig /all` | `192.168.1.x`, gateway `192.168.1.1`, DNS `192.168.20.10` | Client is in the LAN zone but uses the DC for DNS across zones |
| **Domain trust** | `nltest /sc_query:corp.lab` | `\\DC01.corp.lab`, `NERR_Success` | Live authentication to the DC succeeds through the AD rules |
| **Everything-else blocked** | `ping 192.168.20.10` | Request timed out | ICMP is not an allowed AD port, so the block rule drops it |

The headline result is the **contrast between the last two rows**: the client can *authenticate* to a domain controller in a separate firewalled zone, but cannot even *ping* it — because only the four AD service ports to that one host are permitted, and nothing else. That is least privilege, enforced across a firewall boundary and demonstrated end to end.

---

## Concepts demonstrated

- Network segmentation using isolated virtual switches
- Firewall dual-homing (WAN/LAN) and the role of a boundary device
- Static interface addressing and subnetting (`/24`)
- DHCP server vs. DHCP client roles
- Default-gateway routing (inside → firewall → outside)
- Host-only vs. NAT virtual networking and their security implications
- Self-signed certificates and the basics of the certificate trust chain
- Multi-zone network segmentation and zone isolation
- Least-privilege and default-deny firewall design
- Stateful firewall rule evaluation (top-down, first-match-wins) and rule ordering
- Filtering traffic at the source interface (point of entry)
- Integrating an existing Active Directory environment into a segmented network without re-addressing it
- Host-based least privilege (allow to a single host `/32` on specific service ports only)
- Core Active Directory service ports (DNS, Kerberos, LDAP, SMB) and cross-zone domain authentication
- DHCP-provided DNS to keep a relocated domain client pointed at its DC
- Stateful-firewall behaviour and clearing the state table so new rules take effect

---

## Next steps

- Extend isolation to the SERVERS zone (block SERVERS ↔ CORP/LAN as appropriate) for full multi-zone least privilege
- Introduce an IDS/IPS (e.g. Suricata) on the network and forward logs to a SIEM (Splunk) for monitoring and detection
- Simulate an attack from a Kali host and map the resulting detections to MITRE ATT&CK

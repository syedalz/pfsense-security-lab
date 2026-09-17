# pfSense Firewalled Lab Network

A virtualized network built to demonstrate core firewall, routing, and network-segmentation fundamentals. A pfSense firewall sits between an untrusted "outside" network and a sealed "inside" network, controlling all traffic between them. A client machine lives behind the firewall and reaches the internet only by routing through it.

**Status:** Milestone 2 complete — second internal zone added, segmentation policy written and verified (zone reaches the internet but is firewalled off from the user zone).

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
   ┌──┴───────────────────────────────────────────┐
   │  WAN  em0  192.168.150.128/24  (leased by NAT)│
   │                                               │
   │                   pfSense                     │
   │          firewall / router / DHCP             │
   │                                               │
   │  LAN  em1            SERVERS  em2              │
   │  192.168.1.1/24      192.168.2.1/24           │
   └───┬──────────────────────┬────────────────────┘
       │ VMnet3 (Host-only)   │ VMnet4 (Host-only)
       │ user zone            │ server zone
       │                      │
 ┌─────┴──────────┐    ┌──────┴───────────┐
 │ Client-Ubuntu  │    │ Client-Servers   │
 │ 192.168.1.100  │    │ 192.168.2.100    │
 │ gw 192.168.1.1 │    │ gw 192.168.2.1   │
 └────────────────┘    └──────────────────┘

 Policy: SERVERS → internet ALLOWED,  SERVERS → LAN BLOCKED (zone isolation)
```

---

## Components

| Component | Role | Key specs |
|---|---|---|
| **pfSense CE 2.9.0** | Firewall / router / DHCP server | FreeBSD-based; 1 GB RAM, 1 vCPU, 20 GB disk; **three** network adapters (WAN, LAN, SERVERS) |
| **Client-Ubuntu** | User-zone test machine (LAN) | Ubuntu Desktop; 4 GB RAM, 2 vCPU; one adapter on VMnet3 |
| **Client-Servers** | Server-zone test machine (SERVERS) | Linked clone of Client-Ubuntu; one adapter on VMnet4 |
| **Host** | Virtualization platform | VMware Workstation Pro on Windows (Intel i7, 16 GB RAM) |

---

## Network configuration

| Setting | Value | Notes |
|---|---|---|
| WAN interface | `em0` on VMnet8 (NAT) | Address `192.168.150.128/24`, leased automatically by VMware NAT |
| LAN interface | `em1` on VMnet3 (Host-only) | Static `192.168.1.1/24`, set manually via pfSense console |
| SERVERS interface | `em2` on VMnet4 (Host-only) | Static `192.168.2.1/24`, assigned and configured manually via the GUI |
| DHCP server (LAN) | Enabled | Pool `192.168.1.100 – 192.168.1.200` |
| DHCP server (SERVERS) | Enabled | Pool `192.168.2.100 – 192.168.2.200` |
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

---

## Next steps

- Tighten the LAN side and add a single least-privilege exception (allow LAN to one specific service on one host in SERVERS, deny the rest)
- Bring the Active Directory domain controller into the SERVERS zone and validate segmented authentication traffic
- Introduce an IDS/IPS (e.g. Suricata) and forward logs to a SIEM (Splunk) for monitoring and detection
- Simulate an attack from a Kali host and map the resulting detections to MITRE ATT&CK

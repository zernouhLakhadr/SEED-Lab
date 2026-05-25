# BGP Attack Lab

<div align="center">

![Lab](https://img.shields.io/badge/SEED%20Lab-BGP%20Attacks-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Tools](https://img.shields.io/badge/Tools-BIRD%20%7C%20SEED%20Emulator%20%7C%20Wireshark-blue?style=for-the-badge)

</div>

**Student:** Zernouh Lakhdar  
**University:** Amar Telidji University, Laghouat, Algeria  
**Lab Series:** SEED Labs — Syracuse University  

---

## 🎯 Objective

This lab explores the security vulnerabilities of the **Border Gateway Protocol (BGP)** — the routing protocol that governs how traffic flows across the internet. Through a simulated internet environment (SEED Internet Emulator), this lab demonstrates **BGP prefix hijacking**, where a malicious Autonomous System (AS) announces ownership of IP prefixes it does not legitimately control. The lab also covers BGP path selection, route manipulation, Anycast routing, and examines countermeasures such as RPKI.

---

## 🛠️ Environment Setup

| Component | Details |
|-----------|---------|
| **Platform** | SEED Internet Emulator (Docker-based virtual internet) |
| **Host OS** | Ubuntu 20.04 / Kali Linux |
| **Routing Daemon** | BIRD 2.0 (Internet Routing Daemon) |
| **Topology** | Multiple ASes with iBGP/eBGP sessions, simulated IXPs |
| **Tools** | Docker, Docker Compose, BIRD CLI (`birdc`), Wireshark, tcpdump |
| **Emulator Source** | https://github.com/seed-labs/seed-emulator |

---

## ✅ Tasks Completed

- ✅ **Task 1** — Deploy the SEED Internet Emulator and explore the simulated topology
- ✅ **Task 2** — Inspect BGP routing tables using `birdc show route` and `birdc show protocols`
- ✅ **Task 3** — Understand BGP path selection (AS-PATH, LOCAL-PREF, MED attributes)
- ✅ **Task 4** — Launch **BGP prefix hijacking** — announce a victim AS's prefix from a rogue AS
- ✅ **Task 5** — Observe traffic redirection in Wireshark and routing table changes
- ✅ **Task 6** — Perform **more-specific prefix hijacking** (longest-prefix-match wins)
- ✅ **Task 7** — Study **Anycast routing** and BGP's role in load distribution
- ✅ **Task 8** — Review **RPKI** (Resource Public Key Infrastructure) as a hijacking countermeasure

---

## 🧠 Key Concepts Covered

- **BGP fundamentals** — eBGP (between ASes), iBGP (within an AS), peer/transit/customer relationships
- **BGP path attributes** — AS-PATH, NEXT-HOP, LOCAL-PREF, MED, COMMUNITY
- **BGP route selection** — longest prefix match, shortest AS-PATH, lowest MED
- **Prefix hijacking** — announcing a prefix belonging to another AS
- **Sub-prefix hijacking** — announcing a more-specific prefix to override legitimate routes
- **Traffic blackholing** — hijacking traffic and dropping it (DoS via routing)
- **Traffic interception** — hijacking and re-routing traffic through attacker AS (MITM at internet scale)
- **Anycast** — multiple ASes announcing the same prefix for distributed routing
- **RPKI** — cryptographic validation of prefix-to-AS ownership using Route Origin Authorizations (ROAs)
- **BGP filtering** — prefix lists, AS-PATH filters, max-prefix limits

---

## 📊 Key Findings

| Attack | Technique | Outcome | Countermeasure |
|--------|-----------|---------|---------------|
| Prefix Hijacking | Rogue AS announces victim's /16 | ✅ Some routers prefer rogue route | RPKI ROA validation |
| Sub-prefix Hijacking | Rogue AS announces /24 within victim's /16 | ✅ All traffic hijacked (longest match) | RPKI + IRR filtering |
| Traffic Blackholing | Hijack + drop all packets | ✅ Effective internet-scale DoS | RPKI, monitoring |
| Traffic Interception | Hijack + re-route through rogue AS | ✅ MITM at BGP level | RPKI, BGPSec |
| With RPKI Validation | ROA exists for victim prefix | ❌ Rogue announcement rejected as INVALID | Widespread RPKI deployment |

**Key Takeaway:** BGP was designed for trust among cooperating ISPs — it has no built-in authentication. Any AS can announce any prefix. Sub-prefix hijacking is particularly powerful because it exploits BGP's longest-prefix-match rule, overriding even legitimate more-general routes. RPKI provides cryptographic origin validation and is the primary deployed countermeasure, though global adoption is still incomplete.

---

## 💻 Code & Scripts

Configuration files and scripts are in the [`code/`](./code/) folder.

### BIRD BGP Configuration — Legitimate AS
```
# /etc/bird/bird.conf — AS 100 (legitimate owner of 10.100.0.0/16)
router id 10.0.0.1;

protocol bgp upstream {
    local as 100;
    neighbor 10.0.0.254 as 200;
    ipv4 {
        import all;
        export where net ~ [10.100.0.0/16];
    };
}

protocol static owned_prefix {
    ipv4;
    route 10.100.0.0/16 blackhole;
}
```

### BIRD BGP Configuration — Rogue AS (Prefix Hijack)
```
# /etc/bird/bird.conf — AS 666 (rogue, hijacking 10.100.0.0/16)
router id 10.6.6.6;

protocol bgp upstream {
    local as 666;
    neighbor 10.0.0.254 as 200;
    ipv4 {
        import all;
        export where net ~ [10.100.0.0/16+];  # Announce hijacked prefix
    };
}

protocol static hijacked {
    ipv4;
    route 10.100.0.0/16 blackhole;    # Exact hijack
    route 10.100.1.0/24 blackhole;    # Sub-prefix hijack (more specific)
}
```

### BIRD CLI — Inspect Routing Tables
```bash
# Connect to BIRD daemon
sudo birdc

# Show all BGP sessions
show protocols all

# Show routing table
show route

# Show route for specific prefix
show route for 10.100.0.0/16

# Show route with BGP attributes (AS-PATH, etc.)
show route for 10.100.0.0/16 all

# Filter routes learned from specific AS
show route where bgp_path ~ [= * 666 * =]
```

### Deploy SEED Internet Emulator
```bash
# Clone the SEED emulator
git clone https://github.com/seed-labs/seed-emulator.git
cd seed-emulator

# Build and launch the simulated internet
docker-compose build
docker-compose up -d

# List running AS containers
docker ps --format "table {{.Names}}\t{{.Status}}"

# Enter a router container
docker exec -it as100-router bash

# Capture inter-AS BGP traffic
docker exec as200-router tcpdump -i any port 179 -w /tmp/bgp.pcap
```

---

## 🖼️ Screenshots

Screenshots are in the [`screenshots/`](./screenshots/) folder:

| File | Description |
|------|-------------|
| `01-emulator-topology.png` | SEED Internet Emulator network topology diagram |
| `02-bgp-sessions.png` | `birdc show protocols` — all BGP sessions established |
| `03-legitimate-routes.png` | Routing table before attack — legitimate prefixes |
| `04-hijack-config.png` | Rogue AS BIRD config announcing hijacked prefix |
| `05-routes-after-hijack.png` | Routing table showing rogue AS route propagated |
| `06-wireshark-bgp-update.png` | Wireshark BGP UPDATE message with hijacked prefix |
| `07-traffic-redirected.png` | traceroute showing traffic going through rogue AS |
| `08-subprefix-hijack.png` | More-specific /24 completely overrides victim's /16 |
| `09-anycast-routing.png` | Multiple ASes announcing same prefix — closest wins |
| `10-rpki-invalid.png` | RPKI marking rogue announcement as INVALID |
| `11-rpki-block.png` | Route filtered after RPKI validation failure |

---

## 📄 References

- **Lab Manual:** [`lab-instructions.pdf`](./lab-instructions.pdf) — included in this folder
- **SEED Labs:** https://seedsecuritylabs.org/Labs_20.04/Networking/BGP/
- **SEED Internet Emulator:** https://github.com/seed-labs/seed-emulator
- **RFC 4271** — A Border Gateway Protocol 4 (BGP-4)
- **RFC 6811** — BGP Prefix Origin Validation (RPKI)
- **RIPE NCC RPKI Dashboard:** https://rpki-monitor.antd.nist.gov/

---

> 🔬 *This lab is part of the [SEED Labs](https://seedsecuritylabs.org/) series by Syracuse University. All experiments conducted in isolated virtual environments for educational purposes only.*

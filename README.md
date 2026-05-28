# SEED-Lab
# 🔐 SEEDLabs – Cybersecurity Hands-on Labs

<div align="center">

![Security](https://img.shields.io/badge/Category-Network%20Security-red?style=for-the-badge&logo=shield)
![Status](https://img.shields.io/badge/Labs-4%20Completed-brightgreen?style=for-the-badge)
![University](https://img.shields.io/badge/University-Amar%20Telidji-blue?style=for-the-badge)

</div>

---

**Student:** Zernouh Lakhdar  
**Program:** Cyber Security Engineering  
**University:** Amar Telidji University, Laghouat, Algeria  

This repository contains my completed **SEED Lab** exercises — a practical cybersecurity curriculum developed by **Professor Wenliang (Kevin) Du** at Syracuse University. Each lab demonstrates real-world attack techniques, defense mechanisms, and network protocol vulnerabilities through hands-on experimentation in controlled, isolated environments.

> ⚠️ All work is performed strictly for **educational purposes** in virtualized/sandboxed lab environments. No production systems were targeted or affected.

---

## 📋 Lab Overview

| # | Lab Name | Topic | Tools Used | Status |
|---|----------|-------|------------|--------|
| 1 | [ARP Cache Poisoning Attack Lab](./ARP-Cache-Poisoning/) | ARP spoofing, MITM, HTTPS interception | Scapy, Wireshark, OpenSSL, Apache | ✅ Completed |
| 2 | [TCP Attack Lab](./TCP-Attack/) | SYN flood, TCP RST, session hijacking | Scapy, Python, C, Wireshark | ✅ Completed |
| 3 | [Mitnick_Attack](./Mitnick_Attack/) | DNS cache poisoning, spoofing, rebinding | Scapy, BIND9, dig, Wireshark | ✅ Completed |
| 4 | [BGP Attack Lab](./BGP-Attack/) | BGP prefix hijacking, route manipulation | SEED Internet Emulator, BIRD, Wireshark | ✅ Completed |


---

## 🗂️ Repository Structure

```
SEEDLabs/
│
├── ARP-Cache-Poisoning/          # ARP spoofing + HTTPS MITM attack
│   ├── README.md                 # Lab writeup and findings
│   ├── lab-instructions.pdf      # Original SEED Lab manual                
│   └── report.pdf              # the report that anwer the seedlab instructeur
│
├── TCP-Attack/                   # SYN flood, RST attack, session hijack
│   ├── README.md
│   ├── lab-instructions.pdf
│   └── report.pdf
│
├── Mitnick_Attack/                   # DNS cache poisoning & spoofing
│   ├── README.md
│   ├── lab-instructions.pdf
│   └── report.pdf
│
├── BGP-Attack/                   # BGP prefix hijacking & route manipulation
│   ├── README.md
│   ├── lab-instructions.pdf
│   └── report.pdf
│
└── README.md                     # This file
```

Each lab folder contains:
- **`README.md`** — Detailed writeup (objective, tasks, findings, analysis)
- **`lab-instructions.pdf`** — Original SEED Lab PDF manual
- **`code/`** — All scripts written during the lab (Python, Scapy, Bash, C)
- **`screenshots/`** — Terminal captures, Wireshark screenshots, proof of execution

---

## 🖥️ Environment & Tools

All labs were executed in virtualized or containerized environments:

| Category | Tools / Technologies |
|----------|---------------------|
| **Operating Systems** | Kali Linux, Ubuntu 20.04 / 22.04 |
| **Virtualization** | VirtualBox (Internal Networking), Docker |
| **Network Emulation** | SEED Internet Emulator (Docker-based), GNS3 |
| **Packet Analysis** | Wireshark, tcpdump |
| **Packet Crafting** | Scapy (Python 3) |
| **DNS / Routing** | BIND9, BIRD Internet Routing Daemon |
| **Web / PKI** | Apache2, OpenSSL, curl |
| **Development** | Python 3, C, Bash |

---

## 📚 About SEED Labs

The **SEED** (Security EDucation) project is an open-source, NSF-funded initiative providing hands-on cybersecurity labs for students and educators worldwide. Labs are designed to bridge the gap between theory and practice — requiring students to write attack code, analyze network traffic, understand protocol weaknesses, and apply countermeasures.

For original lab materials and documentation, visit: **https://seedsecuritylabs.org/**

---

## 📬 Contact

**Zernouh Lakhdar** — Cyber Security Engineering Student

[![GitHub](https://img.shields.io/badge/GitHub-zernouh--lakhdar-181717?style=flat-square&logo=github)](https://github.com/zernouhLakhadr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-zernouh--lakhdar-0A66C2?style=flat-square&logo=linkedin)](www.linkedin.com/in/aymen-zernouh-1b690925b)

---

<div align="center">

*Part of the [SEED Labs](https://seedsecuritylabs.org/) series by Syracuse University*  
*Amar Telidji University · Laghouat, Algeria · Cyber Security Engineering*

</div>

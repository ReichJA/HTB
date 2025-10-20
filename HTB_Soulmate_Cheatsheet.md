# HTB — Expressway HackNez

**Metadaten**

```text
Target: 10.10.11.85
Name: HTB - Expressway
Datum: 2025-10-13
Autor: J0rg3
```

---

## Inhaltsverzeichnis


---

# Server Status

┌──(kali㉿kali)-[~]
└─$ ping 10.10.11.85 -c1 
PING 10.10.11.85 (10.10.11.85) 56(84) bytes of data.
64 bytes from 10.10.11.85: icmp_seq=1 ttl=63 time=34.9 ms

--- 10.10.11.85 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 34.941/34.941/34.941/0.000 ms


---

# Nmap-Scan (TCP / UDP)

## TCP Scan 

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    syn-ack ttl 63 nginx 1.22.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
---

22/tcp — SSH: OpenSSH 9.2p1 (Debian package 2+deb12u7), Protokoll 2.0.
80/tcp — HTTP: nginx 1.22.1.

## UDP Scan 

```bash
└─$ sudo nmap -sU -p- -T4 10.10.11.85 -vv --top-ports 25
[sudo] Passwort für kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-13 20:33 CEST
Initiating Ping Scan at 20:33
Scanning 10.10.11.85 [4 ports]
Completed Ping Scan at 20:33, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 20:33
Completed Parallel DNS resolution of 1 host. at 20:33, 0.00s elapsed
Initiating UDP Scan at 20:33
Scanning 10.10.11.85 [25 ports]
Completed UDP Scan at 20:33, 8.76s elapsed (25 total ports)
Nmap scan report for 10.10.11.85
Host is up, received echo-reply ttl 63 (0.039s latency).
Scanned at 2025-10-13 20:33:23 CEST for 9s

PORT      STATE         SERVICE      REASON
53/udp    closed        domain       port-unreach ttl 63
67/udp    closed        dhcps        port-unreach ttl 63
68/udp    open|filtered dhcpc        no-response
69/udp    open|filtered tftp         no-response
111/udp   closed        rpcbind      port-unreach ttl 63
123/udp   closed        ntp          port-unreach ttl 63
135/udp   closed        msrpc        port-unreach ttl 63
137/udp   open|filtered netbios-ns   no-response
138/udp   open|filtered netbios-dgm  no-response
139/udp   closed        netbios-ssn  port-unreach ttl 63
161/udp   open|filtered snmp         no-response
162/udp   open|filtered snmptrap     no-response
445/udp   closed        microsoft-ds port-unreach ttl 63
500/udp   closed        isakmp       port-unreach ttl 63
514/udp   open|filtered syslog       no-response
520/udp   closed        route        port-unreach ttl 63
631/udp   closed        ipp          port-unreach ttl 63
998/udp   open|filtered puparp       no-response
1434/udp  open|filtered ms-sql-m     no-response
1701/udp  closed        L2TP         port-unreach ttl 63
1900/udp  closed        upnp         port-unreach ttl 63
4500/udp  closed        nat-t-ike    port-unreach ttl 63
5353/udp  open|filtered zeroconf     no-response
49152/udp open|filtered unknown      no-response
49154/udp open|filtered unknown      no-response
```


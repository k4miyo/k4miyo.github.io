---
title: Try Hack Me Roasted
author: k4miyo
date: 2023-06-28
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Windows]
tags: [Web, SSH]
ping: true
---

## Máquina Roasted
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.124.121.

```bash
❯ ping -c 1 10.10.124.121
PING 10.10.124.121 (10.10.124.121) 56(84) bytes of data.
64 bytes from 10.10.124.121: icmp_seq=1 ttl=127 time=218 ms

--- 10.10.124.121 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 217.890/217.890/217.890/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.124.121 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-28 19:42 CST
Initiating Parallel DNS resolution of 1 host. at 19:42
Completed Parallel DNS resolution of 1 host. at 19:42, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 19:42
Scanning 10.10.124.121 [65535 ports]
Discovered open port 53/tcp on 10.10.124.121
Discovered open port 139/tcp on 10.10.124.121
Discovered open port 445/tcp on 10.10.124.121
Discovered open port 135/tcp on 10.10.124.121
Discovered open port 49668/tcp on 10.10.124.121
Discovered open port 593/tcp on 10.10.124.121
Discovered open port 464/tcp on 10.10.124.121
Discovered open port 5985/tcp on 10.10.124.121
Discovered open port 88/tcp on 10.10.124.121
Discovered open port 49670/tcp on 10.10.124.121
Discovered open port 49673/tcp on 10.10.124.121
Discovered open port 49669/tcp on 10.10.124.121
Discovered open port 3269/tcp on 10.10.124.121
Discovered open port 3268/tcp on 10.10.124.121
Discovered open port 636/tcp on 10.10.124.121
Discovered open port 389/tcp on 10.10.124.121
Discovered open port 49690/tcp on 10.10.124.121
Discovered open port 49665/tcp on 10.10.124.121
Completed SYN Stealth Scan at 19:42, 26.54s elapsed (65535 total ports)
Nmap scan report for 10.10.124.121
Host is up, received user-set (0.18s latency).
Scanned at 2023-06-28 19:42:32 CST for 27s
Not shown: 65517 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
49665/tcp open  unknown          syn-ack ttl 127
49668/tcp open  unknown          syn-ack ttl 127
49669/tcp open  unknown          syn-ack ttl 127
49670/tcp open  unknown          syn-ack ttl 127
49673/tcp open  unknown          syn-ack ttl 127
49690/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.66 seconds
           Raw packets sent: 131065 (5.767MB) | Rcvd: 31 (1.364KB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.124.121
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,3268,3269,5985,49665,49668,49669,49670,49673,49690
   6   │ 
   7   │ [*] Ports copied to clipboard
   8   │ 
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,49665,49668,49669,49670,49673,49690 10.10.124.121 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-28 19:43 CST
Nmap scan report for 10.10.124.121
Host is up (0.17s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-06-29 01:44:00Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49665/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: WIN-2BO8M1OE1M1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-06-29T01:45:01
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 108.51 seconds
```


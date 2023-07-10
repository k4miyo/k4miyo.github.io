---
title: Hack The Box Timelapse
author: k4miyo
date: 2023-07-06
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Network, Active Directory, Common Services, SMB, LAPS, Powershell, Reconnaissance, Password Cracking, Group Membership, Misconfiguration, Anonymous/Guest Access]
ping: true
---

## Máquina Timelapse
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.11.152.

```bash
❯ ping -c 1 10.10.11.152
PING 10.10.11.152 (10.10.11.152) 56(84) bytes of data.
64 bytes from 10.10.11.152: icmp_seq=1 ttl=127 time=161 ms

--- 10.10.11.152 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 160.762/160.762/160.762/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.11.152 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-06 20:20 CST
Initiating Parallel DNS resolution of 1 host. at 20:20
Completed Parallel DNS resolution of 1 host. at 20:20, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 20:20
Scanning 10.10.11.152 [65535 ports]
Discovered open port 139/tcp on 10.10.11.152
Discovered open port 135/tcp on 10.10.11.152
Discovered open port 445/tcp on 10.10.11.152
Discovered open port 53/tcp on 10.10.11.152
Discovered open port 389/tcp on 10.10.11.152
Discovered open port 593/tcp on 10.10.11.152
Discovered open port 5986/tcp on 10.10.11.152
Discovered open port 49674/tcp on 10.10.11.152
Discovered open port 49673/tcp on 10.10.11.152
Discovered open port 49696/tcp on 10.10.11.152
Discovered open port 49667/tcp on 10.10.11.152
Discovered open port 464/tcp on 10.10.11.152
Discovered open port 55723/tcp on 10.10.11.152
Discovered open port 9389/tcp on 10.10.11.152
Discovered open port 3269/tcp on 10.10.11.152
Discovered open port 3268/tcp on 10.10.11.152
Discovered open port 88/tcp on 10.10.11.152
Discovered open port 636/tcp on 10.10.11.152
Completed SYN Stealth Scan at 20:20, 26.46s elapsed (65535 total ports)
Nmap scan report for 10.10.11.152
Host is up, received user-set (0.14s latency).
Scanned at 2023-07-06 20:20:00 CST for 27s
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
5986/tcp  open  wsmans           syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49673/tcp open  unknown          syn-ack ttl 127
49674/tcp open  unknown          syn-ack ttl 127
49696/tcp open  unknown          syn-ack ttl 127
55723/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.57 seconds
           Raw packets sent: 131064 (5.767MB) | Rcvd: 30 (1.320KB)
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
   4   │     [*] IP Address: 10.10.11.152
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49696,55723
   6   │ 
   7   │ [*] Ports copied to clipboard
   8   │ 
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49696,55723 10.10.11.152 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-06 20:22 CST
Nmap scan report for 10.10.11.152
Host is up (0.15s latency).

PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2023-07-07 10:22:20Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
|_http-server-header: Microsoft-HTTPAPI/2.0
|_ssl-date: 2023-07-07T10:23:53+00:00; +7h59m59s from scanner time.
|_http-title: Not Found
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc             Microsoft Windows RPC
49696/tcp open  msrpc             Microsoft Windows RPC
55723/tcp open  msrpc             Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m58s, deviation: 0s, median: 7h59m58s
| smb2-time: 
|   date: 2023-07-07T10:23:12
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 102.41 seconds
```

Vemos varios puertos abiertos, vamos a tratar de ver que recursos podemos observar bajo el servicio SMB con una ***null session***:

```bash
❯ smbclient -L //10.10.11.152/ -N

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Shares          Disk      
	SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available

```

Podemos acceder al recurso `Shares`:

```bash
❯ smbclient //10.10.11.152/Shares -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Oct 25 10:39:15 2021
  ..                                  D        0  Mon Oct 25 10:39:15 2021
  Dev                                 D        0  Mon Oct 25 14:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 10:48:42 2021

		6367231 blocks of size 4096. 1255166 blocks available
smb: \>
```

Para trabajar más cómodos, vamos a crearnos una montura del recurso compartido.

```bash
❯ mkdir /mnt/timelapse
❯ mount -t cifs //10.10.11.152/Shares /mnt/timelapse -o username='null',password='',rw
❯ cd /mnt/timelapse
❯ dir
Dev  HelpDesk
❯ tree
.
├── Dev
│   └── winrm_backup.zip
└── HelpDesk
    ├── LAPS_Datasheet.docx
    ├── LAPS_OperationsGuide.docx
    ├── LAPS_TechnicalSpecification.docx
    └── LAPS.x64.msi

2 directories, 5 files
```

Vemos algunos recursos, vamos a llevarlos a nuestra máquina de atacante para echarles un ojo:

```bash
❯ cp -r * /home/k4miyo/Documents/HackTheBox/Timelapse/content/.
❯ cd !$
cd /home/k4miyo/Documents/HackTheBox/Timelapse/content/.
❯ umount /mnt/timelapse
❯ ll
drwxr-xr-x root root  32 B Thu Jul  6 20:48:48 2023  Dev
drwxr-xr-x root root 176 B Thu Jul  6 20:48:53 2023  HelpDesk
```


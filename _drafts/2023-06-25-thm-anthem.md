---
title: Try Hack Me Anthem
author: k4miyo
date: 2023-06-25)
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Windows]
tags: [Web, SSH]
ping: true
---

## Máquina Anthem
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP 10.10.199.139.

```bash
❯ ping -c 1 10.10.199.139
PING 10.10.199.139 (10.10.199.139) 56(84) bytes of data.

--- 10.10.199.139 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

No nos responde el `ping`, sin embargo, de acuerdo con el autor de la máquina, presenta un sistema operativo  Windows. A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash
❯ nmap -p- --open -sS --min-rate 5000 -vvv -Pn 10.10.199.139 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-25 16:11 CST
Initiating Parallel DNS resolution of 1 host. at 16:11
Completed Parallel DNS resolution of 1 host. at 16:11, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 16:11
Scanning 10.10.199.139 [65535 ports]
Discovered open port 80/tcp on 10.10.199.139
Discovered open port 3389/tcp on 10.10.199.139
Completed SYN Stealth Scan at 16:12, 26.50s elapsed (65535 total ports)
Nmap scan report for 10.10.199.139
Host is up, received user-set (0.15s latency).
Scanned at 2023-06-25 16:11:55 CST for 27s
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE       REASON
80/tcp   open  http          syn-ack ttl 127
3389/tcp open  ms-wbt-server syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.60 seconds
           Raw packets sent: 131086 (5.768MB) | Rcvd: 20 (880B)
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
   4   │     [*] IP Address: 10.10.199.139
   5   │     [*] Open ports: 80,3389
   6   │ 
   7   │ [*] Ports copied to clipboard
```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash

```


---
title: Función extractPorts
author: k4miyo
date: 2021-09-06
math: true
mermaid: true
image:
  path: /assets/images/htb/hackthebox.png
categories: [Scripting, Tools]
tags: [Bash]
ping: true
---

## extractPorts

Herramienta desarrollada por [s4vitar](https://s4vitar.github.io/). Función a nivel de `zsh` o `bashrc` para extraer los puertos de un archivo grepeable, como una salida -oG de nmap; así como también representar la información más relevante de la captura.

[extractPorts](https://pastebin.com/X6b56TQ8)

```bash
# Extract nmap information
function extractPorts(){
	ports="$(cat $1 | grep -oP '\d{1,5}/open' | awk '{print $1}' FS='/' | xargs | tr ' ' ',')"
	ip_address="$(cat $1 | grep -oP '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}' | sort -u | head -n 1)"
	echo -e "\n[*] Extracting information...\n" > extractPorts.tmp
	echo -e "\t[*] IP Address: $ip_address"  >> extractPorts.tmp
	echo -e "\t[*] Open ports: $ports\n"  >> extractPorts.tmp
	echo $ports | tr -d '\n' | xclip -sel clip
	echo -e "[*] Ports copied to clipboard\n"  >> extractPorts.tmp
	cat extractPorts.tmp; rm extractPorts.tmp
}
```
Para su ejecución es necesario instalar `xclip`:

```bash
sudo apt-get install xclip
```

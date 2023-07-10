---
title: (Plataforma) (Máquina)
author: k4miyo
date: 2023-07-09
math: true
mermaid: true
image:
  path: /assets/images/thm/tryhackme.png
categories: [Easy, Linux]
tags: [Web, SSH]
ping: true
---

## Máquina (Máquina)
Se procede con la fase de reconocimiento lanzando primeramente un `ping` a la dirección IP (ip address).

```bash

```

De acuerdo con el TTL de traza ICMP, se puede determinar que se trata de una máquina con sistema operativo (sistema operativo). A continuación se procede con la ejecución de `nmap` para determinar los puertos abiertos de la máquina y exportanto la información al archivo **allPorts**.

```bash

```

Mediante la función [extractPorts](/posts/extractPorts) definida a nivel de `zsh` , se obtiene la información más relevante de la captura grepeable.

```bash

```

A continuación se lanza una serie de scripts para determinar el servicio y versión que corren para los puertos detectados.

```bash

```


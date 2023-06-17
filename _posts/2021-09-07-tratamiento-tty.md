---
title: Tratamiento de la TTY
author: k4miyo
date: 2021-09-07
math: true
mermaid: true
image: 
  path: /assets/images/bash/bash.jpg
categories: [Easy, Linux]
tags: [Terminal,TTY,Interactive]
ping: true
---

## Tratamiento de la TTY

Ejecutamos los siguientes comandos cuando estemos dentro de un sistema linux a través de una reverse shell:

```bash
bash-3.2$ script /dev/null -c bash
```

Presionamos `Ctrl + z` y ejecutamos los siguientes comandos en la terminal de nuestro equipo de atacante:

```bash
 stty raw -echo; fg
[1]  + continued  nc -nlvp 443
```

Ahora escribimos la palabra **reset**, aunque no se muestre. Es posible que en algunos casos nos indique el tipo de terminal que deseamos utilizar, para este caso le indicaremos **xterm**. Posterior modificamos el valor de las siguientes variables de entorno: 

```bash
bash-3.2$ export TERM=xterm
bash-3.2$ export SHELL=bash
```

Por último, modificamos las proporciones la *stty* de acuerdo a los valores de nuestra terminal. Ingresamos el siguiente comando en nuestro equipo:

```bash
❯ stty size
46 131
```
Aqui vemos que para este caso tenemos 46 renglones y 131 columnas; por lo tanto asignamos dichos valores a la tty de la máquina víctima:

```bash
bash-3.2$ stty rows 46 columns 131
```

Ya tenemos una tty totalmente interactiva, podemos hacer `Ctrl + c`, `Ctrl + l` y desplazarnos sin problemas y sin temor que se nos muera la conexión.

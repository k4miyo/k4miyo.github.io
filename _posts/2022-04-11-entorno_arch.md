---
title: Entorno Arch Linux
author: k4miyo
date: 2022-04-11
math: true
mermaid: true
image:
  path: /assets/images/entorno/entorno.jpg
categories: [Configuration, ArchLinux]
tags: [ArchLinux, Configuration, ZSH, Kitty, AwesomeWM, AUR, BlackArch, locate, lsd, bat, scrub, NeoVim, NerdFonts, Copy/Paste VMware]
ping: true
---

# Entorno Arch Linux

A continuación se presenta la configuración de un entorno de hacking en **Arch Linux** de acuerdo con el video de [YouTube de S4vitar](https://www.youtube.com/watch?v=fshLf6u8B-w). Primero, comenzaremos con al descarga de la ISO de [**Arch Linux**](https://archlinux.org/download/).

![""](/assets/images/entorno_arch/arch1.png)

## Configuración de la máquina virtual

Una vez descargada la ISO, vamos a proceder a crear una máquina vírtual con la herramienta VMware WorkStation en la opción **File > New Virtual Machine...** y de la ventanita seleccionamos las siguientes opciones:

- Seleccionamos *Typical (recommended)*
- Seleccionamos *Installer disc image file (iso)* y colocamos el archivo ISO que descargamos.
- Seleccionamos *Linux* y de versión *Debian 10.x 64 bit* (en caso de que nuestra máquina sea de x64).
- Colocamos un nombre que queramos y la ruta donde se guardará nuestra máquina virtual.
- Ahora el tamaño del disco seleccionamos el que nosotros queramos con un mínimo de 20 GB y la opción *Store virtual disk as a single file*.
- Seleccionamos *Customize Hardware...* y la parte de *Network Adapter* ponemos la opción *Bridged: Connected directly to the physical network*.
- Seleccionamos *Memory* y le asignamos 4 GB y en *¨Processors* colocamos 2.

Ejecutamos la máquina virutal y seleccionamos *Arch Linux install medium (x86_64, BIOS)*, la primera opción.

![""](/assets/images/entorno_arch/arch2.png)

Al terminar el proceso, nos arroja una linea de comandos como **root** en la cual primero vamos a validar si tenemos conexión a internet con una traza ICMP (**Nota**: El teclado lo tendremos en configuración en-US, por lo que si lo queremos en español utilizaremos el comando `loadkeys es`):

```bash
root@archiso ~ # ping -c 1 8.8.8.8
```

## Configuracón de las particiones

Ahora escribimos el comando `cfdisk` y nos arrojará un panel de opción; como nos encontramos en máquina virtual, seleccioamos **dos**. Nos toca crear las particiones de nuestro sistema, por lo tanto, en la parte de abajo seleccionamos **New** y

- La primera partición le asignaremos un espacio de **512M** y seleccionamos **primary**.
- Nos colocamos en **Free space** y la segunda partición le asignaremos un espacio de **15G**  y seleccionamos **primary**.
- Nos colocamos nuevamente en **Free space** y la tercera partición le asignaremos el resto (**4.5G**) y seleccionamos **primary**.
- Ahora si seleccionamos `/dev/sda3` (la partición de 4.5G), en la opción de bajo eligimos **Type** y ponemos **82 Linux swap / Solaris**
- Por último, del menú de abajo le damos a **Write** y confirmamos y posteriormente le damos a **Quit**.

Con el comando `lsblk` podemos ver las particiones que hemos creado.

![""](/assets/images/entorno_arch/arch3.png)

Ahora vamos a darle formato a las particiones que tenemos con `mkfs.vfat -F 32 /dev/sda1` para la partición de 512M, `mkfs.ext4 /dev/sda2` para la partición de 15G y `mkswap /dev/sda3` para la partición de 4.5G y aplicamos los cambios `swapon`.

Posterior a esto, vamos a montar lo que hay en `/dev/sda2` en `/mnt` con el comando `mount /dev/sda2 /mnt`, creamos el directorio `boot` con `mkdir /mnt/boot` y montamos lo que hay en `/dev/sda1` en el directorio que creamos con `mount /dev/sda1 /mnt/boot`. Con esto, podemos procede a instalar los paquetes necesarios:

```bash
pacstrap /mnt linux linux-firmware networkmanager grub wpa_supplicant base base-devel
```

Esperamos que se instalen lo paquetes que hemos definido sin errores. Ahora ejecutamos el comando `genfstab -U /mnt > /mnt/etc/fstab` vamos a generar un archivo que contenga la información relacionada a las particiones del sistema y lo validamos con `cat /mnt/etc/fstab`.

Posterior a esto, vamos a cambiar la contraseña del usuario **root** y generar un usuario; por lo tanto, primero ejecutamos `arch-chroot /mnt` para ingresar a lo que va a ser nuestro sistema y definimos la contraseña de **root** con `passwd`. Ahora creamos el usuario que queramos, para este caso **k4miyo** con `useradd -m k4miyo` y también le generamos una contraseña con `passwd k4miyo`.

El usuario que hemos creado lo agregamos al grupo **wheel** con `usermod -aG wheel k4miyo` y lo validamos con `groups k4miyo`. Instalamos los paquetes **sudo**, **vim** y **nano** con `pacman -S sudo vim nano`, abrimos el `/etc/sudoers` con **nano** o **vim** y buscando por **wheel** descomentamos la linea `%wheel ALL=(ALL:ALL) ALL`; guardamos cambios y cerramos. Lo anterior lo hicimos para que cuando del usuario **k4miyo** hacemos un **sudo su**, se nos solicite una contraseña para poder operar como **root**.

Vamos a retocar un poco las regiones con `nano /etc/locale.gen` y descomentamos `en_US.UTF-8 UTF-8` y `es_XX.UTF-8 UTF-8`; en donde XX se refiere a la región donde nos encontremos. Guardamos, salimos y aplicamos un `locale-gen`. Para que nos tome siempre nuestro teclado y no andar cambiando, vamos a crear con **nano** el archivo `nano /etc/vconsole.conf` y le agregamos `KEYMAP=es` 

Proseguimos con la instalación del **bootloader** con `grub-install /dev/sda` y generamos el archivo de configuración con `grub-mkconfig -o /boot/grub/grub.cfg`.

Si queremos definir el nombre de nuestro equipo, vamos a ejecutar `echo k4mi4rch > /etc/hostname` y ahora retocamos el `/etc/hosts` agregando las siguientes linea:

```bash
127.0.0.1 	localhost
::1 		localhost
127.0.0.1 	k4mi4rch.localhost k4mi4rch
```

Y para presumir que tenemos instalado **Arch Linux**, vamos a instalar `neofetch` con `pacman -S neofetch`; que una vez instalado, al ejecutar el comando `neofetch`, nos aparece el logo de **Arch**. Para finalizar, ejecutamos un `exit` y `reboot now`.

## Salida a internet

Ya tenemos nuestro sistema pero sin interfaz gráfica todavía, por lo que nos logueamos como el usuario que definimos y vamos a ver que no tenemos salida a internet; por lo tanto ejecutamos `sudo systemctl start NetworkMaanger.service` y `sudo systemctl enable NetworkMaanger.service`; con esto ya contamos con salida a internet. También vamos a hacer lo mismo para el servicio `wpa_supplicant` con los comandos `sudo systemctl start wpa_supplicant.service` y `sudo systemctl enable wpa_supplicant.service`.


## AUR y BlackArch

Para tener más alcance a los recursos que podemos instalar, vamos a agregar **AUR** que es un repositorio de **Arch**; por lo tanto, primero instalamos `git` con `sudo pacman -S git`. Ahora, como el usuario no privilegiado, nos dirigimos a nuestro directorio asignado y dentro de `Desktop` creamos una carpeta llamada `repos`, que será en donde tengamos los repositorios que queramos descargar `cd; mkdir -p Desktop/repos; cd !$`.

Una vez dentro de dicha carpeta, nos descargamos `paru` con `git clone https://aur.archlinux.org/paru-bin.git`, nos metemos en el directorio `paru-bin` y procedemos a instalarlo con `makepkg -si`; si nos pide contraseña, se la proporcionamos y le damos enter.

Ahora vamos a crear el directorio `/home/k4miyo/Desktop/repos/blackarch` y nos descargaremos un script en bash `curl -O https://blackarch.org/strap.sh`, le damos permisos de ejecución con `chmod +x strap.sh` y ejecutamos el script como el usuario **root**. Para validar los repos y/o bases de datos a las que tenemos acceso, lo hacemos con el comando `sudo pacman -Sy`. Con esto, podemos instalarnos múltiples paquetes, como por ejemplo `impacket` con `sudo pacman -S impacket`. Para buscar los paquetes que podemos instalar, lo hacemos con  `sudo pacman -Sgg | grep blackarch`.

## Interfaz gráfica

Para la interfaz gráfica, vamos a instalar `xorg` con `sudo pacman -S xorg xorg-server`, posteriormente `gnome` con `sudo pacman -S gnome`. Antes de habilitar y arrancar la interfaz gráfica, vamos a instalar `kitty` con `sudo pacman -S kitty`. Ahora ejecutamos `sudo systemctl enable gdm.service` y `sudo systemctl start gdm.service` y con esto ya contamos con una interfaz gráfica; por lo tanto aplicamos un `reboot now`.

Una vez que ingresamos a nuestra máquina, vemos que el teclado está en inglés; por lo que si queremos cambiar, lo haremos de manera manuel dentro del sistema y buscando **Keyboard** y agregando nuestro idioma para el teclado. Si presionamos **Ctrl + Alt + F3**, nos abre la consola y con **Ctrl + Alt + F2** regresamos a la interfaz gráfica. En modo consola, instalamos `gtkmm` con `sudo pacman -S gtkmm` y en caso de que estemos utilizando VMware, instalaremos `sudo pacman -S open-vm-tools xf86-video-vmware xf86-input-vmmouse` y habilitamos el demonio con `sudo systemctl enable vmtoolsd`. Con esto, ya podemos volver a reiniciar la máquina con `sudo reboot now`. Con esto ya tenemos la máquina instalada, ahora nos falta personalizarla, por lo que antes vamos a realizar un **Snapshot** por cualquier cosa que nos llegue a pasar a continuación.

## Personalización

### Awesome

Nos abrimos una **kitty** e instalamos firefox con `sudo pacman -S firefox` y nos dirigimos al repositorio [Awesome](https://github.com/rxyhn/dotfiles), vamos a la parte de **Setup** e instalamos lo que nos indica como un usuario no privilegiado:

```bash
paru -S awesome-git picom-git alacritty rofi todo-bin acpi acpid \
wireless_tools jq inotify-tools polkit-gnome xdotool xclip maim \
brightnessctl alsa-utils alsa-tools pulseaudio lm_sensors \
mpd mpc mpdris2 ncmpcpp playerctl --needed
```

Ahora ejecutamos lo siguiente:

```bash
systemctl --user enable mpd.service
systemctl --user start mpd.service
sudo systemctl enable acpid.service
sudo systemctl start acpid.service
```

Vamos a instalar las fuentes, de acuerdo con al autor del repositorio y vamos a agregar algunas más; por lo que primero nos instalamos `wget` con `sudo pacman -S wget`, nos dirigimos a `cd /usr/share/fonts` y descargamos el un recurso con `sudo wget http://fontlot.com/downfile/5baeb08d06494fc84dbe36210f6f0ad5.105610`. Renombramos el archivo `5baeb08d06494fc84dbe36210f6f0ad5.105610` por uno más corto como `comprimido.zip` con `sudo mv 5baeb08d06494fc84dbe36210f6f0ad5.105610 comprimido.zip`, descomprimimos el archivo `sudo unzip comprimido.zip`, borramos el comprimido `sudo rm comprimido.zip` y ahora vamos a hacer un filtrado de los archivos que nos interesan. (**Nota**: Si quieren pueden descomprimir con 7z instalando el paquete con `sudo pacman -S p7zip`)

Dentro de la ruta `/usr/share/fonts` ejecutamos lo siguiente:
```bash
find . | grep "\.ttf$" | while read line; do sudo cp $line .; done
sudo rm -r iosevka-2.2.1/
sudo rm -r iosevka-slab-2.2.1/
```

Ahora nos descargamos **icomoon.zip** de [Dropbox](https://dropbox.com/s/hrkub2yo9iapljz/icomoon.zip?=0) y movemos el contenido en la carpeta en la que nos encontramos `/usr/share/fonts`:

```bash
sudo mv /home/k4miyo/Downloads/icomoon.zip .
sudo unzip icomoon.zip
sudo rm icomoon.zip
sudo mv icomoon/*.ttf .
sudo rm -rf icomoon/
```

Como un usuario no privilegiado instalamos las fuentes que nos hacen falta:

```bash
paru -S nerd-fonts-jetbrains-mono ttf-font-awesome ttf-font-awesome-4 ttf-material-design-icons
```

De acuerdo con el autor del repositorio, nos descargamos la repo y copiamos los archivos de configuración:

```bash
cd /home/k4miyo/Desktop/repos/
git clone https://github.com/rxyhn/dotfiles.git
cd dotfiles
cp -r config/* ~/.config/
mkdir ~/.local/bin/
cp -r bin/* ~/.local/bin/
cp -r misc/. ~/
```

Reiniciamos la máquina con `sudo reboot now`. Al iniciar la máquina otra vez, al darle click en nuestro usuario, vemos un engrane en la parte inferior derecha, damos click y seleccionamos **awesome**. 

![""](/assets/images/entorno_arch/arch4.png)

Nuestro entorno va a estar un poco raro; esto lo arreglamos ingresando a modo consola con **Ctrl + Alt + F3**, nos logueamos, ingresamos a la ruta `cd /Desktop/repos/dotfiles` y ejecutamos un `git log`; de las opciones que nos aparecen, la quinta nos dice **fix: ui and widgets**.

![""](/assets/images/entorno_arch/arch5.png)

Podemos guardar el id asociado `c1e2eef2baa91aebd37324891cb282666beae04f`  y ejecutamos

```bash
git checkout c1e2eef2baa91aebd37324891cb282666beae04f
```

O ejecutamos el comando siguiente

```bash
git checkout $(git log | grep commit | grep "c1"| awk 'NR==3' | awk 'NF{print $NF}')
```

Volvemos a copiar los binarios y reiniciamos:

```bash
cp -r config/* ~/.config/
cp -r bin/* ~/.local/bin/
cp -r misc/. ~/
sudo reboot now
```

### Idioma del teclado

Si iniciamos sessión, vemos que el teclado se encuentra en inglés y que al hacer **Ctrl + Enter** no se nos abre la terminal, por lo tanto, hacemos otra vez **Ctrl + Alt + F3**; nos logueamos en modo consola y editamos el archivo `nano .config/awesome/rc.lua` y en la parte donde nos dice `terminal` le asignamos el valor de `kitty`. Ahora, hacemos **Ctrl + Alt + F2** y en pantalla completa, debemos hacer un **Ctrl + Windows + R** para aplicar los cambios y ahora si hacemos **Ctrl + Enter** ya se nos despliega la terminal.

Para cambiar el idioma del teclado, ejecutamos `localectl set-x11-keymap es` y al hacer **Ctrl + Windows + Q**, nos deslogueamos y al iniciar sesión, ya tenemos el idioma español

### Instalación ZSH

Instalamos la `zsh` y le cambiamos a nuestro usuario y a **root** la `bash` por la `zsh`.

```bash
sudo pacman -S zsh
sudo usermod --shell /usr/bin/zsh k4miyo
sudo usermod --shell /usr/bin/zsh root
```

Ahora nos dirigimos a [bspwm-configuration-files](https://s4vitar.github.io/bspwm-configuration-files/), borramos el archivo `.zshrc` y copiamos todo lo que se encuentra en el archivo `zshrc` de la págna para generar uno nuevo. Si abrimos una nueva terminal, vemos que nos aparecen unos errores de archivos que no se encuetran en el sistema, por lo tanto los instalmos:

```bash
paru -S zsh-syntax-highlighting zsh-autosuggestions
```

Descargamos el archivo `sudo.plugin.zsh` en la ruta `/usr/share/zsh/plugins/zsh-sudo/`. En caso de no existir los directorios, los creamos:

```bash
cd /usr/share/zsh/plugins/
sudo mkdir zsh-sudo
chown k4miyo:k4miyo zsh-sudo/
cd !$
wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/plugins/sudo/sudo.plugin.zsh
```

Y cambiamos la ruta de los archivos dentro de la `.zshrc`:

```bash
...
# Plugins
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh/plugins/zsh-sudo/sudo.plugin.zsh
...
```

### Instalación de locate, lsd, bat y scrub

Ahora instalamos `locate`,`lsd` y `bat` con `sudo pacman -S locate lsd bat`; así como `scrub` con `paru -S scrub`. Para que el usuario **root** tenga la misma presentación, vamos a crear un link simbólico de nuestra `.zshrc` con `sudo ln -s -f /home/k4miyo/.zshrc /root/.zshrc`.

### Nerd Fonts

Cambiaremos la fuente de nuestra termina descargando [Hack.zip](https://github.com/ryanoasis/nerd-fonts/releases/download/v2.1.0/Hack.zip) y su contenido lo colocamos en `/usr/share/fonts`:

```bash
cd /usr/share/fonts
sudo mv /home/k4miyo/Downloads/Hack.zip .
sudo unzip Hack.zip
sudo rm Hack.zip
```

### Configuración kitty

Vamos a retocar un poco la `kitty`, por lo que vamos descargarnos el archivo `kitty.conf`:

```bash
cd ~/.config/kitty/
wget https://raw.githubusercontent.com/rxyhn/bspdots/main/config/kitty/kitty.conf
wget https://raw.githubusercontent.com/rxyhn/bspdots/main/config/kitty/color.ini
```

Y ahora vamos a retocar el archivo `kitty.conf`:

```bash
enable_audio_bell no

include color.ini

font_family		HackNerdFonts
font_size 12

url_color	#61afef

url_style	curly

map ctrl+left neighboring_window left
map ctrl+right neighboring_window right
map ctrl+up neighboring_window up
map ctrl+down neighboring_window down

map f1 copy_to_buffer a
map f2 paste_from_buffer a
map f3 copy_to_buffer b
map f4 paste_from_buffer b

cursor_shape beam
cursor_beam_thickness 1.8

mouse_hide_wait 3.0
detect_urls yes

repaint_delay 10
input_delay 3
sync_to_monitor yes

map ctrl+shift+z toggle_layout stack
tab_bar_style powerline

inactive_tab_background #e06c75
active_tab_background #98c379
inactive_tab_foreground #000000
tab_bar_margin_color black

map ctrl+shift+enter new_window_with_cwd
map ctrl+shift+t new_window_with_cwd

brackground_opacity 0.95

shell zsh
```

Con esta configuración, podemos hacer lo siguiente:
- **Ctrl + Shift + Enter** - Abrir terminal secundaria
- **Ctrl + Shift + w** - Cerrar terminal secundaria
- **Ctrl + Shift + r** - Resize
- **Ctrl + Shift + l** - Acomodar ventanas
- **Ctrl + Shift + Alt + t** - Nombre de ventana
- **Ctrl + Shift + t** - Crear más ventanas

### Configuración Awesome

Ahora retocremos el archivo `~/.config/awesome/ui/decorations/init.lua`:

```bash
cd ~/.config/awesome/ui/decorations
nano init.lua
```

Y comentaremos las últimas dos lineas con `--`:

```bash
-- require("ui.decorations.titlebar")
-- require("ui.decorations.music")
```

Para la transparencia de la terminal, vamos a borrar el archivo `~/.config/awesome/theme/picom.conf` y vamos a sustituirlo por lo que se encuentra en el siguiente repositorio:

```bash
cd ~/.config/awesome/theme/
rm picom.conf
wget https://raw.githubusercontent.com/rxyhn/bspdots/main/config/picom/picom.conf
```

Ahora vammos a realizar unas modificaciones, por lo que nos abrimos el archivo y cambiamos lo siguiente:

```bash
##############################################################################
#                                  CORNERS                                   #
##############################################################################
# requires: https://github.com/sdhand/compton
corner-radius = 20;
rounded-corners-exclude = [
  #"window_type = 'normal'",
  #"class_g = 'firefox'",
];

round-borders = 20;
round-borders-exclude = [
  #"class_g = 'TelegramDesktop'",
];

...

# Enable/disable VSync.
# vsync = false
vsync = false

...

# Default opacity for dropdown menus and popup menus. (0.0 - 1.0, defaults to 1.0)
# menu-opacity = 1.0
opacity = 1.0

...

##############################################################################
#                                    BLUR                                    #
##############################################################################

# Parameters for background blurring, see the *BLUR* section for more information.
# blur-method = "gaussian"
# blur-size = 14
# blur-strength = 10

# Blur background of semi-transparent / ARGB windows.
# Bad in performance, with driver-dependent behavior.
# The name of the switch may change without prior notifications.
#
# blur-background = false

# Blur background of windows when the window frame is not opaque.
# Implies:
#    blur-background
# Bad in performance, with driver-dependent behavior. The name may change.
#
# blur-background-frame = false

# Use fixed blur strength rather than adjusting according to window opacity.
# blur-background-fixed = false

# Specify the blur convolution kernel, with the following format:
# example:
#   blur-kern = "5,5,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1";
#
# blur-kern = ''
# blur-kern = "3x3box";

# Exclude conditions for background blur.
# blur-background-exclude = []
# blur-background-exclude = [
#     "! name~=''",
#     "name *= 'slop'",
#     "window_type = 'dock'",
#     "window_type = 'desktop'",
#     "_GTK_FRAME_EXTENTS@:c"
# ];

...

# Disable the use of damage information.
# This cause the whole screen to be redrawn everytime, instead of the part of the screen
# has actually changed. Potentially degrades the performance, but might fix some artifacts.
# The opposing option is use-damage
# 
# no-use-damage = false
use-damage = false

...

# Specify refresh rate of the screen. If not specified or 0, picom will
# try detecting this with X RandR extension.
# 
# refresh-rate = 60
# refresh-rate = 0

...
# Specify the backend to use: `xrender`, `glx`, or `xr_glx_hybrid`. 
# `xrender` is the default one.
#
# backend = 'glx'
backend = "xrender"
```

### Fondo de pantalla

Para modificar el fondo de pantalla, primero nos descargamos la imagen que queramos para el fondo y agregamos la siguiente linea en el archivo `~/.config/awesome/rc.lua`. Adicional, nos descargaremos `feh` con el comando `paru -S feh`:

```bash
-- Wallpaper
local wallpaper_cmd="feh --bg-fill /home/k4miyo/Desktop/k4miyo/images/Wallpaper.jpg"
os.execute(wallpaper_cmd)
```

### PowerLevel10k

Ya toma instalarse la **powerlevel10k**; por lo tanto nos dirigimos al repositorio [powerlevel10k](https://github.com/romkatv/powerlevel10k) y seguimos las intrucciónes de instalación; tanto para el usuario normal y **root**.

```bash
cd
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
echo 'source ~/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
zsh
```

Para el usuario no privilegiado, modificaremos parte de la configuración de la **powerlevel10k** en el archivo `~/.p10k.zsh`:

```bash
# The list of segments shown on the left. Fill it with the most important segments.
  typeset -g POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(
    os_icon                 # os identifier
    dir                     # current directory
    vcs                     # git status
    command_execution_time 
    context
    # prompt_char           # prompt symbol
  )

# The list of segments shown on the right. Fill it with less important segments.
  # Right prompt on the last prompt line (where you are typing your commands) gets
  # automatically hidden when the input line reaches it. Right prompt above the
  # last prompt line gets hidden if it would overlap with left prompt.
  typeset -g POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(
    # status                  # exit code of the last command
    # command_execution_time  # duration of the last command
    # background_jobs         # presence of background jobs
    # direnv                  # direnv status (https://direnv.net/)
    # asdf                    # asdf version manager (https://github.com/asdf-vm/asdf)
    # virtualenv              # python virtual environment (https://docs.python.org/3/library/venv.html)
    # anaconda                # conda environment (https://conda.io/)
    # pyenv                   # python environment (https://github.com/pyenv/pyenv)
    # goenv                   # go environment (https://github.com/syndbg/goenv)
    # nodenv                  # node.js version from nodenv (https://github.com/nodenv/nodenv)
    # nvm                     # node.js version from nvm (https://github.com/nvm-sh/nvm)
    # nodeenv                 # node.js environment (https://github.com/ekalinin/nodeenv)
    # node_version          # node.js version
    # go_version            # go version (https://golang.org)
    # rust_version          # rustc version (https://www.rust-lang.org)
    # dotnet_version        # .NET version (https://dotnet.microsoft.com)
    # php_version           # php version (https://www.php.net/)
    # laravel_version       # laravel php framework version (https://laravel.com/)
    # java_version          # java version (https://www.java.com/)
    # package               # name@version from package.json (https://docs.npmjs.com/files/package.json)
    # rbenv                   # ruby version from rbenv (https://github.com/rbenv/rbenv)
    # rvm                     # ruby version from rvm (https://rvm.io)
    # fvm                     # flutter version management (https://github.com/leoafarias/fvm)
    # luaenv                  # lua version from luaenv (https://github.com/cehoffman/luaenv)
    # jenv                    # java version from jenv (https://github.com/jenv/jenv)
    # plenv                   # perl version from plenv (https://github.com/tokuhirom/plenv)
    # perlbrew                # perl version from perlbrew (https://github.com/gugod/App-perlbrew)
    # phpenv                  # php version from phpenv (https://github.com/phpenv/phpenv)
    # scalaenv                # scala version from scalaenv (https://github.com/scalaenv/scalaenv)
    # haskell_stack           # haskell version from stack (https://haskellstack.org/)
    # kubecontext             # current kubernetes context (https://kubernetes.io/)
    # terraform               # terraform workspace (https://www.terraform.io)
    # terraform_version     # terraform version (https://www.terraform.io)
    # aws                     # aws profile (https://docs.aws.amazon.com/cli/latest/userguide/cli-configu>
    # aws_eb_env              # aws elastic beanstalk environment (https://aws.amazon.com/elasticbeanstal>
    # azure                   # azure account name (https://docs.microsoft.com/en-us/cli/azure)
    # gcloud                  # google cloud cli account and project (https://cloud.google.com/)
    # google_app_cred         # google application credentials (https://cloud.google.com/docs/authenticat>
    # toolbox                 # toolbox name (https://github.com/containers/toolbox)
    # context                 # user@hostname
    # nordvpn                 # nordvpn connection status, linux only (https://nordvpn.com/)
    # ranger                  # ranger shell (https://github.com/ranger/ranger)
    # nnn                     # nnn shell (https://github.com/jarun/nnn)
    # xplr                    # xplr shell (https://github.com/sayanarijit/xplr)
    # vim_shell               # vim shell indicator (:sh)
    # midnight_commander      # midnight commander shell (https://midnight-commander.org/)
    # nix_shell               # nix shell (https://nixos.org/nixos/nix-pills/developing-with-nix-shell.ht>
    # vi_mode                 # vi mode (you don't need this if you've enabled prompt_char)
    # vpn_ip                # virtual private network indicator
    # load                  # CPU load
    # disk_usage            # disk usage
    # ram                   # free RAM
    # swap                  # used swap
    # todo                    # todo items (https://github.com/todotxt/todo.txt-cli)
    # timewarrior             # timewarrior tracking status (https://timewarrior.net/)
    # taskwarrior             # taskwarrior task count (https://taskwarrior.org/)
    # time                  # current time
    # ip                    # ip address and bandwidth usage for a specified network interface
    # public_ip             # public IP address
    # proxy                 # system-wide http/https/ftp proxy
    # battery               # internal battery
    # wifi                  # wifi speed
    # example               # example user-defined segment (see prompt_example function below)
  )
```

Para el usuario **root** modificaremos lo mismo que el usuario no privilegiado, y también modificaremos lo siguiente:

```bash
# Context format when running with privileges: bold user@hostname.
  typeset -g POWERLEVEL9K_CONTEXT_ROOT_TEMPLATE=' ^~^`'
  # Context format when in SSH without privileges: user@hostname.
  typeset -g POWERLEVEL9K_CONTEXT_{REMOTE,REMOTE_SUDO}_TEMPLATE='%n@%m'
  # Default context format (no privileges, no SSH): user@hostname.
  typeset -g POWERLEVEL9K_CONTEXT_TEMPLATE='%n@%m'
```

En caso de que cuando migremos al usuario **root** nos aparezca la configuración de la **powerlevel10k**, cancelamos dicha configuración, si tenemos la carpeta `powerlevel10k` en el directorio `/root` la borramos y ejecutamos los comandos siguientes:

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
p10k configure
```

### Instalación FZF

Ahora instalaremos la `fzf` de acuerdo a lo que nos indica el repositorio [fzf](https://github.com/junegunn/fzf); tanto para el usuario no privilegiado como **root**.

```bash
cd
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

### NeoVim

Para instalar `neovim`, ejecutaremos lo siguiente:

```bash
sudo pacman -S neovim
```

Y ahora lo complementaremos **NvChad** del repositorio [NvChad](https://github.com/NvChad/NvChad). Para instalarlo, los dirigiremos al recurso [nvchad.github.io](https://nvchad.github.io/getting-started/setup) y seguiremos los pasos que nos indican como un usuario no privilegiado:

```bash
cd
git clone https://github.com/NvChad/NvChad ~/.config/nvim --depth 1
nvim +'hi NormalFloat guibg=#1e222a' +PackerSync
```

Para abrir cualquier archivo utilizaremos `nvim` y para que nos muestre archivos que forma bonita, utilizaremos el comando `NvimTreeToggle`:

![""](/assets/images/entorno_arch/arch6.png)

Recordar que tenemos que hace lo mismo para el usuario **root**:

### Icono Awesome

Si queremos cambiar el icono de **Awesome** en la barra lateral izquierda, nos dirigimos a `~/.config/awesome/theme/assets/icons` y cambiamos el archivo `awesome.png` por uno que nosotros queramos. Lo podemos visualizar con el comando `kitty +kitten icat awesome.png`.

### ShortCuts

Para agregar shortcuts y poder personalizar las aplicaciones que abramos, vamos a retocar algunos archivos de configuración de **Awesome**. Vamos a empezar primero con modificar las teclas para Firefox; por lo que nos abrimos el archivo `~/.config/awesome/configuration/keys.lua`:

```bash
		awful.key({modkey,"Shift"}, "f", function()
            awful.spawn.with_shell(browser)
        end,
        {description = "open web browser", group = "launcher"}),
```

Ahora nos descargamos el BurpSuite con `sudo pacman -S burpsuite` y vamos a asignarle una combinación de teclas para poder abrirlo. Por lo que primero creamos la variable en el archivo `~/.config/awesome/rc.lua`:

```bash
-- Defalt Applications
terminal = "kitty"
...
browser = "firefox"
proxy_burp = "burpsuite" --La linea que agregamos para burpsuite
```

Ahora agregamos la ejecución en el archivo `~/.config/awesome/configuration/keys.lua`:

```bash
		awful.key({modkey,"Shift"}, "b", function()
            awful.spawn.with_shell(proxy_burp)
        end,
        {description = "open burpsuite", group = "launcher"}),
```

Si aplicamos los cambios de la configuración con **Ctrl + Windows + r**, vemos que podemos hacer **Windows + Shift + b** para abrir el BurpSuite. Cualquier otro programa que necesitemos un shortcut, podemos configurarlo de esta manera.

### Instalación de herramienta de pentesting

Vamos a instalarnos algunas herramientas que nos puedan ayudar para realizar pentesting:

```bash
sudo pacman -S evil-winrm python-pip impacket responder nmap whatweb wfuzz gobuster
sudo pacman -S metasploit
```

### Detalles de teclas

Tenemos una teclas que no nos funcionan como el **HOME**, **END** y **DEL**; por lo tanto, abrimos el archivo `~/.zshrc` como un usuario no privilegiado y agregamos lo siguiente:

```bash
bindkey "^[[H" beginning-of-line
bindkey "^[[F" end-of-line
bindkey "^[[3~" delete-char
bindkey "^[[1;3C" forward-word 
bindkey "^[[1;3D" backward-word
```

### Instalación de mdcat

Utilizando el comando `sudo pacman -S mdcat` para que veamos de forma presentable los archivos de extensión `md`.

### Copy/Paste VMware

En caso de que necesitemos la funcionalidad de **Copy and Paste** sobre WMware, necesitamos agregar las siguientes lineas en el archivo `~/.config/awesome/rc.lua`:

```bash
-- VMware
awful.util.spawn_with_shell("vmware-user-suid-wrapper --no--startup-id")
```


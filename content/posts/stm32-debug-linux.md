+++
title = "Depurando un STM32 con OpenOCD, GDB y GDBFrontend en Linux"
date = 2022-03-19T13:04:02-06:00
lastmod = 2022-03-19T13:04:02-06:00
tags = ["STM32", "Linux", "GDB", "OpenOCD"]
categories = ["Embedded"]
imgs = []
cover = "/img/covers/stm32_dev_debug.png"
readingTime = true
toc = true
comments = false
justify = false
single = false
license = ""
draft = false
[twitter]
  card = "summary_large_image"
  site = "@jhestolano"
  title = "Depurando un STM32 con OpenOCD, GDB y GDBFrontend en Linux"
  description = "En esta entrada les cuento como pueden usar un GDB y OpenOCD para depurar un STM32 en Linux"
  image = "https://elrobotista.com/img/covers/stm32_dev_debug.png"
+++

En la entrada anterior les mostré como pueden iniciar rápidamente un proyecto con un STM32 utilizando herramientas open source en Linux. En este post vamos a continuar con la siguiente cuestión: ¿Cómo podemos depurar el código utilizando herramientas open source? Esto es de vital importancia porque tarde o temprano en el proyecto, llegaremos a un punto en el cual será necesario examinar con lupa los registros / estado del procesador; en estas situaciones llenar el código de ```printf``` puede no ser una solución práctica por muchas razones: no tener acceso al hardware necesario, no tener configurado un puerto serial (UART) en esta etapa del proyecto, la ejecución de ```printf``` podría cambiar el timing del código y cambiar el comportamiento del sistema, etc.

Por estas razones, veremos cómo podemos configurar un ambiente de depuración para nuestro código utilizando [GDB](https://www.sourceware.org/gdb/), [Opencd](https://openocd.org/) y [GDBFrontend](https://github.com/rohanrhu/gdb-frontend).

## OpenOCD

El "Open On-Chip Debugger" es una herramienta que soporta depuración, programación y "boundary scanning" para múltiples plataformas como:

* ARMv5 through latest ARMv8
* MIPS
* AVR (incomplete)
* Andes
* RISC-V

En esta ocasión lo utilizaremos para depuración de un ARM Cortex-M4 a través del debugger GDB. Para instalarlo ejecuten el siguiente comando en su terminal:
``` none
git clone https://github.com/openocd-org/openocd
cd openocd
./bootstrap
./configure
make && sudo make install
```

Si openocd se instaló correctamnete, deberían poder ejecutar el comando ```opencd --version``` y ver algo similar a esto:

``` none
❯ openocd --version
Open On-Chip Debugger 0.11.0+dev-00615-gbe0d68eb6 (2022-03-12-20:07)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
```

Para utilizar openocd para depurar nuestro código, es necesario proveer dos argumentos: el archivo de configuración para el st-link que estén utilizando, y el archivo para la tarjeta / procesador que estén utilizando. Para este post estoy usando una [NUCLEO-F303RE](https://www.st.com/en/evaluation-tools/nucleo-f303re.html), con un [STLINK](https://www.st.com/en/development-tools/st-link-v2.html):
``` none
openocd -f /usr/local/share/openocd/scripts/interface/stlink.cfg -f /usr/local/share/openocd/scripts/board/st_nucleo_f3.cfg
```
Las tarjetas NUCLEO traen incluido el depurador y programador STLINK:

![Embedded STLINK](/img/posts/nucleo_stlink.jpg#center)

Al lanzar opencd, el servidor inicia y deberían ver el siguiente mensaje:
```none
Open On-Chip Debugger 0.11.0+dev-00615-gbe0d68eb6 (2022-03-12-20:07)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Warn : Interface already configured, ignoring
Error: already specified hl_layout stlink
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
Info : DEPRECATED target event trace-config; use TPIU events {pre,post}-{enable,disable}
srst_only separate srst_nogate srst_open_drain connect_deassert_srst

Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : clock speed 1000 kHz
Info : STLINK V2J30M19 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.244649
Info : [stm32f3x.cpu] Cortex-M4 r0p1 processor detected
Info : [stm32f3x.cpu] target has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f3x.cpu on 3333
Info : Listening on port 3333 for gdb connections
```
Aún más, deberían ver el LED (LD1) del ST-LINK parpadear rápidamente entre verde y rojo:

![OpenOCD STM32](/img/posts/stm32_openocd_blink.gif#center)

¡Listo! Podemos ver que el servidor está esperando la conexión de GDB en ```localhost:3333```

## GDB

Instalar GDB utilizando el package manager es la forma más sencilla de hacerlo en Linux. El problema aquí es que el GDBFrontend que vamos a utilizar necesita GDB compilado con soporte para Python3 y otras configuraciones específicas. Por lo menos en Artix, no pude encontrar a través del package manager exactamente lo que necesitaba. Entonces vamos a instalarlo desde el código fuente. Es bastante sencillo, aunque sí toma un poco de tiempo el proceso de compilación: alrededor de 15 minutos.

Los pasos a seguir para el proceso de compilación están bien descritos en https://github.com/rohanrhu/gdb-frontend/wiki/Embedded-Debugging. Los comandos para instlar GDB y compilarlo son:
``` none
# Este comando descarga una carpeta comprimida con el código de GDB.
wget https://ftp.gnu.org/gnu/gdb/gdb-11.2.tar.xz

# En este paso se descomprime el contenido de la carpeta.
tar zxvf gdb-11.2.tar.gz

# Se crea la carpeta de "build" para desde ahí lanzar la compilación con make.
mkdir gdb-11.2-build
cd gdb-11.2-build
../configure --with-python=/usr/bin/python3 --target=arm-none-eabi --enable-interwork --enable-multilib

# Iniciar el proceso de compilación.
make
```

Noten que en el paso de configuración ```../configure``` se hace referencia a ```/usr/bin/python3``` por lo tanto, se asume que en este punto ya tienen instalado Python3, como es el caso de distribuciones de Linux como Ubuntu, Debian, Arch / Artix. Si por alguna razón tienen el binario de Python en otra ruta, cambien ```/usr/bin/python3``` con el path hacia su binario. Si no estan seguros si su binario de Python está en el lugar correcto, pueden ejectuar:
```none
❯ which python3
/usr/bin/python3
```

Al terminar, como siempre, para validar que el proceso de compilación funcionó correctamente pueden inspeccionar la versión de GDB: ```cd gdb/ && ./gdb --version```. La versión 11.2 se imprime en la consola:
```none
GNU gdb (GDB) 11.2
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

Para ir un poco más lejos, y realmente asegurarnos que todo está bien, podemos intentar conectar openocd y gdb. Asumiendo que tienen openocd ejecutándose con el comando mencionado anteriormente, ahora ejecuten ```./gdb``` nuevamente desde la carpeta de ```gdb-11.2-build/gdb```. Y dentro de gdb ejecuten ```target extended-remote:3333```. Noten la referencia al puerto ```:3333``` que es donde está escuchando openocd. Deberían ver algo muy similar al siguiente output:
``` none
❯ ./gdb
Python Exception <class 'ModuleNotFoundError'>: No module named 'gdb'
./gdb: warning:
Could not load the Python gdb module from `/usr/local/share/gdb/python'.
Limited Python support is available from the _gdb module.
Suggest passing --data-directory=/path/to/gdb/data-directory.
Exception caught while booting Guile.
Error in function "open-file":
No such file or directory: "/usr/local/share/gdb/guile/gdb/boot.scm"
./gdb: warning: Could not complete Guile gdb module initialization from:
/usr/local/share/gdb/guile/gdb/boot.scm.
Limited Guile support is available.
Suggest passing --data-directory=/path/to/gdb/data-directory.
GNU gdb (GDB) 11.2
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb) target extended-remote :3333
Remote debugging using :3333
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
Python Exception <class 'NameError'>: Installation error: gdb._execute_unwinders function is missing
0x080001be in ?? ()
(gdb)
```
Noten que tenemos algunos errores porque nos hacen falta por ahí unas opciones al lanzar GDB. Pero ya las resolveremos en la siguiente sección.

## GDBFrontend

La motivación de experimentar con GDBFrontend es que, si bien gdb es un debugger completo con toda la funcionalidad necesaria para llevar a cabos grandes proyectos, la curva de aprendizaje puede llegar a ser un poco elevada. Esto debido a su interfaz tipo consola para la que tienes que conocer y recordar los comandos para ser productivo. Hay proyectos que precisamente intentan mejorar la interfaz a través Python, como lo es [GDB dashboard](https://github.com/cyrus-and/gdb-dashboard):

![GDB Dashboard](/img/posts/gdb_dash.jpg#center)

El problema aquí sigue siendo, que a pesar de la mejoría visual, la interacción con GDB sigue siendo a través de la consola. Un frontend gráfico como lo es GDBFrontend suaviza mucho la curva inicial y nos permite familiarizarnos rápidamnete. Vamos a la instalación.

Para clonar el proyecto, ya saben: ```git clone https://github.com/rohanrhu/gdb-frontend```... y listo: así de sencillo. Por estar basado en Python ¡no hay que compilar nada! Lo único que queda es entrar a la carpeta y ejecutarlo con ```./gdbfrontend```. Automaticamente una nueva ventana en su navegador debería abrirse con GDBFrontend listo para comenzar a depurar. Por default, la dirección para lanzarlo es http://127.0.0.1:5550/terminal/. Deberían ver una pantalla azul con la leyenda "No opened source":

![Default GDBFrontend](/img/posts/gdb-frontend-default.jpg#center)

Y ya estamos listos para comenzar a depurar el código.

## Debugging

Vamos a retomar el proyecto "blinky" que comenzamos en la [entrada anterior](/posts/stm32-dev-linux). Vayan y completen ese tutorial si no lo han hecho aún para que puedan seguir exactamente estos pasos.

Si no tienen, el repositorio, clonen, compilen y escriban en flash el proyecto con:

``` none
git clone https://github.com/elrobotista/stm32_dev_linux && cd stm32_dev_linux
git submodule update --init
cd libopencm3 && make
cd ../appl & make
st-flash --reset write blink.bin 0x8000000
```
En este punto, la memoria flash ya está ejecutando el código que enciende y apaga el LED aproximadamente cada segundo. Empecemos a depurar en GDBFrontend. Una vez más, inicien OpenOCD si no lo han hecho aún (recuerden cambiar la configuración si no están usando el mismo hardware):

``` none
openocd -f /usr/local/share/openocd/scripts/interface/stlink.cfg -f /usr/local/share/openocd/scripts/board/st_nucleo_f3.cfg
```

Lo siguiente es lanzar GDBFrontend (que automáticamente ejecuta GDB detrás de cámaras). ¿Pero recuerdan el error que teníamos arriba? Eso se soluciona de forma sencilla siguiendo los pasos mencionados en el repositorio de GDBFrontend. Necesitamos especificar el path a GDB y un directorio específico:

```none
~/Repos/gdb-frontend/gdbfrontend -g $(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/gdb) -G --data-directory=$(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/data-directory/)
```

En este comando se hace referencia al ejecutable ```gdbfrontend``` que guardo en la carpeta ```Repos```. De la misma forma, ```gdb``` también lo compile dentro de la carpeta ```Repos/gdb-11.2/```. Entonces lo que nos hacía falta definir a ```gdbfrontend``` era precisamente dónde se encuentra el ejecutable de ```gdb``` con soporte para ```python3```  y el ```data-directory```. Para no estar tecleando todo este cuento una y otra vez, pueden guardar esta linea dentro de una carpeta de scripts:
```none
mkdir ~/scripts
echo ~/Repos/gdb-frontend/gdbfrontend -g $(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/gdb) -G --data-directory=$(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/data-directory/) >> launch_gdbfrontend.sh
sudo chmod +x
```

Con esto pueden lanzar su ```gdbfrontend``` de forma sencilla con ```~/scripts/launch_gdbfrontend.sh```. Ya con openocd ejecutándose podemos conectarnos a través de gdbfrontend dando click en ```connect``` y utilizando el valor ```localhost:3333```:

![GDBFrontend Connect](/img/posts/gdbf-rename.jpg#center)

Seguido de esto, es necesario cargar el ejecutable con los "debugging symbols"; en este caso, el archivo ```blinky.elf```:

![GDBFrontend Executable](/img/posts/gdbf-load-exec.jpg#center)

Si todo va bien hasta ahorita, ya deberían poder ver las instrucciones en ensamblador (disassembly). Ya lo único que falta es que naveguen al "source code" para que GDBFrontend pueda mostrarles en código las instrucciones que están depurando. Seguido de esto, podemos insertar un breakpoint:

![GDBFrontend Breakpoint](/img/posts/gdbf-bkpt.jpg#center)

Noten a la izquierda el arbol de carpetas, donde navegué hasta ```~/Repos/stm32_dev_linux/appl/main.c```. Y ya están listos para depurar su código de una forma más amigable y visual.

Ejecuten un ```step in``` para entrar a la función ```gpio_toggle```, y pueden ir ejecutando linea por linea exactamente en el momento donde se alterna el estado del pin que controlad el LED. Pueden incluso ir instrucción por instrucción hasta la instrucción específica que escribe en la dirección de RAM mappeada al GPIO. ¡Imaginen el potencial que esta herramienta puede otorgarles en sus futuros proyectos!

En la parte inferior derecha pueden ver las variables locales, sus valores, el call stack y los registros del procesador:

![GDBFrontend Registers](/img/posts/gdbf-registers.png#center)

En la parte inferior está disponible la consola de GDB para correr cualquier comando de GDB que ejecutarías en la consola. Si quisieramos hacer un memory dump de una región de memoria RAM en específico, digamos las primeras 32 palabras con ```x/32xw 0x20000000```, donde 0x20000000 es el inicio de la RAM:

![GDBFrontend Dump](/img/posts/gdbf-dump.png#center)

Para los que ya conocen los comandos de GDB, también pueden contrlarlo desde la consola para con los comandos usuales ```step, continue, next, nexi, break```, etc.

Otro feature interesante que trae GDBFrontend es un evaluador de expresiones. Por ejemplo, puedes hacer operaciones con las variables locales o globales en contexto; o incluso si no estás seguro de una operación que tengas en el código, puedes usar el evaluador para jugar con ella. En la línea 90, en ```gpio_common_all.c``` aparece la operación:
```c
  uint32_t port = GPIO_ODR(gpioport);
  GPIO_BSRR(gpioport) = ((port & gpios) << 16) | (~port & gpios);
```

Vemos que el valor de ```port``` ha sido removido por el compilador en la optimización de código. Pero sin problema podemos cargar la primer operación en el evaluador:

![GDBFrontend Expression-1](/img/posts/gdbf-expression-1.png#center)

Para después evaluar la segunda expresión sustituyendo manualmente el valor de port en la expresión:

![GDBFrontend Expression-2](/img/posts/gdbf-expression-2.png#center)


¿Qué piensan? A mí me parece una herramienta bastante útil y sencilla de usar en comparación con iniciarse en GDB. Seguramente esta herramienta tiene más features, pero por lo pronto sigo explorandola para conocer qué mas tiene que ofrecer. Esto es todo por ahora como una sencilla introducción a la depuración de código embebido utilizando herramientas open source en Linux. ¡Hasta la próxima entrada!

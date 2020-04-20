+++
title = "Gráfica de señales en tiempo real."
date = 2020-04-19T13:51:55-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/serial-plot.gif"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
+++

A la hora de diseñar un controlador, tenemos herramientas como matlab, simulink, python con scipy/numpy, etc; que permiten la creación de gráficas en las disintas señales que podrían sernos de interés a la hora del diseño: señales de error, acción de control, velocidades, posiciones, etc. Pero la cosa cambia un poco cuandoes tiempo de llevar el diseño a la implementación: ¿Cómo podemos usar herramientas similares para visualizar las señales? Les presento: [*serialplot*](https://github.com/hyOzd/serialplot).

# Pasos iniciales
En el link de github del proyecto muestra cada paso que tienes que llevar acabo para instalar la herramienta. Para los usuarios de windows, según se menciona en la página, lo único que tienen que hacer es a la sección de [descargas](https://bitbucket.org/hyOzd/serialplot/downloads/) del repositorio y bajar el ejecutable. Pero debido a que no utilizo windows, sino linux (les aconsejo hacer igual), no podría confirmarles. Inténtenlo y si tienen problemas, escríbanme y puedo ayudarles.

Ahora, para los usuarios de linux: lo que me resultó más sencillo fue descargar el código y compilarlo. En el repositorio menciona que puede instalarse a través del package manager de linux, pero parece no estar disponible para la versión que estoy utilizando (Ubuntu 18.04.4 LTS).

Antes de comenzar el build, instalen las dependencias necesarias mencionadas en el repositorio: `qtbase5-dev`, `libqt5serialport5-dev`, `cmake` y `mercurial`. Para Qwt (librería de widgets gráficos), si instalan SVN, el build script de serial plot instala automáticamente este paquete. Pero si prefieren no instalar SVN, instalen `libqwt-qt5-dev` a través de su package manager (`apt-get install libqwt-qt5-dev` en Ubuntu). No olviden cambiar a `false` la variable `BUILD_QWT` en el archivo `CMakeLists.txt`.

Una vez hecho esto, vayan a la carpeta del proyecto, creen el directorio `build`, entren y corran el siguiente commando en bash: `cmake .. && make`, y ¡listo! Una vez terminado el build, encontraran el ejecutable `serialplot`. Al abrirlo deberían ver algo así:

![serial-plot-window](/img/posts/serial-plot-window.jpg)
*Figura 1. Serialplot*

# Configuración

En el tab de configuración podrán ver lo siguiente:

![serial-plot-config](/img/posts/serial-plot-config.jpg)
*Figura 2. Configuración de Serialplot*

En el campo `port` deberían ver automáticamente preseleccionado el dispositivo de puerto serial a través del cual recibirán los datos: un arduino, mbed, launchpad, etc. En mi caso es una [tarjeta de desarrollo de ST](https://www.st.com/en/evaluation-tools/nucleo-f302r8.html). Pero esto no es importante, el proceso es exactamente el mismo, la diferencia es el nombre del dispositivo asignado por el sistema operativo. Otro nombre común asignado es `ttyACM0`. Si no aparece ningún dispositivo, asegúrense primero que fue reconocido por el sistema operativo: `lsusb`:

``` bash

Bus 001 Device 009: ID 0951:16d2 Kingston Technology 
Bus 001 Device 010: ID 0483:374b STMicroelectronics ST-LINK/V2.1 (Nucleo-F103RB)
Bus 001 Device 002: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

```

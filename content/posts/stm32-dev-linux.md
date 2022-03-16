+++
title = "Hola, mundo embebido: STM32 en Linux"
date = 2022-03-12T05:19:50-06:00
lastmod = 2022-03-12T05:19:50-06:00
tags = []
categories = []
imgs = []
cover = "/img/covers/stm32_dev_linux_600w.png"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
[twitter]
  card = "summary"
  site = "@jhestolano"
  title = "Hola mundo embebido con un STM32 en Linux."
  description = "En esta entrada les muestro cómo pueden instalar las herramientas necesarias para desarrollar con un STM32 en Linux."
  image = "/img/covers/stm32_dev_linux_600w.png"
+++

Hoy en día las tarjetas basadas en microcontroladores [STM32](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html) son muy utilizadas en la industria, por los makers, y en la enseñanza.
¿Las razones? Podrían ser su amplio ambiente de herramientas de desarrollo: una multitud de "development boards" que incluyen el debugger [ST-LINK](https://www.st.com/en/development-tools/st-link-v2.html), infinidad de "capas" a las Raspberry PI. Herramientas sin costo para configurar los periféricos de forma gráfica [TrueSTUDIO](https://www.st.com/en/development-tools/truestudio.html), acceso libre al HAL escrito por el fabricante, pero también versiones open source: ([libopencm3](https://github.com/libopencm3/libopencm3)). Algunas familias cuentan con hardware dedicado para aplicaciones de alto desempeño, como control de servomotores, por ejemplo la utilizada en el proyecto [ODrive](https://odriverobotics.com/). Es por eso que en esta entrada quiero mostrarles como pueden instalar el toolchain de desarrollo para estas tarjetas/procesadores en Linux. Les mostraré como ir desde cero hasta el clásico "Hola, mundo" de los embebidos que es encender y apagar un LED como primer aplicación.

## Software y Herramientas

Sin importar qué "dev. board" vayan a usar, van a necesitar sí o sí ciertas herramientas genéricas: editor de texto, compilador, debugger, etc; empecemos a instalar todo esto.
Cosas como el editor de texto depende totalmente de su preferencia y en este punto es bastante irrelevante. Pero yo les sugeriría algo open source, que tenga un buen ecosistema de desarrollo para tener acceso a plug-ins que nos facilitan la vida. Podría ser algo como [Visual Studio Code](https://code.visualstudio.com/). Yo, personalmente, utilizo [Neovim](https://neovim.io/).

A pesar de utilizar Artix Linux, voy a intentar dejar las instrucciones bastante agnósticas para que los pasos sean similares aún estando en Ubuntu, Debian o alguna otra distro.

### Compilador (GCC)

Empecemos por el compilador. Aquí la opción open source es la versión de GCC para ARM: [arm-gcc-none-eabi](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads). Para instalarlo, descarguen la version más actual (o si buscan una específica); muy probablemente necesiten la versión x86_64 si tienen un PC con procesador intel. Una vez descargado, vayan a la carpeta y ejecuten ```tar gcc-arm-none-eabi-10.3-2021.10-x86_64-linux.tar.bz2``` para descomprimir. En su interior, deberían poder ver los siguientes folders: ```arm-none-eabi, bin, lib y share```.

A mí me gusta tener una carpeta ```~/opt/``` donde voy poniendo todos mis programas, y los uso directamente de ahí, o creo links simbólicos de forma que sean accedibles desde cualquier folder. Para hacer lo último, muevan la carpeta con el compilador ```mv gcc-arm-none-eabi-10.3-2021.10 ~/opt```. Y ya desde ahí podemos crear un link simbólico para acceder al compilador desde cualquier. ```cd ~/opt/gcc-arm-none-eabi-10.3-2021.10/bin && sudo ln -s ./* /usr/local/bin```. Y listo, el compilador está instalado. Para ver que todo esté correcto, deberían poder ver todos los ejecutables del compilador en con el comando: ```ls /usr/local/bin```:

``` none
arm-none-eabi-addr2line
arm-none-eabi-ar
arm-none-eabi-as
arm-none-eabi-c++
arm-none-eabi-c++filt
arm-none-eabi-cpp
arm-none-eabi-elfedit
arm-none-eabi-g++
arm-none-eabi-gcc
arm-none-eabi-gcc-10.3.1
arm-none-eabi-gcc-ar
arm-none-eabi-gcc-nm
arm-none-eabi-gcc-ranlib
arm-none-eabi-gcov
arm-none-eabi-gcov-dump
arm-none-eabi-gcov-tool
arm-none-eabi-gdb
arm-none-eabi-gdb-add-index
arm-none-eabi-gdb-add-index-py
arm-none-eabi-gdb-py
arm-none-eabi-gprof
arm-none-eabi-ld
arm-none-eabi-ld.bfd
arm-none-eabi-lto-dump
arm-none-eabi-nm
arm-none-eabi-objcopy
arm-none-eabi-objdump
arm-none-eabi-ranlib
arm-none-eabi-readelf
arm-none-eabi-size
arm-none-eabi-strings
arm-none-eabi-strip
```

Y al ejecutar ```arm-none-eabi-gcc --version``` desde la carpeta ```~/```, por ejemplo, deberían ver algo así:

``` none
arm-none-eabi-gcc (GNU Arm Embedded Toolchain 10.3-2021.10) 10.3.1 20210824 (release)
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

### HAL (libopencm3)

El Hardware Abstraction Layer (HAL), como su nombre lo indica, es una capa de software que permite desacoplar los detalles del microcontrolador del resto del stack de software. Por ejemplo, los diferentes microcontroladores que STM32 ofrece tienen diferentes periféricos, pero incluso entre los mismo periféricos, pueden tener diferentes features. Aún así, un GPIO sigue siendo un GPIO, y sin importar el microcontrolador o familia, debe soportar operaciones básicas como Set, Reset y Toggle de los pines. Los detalles de cómo lograr / habilitar esto a través de las diferentes familias, son detalles de los que se encarga el HAL.

Entonces, si tu aplicación está diseñada de forma que se aislan los detalles del hardware en la capa de la HAL, podrías tener código portable a través de diferentes microcontroladores del mismo fabricante. Por ejemplo, el siguiente código:

```C
/*
Habilita / deshabilita la etapa de potencia a través del
parámetro boolean enbl.
*/
void appl_mtr_enbl(bool enbl) {
  if(enbl) {
    GPIO_Set((uint16_t)APPL_H_BRIDGE_ENBL_PIN);
  } else {
    GPIO_Reset((uint16_t)APPL_H_BRIDGE_ENBL_PIN);
  }
}
```

Y debajo de la función ```GPIO_Set``` y  ```GPIO_Reset``` aislan la parte de la HAL:
```C
void GPIO_Set(uint16_t pin) {
  HAL_GPIO_WritePin(gsc_gpio_pin_map[pin].port, gsc_gpio_pin_map[pin].pin, GPIO_PIN_SET);
}

void GPIO_Reset(uint16_t pin){
  HAL_GPIO_WritePin(gsc_gpio_pin_map[pin].port, gsc_gpio_pin_map[pin].pin, GPIO_PIN_RESET);
}
```

Podemos ver que a la aplicación de control no le interesan los detalles del microcontrolador que va a habilitar / deshabilitar la etapa de potencia del motor (en este ejemplo). Ahora, no siempre es posible aislar totalmente los detalles del hardware, pero sin duda el esfuerzo necesario para aislar la mayor parte de los detalles bien que vale la pena.

Para instalar usar libopencm3 en nuestro proyecto, la forma más sencilla es clonar el template de un proyecto ejemplo que pueden encontrar aquí: https://github.com/libopencm3/libopencm3-template. Pueden clonar este proyecto ejemplo en la carpeta de su proyecto, y desde ahí compilar la librería junto con su proyecto. Sería algo así: ```git clone https://github.com/libopencm3/libopencm3-template ~/stm32-blinky```. El proyecto ejemplo tiene la siguiente estructura:

``` none
.
├── libopencm3
├── LICENSE
├── my-common-code
│   ├── api-asm.h
│   ├── api-asm.S
│   ├── api.c
│   └── api.h
├── my-project
│   ├── Makefile
│   └── my-project.c
├── README.md
└── rules.mk
```
La carpeta que contiene el HAL es ```libopencm3```, las carpeta ```my-common-code``` no nos sirve y podemos removerla. También podemos eliminar el archivo ```my-project.c``` en la carpeta ```my-project```. El Makefile lo vamos a utilizar pero le haremos modificaciones. Ya les iré diciendo qué vamos a cambiar. Primero cambiemos el nombre de la carpeta ```my-project``` a ```appl``` o el nombre de su preferencia. La estructura del proyecto debería quedarles algo así ahora:

```none
.
├── appl
│   ├── Makefile
├── libopencm3
│   ├── COPYING.GPL3
│   ├── COPYING.LGPL3
│   ├── doc
│   ├── HACKING
│   ├── HACKING_COMMON_DOC
│   ├── include
│   ├── ld
│   ├── lib
│   ├── locm3.sublime-project
│   ├── Makefile
│   ├── mk
│   ├── README.md
│   ├── scripts
│   └── tests
├── LICENSE
├── README.md
└── rules.mk
```

Para hacer el build de la librería, se puede especificar el Target (procesador) para evitar compilar para todos los targets que soporta la librería y ahorrar tiempo. Pero realmente el proyecto no es tan grande y no tarda mucho. Para compilar y generar documentación ejecuten en la consola: ```cd ~/Repos/libopencm3 && make && make doc```.

Como resultado van a tener los archivos ```*.a``` en la carpeta de ```lib/``` que se utilizarán en la etapa de linking para crear el binario que vamos a flahsear en el procesador. La documentación la encutran en la carpeta ```doc/```, en el archivo ```index.html``` que pueden abrir con cualquier navegador web. Y listo con el HAL por ahora.

En el archivo ```Makefile``` vamos a quitar lo que no necesitamos. Originalmente deberían tener algo así:

```makefile
PROJECT = awesomesauce
BUILD_DIR = bin

SHARED_DIR = ../my-common-code
CFILES = my-project.c
CFILES += api.c
AFILES += api-asm.S

# TODO - you will need to edit these two lines!
DEVICE=stm32f407vgt6
OOCD_FILE = board/stm32f4discovery.cfg

# You shouldn't have to edit anything below here.
VPATH += $(SHARED_DIR)
INCLUDES += $(patsubst %,-I%, . $(SHARED_DIR))
OPENCM3_DIR=../libopencm3

include $(OPENCM3_DIR)/mk/genlink-config.mk
include ../rules.mk
include $(OPENCM3_DIR)/mk/genlink-rules.mk
```
cambien a variable ```PROJECT``` con el nombre de su proyecto (blinky); pueden eliminar la variable ```SHARED_DIR```. La variable ```C_FILES``` solo va a contener el archivo ```main.c```, pueden eliminar los demás. En ```DEVICE``` utilicen el nombre de su procesador, para mí es un stm32f303ret6. Y para el ```OOCD_FILE``` ahorita no tiene mucha relevancia porque no estamos usando openocd, pero pueden configurarlo si gustan con el nombre de su tarjeta o procesador. Si instalan openopcd con el package manager de su distro, los archivos de configuración ```*.cfg``` están en ```/usr/local/share/openocd```, y ahí pueden buscar el archivo adecuado para su hardware. Yo usaré ```board/st_nucleo_f3.cfg```. Y es todo. El Makefile queda así:
```makefile
PROJECT = blinky
BUILD_DIR = bin

SHARED_DIR =
CFILES = main.c

# TODO - you will need to edit these two lines!
DEVICE=stm32f303ret6
OOCD_FILE = board/st_nucleo_f3.cfg

# You shouldn't have to edit anything below here.
VPATH += $(SHARED_DIR)
INCLUDES += $(patsubst %,-I%, . $(SHARED_DIR))
OPENCM3_DIR=../libopencm3

include $(OPENCM3_DIR)/mk/genlink-config.mk
include ../rules.mk
include $(OPENCM3_DIR)/mk/genlink-rules.mk
```



### st-link
Para instalar los tools de stlink, es mucho más sencillo hacerlo a través del package manager. Incluso es lo recomendado en su repositorio de github: https://github.com/stlink-org/stlink#Installation. Seleccionen el link que les funcione para la distro que utilizan. En mi caso, en Arch / Artix Linux a través del AUR: ```yay -S stlink-git```. Y listo.

Para asegurarnos que salió bien, el comando ```st-util --version``` debería mostrarles la versión. Algo similar a esto: ```v1.7.0-186-gc4762e6```. Aún mejor, conecten su tarjeta de desarrollo; yo estoy usando una [NUCLEO-F303RE](https://www.st.com/en/evaluation-tools/nucleo-f303re.html) con un procesador STMF303RE. Al ejecutar el comando ```st-info --probe``` obtengo lo siguiente:

``` none
Failed to parse flash type or unrecognized flash type

detected chip_id parametres

# Device Type: STM32F302_F303_F398_HD
# Reference Manual: RM0365           // also RM0316 (Rev 5)
#
chip_id 0x446
flash_type 1
flash_size_reg 0x1ffff7cc
flash_pagesize 0x800
sram_size 0x10000
bootrom_base 0x1fffd800
bootrom_size 0x2000
option_base 0x1ffff800
option_size 0x10
flags 2

Found 1 stlink programmers
  version:    V2J30S19
  serial:     066CFF515751836687133543
  flash:      524288 (pagesize: 2048)
  sram:       65536
  chipid:     0x446

detected chip_id parametres

# Device Type: STM32F302_F303_F398_HD
# Reference Manual: RM0365           // also RM0316 (Rev 5)
#
chip_id 0x446
flash_type 1
flash_size_reg 0x1ffff7cc
flash_pagesize 0x800
sram_size 0x10000
bootrom_base 0x1fffd800
bootrom_size 0x2000
option_base 0x1ffff800
option_size 0x10
flags 2

  dev-type:   STM32F302_F303_F398_HD
```

Estamos listos para escribir nuestro código.

## Hola, mundo embebido!

Volvamos al root de la carpeta ```st-blinky```. Creamos el main con el comando ```touch appl/main.c```. Aquí vamos a empezar la ejecución de nuestro primer programa de ejemplo.

Primero vamos a asegurarnos que el "toolchain" está bien instalado y podemos compilar un programa sencillo que ni siquiera haga uso del HAL; esto para aislar que el único punto de error pueda ser un toolchain mal instalado. Entonces vamos a escribir el ```main``` más simple posible:

```c
int main(void)
{
  return 0;
}
```

Navegamos a la carpeta ```appl``` e iniciamos el build: ```cd appl && make```. Si todo está correcto, deberían ver un mensaje más o menos así:

``` none
❯ make
  CC    main.c
  GENLNK  stm32f303ret6
  LD    blinky.elf
  OBJCOPY       blinky.bin
```

Ahora sí podemos continuar con el "Hola, mundo embebido!". Lo siguiente que tenemos que revisar es el diagrama de pines de la tarjeta; necesitamos saber a qué GPIO (General Purpose Input / Output) está conectado el pin que controla el LED que vamos a encender y apagar. Para este tipo de tarjetas nucleo, me gusta consultar el diagrama en la página de mbed (otro framework similar a Arduino para desarrollo de embebidos). Tienen unos diagramas muy amigables y rápidos de consultar en https://os.mbed.com/platforms/ST-Nucleo-F303RE/:

![MBED STM32 Pinout](/img/posts/stm32_mbed_pinout.png)

Podemos ver que el LED1 está conectado en el GPIO-A, pin 5. Comencemos a modificar el ```main```.

Lo primero que tenemos que hacer es incluir los "headers" para configurar el reloj del periférico y configurar el pin como salida digital. Es necesario configurar el reloj (clock) del periférico porque sin él, podemos configurarlo todo lo que queramos, pero la lógica digital no estará habilitada y no hará absolutamente nada. Los headers que tenemos que incluir son:

```c
/* Continúe funciones de configuración del reloj de periféricos
como: rcc_periph_clock_enable */
#include "libopencm3/stm32/rcc.h"

/* Contiene funciones de configuración y uso de GPIO como:
gpio_toggle */
#include "libopencm3/stm32/gpio.h"

int main(void)
{
  return 0;
}
```

Para habilitar el reloj del GPIO-A vamos a utilizar la función ```rcc_periph_clock_enable```. Esto es lo que podemos encontrar en la documentación de dicha función (recuerden el ```index.html``` que mencionamos arriba):

``` none
◆ rcc_periph_clock_enable()
void rcc_periph_clock_enable 	( 	enum rcc_periph_clken  	clken	) 	

Enable Peripheral Clock in running mode.

Enable the clock on particular peripheral.

Parameters
    [in]	clken	rcc_periph_clken Peripheral RCC

For available constants, see rcc_periph_clken (RCC_UART1 for example)

Definition at line 134 of file rcc_common_all.c.
```

Bastante útil, ¿no.? Aquí podemos ver que sólo necesitamos pasarle como parámetro de entrada el clock que queremos habilitar, y listo. El ```main``` quedaría algo así:

```c
/* Continúe funciones de configuración del reloj de periféricos
como: rcc_periph_clock_enable */
#include "libopencm3/stm32/rcc.h"

/* Contiene funciones de configuración y uso de GPIO como:
gpio_toggle */
#include "libopencm3/stm32/gpio.h"

int main(void)
{
  rcc_periph_clock_enable(RCC_GPIOA);
  return 0;
}
```

El código no es muy útil aún porque no hace nada, lo único que estamos haciendo es habilitar el periférico. Lo siguiente es configurar el pin del LED como salida (tipo push-pull para que pueda energizar directamente el LED):

```c
/* Continúe funciones de configuración del reloj de periféricos
como: rcc_periph_clock_enable */
#include "libopencm3/stm32/rcc.h"

/* Contiene funciones de configuración y uso de GPIO como:
gpio_toggle */
#include "libopencm3/stm32/gpio.h"

int main(void)
{
  /* Habilita el reloj hacia el puerto A */
  rcc_periph_clock_enable(RCC_GPIOA);

  /* Configura el puerto A, pin 5 como salida,
  con las resistencias de pull-up/down desactivadas. */
  gpio_mode_setup(
    GPIO_PORT_LED1,
    GPIO_MODE_OUTPUT,
    GPIO_PUPD_NONE,
    GPIO_PIN_LED1
  );

  /* Configura el pin 5 como salida tipo push-pull,
  a baja frecuencia (2MHz, la opción máxima es 100MHz) */
  gpio_set_output_options(
    GPIO_PORT_LED1,
    GPIO_OTYPE_PP,
    GPIO_OSPEED_2MHZ,
    GPIO_PIN_LED1
  );
  return 0;
}
```

Pero esto sigue sin ser totalmente útil. Sólo se está configurando el pin como salida, pero aún no existe el código que encienda y apague el LED de forma intermitente. Para esto haremos uso de la función ```gpio_toggle```, que hace justamente lo que el nombre sugiere: alterna el estado del pin, si está encendido, lo apaga, y vice versa. Esta es la documentación:

``` none
◆ gpio_toggle()
void gpio_toggle(uint32_t gpioport, uint16_t gpios)

Toggle a Group of Pins.

Toggle one or more pins of the given GPIO port. The toggling is not atomic, but the non-toggled pins are not affected.

Parameters
    [in]	gpioport	Unsigned int32. Port identifier GPIO Port IDs
    [in]	gpios	Unsigned int16. Pin identifiers GPIO Pin Identifiers If multiple pins are to be changed, use bitwise OR '|' to separate them.

Definition at line 87 of file gpio_common_all.c.
```
Suficientemente sencillo: sólo hay que pasarle el puerto (A) y el pin (5) como parámetro y listo.

Ahora, si lo hacemos de forma continua en el ```main``` el procesador lo hará tan rápido que sólo veremos el LED encendido siempre, sin apagarse. Para hacerlo visible necesitamos agregar un retardo entre cada toggle. Simularemos este retardo en un loop cuya única función es hacer perder tiempo al procesador par que nuestro ojo pueda notar el cambio en el LED. El código queda así:

```c
/* Continúe funciones de configuración del reloj de periféricos
como: rcc_periph_clock_enable */
#include "libopencm3/stm32/rcc.h"

/* Contiene funciones de configuración y uso de GPIO como:
gpio_toggle */
#include "libopencm3/stm32/gpio.h"

/* Macros para ayudar en la legibilidad del código. */
#define GPIO_PORT_LED1 (GPIOA)
#define GPIO_PIN_LED1 (GPIO5)
#define RCC_PORT_LED1 (RCC_GPIOA)
#define GPIO_LED1 (GPIO_PORT_LED1, GPIO_PIN_LED1)

int main(void)
{
  /* Habilita el reloj hacia el puerto A */
  rcc_periph_clock_enable(RCC_PORT_LED1);

  /* Configura el puerto A, pin 5 como salida,
  con las resistencias de pull-up/down desactivadas. */
  gpio_mode_setup(
    GPIO_PORT_LED1,
    GPIO_MODE_OUTPUT,
    GPIO_PUPD_NONE,
    GPIO_PIN_LED1
  );

  /* Configura el pin 5 como salida tipo push-pull,
  a baja frecuencia (2MHz, la opción máxima es 100MHz) */
  gpio_set_output_options(
    GPIO_PORT_LED1,
    GPIO_OTYPE_PP,
    GPIO_OSPEED_2MHZ,
    GPIO_PIN_LED1
  );

  /* Loop infinito para que el procesador siempre esté ejecutando
  la aplicación. */
  while(1) {

    /* Ocasiona que el procesador se retarde entre cada ejecución de la
    función gpio_toggle. Desperdicia tiempo a propósito. */
    for(volatile unsigned int tmr = 1e6; tmr > 0; tmr--);

    /* Enciende / apaga el LED. */
    gpio_toggle(GPIO_PORT_LED1, GPIO_PIN_LED1);
  }
  return 0;
}
```

¡Y listo! Ahora ejecuten de nuevo el comando ```make``` para compilar el código, y deberían tener en la carpeta ```appl``` el binario que escribiremos en la memoria flash del procesador, llamado ```blinky.bin```.

Lo siguiente sería escribir el código en la memoria flash del procesador. Para hacer esto vamos a utilizar el comando ```st-flash``` que instalamos arriba:

```bash
st-flash --reset write blinky.bin 0x8000000
```
Si lo hicieron correctamente, en la consola deberían tener un texto similar a este:
```none
❯ st-flash --reset write blinky.bin 0x8000000
st-flash 1.7.0-186-gc4762e6
Failed to parse flash type or unrecognized flash type

detected chip_id parametres

# Device Type: STM32F302_F303_F398_HD
# Reference Manual: RM0365           // also RM0316 (Rev 5)
#
chip_id 0x446
flash_type 1
flash_size_reg 0x1ffff7cc
flash_pagesize 0x800
sram_size 0x10000
bootrom_base 0x1fffd800
bootrom_size 0x2000
option_base 0x1ffff800
option_size 0x10
flags 2

2022-03-13T14:28:07 INFO common.c: STM32F302_F303_F398_HD: 64 KiB SRAM, 512
 KiB flash in at least 2 KiB pages.
file blinky.bin md5 checksum: da57d69c53496eb4ec76c198a846f154, stlink chec
ksum: 0x0000bea9
2022-03-13T14:28:07 INFO common_flash.c: Attempting to write 788 (0x314) by
tes to stm32 address: 134217728 (0x8000000)
-> Flash page at 0x8000000 erased (size: 0x800)

2022-03-13T14:28:07 INFO flashloader.c: Starting Flash write for VL/F0/F3/F
1_XL
2022-03-13T14:28:07 INFO flash_loader.c: Successfully loaded flash loader i
n sram
2022-03-13T14:28:07 INFO flash_loader.c: Clear DFSR
  1/  1 pages written
2022-03-13T14:28:07 INFO common_flash.c: Starting verification of write com
plete
2022-03-13T14:28:07 INFO common_flash.c: Flash written and verified! jolly
good!
```

Deberían ver su tarjeta encendiendo y apagando el LED cada segundo, aproximadamente:
![blinky stm32](/img/posts/stm32_dev_linux.gif)

Pueden encontrar el código en el Github de El Robotista; específicamente aquí: https://github.com/elrobotista/stm32_dev_linux.

¡Hasta la siguiente entrada!


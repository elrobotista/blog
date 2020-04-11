---
title: "Ejecución de comandos por puerto serial" 
date: 2020-04-09T20:37:12-05:00
draft: false
---

No sé si te ha pasado; pero al trabajar en le etapa de desarrollo de un proyecto, es común necesitar un tipo de interfaz a través de la cual ejecutar comandos de forma sencilla. Por ejemplo: ejecutar un test simple que encienda y apague leds o indicadores conectados a la salidas digitales para validar las conexiones, cambiar los parámetros de un controlador PID, cambiar el periodo o frecuencia de un PWM, cambiar la direccion de giro de un motor, etc.

Una manera sencilla de cambiar la funcionalidad del código es con switches conectados a las entradas digitales. El problema es que utilizarías entradas, de por sí limitadas, que tal vez sean necesarias para la finalidad de tu proyecto. Una mejor manera es a través del puerto serial de tu microcontrolador: implementar una terminal de comandos sencilla que permita ejecutar comandos, y enviar parámetros para alterar la funcionalidad de tu código.

Una de las formas más sencillas de implementar esta funcionalidad, sería similar a la propuesta en este link: https://www.baldengineer.com/simple-serial-commands-arduino.html. El problema surge en que no tiene una manera específica de enviar argumentos a las funciones, y que tendrías que recordar qué letra representa cuál domando.

Una mejor manera, propuesta por nosotros, sería algo así: como caso de uso, supongamos que estamos intentando evaluar el funcionamiento de un motor controlado por PWM. Nos interesa poder modificar la frecuencia y el ciclo de trabajo. Entonces podrías escribir las siguientes funciones:

``` c
extern Pwm_t pwm_obj;

void pwm_freq(int32_t freq_hz) {
    pwm_set_freq(&pwm_obj, freq_hz);
    return;
}
void pwm_dc(int32_t dc_pct) {
    pwm_set_dc(&pwm_obj, dc_pct);
    return;
}
```
que se ejecuten al recibir un comando a través del puerto serial:
``` c
> pwmfreq 23000
> pwmdc 55
```
donde el primer “string” denote el nombre del comando a ejecutar, seguido de un espacio, y por último el argumento a utilizar en la función. De manera general:
``` c
> cmd_name arg_value
```
Entonces, para conocer el comando a ejecutar, necesitamos una función que busque el índice del caracter de la barra de espacio. Lo que se encuentra antes del índice es el comando; y después, el argumento.
La función para encontrar la posición del caracter de la barra de espacios sería algo así:
``` c
/*
 * char ch: The character to be found.
 * char* raw_string: Pointer to the string that will be searched.
 * returns: Position of the @ch if found, -1 if @ch is not contained
 *          within @raw_string.
 */
int16_t find_char(char ch, char* raw_string) {
    uint16_t i = 0;
    int32_t idx = -1;
    for(i = 0; i < strlen(raw_string); i++) {
        if(raw_string[i] == ch) {
            idx = i;
            break;
        }
    }
    return idx;
}
```
NOTA: Si te das cuenta, la función `find_char` no valida los datos de entrada. ¿Qué problema podría haber? Te dejo esta pregunta, y encontrar una manera para validar los datos de entrada como ejercicio.
Ya que la función para encontrar la posición del caracter de espacio, podemos escribir la función para leer el argumento y el comando. Quedaría algo así:

``` c
#include <string.h>

const char SpaceChar = 0x20;
const char NullChar = 0x00;

/*
 * char* cmd_name: Pointer to the memory address where the @cmd_name will be stored.
 * char* arg_name: Pointer to the memory address where the @arg will be stored.
 * char* raw_string: Pointer to the @raw_string to be parsed. 
 * returns: value of 0 if parsed correctly, -1 otherwise.
 */
static int8_t read_cmd(char* raw_string, char* cmd_name, char* arg) {
    int err = -1;
    int32_t i = 0;
    uint8_t cmd_done = 0;
    size_t slen = strlen(raw_string);
    i = find_char(SpaceChar, raw_string);
    if(i > 0) {
        // Copy command name from raw_string into cmd_name.
        memcpy(cmd_name, raw_string, i);
        // In C, strings have to be null-terminated.
        cmd_name[i] = NullChar;
        // Now, copy the argument from raw_string into arg.
        // i + 1 acconts for skipping SpaceChar.
        memcpy(arg, &raw_string[i + 1], (slen - i));
        arg[slen - i + 1] = Nullchar;
        err = 0;
    }
    else {
        // Handle Error: invalid command.
        err = -1;
    }
    return err;
}
```
Excelente. Ya podemos leer el nombre del comando y el argumento. Pero aún no terminamos. Hace falta encontrar la dirección de memoria de la función que necesitamos ejecutar según el nombre del comando guardado en el apuntador `cmd_name`. Además, necesitamos convertir el string arg al valor numérico correspondiente.

Para convertir el argumento de string a decimal, la librería estandar de C cuenta con una función llamada `strtol`. Se usa de la siguiente forma:

``` c
char arg[] = "1200";

/* El argumento arg es el string que se convertirá a valor numérico.
 * El segundo argumento, en este caso NULL, es un char* que la función
 * modifica para apuntar al primer valor no-numérico encontrado en el 
 * string. Para nuestro caso, esto no nos interesa, de ahí usar el 
 * valor NULL. El última argumento: 10, representa que el número está 
 * en base decimal.
 */
int32_t argi32 = strtol(arg, NULL, 10);
```
Para encontrar la dirección de memoria propongo lo siguiente: crear una estructura que contenga como miembros el nombre del comando y la dirección de memoria que se debe llamar para ejecutar el comando. Algo así:

``` c
#define CMD_NAME_LEN_MAX (32)

typedef struct Command {
    char cmd_name[CMD_NAME_LEN_MAX];
    void (*callback)(int32_t arg);
} Command_s;
```
El miembro Callback es sólo un apuntador a la dirección de memoria donde se encuentra la función que necesitamos ejecutar. La Sintaxis es un poco confusa, pero es sólo eso: un apuntador.
Con esta estructura, podemos definir una tabla de comandos que nos permitirá agregar o quitar comandos de una forma muy sencilla:

``` c
#define CMD_TABLE_END {0x00, NULL}
/* Initialized somwhere in main by calling pwm_init. */
Pwm_t pwm_obj;
void pwmfreq_callback(int32_t freq_hz) {
    pwm_set_freq(&obj, freq_hz);
}
void pwmdc_callback(int32_t dc_pct) {
    pwm_set_dc(&obj, dc_pct);
}
const Command_s CmdTable[] = {
    {"pwmfreq", pwmfreq_callback},
    {"pwmdc",   pwmdc_callback},

    /* Keep this element last always.
      Used to calculate table size!*/
    CMD_TABLE_END,
};
```
En este caso es necesario definir el tamaño de la tabla. El compilador puede calcularlo de manera automática, y esto nos permite agregar y eliminar entradas de la tabla de forma sencilla. Probablemente te preguntes cuál es la funcionalidad del último elemento: `CMD_TABLE_END`. Sirve para que el código pueda determinar dónde se encuentra el final de la tabla; más delante verás la utilidad.
Entonces, la funcionalidad completa sería implementada de la siguiente forma:

``` c
#define ARG_NAME_LEN_MAX (16)
int16_t Cmd_Run(char* raw_str, Command_s* cmd_table) {
   int16_t err = -1;
   uint16_t i = 0;
   char cmd_name[CMD_NAME_LEN_MAX] = { 0 };
   char arg[ARG_NAME_LEN_MAX] = { 0 };
   int32_t arg_val;
   read_cmd(raw_str, cmd_name, arg);
   for (i = 0; (cmd_table[i].callback != NULL); ++i) {
      if (strcmp(cmd_name, cmd_table[i].cmd_name) == 0) {
         arg_val = strtol(arg, NULL, 10);
         cmd_table[i].callback(arg_val);
         err = 0;
         break;
      }
   }
   return err;
}
```
En el ciclo `for` se utiliza la funcionalidad que te había comentado: el `CMD_TABLE_END` es un `#define` cuyo callback es un apuntador `NULL`. Por lo tanto, podemos detectar cuando hemos llegado al final de la tabla, y evitar intentar acceder direcciones de memoria inválidas.

¡Listo! Está lista una mini-terminal de comandos sencillas que puedes utilizar a través de cualquier puerto serial. Gracias a que el código está hecho para trabajar con “strings”, independientemente de la interfaz: puedes leer una línea de caractéres a través de UART, CAN, SPI, Ethernet, o lo que sea.

Esta pequeña y simple librería para ejecutar comandos a través de un puerto serial puede ser mejorada de diferentas formas:

1. Validación de parámetros de entrada: en ningún lugar se valida si los apuntadores recibidos por las funciones son válidos. Esto puede ocasionar que el microcontrolador haga un “crash” por intentar leer una dirección de memoria inválida.
2. Qué tal si queremos implementar un comando que no reciba ningúna entrada? p.ej. hacer un “toggle” en una salida digital específica.
 3. Diferentes tipos de dato: `uint8_t`, `float`, `uint16_t`, etc.
 4. Implementar sistema de detección de errores más descriptivo. En este caso sólo utilizamos 0 para OK, y -1 para ERROR.
 5. Agregar un miembro “string” a la estructura `Command_s` que sirva como información de cómo utilizar el comando.

Sin embargo, para propósitos de esta entrada atenderemos estos puntos en una entrada futura, si les interesa. Por el momento, esta sencilla terminal puede ser utilizada en cualquier microcontrolador, debido a que el código está escrito en ANSI C. El problema podría surgir en el uso de funciones como `memcpy` y `strtol` en caso de su entorno de desarrollo no las proporcionara. Y en este caso, tendrían que ser implementadas por ustedes mismos, pero no debería ser mucho problema. Sin embargo, dejen sus comentarios si tienen este error y ¡con gusto les ayduamos!

Como siempre, podrán encontrar el código completo en https://github.com/elrobotista/simplecmd

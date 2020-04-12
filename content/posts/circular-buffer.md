+++
title = "Buffer circular en C"
date = 2017-04-06T12:25:09-05:00
tags = []
categories = []
imgs = []
cover = "/img/circular-buffer.jpg"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
+++

Los buffer circulares son bastante útiles en aplicaciones para sistemas embebidos: Para añadir la capacidad de “buffering” a los perifericos seriales del microcontrolador o para el ADC (convertidor analógico-digital), por ejemplo. Los buffer circulares no son más que “arrays” a los cuales se añaden datos. Cuando el buffer se llena, se debe tomar una de las siguientes dos decisiones: bloquear la escritura en el buffer hasta que haya espacio de nuevo; o permitir la escritura y desechar el dato más antiguo. La decisión depende de la aplicación. Eliminar el dato más antiguo permite olvidarnos de reservar memoria que no será utilizada al final sólo por evitar perder información, o preocuparnos por escribir datos fuera de la memoria reservada para el “array”; y malo ya que información es perdida si esta no es leída antes de que el buffer se llene. Una de las desventajas es que dificulta la implementación en un sistema concurrente (el cual es el caso, generalmente, en sistemas embebidos) si no se encuentra la manera de hacer las operaciones de escritura y lectura de forma atómica.

Podemos concluir que un buffer es una estructura de datos FIFO (First in, first out) relativamente sencilla de implementar. Lo especial del asunto viene cuando queremos diseñar un buffer circular para cualquier tipo de datos. En esta entrada los mostraré una implementación que no se enfoca en la eficiencia, sino en la claridad del código. En futuras entradas veremos cómo podemos mejorar el código presentado en esta entrada. Para facilitar la reusabilidad y claridad del código tomaremos un enfoque orientado a objetos. Esto podrá resultarles raro; hace unos años yo también estaba en la idea de que no se podía hacer programación orientada a objetos en C. Y en parte es verdad: C no soporta clases, herencia, polimorfismo, etc. Sin embargo, la programación orientada a objetos es un paradigma que va mas allá de si el lenguaje define el keyword “class”. Les dejo el link al siguiente video donde [Bob Martin explica cómo sí que es posible en C](#https://www.youtube.com/watch?v=TMuno5RZNeE).

La API del la librería muy sencilla. Tendrá las siguientes funciones:

``` C

/*
Inicializa el estado de la instancia @buf. @mem es la memoria definida para el buffer.
@datasz determina el tamaño de los datos. @bufsz determina el número de elementos que puede contener el buffer.
*/
int8_t Ring_buffer_init(Ring_buffer_t* buf, void* mem, uint8_t datasz, uint8_t bufsz);

/*
Guarda el dato @data en el buffer @buffer.
*/
int8_t Ring_buffer_push(Ring_buffer_t* buffer, void* data);

/*
Se retira el siguiente dato en el @buffer y se gurada en @data.
*/
int8_t Ring_buffer_pop(Ring_buffer_t* buf, void* data);

/*
Regresa 1 si @buffer esta lleno, de lo contrario: 0.
*/
uint8_t Ring_buffer_is_full(Ring_buffer_t* buffer);

/*
Regresa 1 si @buffer esta vacio, de lo contrario: 0.
*/
uint8_t Ring_buffer_is_empty(Ring_buffer_t* buffer);

```

Ahora les explico el por qué de la definición de las funciones. Lo primero que puede saltar a la vista es el uso de `void` como tipo para los datos que se desea guardar en el buffer. La palabra `void` nos permite decirle al compilador que no sabemos sobre qué tipo de datos serán utilizadas las funciones.
Esta característica es a que nos permite almacenar datos de tipo `int`, `float`, `char`, o cualquier tipo de struct que quieran utilizar. Debido a que el tipo de dato no está definido, entonces tampoco la memoria necesaria para almacenar los datos está definida: tenemos que llevar la cuenta de forma “manual”, por esto es que como argumento de la función se utiliza el tamaño del dato a almacenar en conjunto con el número de datos (tamaño del buffer) que queremos almacenar. La siguiente cuestión que salta a la vista es el argumento  `void* mem`. En una computadora podemos pedir al sistema operativo memoria de forma dinámica. En un sistema embebido con recursos tan limitados, se considera de mala práctica reservar memoria de forma dinámica. Entonces, para resolver esto, la memoria debe ser provista por el usuario. Todos estos parámetro son almacenados dentro del “objeto” para retener el estado de la memoria, el tamaño de los datos, etc.

El tipo de dato `Ring_buffer_t` está definido como:

```c

typedef struct Ring_buffer_s {
   uint8_t len;
   void* head;
   void* tail;
   void* first;
   void* last;
   uint8_t datasz;
   uint8_t bufsz;
   uint32_t count;
} Ring_buffer_t;


```

La variable `len` almacena el número de elementos que puede almacenar el buffer. La variable `head` apunta a la dirección de memoria que será escrita la siguiente vez que se introduzca un dato nuevo al buffer. La variable `tail` apunta al dato que sera devuelto la próxima vez que se decida retirar un dato del buffer. los apuntadores  `first` y `last` apuntan al inicio y al final de la memoria designada al buffer, respectivamente. `datasz` almacena el tamaño en memoria del tipo de dato que se almacena en el buffer. `datasz` almacena el número máximo de elementos que puede almacenar el buffer. `count` denota el número de datos que contiene el buffer actualmente. Cabe destacar que no es necesario que el usuario modifique las variables dentro de la estructura.

Veamos la función que inicializa la estructura:

``` c

int8_t Ring_buffer_init(Ring_buffer_t* buf, void* mem, uint8_t datasz, uint8_t bufsz)
{
   int8_t ret = -1;
   if(buf && mem && (datasz > 0U) && (bufsz > 0U) &&
   ((bufsz * datasz) <= BUFFER_SIZE_MAX) && (datasz <= bufsz)) {
      ret = 0;
      buf->head = mem;
      buf->tail = mem;
      buf->datasz = datasz;
      buf->bufsz = bufsz;
      buf->first = mem;
      buf->last = (mem + (bufsz - 1U) * datasz);
      buf->count = 0U;
   }
   return ret;
}

```

La función de inicialización regresa un dato `uint8_t` con la finalidad de poder informar al usuario si ocurre un error. Por esta razón, lo primero que se hace al entrar en la función es establecer el error. En las líneas 5 y 6 se pasa a probar si los parámetros sobre los cuales se llama la función son validos: se prueba que todos los apuntadores tengan un valor diferente de  `NULL`, que el tamaño de dato sea estrictamente positivo, que el tamaño del buffer pueda almacenar por lo menos un elemento. Se prueba, también, que el tamaño en memoria del buffer no exceda un máximo predefinido: como mencioné anteriormente, en un sistema embebido los recursos son limitados. Si estas condiciones se cumplen, entonces se tiene la seguridad que el resto de la función no puede fallar; por lo tanto, podemos establecer el valor de retorno como exitoso (0, en este caso). Como condición inicial, los apuntadores de lectura y escritura apuntan al mismo elemento ya que el buffer está vacío. El apuntador  `first`  apunta al primer elemento del buffer. Y el apuntador `last` al último elemento, y la dirección se define calculando el número de bytes máximo que puede almacenar el buffer.

Esta es la función para almacenar un nuevo dato:

``` c

int8_t Ring_buffer_push(Ring_buffer_t* buf, void* data)
{
    int8_t ret = -1;
    if(data && buf) {
        ret = 0;
        memcpy(buf->head, data, buf->datasz);
        buf->count = (buf->count == buf->bufsz) ? buf->count : buf->count + 1U;
        buf->head = (buf->head == buf->last) ? buf->first :
                                               buf->head + buf->datasz;
        if(buf->count == buf->bufsz) {
            buf->tail = buf->head;
        }
    }
    return ret;
}

```

De manera similar a la función de inicialización esta también puede fallar, por ello se establece el error al entrar en la función. A continuación se prueba que los apuntadores sean válidos. Si es así, entonces la función ya no puede fallar y se continúa con la lógica. En la línea #6 se copian los valores al buffer, pero para esto, el compilador necesita saber cuántos bytes va a copiar a la dirección de memoria deseada. Por esto es necesario inicializar el buffer con el número de bytes que ocupará en memoria los datos que serán almacenados en el buffer. En la línea #7, se actualiza la variable que almacena el número de elementos en el buffer. Si aún hay espacio para el nuevo elemento, se suma 1 a la variable. Si el buffer está lleno, se mantiene el valor ya que no hay espacio para otro elemento en el buffer. En la línea #8 se actualiza el apuntador de escritura. Esta es la parte peculiar de un buffer circular. Cuando los apuntadores (de escritura o lectura) llegan al final del buffer, los apuntadores son devueltos al inicio del buffer. Esta es la condición que se está evaluando en la línea #8. Si el apuntador ya está en el final del buffer, entonces se regresa al inicio del buffer; de lo contrario, se apunta al siguiente elemento en el buffer.

Ahora, echen un vistazo a lo que sucede en las lineas #10-#12. Ahí se maneja una condición peculiar: si el apuntador de escritura “dio vuelta” y ha alcanzado al apuntador de lectura, entonces en la presente llamada a la función  `Ring_buffer_push`, ¡el apuntador de escritura pasará al apuntador de lectura! Esta condición es invalida y se remedia actualizando el apuntador de lectura a la misma dirección que el de escritura. Esto solo sucede cuando el buffer está lleno y se intenta almacenar otro dato.

La siguiente función es la de lectura:

``` c

int8_t Ring_buffer_pop(Ring_buffer_t* buf, void* data) {
    int ret = -1;
    if(data && buf) {
        if(buf->count > 0U) {
            ret = 0;
            memcpy(data, buf->tail, buf->datasz);
            buf->count = (buf->count == 0U) ? : buf->count - 1U;
            buf->tail = (buf->tail == buf->last) ? buf->first :
                                                   buf->tail + buf->datasz;
        }
    }
    return ret;
}

```

Ya se lo imaginarán: establecer el código de error, y si los apuntadores son válidos, entonces continuamos con la ejecución. La condición de la línea #4 es clara: si el buffer esta vacío regresamos el código de error. En la línea #6 es donde copiamos el valor del buffer a la dirección de memoria a la que apunta  data. Les recuerdo: es de tipo `void` para poder utilizar estas funciones para cualquier tipo de dato. En las siguiente línea se actualiza el número de elementos pendientes en el buffer, y en la #8 se actualiza el apuntador de lectura. Si el apuntador llega al final del buffer, entonces se devuelve hacia el inicio de la dirección de memoria del buffer.

Y al final están las funciones para probar si el buffer está lleno o vacío:

``` c

uint8_t Ring_buffer_is_full(Ring_buffer_t* buf) {
    uint8_t ret = 0U;
    if(buf) {
        ret = (buf->count == buf->bufsz);
    }
    return ret;
}

uint8_t Ring_buffer_is_empty(Ring_buffer_t* buf) {
    uint8_t ret = 0U;
    if(buf) {
        ret = (buf->count == 0U);
    }
    return ret;
}

```
Estas son más sencillas: si el número de elementos pendientes en el buffer es igual al número de elementos que puede guardar el buffer, entonces el buffer está lleno. Y el caso contrario: si el número de elementos pendientes es 0, entonces el buffer está vacío.

Por último, déjenme les platico cómo se usaría esta librería con un ejemplo. Supongamos que queremos almacenar enteros  `uint32_t` (sin signo, de 32 bits):

``` c

#include <stdio.h>
#include <stdint.h>
#include "ring_buffer.h"

int main(void)
{
    Ring_buffer_t buffer;
    uint32_t mem[64U];
    Ring_buffer_init(&buffer, mem, sizeof(uint32_t), 64U);
    uint32_t tmp;
    /* Llenemos el buffer. */
    while(!Ring_buffer_is_full(&buffer)) {
        tmp = 2U * i;
        printf("Agregando: %u\n\r", tmp);
        Ring_buffer_push(&buffer, (void*)&tmp);
    }
    /* Ahora leamos y vaciemos el buffer. */
    while(!Ring_buffer_is_empty(&buffer) {
        Ring_buffer_pop(&amp;buffer, (void*)&amp;tmp);
        printf("Valor leido: %u\n\r", tmp);
    }
}

```
En la linea #8 se declara la memoria que será asignada al buffer para almacenar datos. Observen que en la inicialización, se le deja saber al módulo que cada dato que sea almacenado tendrá la medida de un tipo `uint32_t`; es decir, 4 bytes, y el buffer tendrá capacidad para 64 elementos de tipo `uint32_t`. En el ciclo `while`, se añaden datos al buffer hasta que este se llene. En el siguiente ciclo `while` se procede a obtener datos del buffer hasta que este quede vacío. Y esta es la manera como podemos hacer uso de un buffer circular en C para cualquier tipo de datos. Para otro tipo de datos sólo es necesario cambiar la medida del tipo de datos:  `sizeof(float)`, `sizeof(char)`, `sizeof(Cualquier_tipo_t)`, etc.

Podrán observar un problema con esta implementación: ¡La escritura y lectura no son atómicas! Es decir, si sucede una interrupción mientras se está leyendo el buffer, y el código de la interrupción modifica el apuntador de lectura (que sucede cuando el buffer se llena), entonces el dato regresado por  `Ring_buffer_pop`  estará corrupto. Y está situación no es improbable, ya que una de las principales aplicaciones para este tipo de buffer es dotar al módulo UART de un microcontrolador con un buffer de software cuando el microcontrolador no cuenta con uno en hardware. Entonces, la introducción del módulo UART alimenta el buffer con nuevos datos, mientras el `main` o el código de aplicación se alimenta de los datos del buffer. En una futura entrada veremos cómo podemos modificar está pequeña librería para que pueda ser “thread-safe”, para evitar este tipo de problemas y hacerla más eficiente.

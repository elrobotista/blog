+++
title = "Modelado de un motor de corriente directa"
date = 2017-04-05T16:43:36-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/modelado-motor-cd.jpg"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
+++

Si ustedes llevan o han llevado materias como dinámica de sistemas o control, entonces muy probablemente les haya sucedido lo siguiente. El profesor escribe algo así en el pizarrón:

Se tiene el siguiente sistema dinámico en lazo abierto en el dominio de la frecuencia:

$$G(s)=s+3s2+10s+1$$

Diseñe un controlador tipo PI para estabilizar el sistema y lograr el dempeño deseado…

la primer pregunta que se me vino a la mente fue:

¿Qué significa la $S$?

Las siguientes fueron: ¿Qué sistema es ese? ¿cómo “eso” describe el comportamiento de un sistema real? ¿cómo convierto un sistema real a “eso”? Y una vez que diseñe el controlador, ¿cómo llevo “eso” a la implementación real?

Esa S obviamente es la variable resultante de la transformación de Laplace. Pero esa no es realmente la respuesta a la pregunta. Como ingenieros, que no matemáticos, nos interesa el aspecto práctico de las ciencias. Ese es el trabajo de los ingenieros: apoyarse en la ciencia para crear/desarrollar tecnologías. ¿Cuál es el significado en la realidad de una transformación de Laplace?, ¿qué significa multiplicar o dividir por $S$ una señal?

Y si ya andamos en preguntas del significado físico de operadores matemáticos, ¿qué es una derivada o integral en la realidad?, ¿qué problema puedo resolver, usando como herramienta derivadas e integrales? Este tipo de preguntas me hice alguna vez, y que algunos profesores fallaron en responder, mientras que otros ampliaron la comprensión que tenía de estas herramientas. Entonces, intentaré responder estas preguntas apoyándome en la experiencia y conocimiento adquirido en mis años de practica ingenieril.

Empecemos con un ejemplo sencillo, pero que tiene muchísimas aplicaciones en la industria: el control de un motor de corriente continua. Un motor eléctrico es una maquina capaz de convertir energía eléctrica a trabajo mecánico.

![dc-motor-model](/img/posts/modelo-motor-cd.png)
*Figura 1.- Modelo de un motor de corriente directa.*

Entonces, podemos esperar que en el modelo matemático aparescan por allí ecuaciones de voltajes y corrientes acopladas a ecuaciones de fuerzas y velocidades.Confíen un poco en mi y acepten por un momento que la parte eléctrica del motor se puede modelar como se muestra a continuación:

$$L\frac{\mathrm{d}i}{\mathrm{d}t}+Ri+v_b+=v_{in},$$

donde $i$ es la corriente que consume el motor, $R$ y $L$ son la resistencia e inductancia del embobinado del motor, $v_{in}$ es el voltaje de entrada y $v_b$ es el voltaje de fuerza contra electromotriz.

Esa fue la parte eléctrica del motor. Ahora, la parte mecánica. La corriente que fluye por el motor, a través de inducción genera un torque sobre el eje del motor, entonces podemos escribir ecuaciones de movimiento de la siguiente manera:

$$J\frac{\mathrm{d}\omega}{\mathrm{d}t}+k_f\omega=\tau_m+\tau_l.$$

En esta ecuación ω es la velocidad del motor, J y kf son el momento de inercia y el coeficiente de fricción viscosa, τm es el torque que genera el motor y τl es la carga que afecta al motor: lo que quieres mover.

¿Pero cómo combinamos ambas ecuaciones? Qué bueno que preguntan. Resulta que el torque producido por el motor, depende de la corriente que consume el motor:

$$\tau_m=k_{\tau}i,$$

donde $k_{\tau}$ es conocida como constante de torque. Y la fuerza contra electromotriz, depende de la velocidad de giro del eje del motor:

$$v_b=k_b\omega,$$

y kc es la constante de fuerza contra electromotriz. Si sustituimos estas ecuaciones que relacionan variables eléctricas y mecánicas en las ecuaciones diferenciales, estamos combinando ambos sistemas: mecánico y eléctrico en un solo. El modelo quedaría algo así:

$$L\frac{\mathrm{d}i}{\mathrm{d}t}+Ri+k_b\omega+=v_{in},$$

$$J\frac{\mathrm{d}\omega}{\mathrm{d}t}+k_f\omega=k_{\tau}i+\tau_l.$$

¿Y de qué me sirve ese modelo a mí?

tal vez se preguntarán… como yo lo hice. Verán, tener un modelo matemático de un sistema tiene muchas ventajas. Algunas de estas son: permite estimar la respuesta del sistema a diferentes entradas. Estudiar el desempeño del sistema en lazo cerrado variando los parámetros del controlador. Conocer las condiciones de operación del motor para el desempeño deseado: cuanto voltaje, corriente o torque necesita el motor para mover una carga o seguir un perfil de movimiento. Así, podemos dimensionar adecuadamente el motor, la circuitería de potencia, la fuente de alimentación. O por otro lado, ¿qué tal si no contamos con sensor de velocidad, solo de posición, y necesitamos controlar la velocidad? Entonces nos vemos bajo la necesidad de diseñar, a parte del controlador, un algoritmo para estimar la velocidad a partir de una señal de posición. Y es aquí donde entran las derivadas. ¿Recuerdan que la velocidad es la derivada de la posición en el tiempo? Las simulaciones nos permitirán ver (¡antes de tener el sistema físico!) que derivar una señal de posición no siempre entrega resultados aceptables para estimar velocidad. Y veremos algoritmos un poco más complejos que sí que lo hacen.

Estas son algunas de las ventajas del diseño basado en modelo. Es un paradigma bastante utilizado en la industria que permite reducir los tiempos de entrega de productos y los costos de diseño, ya que permite la iteración rápida de cambios de diseño: ¿No funcionó? Bien, hagamos modificaciones y simulemos de nuevo. ¿Funcionó? ¿no? Repitamos. Funcionó, ahora simulemos con un modelo más complejo y preciso del sistema real. ¿Funcionó? Excelente, construyamos el prototipo.

Ahora, ¿qué hacemos con el modelo que tenemos? De seguro ya estarán pensando: ¿cómo sé cuál es el valor de $R,L,k_f,k_b,J$ para mi motor? Cuando estuve en ingeniería intenté medirlo… con un multímetro. No hagan eso. Resulta que hay maneras más sencillas: la hoja de datos del fabricante contiene estos valores. Este método no me ha resultado muy favorable, los valores divergen bastante del valor real del motor. Otra opción es usar herramientas de identificación de parámetros, pero al final de cuentas, no te arroja los valores reales del motor, ya que hay un conjunto infinito de soluciones si no conoces ninguno de los valores del motor. Hay un método más sencillo: no se necesita conocer los valores del motor. Veamos cómo.

Si aplicamos la transformación de Laplace en el sistema para encontrar la función de transferencia entre voltaje y salida, llegamos al siguiente sistema:

$$\frac{\Omega(s)}{V(s)}=\frac{k_{\tau}}{(Ls+R)(Js+k_f)+k_{\tau}k_b}.$$

Ahora, debido a la construcción de los motores de corriente directa, la inductancia $L$ es mucho menor que la resistencia $R$, entonces el termino $L/R$ se puede despreciar. ¿Por qué deberíamos hacer esto? Bueno, porque si despreciamos este factor, el sistema se convierte, de uno segundo orden, a uno de primer orden. Ingeniería es sobre hacer las cosas lo más simple posibles logrando el desempeño  adeucado. Más adelante les mostraré por qué tiene sentido despreciar este termino, analizando la respuesta del motor. Entonces, la función de transferencia toma la siguiente forma:

$$\frac{\omega(s)}{V(s)}=\frac{\frac{k_{\tau}}{R}}{Js+(k_f+\frac{k_{\tau}k_b}{R})}$$

¡El sistema tiene forma de filtro pasabajas! y se puede representar de la forma:

$$\frac{\Omega(s)}{V(s)}=\frac{K}{\tau s+1}.$$

¡Y listo! Nos olvidamos de $L,R,J$… y esas cosas que no podemos medir directamente. Sólo queda determinar la denominada ganancia de DC $K$ y la constante de tiempo del sistema $\tau$ (no confundir con torque). Estas se obtienen fácilmente a partir de la respuesta del motor a una entrada constante de voltaje. En la siguiente figura les muestro un ejemplo la respuesta real del motor en comparacion con el modelo a una entrada constante de voltaje:

![dc-motor-response](/img/posts/respuesta-modelo-motor-cd.png)
*Figura 2.- Respuesta del motor a un escalón de voltaje.*

La gráfica azul es la velocidad real del motor; la roja, el modelo. En la grafica se muestran dos puntos importantes: el tiempo que toma al motor en alcanzar por primera vez el 98% de la velocidad para un determinado voltaje, y la velocidad maxima del motor a 24v. A partir de estos dos puntos podemos caracterizar el motor con las siguientes ecuaciones:

$$K=\frac{\omega_{max}}{v_{in}},\tau=4T_s$$

Sustituyendo $\omega_{max},v_{in},T_s$, el modelo de primer orden del motor es

$$\frac{\Omega(s)}{V(s)}=\frac{145.47}{0.087s+1}.$$

Noten que las unidades de la ganancia $K$ son RPM/Volts. Es importante ser congruentes en las unidades a lo largo de toda la etapa de diseño e implementación.

Como se aprecia en la , el modelo de primer orden es adecuado para describir el comportamiento del sistema. Claramente no lo describe a la perfeccion. Un motor tiene efectos no lineales como friccion seca y juego mecanico, los cuales son bastante dificiles de modelar. En futuras entradas veremos como, a pesar de que el modelo es bastante sencillo, es de gran utilidad para simulación y diseño de controladores.

Ahora, ya que tenemos un modelo adecuado:

### ¿Cómo diseño un controlador para el sistema?

Este sera el tema que discutiremos en el siguiente post. En futuras entradas tambien veremos cómo escribir simulaciones para los sistemas que modelamos utilizando herramientas de codigo abierto. Y una vez diseñado el controlador, veremos como ir de la teoría y simulaciones a la implementación del controlador en un sistema embebido real.

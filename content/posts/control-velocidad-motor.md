+++
title = "Control de velocidad de un motor de corriente directa"
date = 2017-04-17T20:41:59-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/cover-control-velocidad-motor-cd.jpg"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
+++

En la [entrada anterior](https://jhestolano.com/posts/simulacion-sistemas/) vimos cómo podemos escribir una simulación en Python para estudiar la respuesta de un motor de corriente directa. En esta, estudiaremos cómo podemos utilizar el modelo del motor para diseñar un controlador de velocidad PI (Proporcional-Integral).

Más del 95% de las aplicaciones industriales de control, utilizan el control PID de alguna u otra forma. Esto, debido a la estructura sencilla del controlador y a que ha sido estudiado por más de 50 años. Pero al ser un controlador lineal, el desempeño del sistema se ve afectado frente a efectos no lineales despreciados en la etapa de modelado: fricción seca en sistemas mecánicos, variación de parámetros del sistema, retardos en la implementación digital del controlador, perturbaciones, dinámica de los sensores y actuadores, etc. A pesar de todo esto, si el controlador es diseñado de la manera adecuada, se puede lograr el desempeño deseado en muchos de los casos.

La estructura de un controlador PID se define de la siguiente forma:

$$u(t) = k_p e(t) + k_d\frac{\mathrm{d}e(t)}{\mathrm{d}t}+k_i\int_{t_0}^{t}e(\tau)\mathrm{d}\tau,$$

$$e(t)=r(t)-y(t),$$

donde $ki,kd,kp$ son las ganancias del controlador, $u$ es la señal de control, $e$ es el error, $r$ es el objetivo, e y la salida del sistema.El objetivo es encontrar la combinación de ganancias que permitirán que el sistema en lazo cerrado tenga el desempeño deseado.

En este caso, la señal de referencia $r$ es la velocidad deseada, que denotaremos como $\omega_r$, y la salida $y$ es la velocidad real del motor: $\omega$. El voltaje de entrada al motor $v_in$ será la salida del controlador $u$. El diagrama a bloques del sistema de control se aprecia en la Figura 1.

![closeed-loop-system](/img/posts/sistema-lazo-cerrado-control-velocidad.png)
*Figura 1. Sistema en lazo cerrado.*

Existen diversos métodos para sintonizar las ganancias del controlador: posicionamiento de polos, lugar de las raíces, diagramas de Bode, reglas de Ziegler-Nichols, etc. En esta entrada discutiremos el diseño del controlador utilizando el posicionamiento de polos. Esto debido a que me parece un método analítico y más intuitivo que los métodos de frecuencia y lugar de las raíces. Los métodos heurísticos de Ziegler-Nichols son de mucha utilidad cuando no se tiene un modelo preciso del sistema. En este caso, el modelo que tenemos es suficiente.

# Posicionamiento de polos
El método consiste en escoger las ganancias de tal forma que los polos del polinomio característico den al sistema el desempeño deseado. Esto puede parecer complicado, pero no lo es tanto. Vamos por pasos. Primero, definamos la función de transferencia del sistema en lazo cerrado:

$$G(s) = \frac{C(s)H(s)}{C(s)H(s)+1},$$

donde $H(s)$ es la función de transferencia del motor, y C(s) es la función de transferencia del controlador. El polinomio característico es el denominador de la función de transferencia del sistema cuando se vuelve cero:

$$H(s)C(s)+1=0 .$$

Si hacemos un poco de álgebra, podemos definir el polinomio característico del sistema en función de las ganancias del controlador:

$$s^2+ \frac{Kk_p+1}{\tau+Kk_d}s+\frac{Kk_i}{\tau+Kk_d}=0.$$

La ganancia $k_d$ no aporta beneficio claro al sistema, en el sentido de que podemos controlar los coeficientes del término $s$ y el término lineal sin necesidad de $k_d$, entonces desde el inicio definiremos $k_d=0$. Si se dan cuenta, el polinomio característico es de segundo orden tiene la forma:

$$s^2+2\zeta\omega_ns+w_n^2 =0.$$

Esto es importante, ya que la respuesta para sistemas de segundo orden está bien definida en función de los parámetros $\zeta$, denominado factor de amortiguamiento y la frecuencia natural $\omega_n$. Podemos usar parámetros de desempeño como el máximo “sobretiro” del sistema: cuanto sobrepasa el valor deseado el motor cuando se acerca a este por primer vez. El “tiempo de asentamiento”: cuánto tiempo tarda el sistema en lazo cerrado en reducir el error a un valor menor al 2%. El “tiempo pico”: el tiempo que tarda el sistema en llegar al momento de máximo sobretiro; entre otros. En este caso, utilizaremos el tiempo de asentamiento, que nos dará una manera de definir la rapidez con la que el sistema llega al valor deseado y el máximo sobretiro, que definirá la agresividad de la respuesta del sistema.

Utilizando estos parámetros de diseño, podemos definir la frecuencia natural $\omega_n$ y el factor de amortiguamiento $\zeta$ en función de los parámetros de diseño:

$$\zeta = \frac{|\ln{P_{os}}|}{\sqrt{\pi^2+\left(\ln P_{os}\right)^2}} , \quad \omega_n \approx \frac{4}{\zeta T_s},$$

donde $P_{os}$ es el máximo sobretiro en porcentaje, y $T_s$ es el tiempo de asentamiento deseado. Estas ecuaciones se encuentran en los libros de dinámica de sistemas en la sección de respuesta en el tiempo de sistemas de segundo orden.

Basándonos en la forma general de sistemas de segundo orden, podemos definir las ganancias del controlador en función de los parámetros de un sistema general de segundo orden:

$$k_p=\frac{2\zeta\omega_n\tau – 1}{K}, \quad k_i=\frac{\tau\omega_n^2}{K}.$$

Entonces, con esta ecuación y la anterior podemos encontrar las ganancias del controlador para lograr un comportamiento aproximado al deseado. Digo aproximado porque el sistema en lazo cerrado esta afectado por “ceros” que afectan la fase del sistema que no tomamos en cuenta a la hora de hacer nuestro análisis de posicionamiento de polos. Sin embargo, nos dará un excelente punto inicial, después del cuál podemos afinar la respuesta del sistema. Utilizando simulaciones numéricas podemos iterar rápidamente y evaluar el impacto que tiene en el sistema la variación de parámetros de $k_p$ y $k_i$. Es por eso que en la entrada anterior escribimos una simulación que hacía justo esto. Ahora, modificaremos ese código para adaptarlo a la variación de parámetros del controlador. Pero primero:

# ¿Cómo simular un controlador PI?
En la entrada [simulación de sistemas dinámicos](https://jhestolano.com/posts/simulacion-sistemas/) platicamos cómo podemos calcular derivadas e integrales usando métodos numéricos. Exactamente lo mismo se aplicará aquí. Utilizando la aproximación de la integral del error para el término integral del controlador:

$$\int_{t_i}^{t} e( \tau ) \mathrm{d} \tau  \approx \sum_{k=0}^{n} e[k] \Delta{t},$$

y la aproximación de la derivada para la parte derivativa:

$$\frac{\mathrm{d}e(t)}{\mathrm{d}t} \approx \frac{e[k]-e[k-1]}{\Delta{t}}.$$

Aunque en esta aplicación no usemos la parte derivativa del controlador ($k_d=0$), la implementaremos en la simulación, pues tal vez en futuras aplicaciones sea necesaria. El código de la simulación es muy similar al del motor de entradas anteriores. La diferencia es que ahora la entrada de voltaje no es un valor constante, sino que varía en función del error del sistema. El código es como sigue:

``` python
import matplotlib.pyplot as plt
import math
# Variables de simulacion.
ti = 0. # Tiempo inicial.
tf = 2. # Tiempo final.
ts = 0.001 # Tiempo de muestreo.
t = ti # Tiempo actual de simulacion.

# Variables del motor
K = 145.47 # Ganancia de DC del motor.
tau = 0.087 # Constante de tiempo,
w = 0. # Velocidad inicial del motor.
wr = 1000.0 # Velocidad deseada.

# Vectores de solucion.
W = [w] # Vector de soluciones.
T = [t] # Vector de tiempo.
U = [0.] # Vector de voltaje.
Wr = [wr / 1000.0] # Vector de velicidad deseada.

# Variables de PID
e = 0. # Error.
ie = 0. # Integral del error.
de = 0. # Derivada del error.
prev_e = 0. # Variable auxiliar: error anterior.

# Comportamiento deseado.
Ts = 0.2 # Tiempo de estabilizacion
Pos = 0.05 # Sobretiro maximo de 5%.

# Parametros de sistema de segundo orden.
z = abs(math.log(Pos)) / math.sqrt((math.log(Pos))**2 + math.pi**2)
wn = 4 / (z * Ts)

# Calcular ganancias en funcion de
# respuesta deseada.
kp = (2 * z * wn * tau - 1) / K
ki = (tau * wn ** 2) / K
kd = 0.

while t < tf:
   # Calcular error
   e = wr - w

   # Calcular derivada
   de = (e - prev_e) / ts
   prev_e = e

   # Calcular integral
   ie += e * ts

   # Calcular PID
   Vin = kp * e + kd * de + ki * ie

   # Calcular dinamica del motor
   dw = (K * Vin - w) / tau

   # Integrar para obtener velocidad.
   w += dw * ts

   t += ts # Siguiente muestra de tiempo.

   # Guardar en vectores para graficar.
   W.append(w / 1000.) # Convertir a RPM x 1000
   U.append(Vin)
   Wr.append(wr / 1000.) # Convertir a RPM x 1000
   T.append(t)

# Desplegar resultados.
plt.plot(T, W, linewidth = 2.)
plt.title('Respuesta del Motor con control PD')
plt.xlabel('Tiempo (s)')
plt.ylabel('Velocidad (RPM x 1000)')
plt.grid()
plt.show()

```
Se puede apreciar en la Figura 1 se puede apreciar la respuesta del motor. Si se dan cuenta, el motor se estabiliza en el valor deseado en un tiempo aproximado a los $0.2$ segundos deseados. Y el sobretiro es cercano al $5$%.

![closeed-loop-system-response](/img/posts/control-velocidad-respuesta.png)
*Figura 2. Respuesta del motor con control PI.*

Entonces podemos concluir que las ecuaciones que definimos son un buen punto de inicio para una sintonización más fina del controlador, si fuera necesario. Podemos correr múltiples simulaciones variando los parámetros de diseño (tiempo de asentamiento y sobretiro máximo), o los parámetros del controlador para evaluar el impacto que tienen en la respuesta del motor:

control-velocidad-respuesta-parametros.png
![closeed-loop-system-response-parameter-variation](/img/posts/control-velocidad-respuesta-parametros.png)
*Figura 3. Respuesta del motor a variación de parámetros de diseño.*

Se aprecia en la Figura 3 que la respuesta del sistema se ajusta decentemente a nuestra especificación, y por un instante pareciera que podemos definir el tiempo de respuesta $T_s$ del sistema tan pequeño como deseemos.  Pero en la Figura 4 podemos observar que para que el motor responda de la manera deseada, ¡el voltaje de entrada es superior a los $60$ volts! Este voltaje es muy superior a voltaje máximo especificado del motor que se está estudiando.

![closeed-loop-system-response-input-voltage](/img/posts/control-velocidad-respuesta-voltaje.png)
*Figura 4. Voltaje de alimentación del motor.*

Esta es la importancia del diseño basado en modelos. Si hubiéramos ido directo a la implementación, podríamos haber dañado motores diseñando el controlador. Sin embargo, con simulaciones podemos darnos cuenta que probablemente no podamos obtener un desempeño tan agresivo de parte del motor sin exceder las capacidades físicas de este. Entonces, podemos decidir un compromiso entre desempeño y longevidad del equipo electrónico y mecánico.

¿Se dan cuenta que esta simulación sólo nos da una estimación del voltaje de entrada?, ¿qué pasa con la corriente?, ¿cuánta corriente es necesaria, ademas de voltaje, para obtener el desempeño deseado? Esto es importante porque, el motor también tiene una especificación de corriente máxima, y el circuito de potencia que diseñemos debe ser capaz de proveer la corriente necesaria para que el motor funcione de la manera esperada. ¿Cómo podemos obtener una estimación de la corriente consumida por el motor? En futuras entradas volveremos al modelo en espacio de estados de segundo orden que obtuvimos en [modelado de un motor de corriente directa](https://jhestolano.com/posts/modelado-motor-cd/).

Esto no significa que después de todo, el modelo de primer orden que hemos estado estudiando sea inútil, ya que cumplió su propósito: nos permitió encontrar una relación para diseñar las ganancias del controlador. Ahora, para llevar a cabo simulaciones un poco más reales es común utilizar modelos un poco más complejos que modelen un comportamiento más acertado del sistema. Esto exploraremos en futuras entradas: cómo simular un sistema de orden $n>1$,  y empezaremos a platicar sobre la implementación en un sistema embebido real utilizando microcontroladores.

Acertijo: Existen aplicaciones donde es necesario controlar la posición de un motor, en lugar de la velocidad: fresadoras y tornos de control numérico, impresoras 3D, robótica, seguimiento de trayectorias con drones, etc. Siendo este el caso, ¿Cómo utilizarían el método de posicionamiento de polos para calcular las ganancias del controlador PID para lograr un desempeño de $Ts≈0.2$ y $Pos≈5$%?

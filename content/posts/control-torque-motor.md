+++
title = "Control de torque de un motor de corriente directa"
date = 2017-05-25T18:15:37-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/current-control-servo.jpg"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
math = true
+++

En esta entrada abordaremos el control PID de torque de un motor de corriente directa. En entradas anteriores les comenté cómo diseñar controladores de posición y velocidad, en esta entrada seguiremos un procedimiento similar, pero diferente para diseñar el controlador de torque.

¿Y por qué nos interesa el control PID de torque?, podrías preguntarte. Existen aplicaciones de robótica donde es necesario controlar el torque/fuerza producido por el actuador; es también común que el torque guarde una relación lineal con la corriente que consume el actuador. Debido a esto, el control de torque es importante en sistemas hápticos, o al manipular objetos que pueden ser dañados por esfuerzos elevados debido a los actuadores. Otra aplicación es el algoritmo de control utilizado en servodrives o servoamplificadores: es común encontrar el control PID “en cascada” o “anidado”. Y en futuras entradas exploraremos este algoritmo de control y sus ventajas frente a un controlador de posición o velocidad simple.

Conocemos bien el modelo lineal de segundo orden que hemos estado utilizando en las entradas anteriores:

$$ L\frac{di}{dt} + Ri +k_c\omega+ = v_{in}, \quad J\frac{d\omega}{dt}+k_f\omega=k_{\tau}i+\tau_l, $$

donde $L$ es la inductancia del motor, $R$ es la resistencia, $J$ es la inerncia del rotor, $k_f$ es el coeficiente de viscosidad, $k_c$ es la constante de fuerza contra-electromotriz, $v_{in}$ es el voltaje de alimentación del motor, o la señal de control. $k_{\tau}$ es la constante de torque del motor y $\tau_l$ es la carga del motor. i es la corriente y $\omega$ es la velocidad del rotor.

El torque producido por el motor debido a la energía eléctrica que consume guarda la relación lineal $\tau_m=k_{\tau}i$. La dinámica de la corriente que circular a través del embobinado del motor está definida por la primer ecuación del modelo dinámico. Si tomamos la ecuación en el dominio de Laplace, se tiene:

$$(Ls+R)I(s) = V_{in}(s)-k_b\Omega(s).$$

El término $k_b\Omega(s)$ es debido al voltaje que se opone al movimiento del motor, denominado fuerza contra-electromotriz, del inglés Back Electromotive Force. En la literatura y en las aplicaciones mencionadas anteriormente, este termino se omite en el diseño del controlador. Este termino es función de la velocidad del motor, que en comparación con la dinámica eléctrica del motor, es lenta, y se considera que no interfiere con el lazo de control de corriente. Otra razón es que en aplicaciones de velocidad constante, o donde la velocidad varía lentamente, este termino permanece constante con respecto al lazo de control de corriente, y el término integral del controlador compensa esta perturbación.

# Posicionamiento de polos
En nuestro caso haremos algo distinto: en aplicaciones de control de motores, es común tener un sensor del cual se puede obtener la velocidad, ya sea utilizando filtros, diferenciadores, o directamente si se utiliza un tacómetro. Asumiendo que la señal de velocidad está disponible, cancelaremos éste término introduciendo un voltaje equivalente $v_{eq}=v_{in}–k_b\omega$. Entonces la función de transferencia del sistema eléctrico es:

$$\frac{I(s)}{V_{eq}(s)} = \frac{\frac{1}{R}}{\frac{L}{R}s+1}.$$

¿Ves lo que yo estoy viendo? Otra vez el sistema de primer orden… ¡Para el que ya sabemos calcular las ganancias a partir de posts anteriores! Pero para evitar la monotonía esta vez seguiremos una ruta distinta. La constante de tiempo del sistema es $\tau=L/R$ y la ganancia en frecuencia cero es $K=R^{-1}$ (no confundir con la ganancia y la constante de tiempo utilizada en entradas anteriores).  Introduzcamos el controlador PI en el sistema en lazo cerrado. La función de transferencia sería:

$$\frac{I(s)}{I_d(s)}= \frac{\left(\frac{K}{\tau{s}+1}\right)(k_p + k_i/s)}{\left(\frac{K}{\tau{s}+1}\right)(k_p + k_i/s) + 1}.$$

No se ve muy sencillo, pero no te preocupes. Usemos un poco de algebra para simplificar la función de transferencia:

$$\frac{I(s)}{I_d(s)}= \frac{K(k_ps + k_i)}{\tau{s^2} + (Kk_p + 1)s + Kk_i}.$$

Ya no se ve tan mal, ¿no? Nota que el denominador de la función de transferencia es de segundo orden, como esperamos al introducir el control PID (con $k_d=0$) en el sistema. Podemos escoger la ubicación de los polos escogiendo de manera adecuada las ganancias $k_p$, $k_i$ para cancelar el cero en el numerador. Esto significa que queremos posicionar un polo en $s=−k_i/k_p$:

$$ \frac{I(s)}{I_d(s)}= \frac{K(k_ps + k_i)}{(k_ps+ki)(p_1s+p_0)}. $$

El valor del polo restante $s=−p_0/p_1$ está por determinar ya que depende de la posición del polo que queremos eliminar, y del desempeño en lazo cerrado deseado. Desarrollando la multiplicación del denominador, y por comparación, podemos establecer la siguiente relación:

$$ \tau{s^2}+(Kk_p+1)s+Kk_i=k_p{p_1}s^2+(k_p{p_0}+k_i{p_1})s+k_i{p_0}; $$

Podemos concluir que si posicionamos el polo restante en $s=−Kk_i$, el cero es cancelado con un polo y el grado del sistema se reduce a primer orden:

$$ \frac{I(s)}{I_d(s)} = \frac{1}{\frac{1}{Kk_i}s+1} $$

¡Y listo! No tenemos que preocuparnos por oscilaciones, sobretiro, factor de amortiguamiento ni nada de estas cosas de sistemas de segundo orden. Nota que la constante de tiempo del sistema se puede diseñar con la ganancia ki. Un relación útil para escoger la constante de tiempo del sistema en lazo cerrado es:

$$ T_s= 4\tau_{d}, $$

donde $\tau$ es la constante de tiempo deseada del sistema en lazo cerrado, y $Ts$ es el tiempo que le toma al sistema alcanzar el 98% de la corriente deseada, en nuestro caso. En una aplicación real no es posible escoger este valor arbitrariamente. Entre menor sea el tiempo requerido, mayor será el voltaje necesario, mayor será la frecuencia a la que deberá ser implementado el algoritmo digital, mayor deberá ser la frecuencia de de la modulación de ancho de pulso (PWM) de la etapa de potencia, con mayor cuidado deberá ser diseñada la circuitería para no introducir dinámica parásita, etc. Pero el propósito de esta entrada no es profundizar en estos aspectos, esto lo dejaremos para futuras entradas. Escogeremos un tiempo de respuesta $Ts=50ms$, ¿y el sobretriro? Bueno, pues recuerda que la respuesta del sistema eléctrico ha sido diseñada para ser de primer orden, entonces no tenemos que preocuparnos por el sobretiro y el factor de amortiguamiento como en entradas anteriores.

A partir de las relaciones anteriores concluimos que para que el controlador reduzca el orden del sistema, las ganancias deben ser escogidas según la siguiente relación:

$$k_i= \frac{4R}{T_s},\quad k_p=\frac{4L}{T_s}.$$

Vemos que es muy sencillo, entonces, escoger las ganancias del controlador cuando conocemos los parámetros eléctricos del motor. Fabricantes de motores utilizados en aplicaciones industriales comúnmente especifican los parámetros del motor la hoja de datos. Por ejemplo el motor fabricado por Pittman en cuál me basé para escribir esta entrada. Pero infinidad de motores que se venden por ahí en internet no tienen una hoja de especificaciones, y ahí está el problema. En futuras entradas implementaremos los algoritmos de control y describiré un método para estimar los parámetros del motor a través de experimentos.

# Simulación y control PID en python con scipy

Extenderemos el código que hemos venido utilizando en entradas anteriores para simular el control de corriente. En entradas anteriores programamos el control PID de una manera no muy… “elegante”. En esta entrada utilizaremos una clase para poder utilizarlo en futuras entradas, y no tener que reescribir el código una y otra vez. La case del controlador es así:

``` python
class PID_Controller(object):

    def __init__(self, kp=0., ki=0., kd=0., target=0., I_MAX=None, I_MIN=None,
                 ts = 1e-3):
        self.kp = kp
        self.ki = ki
        self.kd = kd
        self.ts = ts
        self.target = target
        self.error = 0.
        self._ie = 0.
        self._de = 0.
        self._preverr = 0.
        self.I_MIN = I_MIN
        self.I_MAX = I_MAX
        self.output = 0.
        self.feedback = 0.

    def reset(self):
        self._ie = 0.

    def step(self, y):
        self.error = self.target - y
        self._ie += self.error * self.ts
        if self.I_MAX is not None:
            self._ie = np.max((self._ie, self.I_MAX))
        if self.I_MIN is not None:
            self._ie = np.min((self._ie, self.I_MIN))
        _de = (self.error - self._preverr) / self.ts
        self._preverr = self.error
        self.output = self.kp * self.error + self.kd * _de + self.ki * self._ie
        return self.output
```

La clase es bastante sencilla. Acepta como argumentos opcionales las ganancias del controlador, que si no son especificadas, son cero. Esto es útil en caso de no necesitar las tres ganancias del control PID y solo un PD, un PI, un simple P. Los siguientes argumentos son los límites del integrador. Esto es para evitar que el algoritmo siga integrando sin límite. En la práctica es conocido como anti-windup, y la manera en la que está implementado aquí, es la más sencilla. De no ser especificado un valor para el integrador, no se limita a ningún valor. El último argumento es el periodo de muestreo del algoritmo. Este determina que tan frecuente se toma la integral y la derivada del controlador. En la práctica este valor depende de la dinámica del sistema y el poder de procesamiento del procesador que ejecuta el lazo de control. El método `step` calcula la acción de control según el estado del controlador, el valor actual de la señal a controlar y el valor deseado definido por la propiedad `target`. En este caso la señal a controlar es la corriente del motor. El método `reset` reinicia el integrador.

La siguiente clase es la que calcula la dinámica del motor, y que conocemos de entradas anteriores. Te recomiendo visitar la entrada de control de posición donde se explica a profundidad la clase:

La siguiente clase es la que calcula la dinámica del motor, y que conocemos de entradas anteriores. Te recomiendo visitar la entrada de control de posición donde se explica a profundidad la clase:

``` python

    def __init__(self, R = 1., L = 1., J = 1., kc = 1., kt = 1., kf = 1.):
        assert(R > 0. and L > 0. and J > 0. and kc > 0. and kt > 0. and kf > 0.)
        self.params = {
            'R':  R,   # Armature resistance.
            'kf': kf,  # Friction coefficient.
            'kc': kc,  # Back-emf constant.
            'L':  L,   # Armature inductance.
            'kt': kt,  # Torque constant.
            'J':  J,   # Rotor's moment of inertia.
        }
        self.vin = 0.
        self.load = 0.
        self.A = np.array([[0.,          1.,     0.],
                           [0.,    -kf / J,  kt / J],
                           [0.,    -kc / L,  -R / L]])

        self.B = np.array([[0.,    0.,   0.],
                           [0., 1./J,    0.],
                           [0.,    0., 1./L]])

    def dynamics(self, x, t, *args):
        try:
            self.A.reshape(3, 3)
            self.B.reshape(3, 3)
            dx = self.A.dot(x.reshape(3, 1)) +\
                 self.B.dot(np.array([0., self.load, self.vin]).reshape(3, 1))
        except Exception:
            raise ValueError('System dynamics dimension mismatch @ time {}'.format(t))
        return dx.flatten()

```

Para obtener las ganancias del controlador, utilizaremos la siguiente función:

``` python

def compute_gains_current_control(Ts, R = 0., L = 0.):
    assert(Ts > 0. and R > 0. and L > 0.)
    ki = 4. * R / Ts
    kp = 4. * L / Ts
    return kp, ki

```

Si te das cuenta, lo único que esta función hace es implementar la relación para calcular las ganancias que obtuvimos anteriormente. Nada especial en esta función, realmente. Ahora, a continuación te muestro el código de la simulación completa. A primera vista se ve bastante extendido, pero en realidad no es tan complicado:

``` python

import numpy as np
import math
import matplotlib.pyplot as plt
import scipy.integrate
from scipy.integrate import odeint
from motor import DCMotor
from controller import PID_Controller, compute_gains_current_control

def simulate(dynamics, x0, t0, tf, ts = 10e-3, simulation = None, args = ()):
    # Runs the simulation for the specified amount of time and given initial conditions.
    # This function also saves the data return by the simulation function specified in the
    # object constructor. The additional data to be saved must be returned as a set of tuples.
    # The data will be stored in a numpy array in the order that the simulation function
    # given by the user returns it. This is useful to store simulation data such as internal
    # signals as controller error, controller signal, time-varying parameters, etc.

    assert(ts > 0.)
    assert(tf > t0 + ts)
    assert(x0)
    assert(dynamics)

    datalog = None
    unpacked_data = None
    ldata = None
    time = np.arange(t0, tf + ts, ts)
    x = np.array(x0)
    solution = np.array(x0)
    for _t in time:
        if simulation is not None:
            ldata = simulation(x, _t, args = args)
        if ldata is not None:
            # If the simulation returned data, unpack the tuple and sotre
            # it in a temp variable to be processed.
            unpacked_data = _unpack_tuple(ldata)
            # If the simulation returned data and it has been unpacked,
            # store in the datalog array.
            datalog = _logdata(datalog, unpacked_data)
        if _t != time[-1]:
            # Integrator runs one step ahead of simulation.
            # Notice: _t  + ts.  It's one timestep ahead.
            x = odeint(dynamics, x, [_t, _t + ts])[1]
            solution = np.vstack((solution, x))
    return (time, solution, datalog) if datalog is not None else (time, solution)

def _unpack_tuple(tdata):
    tmp = np.array([])
    for data in tdata:
        tmp = np.hstack((tmp, data))
    return tmp

def _logdata(datalog, data):
    if datalog is None:
        datalog = data
    else:
        datalog = np.vstack((datalog, data))
    return datalog

def plot_results(t, y):
    for i, name in enumerate(['$\\theta\,[\mathrm{rad}]$', '$\omega\,[\mathrm{rad/s}]$',\
                              '$i\,[\mathrm{amps}]$']):
        ax = plt.subplot(3, 1, i + 1)
        ax.plot(t, y[:, i])
        ax.set_ylabel(name)
        ax.set_xlim([t[0], t[-1]])
    plt.xlabel('$Time\,\mathrm{[s]}$')

def plot_voltage(t, v):
    f, ax = plt.subplots()
    ax.plot(t, v)
    ax.set_xlim(t[0], t[-1])
    ax.set_ylabel('$V_{in}\,[\mathrm{Volts}]$')
    ax.set_xlabel('$Time\,\mathrm{[s]}$')

def run_current_control(m, x0, tf, ts):

    def pid_simulation(x, t, args = ()):
        motor, ctrl = args
        motor.vin = ctrl.step(x[2]) + x[1] * motor.params['kc']
        return (motor.vin,)

    i_des = 1.0 # Desired current: 1 Amp.
    kp, ki = compute_gains_current_control(50e-3, R = m.params['R'], L = m.params['L'])

    ctrl = PID_Controller(kp = kp, ki = ki, ts = ts)
    ctrl.target = i_des

    t, sol, vin = simulate(m.dynamics, x0, 0., tf, ts, simulation = pid_simulation, args = (m, ctrl))
    plot_results(t, sol)
    plot_voltage(t, vin)
    plt.show()

if __name__ == '__main__':
    tf, ts = 0.3, 1.0e-3
    x0 = [0., 0., 0.]
    m = DCMotor(R = 0.83, L = 2.31e-3, J = 2.37e-4, kc = 0.128, kt = 0.128, kf = 0.001697)
    run_current_control(m, x0, tf, ts)

```

La primer función `simulate` ejecuta el cíclo de la simulación. El primer argumento es la función que calcula la dinámica del sistema que se pretende simular, en este caso el motor. El segundo argumento es el vector de condiciones iniciales. El tercero y cuarto argumento son el tiempo inicial y final, respectivamente. Como argumentos opcionales está el “time-step” de la simulación, por defecto 10 milisegundos. El argumento `simulation` es una función que se ejecuta antes de calcular la dinámica del sistema. Esto es útil para calcular el voltaje de entrada del motor, perturbar el motor con un torque de carga, variar los parámetros del motor para probar la robustez del controlador, etc. Y el último argumento es una tupla con argumentos opcionales para la simulación.

las funciones `_logdata`, `_unpack_data` no son muy importantes: solo construyen un vector si la función pasada como argumento opcional `simulation` regresa una tupla. Esto es útil en casos en que se necesita inspeccionar señales que no son parte de la solución de la dinámica del sistema. Por ejemplo, en nuestro caso, las únicas señales que son solución de la dinámica son la corriente, posición y velocidad. Entonces, estas funciones nos ayudarán si necesitamos inspeccionar otras señales como el voltaje aplicado al motor, o la carga del motor regresándolos como tuplas al finalizar la función; ya lo verás más adelante. Al final está el `main` que inicializa y ejecuta la simulación.

Realmente, la función interesante es `run_current_control`. A continuación te muestro el código con comentarios en las partes importantes del mismo:

``` python
def run_current_control(m, x0, tf, ts):

    def pid_simulation(x, t, args = ()):
        # el motor y el controlador se encuentran dentro de los argumentos opcionales args,
        # como veran mas adelante.
        motor, ctrl = args

        # El voltaje de entrada es la salida del controlador, calculado con el metodo step. El
        # argumento x[2] es la corriente del motor. Recuerda que el controlador calcula el error
        # segun el parametro target que sera definido mas adelante. El termino
        # x[1] * motor.params['kc'] es la cancelacion del back-emf del motor.
        motor.vin = ctrl.step(x[2]) + x[1] * motor.params['kc']

        # Regresando el voltaje del motor como una tupla permite obtenerlo en el main
        # para visualizarlo.
        return (motor.vin,)

    # Corriente deseada de 1 Amperio.
    i_des = 1.0

    # Las ganancias del controlador se calculan segun las relaciones que definidas
    # anteriormente.    
    kp, ki = compute_gains_current_control(50e-3, R = m.params['R'], L = m.params['L'])


    # Se construye un controlador PI con las ganancias obtenidas.
    ctrl = PID_Controller(kp = kp, ki = ki, ts = ts)

    # Se programa la corriente deseada en el controlador.
    ctrl.target = i_des


    # Se ejecuta la simulacion, donde la dinamica del sistema esta calculada por el metodo
    # dynamics de la clase DCMotor. Las condiciones iniciales son x0. La simulacion dura
    # un periodo desde el tiempo inicial 0 hasta tf. Se utiliza la funcion pid_simulation
    # definida anteriormente para calcular el nuevo voltaje de entrada cada iteracion.
    # Como argumentos extra se define el motor m, y el controlador ctrl. Esto para poder
    # utilizarlos dentro de la funcion pid_simulation. De no ser asi, estos objetos
    # deben ser definidos como globales para poder ser utilizados en la funcion pid_simulation;
    # pero es considerada mala practica utilizar variables globales por razones que no
    # discutiremos en esta entrada.
    t, sol = simulate(m.dynamics, x0, 0., tf, ts, simulation = pid_simulation, args = (m, ctrl))
    plot_results(t, sol)
    plt.show()

```

Y listo. Si teclearon el código (los invito a teclearlo, y no copiarlo) de forma correcta, deberían ver gráficas similares a las siguientes:

![current response](/img/current-control-response.png)

Nota que el tiempo de respuesta del sistema es de $50ms$, como habíamos definido. Para el voltaje de entrada deberías ver algo similar a la siguiente gráfica:

![voltage-current response](/img/voltage-current-control.png)

Nota que a pesar de que la corriente se estabiliza en el valor deseado, el voltaje sigue aumentando. Esto es debido al término que cancela el “back-emf” del motor. La corriente genera un torque en el motor, lo que acelera el rotor, e incrementa cada vez más la velocidad del motor; por ende, el voltaje generado por el rotor se incrementa, y el controlador debe incrementar el voltaje de alimentación para sostener la corriente deseada.

# Discusión

En aplicaciones reales de control de torque, el rotor del motor no se deja girar libremente. Por ejemplo en una arquitectura de control en cascada, el lazo de control de corriente es utilizado para convertir el motor en un actuador donde la salida es torque, en lugar de velocidad, y la entrada es el torque deseado. Del modelo dinámico del motor puedes observar que el sistema eléctrico está “conectado” al sistema mecánico a través del término de torque kτi; pero ahora podemos controlar la corriente i directamente. Y dado que la respuesta del sistema eléctrico y el controlador de corriente es mucho más rápida que la dinámica del sistema mecánico, es posible hacer la aproximación:

$$J\frac{d\omega}{dt}+k_f\omega=k_{\tau}i_d+\tau_l,$$

donde podemos ignorar la dinámica del sistema eléctrico y podemos asumir que el torque producido por el motor es instantáneo. Esto, además de facilitar el diseño del controlador de velocidad, pues el sistema es, de nuevo, de primer orden, tiene el beneficio de que mejora la robustez del sistema. Dejaremos el análisis para una futura entrada, donde te platicaré cómo diseñar un controlador de velocidad y posición en cascada utilizando el controlador de corriente que diseñamos en esta entrada.

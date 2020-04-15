+++
title = "Control Velocidad Motor"
date = 2017-04-17T20:41:59-05:00
tags = []
categories = []
imgs = []
cover = "/img/cover-control-velocidad-motor-cd.jpg"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = true
+++

En la [entrada anterior](https://jhestolano.com/posts/simulacion-sistemas/) vimos cómo podemos escribir una simulación en Python para estudiar la respuesta de un motor de corriente directa. En esta, estudiaremos cómo podemos utilizar el modelo del motor para diseñar un controlador de velocidad PI (Proporcional-Integral).

Más del 95% de las aplicaciones industriales de control, utilizan el control PID de alguna u otra forma. Esto, debido a la estructura sencilla del controlador y a que ha sido estudiado por más de 50 años. Pero al ser un controlador lineal, el desempeño del sistema se ve afectado frente a efectos no lineales despreciados en la etapa de modelado: fricción seca en sistemas mecánicos, variación de parámetros del sistema, retardos en la implementación digital del controlador, perturbaciones, dinámica de los sensores y actuadores, etc. A pesar de todo esto, si el controlador es diseñado de la manera adecuada, se puede lograr el desempeño deseado en muchos de los casos.

La estructura de un controlador PID se define de la siguiente forma:

$$u(t) = k_p e(t) + k_d\frac{\mathrm{d}e(t)}{\mathrm{d}t}+k_i\int_{t_0}^{t}e(\tau)\mathrm{d}\tau,$$

$$e(t)=r(t)-y(t),$$

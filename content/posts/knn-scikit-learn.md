+++
title = "Clasificacion de images satelitales"
date = 2020-04-11T18:15:37-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/cover-knn.jpg"  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = true  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
math = false
featured_image = ""
+++

En esta entrada abordare la implementacion de un algoritmo de agrupamiento supervisado llamado K-Nearest-Neighbor o KNN de manera abreviada. El objetivo sera tomar un conjunto de datos que contiene imagenes de banda satelites y clasficiar entre tierra y agua. 

El material para todo este ejercicio pueden encotrarlo en las siguientes ligas.

[Conjunto de entrenamiento](https://gist.github.com/Marckhz/593f41a78eeedfc2adc53e187f842027)

[Bandas satelitates](https://gofile.io/d/8elxn3)

Intuicion del algoritmo. La idea es clasificar un punto P en el espacio, donde de los K vecinos el punto P se encuentre el mas frecuente se toma el mas cercano de estos. La distancia es calculada utilizando la distancia euclidiana.

$$d(x_i,x_j) = \sqrt{\sum_{r=1}^{p}(x_ri - x_rj)^2}$$

``` python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from PIL import Image
```

Utilizaremos como apoyo la libreria scikit-learn, en ella se encuentra la clase KNeighborsClassifier.


Primero cargaremos el conjunto de entrenamiento. Este conjunto de datos no es nada mas que una matriz. Por lo tanto utilizaremos numpy.

```python
trainingset = np.loadtxt("rsTrain.txt", dtype="uint8")
trainingset.shape
```

Como pueden observar, es una matriz de 5 columnas con con 200 renglones.Vamos a divir esta matriz en dos variables. Dentro de esta matriz pueden encontrar en la ultima columna los valores 1 y 0. Esos valores indican entre agua o tierra.

```python
X = trainingset[:,[0,1,2] ]
y = trainingset[::,-1]
```

Ahora carguremos las bandas que vamos a clasificar.

```python
list_of_bands = [
                        np.fromfile("band1.irs", dtype="uint8"), 
                        np.fromfile("band2.irs", dtype="uint8"),
                        np.fromfile("band3.irs", dtype="uint8"),
                ]       
```

Como existe curiosidad, vamos a mostrar como lucen estas bandas previo a la aplicacion del algoritmo. 

```python
fig, axes = plt.subplots(1,3, figsize = (12,4) )
for band in range(len(list_of_bands)):
  list_of_bands[band].shape = (512,512)
  axes[band].imshow(list_of_bands[band])
```

Codigo paso a paso. En la primera linea, declaramos el canvas donde vamos plotear las imagenes. Luego, dentro del ciclo, lo que hacemo es iterar sobre la longitud de la lista donde tenemos cargadas las bandas, les damos formato de 512x512 para luego tomar la posicion del iterador, seleccionar la banda y plotear. 

![satelite-bands](/img/posts/satelite.png)

Este siguiente paso es importate, vamos a concatenar las bandas y a cada banda vamos a darle un formato de vector columna, para cuando clasifiquemos cada banda tenga la misma dimension que el conjunto de entrenamiento. Me refiero a Nx1.

```python
mp = np.c_[
           list_of_bands[0].reshape(-1,1),
           list_of_bands[1].reshape(-1,1),
           list_of_bands[2].reshape(-1,1),
          ]
mp.shape
```
```python
from sklearn.neighbors import KNeighborsClassifier
```
Crearemos diferentes modelos para los diferentes cantidades de vecinos.

```python

y_pred = None
list_of_predictions = []
k = [10, 25, 50, 100]
scores = dict()

for num_of_neighbors in k:

  model = KNeighborsClassifier(n_neighbors=num_of_neighbors)
  model.fit(X,y)
  scores[str(num_of_neighbors)] = model.score(X,y)
  y_pred = model.predict(mp)
  y_pred.shape = (512, 512)
  y_pred = np.where(y_pred==1, 255,y_pred)
  list_of_predictions.append( y_pred )
```
Linea por linea. Primero, declaramos una variable y_pred para guardar las predicciones, sera como apoyo.
Luego, una lista para almacenarlo, esto por simple comodidad para luego mostrarlas uno por una. Una lista K que contenga los numeros de vecinos y un diccionario para guardarlos scores segun su K. 

Dentro del ciclo, creamos una instancia de la clase KNeighborsClassifier y le pasamos al constructor el numero de vecinos que queremos. Hacemos un fit(). Fit hace referencia  a lo que es el training y calcula los coeficientes. Obtenemos el score. 
Realizamos lo que es el predict contra el modelo y las bandas. 
Tomamos el objeto y le damos una dimesion de 512x512.
Luego remplazamos dentro del objeto remplazamos los 1 por 255, por que en RGB 255 es el color blanco. 

Ahora revimos cada imagen. 

```python
from PIL import Image
Image.fromarray(list_of_predictions[0])
scores['10']
```
Y asi es como se visualiza la imagen ya clasificada.

![K-equals-ten](/img/posts/prediction-satelite.png)

Y este es el mapa original tomada de Google Maps. [kolkata](https://www.google.com/maps/place/Calcuta,+Bengala+Occidental,+India/@22.6206287,88.3571094,11.04z/data=!4m5!3m4!1s0x39f882db4908f667:0x43e330e68f6c2cbc!8m2!3d22.572646!4d88.363895)

Conclusion. Es facil poder implementar estos tipos de algortimos utilizando librerias como scikit-learn. Sin embargo, hago enfasis en que es importante conocer el algoritmo, analizar la matematica y sobre todo digerirlo. 
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

En esta entrada abordare la implementacion de un algoritmo de agrupamiento supervisado llamado K-Nearest-Neighbor o KNN de manera abreviada. El objetivo sera tomar un conjunto de datos que contiene imagenes de bandas satelitales y clasificar entre tierra y agua. 

Este es un tipo de problema de aprendizaje supervisado. La razon, en el conjunto de datos de entrenamiento conocemos los resultados desados, es decir, las etiquetas. Un problema tipico de aprendizaje supervisado es la de clasificacion. 


Estaremos apoyandonos con la libreria scikit-learn, pandas, numpy, matplot y al final Pillow. Tambien, hablare un poco del tema de paralelizacion. 

El material para todo este ejercicio pueden encotrarlo en las siguientes ligas.

[Conjunto de entrenamiento](https://gist.github.com/Marckhz/593f41a78eeedfc2adc53e187f842027)

[Bandas satelitates]()

Intuicion del algoritmo. La idea es clasificar un punto P en el espacio, donde de los K vecinos, el punto P se encuentre el mas frecuente de estos y se toma el mas cercano. La distancia es calculada utilizando la distancia euclidiana.

$$d(x_i,x_j) = \sqrt{\sum_{r=1}^{p}(x_ri - x_rj)^2}$$

``` python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from PIL import Image
```
Primero cargaremos el conjunto de entrenamiento. Este conjunto de datos no es nada mas que una matriz. Por lo tanto utilizaremos numpy.

```python
trainingset = np.loadtxt("rsTrain.txt", dtype="uint8")
trainingset.shape
```

Como pueden observar, es una matriz de 5 columnas con con 200 renglones.Vamos a dividir esta matriz en dos variables. Dentro de esta matriz pueden encontrar en la ultima columna los valores 1 y 0. Esos valores indican entre agua o tierra.

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

Codigo paso a paso. En la primera linea, declaramos el canvas donde vamos plotear las imagenes. Luego, dentro del ciclo, lo que hacemos es iterar sobre la longitud de la lista donde tenemos cargadas las bandas, les damos formato de 512x512 para luego tomar la posicion del iterador, seleccionar la banda y plotear. 

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
cross_val_holder = []
for num_of_neighbors in k:

  model = KNeighborsClassifier(n_neighbors=num_of_neighbors, n_jobs=-1)
  cross_val_holder.append( cross_val_predict(model, X,y, cv=3) )
  model.fit(X,y)
  y_pred = model.predict(mp)
  y_pred.shape = (512, 512)
  y_pred = np.where(y_pred==1, 255,y_pred)
  list_of_predictions.append( y_pred )
```
Linea por linea. Primero, declaramos una variable y_pred para guardar las predicciones, sera como apoyo.
Luego, una lista para almacenarlo, esto por simple comodidad para luego mostrarlas uno por una. Una lista K que contenga los numeros de vecinos.

Dentro del ciclo, creamos una instancia de la clase KNeighborsClassifier y le pasamos al constructor el numero de vecinos que queremos y el parametro n_jobs=-1. Este ultimo parametro lo que haces utilizar todos los procesadores de nuestro CPU. Mucho de esa funcionalidad, utiliza directivas de C para parelizar, por ejemplo, OPENMP. 


Veamos a que me refiero

```C
int counter = 0;
#pragma omp parallel for reduction(+:counter)
for(int i = 0; i < n; i++){
  counter +=1
}
```
A un nivel, muy superficial e incluso, de manera de intucion podriamos decir que algo similar sucede asi en python. 
Lo que esta sucediendo es que al llamar el #pragma  con reduction y aputando a la variable que queremos proteger, estara dividiendo las operaciones dentro de ciclo y al terminar hara una combinacion de estas, en este caso, la suma de counter +=1;

Imagina un arreglo de una N dimesion, y tu CPU es de 4 cores, tendras 4 hilos, divide el arreglo en 4 y cada hilo hare una operacion en su segmento y al terminar este, hare la combinacion. Algo importante a mencionar, hay muchas cosas mas detras que suceden, como la region critica, el alcance de los datos, y las variables, sin embargo, queria darte una intucion si es que jamas has trabajado con paralelismo.

Regresando al codigo.


En la lista cross_val_holder, vamos guardando los resultados del cross validation Hacemos un fit(). Fit hace referencia  a lo que es el training y calcula los coeficientes. 
Realizamos lo que es el predict contra el modelo y las bandas. 
Tomamos el objeto y le damos una dimesion de 512x512.
Luego remplazamos dentro del objeto remplazamos los 1 por 255, por que en RGB 255 es el color blanco. 

Ahora revisemos cada imagen. 

```python
from PIL import Image
Image.fromarray(list_of_predictions[0])
```
Y asi es como se visualiza la imagen ya clasificada.

![K-equals-ten](/img/posts/prediction-satelite.png)

Podemos analisar la calidad de nuestro modelo. Normalmente, en el proceso de aplicar un modelo predictivo es mandatorio dividir entre un test set y un train set. En nuestro caso, nos saltamos esa parte, dado que este conjunto de dato, esta curado para propositos educativos y prototipado rapido para conocer el algoritmo y sus alcances.

Para analizar la calidad del modelo existen diferetes formas, en nuestro caso, vamos a utilizar una matriz de confusion dada que es muy simple de usar.

```python
from sklearn.metrics import confusion_matrix
y_train_1, y_train_2, y_train_3, y_train_4 = cross_val_holder
confusion_matrix(y_train_1, y)
```

Ahora, que sucede? Estamos visualizando la matriz de confusion para el primer modelo contra los labels reales.
En una matriz de confusion los renglones son las clases actuales y las columnas las predicciones.
Podemos observar en el primer renglon, el primer valor serian, los negativos verdares  y en el segundo los falsos positivos.
En el siguiente renglon, los falsos negativos y los verdaderos positivos.

Si instalaste la version de sklearn 0.23, te sera posible graficar esta matriz, importando plot_confusion_matrix.

Aqui podras encontrar la ubicacion en  Google Maps de la imagen que clasificamos. [kolkata](https://www.google.com/maps/place/Calcuta,+Bengala+Occidental,+India/@22.6206287,88.3571094,11.04z/data=!4m5!3m4!1s0x39f882db4908f667:0x43e330e68f6c2cbc!8m2!3d22.572646!4d88.363895)

Conclusion.
----

 Es relativamente sencillo poder implementar estos tipos de algortimos utilizando librerias como sklearn. Sin embargo, hago enfasis en que es importante conocer el algoritmo, analizar la matematica y sobre todo digerirlo. Definitivamente, hay mejores algoritmos para clasificar imagenes, pero siendo este que utilizamos aqui, uno muy bueno, dado que, al entenderlo, podemos utilizar para prototipar otras cosas que el tipo de problema se pueda resolver mediante clusterizacion.
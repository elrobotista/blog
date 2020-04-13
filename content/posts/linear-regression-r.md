+++
title = "Regresion Lineal aplicada en lenguaje R"
date = 2020-04-11T18:15:37-05:00
tags = []
categories = []
imgs = []
cover = ""  # image show on top
readingTime = true  # show reading time after article date
toc = true
comments = false
justify = false  # text-align: justify;
single = false  # display as a single page, hide navigation on bottom, like as about page.
license = ""  # CC License
draft = false
math = true
+++

En este articulo abordaremos uno de los modelos basicos y mas utilizados en la estadistica, la Regresion Lineal.
Nos apoyaremos del lenguaje R para implementar el algoritmo y lo aplicaremos en un conjunto de datos de peliculas
extraida de diversas fuentes.Exploraremos el conjunto de datos, graficaremos, mostraremos estadisticos y mas. 

El objetivo es tener un modelo que deje el mejor score para predecir que elementos, debe tener una pelicula para
que esta, obtenga un buen score en Rotten Tomatoes y IMBD.

Puedes trabajar desde la consola de R o cualquier editor de texto que preferias, sin embargo, yo sugiero que utilices R Studio.

A continuacion te dejo los paquetes de R para trabajar el material y el conjunto de datos.

* ggplot2
* dplyr
* GGally
* gridExtra
* cowplot
* Movies Dataset

Dividiremos en seis partes o pasos a seguir en la realizacion de esta tarea. 

1. Explicacion del conjunto de datos.
2. El planteamiento de la pregunta de investigacion.
3. Analisis Explotario de Datos.
4. Modelado
5. Prediccion
6. Conclusion.

### Parte 1 Los DATOS:
De donde provienen los datos?
Los datos fueron recolectados de los sitios Rotten Tomatoes y IMDb. El conjunto de datos es una muestra aleatoria 
de ambos sitios que contienen diferentes peliculas, generos y sus respectivas metricas de cada sitio.

### Parte 2 Pregunta de investigacion
Cuales son las caracteristicas minimas adecuadas para obtener una calificacion decente en ambas plataformas?

### Parte 3 EDA (Analisis Exploratorio de Datos)

#### Cargar Paqueterias
``` R
library(ggplot2)
library(dplyr)
library(statsr)
library(GGally)
library(gridExtra)
library(cowplot)
```

#### Cargar conjunto de Datos
```R
load("movies.Rdata")
```

Claramente existen diferentes generos de peliculas, pero, tienen algo en comun? Probablemente si. Entonces podriamos construir algun tipo de formula para elaborar una pelicula que siempre obtenga un buen score por parte del publico y de los criticos? Probablemente.

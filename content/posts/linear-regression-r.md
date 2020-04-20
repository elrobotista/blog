+++
title = "Explorando un conjunto de datos"
date = 2020-04-11T18:15:37-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/movies-cover.jpg"  # image show on top
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


En este articulo abordaremos uno de los modelos basicos y mas utilizados en la estadistica, la Regresion Lineal. Nos apoyaremos del lenguaje R para implementar el algoritmo y lo aplicaremos en un conjunto de datos de peliculas extraida de diversas fuentes .Exploraremos el conjunto de datos, graficaremos, mostraremos estadisticos y mas. 

El objetivo es tener un modelo que deje el mejor score para predecir que elementos, debe tener una pelicula para
que esta obtenga un buen score en Rotten Tomatoes y IMBD.

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
movies <- read.csv(url("https://gist.githubusercontent.com/Marckhz/e187f1114f9cb850600bfa739763703d/raw/a29cc73c593e03671d8c9ffcd3e28bf1a0853d1f/movies.csv") )
```
Claramente existen diferentes generos de peliculas, pero, tienen algo en comun? Probablemente si. Entonces podriamos construir algun tipo de formula para elaborar una pelicula que siempre obtenga un buen score por parte del publico y de los criticos? Probablemente.

Me gustaria saber que columnas contiene el conjunto de datos. Para esto utilizaremos la funcion 

``` R
names(movies)
```
![names data](/img/posts/movies-names-out.png)
Ahora con esto tenemos las columnas y podemos darnos una idea con que tipo de datoes estamos lidiando. En algunos escenearios cuando los datos son recolectados es probable que te briden un CodeBook, donde, se te informa el tipo de datos, que contiene y quiza algo extra.

Si ese es tu caso, donde se dieron los datos, no se te proporciona alguna informacion, el lenguaje R nos proporciona la siguente funcion.

```R
str(movies)
```
![struct data](/img/posts/movies-str-data.PNG)

De esta forma podemos observar cada atributo con su respectivo tipo de dato.

Ahora bien, inspeccionemos el atributo runtime, sabemos que es un atributo numerico y por su nombre runtime sabemos que se habla acerca de la duracion de cada pelicula. Podemos generar estadisticos significativos de la siguiente manera y tambien graficaremos.  

```R
movies %>% select(runtime) %>% summary(runtime)
```
![summary stats](/img/posts/summary_stats_runtime.PNG)

La funcion anterior proporciona los minimos y maximos, los quartiles, la mediana y la media.A continuacion crearemos un histograma del atributo runtime.

```R
ggplot(data = movies, aes(x = runtime)) + geom_histogram(binwidth = 5)
```
![histogram runtime](/img/posts/histogram-runtime.PNG)

Se observan algunos outliers y una ligera inclinacion hacia la derecha. Para observar de una manera mejor estos outliers seria buena idea utilizar un boxplot.

```R
ggplot(data = movies, aes(x= "movies", y  = runtime)) + geom_boxplot()
```
![boxplot runtime](/img/posts/boxplot-runtime.PNG)

Se aprecia de manera los outliers y la media. Esto indica que en promedio la duracion de una pelicula es alrededor de 100 minutos.

Consideremos los casos de clasificacion de categoria en sitio de Rotten tomatoes y la duracion de cada pelicula. Procederemos a generar un histograma para cada uno  y en el mismo grafico posicionaremos la media y la medianaa.
    
```R
na.omit(movies) %>% group_by(critics_rating) %>% summarise(runtime_mean = mean(runtime), runtime_median = median(runtime))
```
![category summary](/img/posts/cat-sum.PNG)
```R
g1 <- na.omit(movies) %>% filter(critics_rating == "Certified Fresh")  %>% ggplot(aes(x = runtime)) + geom_histogram( binwidth = 5) + geom_vline(aes(xintercept= mean(runtime), color="mean" ), linetype="dashed") + geom_vline(aes(xintercept=median(runtime),color="median" ) , linetype="dashed")  + scale_colour_manual(name="statistics", values = c(mean="red", median="green") ) + labs(title= "Certified Fresh") + theme(plot.title = element_text(hjust = 0.5) )

g2 <- na.omit(movies) %>% filter(critics_rating == "Fresh")  %>% ggplot(aes(x = runtime)) + geom_histogram( binwidth = 5) + geom_vline(aes(xintercept= mean(runtime), color="mean"), linetype="dashed") + geom_vline(aes(xintercept=median(runtime), color="median"), linetype="dashed") + scale_colour_manual(name="statistics", values=c(mean="red", median="green")) +  labs(title= "Fresh") + theme(plot.title = element_text(hjust = 0.5)) 

g3 <- na.omit(movies) %>% filter(critics_rating == "Rotten")  %>% ggplot(aes(x = runtime)) + geom_histogram( binwidth = 5) + labs(title= "Rotten")  + geom_vline(aes(xintercept= mean(runtime), color="mean" ), linetype="dashed") + geom_vline(aes(xintercept=median(runtime),color="median" ) , linetype="dashed")  + scale_colour_manual(name="statistics", values = c(mean="red", median="green") ) + theme(plot.title = element_text(hjust = 0.5) )

plot_grid(g1,g2,g3)
```
![cat grid](/img/posts/multiple-hist-cat.PNG)

Una vez mas claramente se observa una inclinacion hacia la derecha y un par de outliers. Apoyemonos una vez mas de un boxplot.

```R
movies %>% ggplot(aes(x = critics_rating, y  = runtime)) + geom_boxplot()
```
![multiple boxplot](/img/posts/boxplot-cat-mul.PNG)

De manera mas comoda se pueden visualizar los outliers y la media.

Continuando ahora con el sitio IMDb, este maneja sus clasificaciones por puntaje por lo tanto para visualizar la relacion puntaje-duracin de pelicula nos apoyaremos de un scatterplot .
```R
 movies %>% select(runtime, imdb_rating) %>% ggplot(aes( x= imdb_rating, y = runtime)) + geom_jitter() + geom_hline(yintercept = 105.8, color="red")
```
![scatterplot runtime imdb](/img/posts/scatter-plot-runtime.PNG)

Sabemos que el promedio de duracion de un pelicula es 105 minutos entonces cortamos horizontalmente los datos y se observa que las peliculas que tienen con la duracion de 105 minutos tienen un mejor puntaje de 6 ~ 8.

Indagemos la popularidad por mes.
```R
sum_scores_crit <- movies %>% group_by(thtr_rel_month) %>% summarise(mean_critics_score = mean(critics_score), median_critics_score = median(critics_score))
ggplot(data= sum_scores_crit, aes(x= thtr_rel_month, y = mean_critics_score )) + geom_point(size=5) + scale_x_continuous(breaks = c(1:12)) + xlab("month") + ylab("average score") + labs(title="Average Score per month") +  theme(plot.title = element_text(hjust = 0.5))
```
![month score](/img/posts/avg_month.PNG)

A simple vista parace ser que las peliculas que son publicadas en el mes de Diciembre tienden a tener un mejor puntaje, sin embargo, estos datos se refieren a la opinion de los criticos. Pensara la audiencia de misma manera?

```R
aud_score <- movies %>% group_by(thtr_rel_month) %>% summarise(mean_audience = mean(audience_score))
g5<-ggplot(data= sum_scores_crit, aes(x= thtr_rel_month, y = mean_critics_score )) + geom_point(size=5) + scale_x_continuous(breaks = c(1:12)) + scale_y_continuous(breaks = c(50:80)) + xlab("month") + ylab("average score") + labs(title="Average Score per month") +  theme(plot.title = element_text(hjust = 0.5))
g5 + geom_point( y = aud_score$mean_audience, size = 5, color="blue") + scale_y_continuous(breaks = c(54:70))
```
![audience score](/img/posts/audience-score.PNG)

Al parecer la audiencia es menos dura que los criticos al momento de calificar, pero, se observa que al igual que  los criticos, las peliculas que son publicadas en Diciembre.

Continuando, visualizando el puntaje por genero de pelicula en relacion al promedio de critica por genero.
```R
na.omit(movies) %>% ggplot(aes(x = genre, y = critics_score)) + geom_boxplot() + theme(axis.text.x = element_text(angle= 90, vjust=0.5))
```
El grid de BoxPlot muestra muestra el promedio de puntaje en relacion con el genero de cada pelicula, cualquier cosa arriba de 50 podria significar un puntaje decente.

Ahora, analicemos los votos de los criticos  en cada categoria.

```R
ggplot(data = movies, aes(x = critics_rating, y = imdb_num_votes/1000, fill = critics_rating ) ) + geom_bar( stat = "identity")+ labs(title = "Relationship Rotten Tomatoes and IMDb votes") + theme(plot.title = element_text(hjust = 0.5))
```
![category critics](/img/posts/critics-category.PNG)

### Parte 4 Modelar

Es hora de construir un modelo linear de tipo regression. Usaremos las siguientes variables.

- genre
- critics_rating
- thtr_rel_month
- runtime
- mdb_num_votes

Utilizaremos la metodologia de eliminacion en reversa para elegir al mejor modelo y el mas simple. Las variables se iran eliminando de acuerdo al valor de p.

```R
movies_model <-  lm(critics_score ~  genre + critics_rating +  thtr_rel_month + runtime + imdb_num_votes, data = movies)
summary(movies_model)
```
![modelo 1](/img/posts/modelo1.PNG)

La tactica es, ir quitando aquellas variables que  tengan el valor de P mas alto con el objetivo de reducir el coeficiente de R squared.

En este modelo la variable htr_rel_month tiene el valor de P mas alto. Este modelo, tuvo un R-squared de 0.7923.Eliminenos esta variable para ver si podemos incrementar el valor de R-squared.

```R
movies_model_2 <- lm(critics_score ~  genre + critics_rating  + runtime + imdb_num_votes, data = movies)
summary(movies_model_2)
```

![modelo 2](/img/posts/model-2.PNG)
El valor de R-squared se ha incrementando un par de puntos, lo cual es bueno, mientras el valor de R-squared siga incrementandose podemos seguir quitando variables del juego, ahora bien, la variable genre NO tiene un valor significante por lo tanto un movimiento sensanto seria quitarla, sin embargo, esta variable es una variable que se compone de diferentes niveles y por lo tanto es necesario tratarla como una sola.

```R
movies_model_3 <- lm(critics_score ~  genre + critics_rating  + runtime, data = movies)
summary(movies_model_3)
```

![modelo 3](/img/posts/model-3.PNG)
El valor  R-squared se ha vuelto de incrementar.Ya no existen mas variables que quitar del modelo. Es hora de realizar un diagnostico de este.

### Dispersion aletoria alrededor de cero
```R
residuals <- c(movies_model_3$residuals, 0)
plot(residuals ~ movies$runtime)
```
![random scatter](/img/posts/random-scatter-around-0.PNG)
Queremos la dispersion alrededor de cero, esto luce justo.

### Residuos de distribucion normal con media cero
```R
hist(movies_model_3$residuals)
```
![nearly normal cero](/img/posts/nearly-normal-cero.PNG)
Es similar a una distribucion normal? analicemos con una QQ plot.

```R
qqnorm(movies_model_3$residuals)
qqline(movies_model_3$residuals)
```
![qqlplot](/img/posts/qqplot.PNG)
La grafica muestra una similitud de forma en S, sin embargo, no hay mucha variabilidad en las colas.

```R
plot(movies_model_3$residuals)
```
![residual patterns](/img/posts/residual-any-pattern.PNG)

No se muestra ningun patron en los residuos.

### Parte 5 prediccion

```R
y_test <- data.frame(genre = "Documentary", critics_rating = "Fresh", runtime = 105)
predict(movies_model_3, y_test, interlval="prediction", level = 0.05)
```
El modelo da una respuesta de 86.65909. 
Con esto, diciendonos que introduciendo estos variables podemos aseguranos que en realidad si es una categoria Fresh, de acuerdo Rotten tomatoes un intervalo   85 a 87 es categoria fresh.

### Conclusion

Claramente esto es considerado un modelo de juguete y solo es considerado para propositivos educativos de manera introductoria. Sin embargo, si tenemos suficientes datos podemos definir un patron y en realidad llegar a un modelo que pueda darnos la respuesta a una formula para siempre tener una pelicula exitosa? Probablemente, aunque, las variables que no se pueden controlar como el humor, la situacion economica, cultural, entre otras pienso que juegan un rol importante al momento de llevar a la implementacion.    
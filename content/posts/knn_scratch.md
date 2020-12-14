+++
title = "KNN version regression"
date = 2020-12-13T18:15:37-05:00
tags = []
categories = []
imgs = []
cover = "/img/covers/targets-cover.jpg"  # image show on top
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



En una entrada anterior, utilizamos el algoritmo KNN  para clasificar imagenes. Algo que no les mencione es que este algoritmo tiene una flexibilidad que nos permite trabajar problemas  de clasificacion asi como de regresion.

En aquella problematica, donde clasificamos imagenes satelitales utilizando bandas, nos apoyamos de la libreria Sklearn. 

En esta entrada trabajemos otro problema de agrupamiento pero esta vez una problematica de regresion. 

De misma manera, utilizaremos Sklearn, pero esta vez, vamos a intentar implementar el algoritmo nosotros mismos y verifiquemos si tenemos los mismos resultados que si utilizaramos la libreria de sklearn. 

Ademas de implementar el algortimo KNN en su version de regresion, tambien, hablaremos de normalizacion de datos, el cual, es un paso muy importante al momento de desarrollar estos prototipos.

Se utilizaran los conjutos de datos kc_house_data. Se proveeran un conjunto de entrenamiento, prueba y validacion. 

Los puedes descargar [aqui](https://www.kaggle.com/marckhi/kc-house-data)


Como primer paso importemos las librerias con las cuales nos estaremos apoyando. 


```python
from sklearn.neighbors import KNeighborsRegressor
```


```python
from sklearn.preprocessing import normalize as sk_norm
```


```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

```


```python
dtype_dict = {
              'bathrooms':float, 
              'waterfront':int, 
              'sqft_above':int, 
              'sqft_living15':float, 
              'grade':int, 
              'yr_renovated':int, 
              'price':float, 
              'bedrooms':float, 
              'zipcode':str, 
              'long':float, 
              'sqft_lot15':float, 
              'sqft_living':float, 
              'floors':float, 
              'condition':int, 
              'lat':float, 
              'date':str, 
              'sqft_basement':int, 
              'yr_built':int, 
              'id':str, 
              'sqft_lot':int, 
              'view':int
          }
```

Creemos un diccionario donde estaremos pasando los tipos de datos al objeto dataframe al momento de leer el archivo CSV. 

Al momento de utilizar la funcion read_csv() es posible pasar como argumento los tipos de datos del archivo, esto ahorra tiempo y memoria al momento de cargar ficheros dado que evita que pandas infiera los tipos de datos y vaya de uno en uno intentando buscar el tipo de dato correcto.

Como siguiente paso, carguemos los ficheros en memoria.


```python
train = pd.read_csv("kc_house_data_small_train.csv", dtype= dtype_dict)
test = pd.read_csv("kc_house_data_small_test.csv", dtype= dtype_dict)
df = pd.read_csv("kc_house_data_small.csv", dtype= dtype_dict)
validation = pd.read_csv("kc_house_data_validation.csv", dtype= dtype_dict)
```

Una vez cargados los archivos en memoria podemos inspeccionarlos utilizando el metodo head()


```python
train.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>date</th>
      <th>price</th>
      <th>bedrooms</th>
      <th>bathrooms</th>
      <th>sqft_living</th>
      <th>sqft_lot</th>
      <th>floors</th>
      <th>waterfront</th>
      <th>view</th>
      <th>condition</th>
      <th>grade</th>
      <th>sqft_above</th>
      <th>sqft_basement</th>
      <th>yr_built</th>
      <th>yr_renovated</th>
      <th>zipcode</th>
      <th>lat</th>
      <th>long</th>
      <th>sqft_living15</th>
      <th>sqft_lot15</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>7129300520</td>
      <td>20141013T000000</td>
      <td>221900.0</td>
      <td>3.0</td>
      <td>1.00</td>
      <td>1180.0</td>
      <td>5650</td>
      <td>1.0</td>
      <td>0</td>
      <td>0</td>
      <td>3</td>
      <td>7</td>
      <td>1180</td>
      <td>0</td>
      <td>1955</td>
      <td>0</td>
      <td>98178</td>
      <td>47.5112</td>
      <td>-122.257</td>
      <td>1340.0</td>
      <td>5650.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>6414100192</td>
      <td>20141209T000000</td>
      <td>538000.0</td>
      <td>3.0</td>
      <td>2.25</td>
      <td>2570.0</td>
      <td>7242</td>
      <td>2.0</td>
      <td>0</td>
      <td>0</td>
      <td>3</td>
      <td>7</td>
      <td>2170</td>
      <td>400</td>
      <td>1951</td>
      <td>1991</td>
      <td>98125</td>
      <td>47.7210</td>
      <td>-122.319</td>
      <td>1690.0</td>
      <td>7639.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>5631500400</td>
      <td>20150225T000000</td>
      <td>180000.0</td>
      <td>2.0</td>
      <td>1.00</td>
      <td>770.0</td>
      <td>10000</td>
      <td>1.0</td>
      <td>0</td>
      <td>0</td>
      <td>3</td>
      <td>6</td>
      <td>770</td>
      <td>0</td>
      <td>1933</td>
      <td>0</td>
      <td>98028</td>
      <td>47.7379</td>
      <td>-122.233</td>
      <td>2720.0</td>
      <td>8062.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2487200875</td>
      <td>20141209T000000</td>
      <td>604000.0</td>
      <td>4.0</td>
      <td>3.00</td>
      <td>1960.0</td>
      <td>5000</td>
      <td>1.0</td>
      <td>0</td>
      <td>0</td>
      <td>5</td>
      <td>7</td>
      <td>1050</td>
      <td>910</td>
      <td>1965</td>
      <td>0</td>
      <td>98136</td>
      <td>47.5208</td>
      <td>-122.393</td>
      <td>1360.0</td>
      <td>5000.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1954400510</td>
      <td>20150218T000000</td>
      <td>510000.0</td>
      <td>3.0</td>
      <td>2.00</td>
      <td>1680.0</td>
      <td>8080</td>
      <td>1.0</td>
      <td>0</td>
      <td>0</td>
      <td>3</td>
      <td>8</td>
      <td>1680</td>
      <td>0</td>
      <td>1987</td>
      <td>0</td>
      <td>98074</td>
      <td>47.6168</td>
      <td>-122.045</td>
      <td>1800.0</td>
      <td>7503.0</td>
    </tr>
  </tbody>
</table>
</div>



Podemos observar en el conjunto de datos,  tipo de fecha, continuos,  y probablemente algunas variables de ordinales.


La variable fecha, puede ser un tipo de dato muy interesante al momento de tratar estos problemas de valuacion de precio o caracteristicas, seria interesante estudiarlo y quiza crear algun tipo de mini proyecto en como podemos utilizar un tipo de serie de tiempo para hacer una valuacion de estas casas y obtener cosas curiosas, sin embargo, eso esta fuera del alcance de esta entrada.

Nos es posible trabajar con las fechas, como dato curioso podemos dividir las fechas entre dia, mes y año y asi convertimos ese tipo de dato fecha en tres variables que mapeen a la fecha. Por cuestiones de simplicidad no haremos eso. Trabajaremos con las siguientes variables. 


```python
feature_list = [
                'bedrooms',  
                'bathrooms',  
                'sqft_living',  
                'sqft_lot',  
                'floors',
                'waterfront',  
                'view',  
                'condition',  
                'grade',  
                'sqft_above',  
                'sqft_basement',
                'yr_built',  
                'yr_renovated',  
                'lat',  
                'long',  
                'sqft_living15',  
                'sqft_lot15'
                ]
```


```python
len(feature_list)
```




    17



Para no modificar nuestros dataframes originales, hagamos una copia de ellos.


```python
train_copy = train.copy()
test_copy = test.copy()
valid_copy = validation.copy()
```


```python
def draw_histograms(df, variables, n_rows, n_cols):
    fig=plt.figure(figsize=(18, 22))
    if not isinstance(df, pd.DataFrame):
      df = pd.DataFrame(df)
      df.columns = variables
    for i, var_name in enumerate(variables):
        ax=fig.add_subplot(n_rows,n_cols,i+1)
        df[var_name].hist(bins=15,ax=ax)
        ax.set_title(var_name+" Distribution")
    #fig.tight_layout()  # Improves appearance a bit.
    plt.show()

draw_histograms(train_copy, feature_list, 6,3)
```


![png](/img/posts/output_17_0.png)


Hablemos de este concepto de *Normalizacion*. 

Cuando Normalizar datos? Cuando nuestros datos no presentan una distrubucion gaussiana (la campana de gauss). Cuando los datos presentan diferentes rangos entre si o bien, al momento de utilizar algun tipo de algortimo donde este no haga asumpciones acerca de la distribucion de los datos. En este caso KNN.

En este caso, usaremos la normalizacion l2.

La normalizacion l2, la cual es la distancia euclidiana, donde la definimos en la entrada de [clasificacion de imagenes](https://elrobotista.com/posts/knn-scikit-learn).

En las graficas podemos observar que las distrubiuciones de los datos no se asemejan a una gausiana y ademas los rangos de ellos varian, por lo tanto vamos normalizar.

Utilizemos sklearn para normalizar.
Dentro de jupyter notebooks podemos acceder a la documentacion del codigo utilizando los signos de interrogacion.


```python
sk_norm?
```
~~~

Signature: sk_norm(X, norm='l2', axis=1, copy=True, return_norm=False)
Docstring:
Scale input vectors individually to unit norm (vector length).

Read more in the :ref:`User Guide <preprocessing_normalization>`.

Parameters
----------
X : {array-like, sparse matrix}, shape [n_samples, n_features]
    The data to normalize, element by element.
    scipy.sparse matrices should be in CSR format to avoid an
    un-necessary copy.

norm : 'l1', 'l2', or 'max', optional ('l2' by default)
    The norm to use to normalize each non zero sample (or each non-zero
    feature if axis is 0).

axis : 0 or 1, optional (1 by default)
    axis used to normalize the data along. If 1, independently normalize
    each sample, otherwise (if 0) normalize each feature.

copy : boolean, optional, default True
    set to False to perform inplace row normalization and avoid a
    copy (if the input is already a numpy array or a scipy.sparse
    CSR matrix and if axis is 1).

return_norm : boolean, default False
    whether to return the computed norms

Returns
-------
X : {array-like, sparse matrix}, shape [n_samples, n_features]
    Normalized input X.

norms : array, shape [n_samples] if axis=1 else [n_features]
    An array of norms along given axis for X.
    When X is sparse, a NotImplementedError will be raised
    for norm 'l1' or 'l2'.

See also
--------
Normalizer: Performs normalization using the ``Transformer`` API
    (e.g. as part of a preprocessing :class:`sklearn.pipeline.Pipeline`).

Notes
-----
For a comparison of the different scalers, transformers, and normalizers,
see :ref:`examples/preprocessing/plot_all_scaling.py
<sphx_glr_auto_examples_preprocessing_plot_all_scaling.py>`.
File:      /usr/local/lib/python3.6/dist-packages/sklearn/preprocessing/_data.py
Type:      function

~~~


Necesitamos obetener nuestra variable X normalizada y ademas la normalizacion. Nota que por defecto este opera por los rows y nosotros necesitamos que opere por las columnas, por lo tanto pasaremos como parametro opcional axis=0 y return_norm = True


```python
tr_normalized, tr_norms = sk_norm(train_copy[feature_list], axis=0, return_norm=True)
tr_normalized.shape, tr_norms.shape
```




    ((5527, 17), (17,))



Ahora tenemos nuestros datos de entrenamiento normalizados. 
Ademas necesitamos normalizar los datos de  prueba y validacion.
Para esta operacion, tomaremos las normas de los datos de entrenamiento para normalizar los otros conjuntos de datos para que estos esten en el mismo contexto


```python
test_normalized = (test_copy[feature_list] / tr_norms).values
valid_normalized = (valid_copy[feature_list] / tr_norms).values
```


```python
valid_normalized.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>bedrooms</th>
      <th>bathrooms</th>
      <th>sqft_living</th>
      <th>sqft_lot</th>
      <th>floors</th>
      <th>waterfront</th>
      <th>view</th>
      <th>condition</th>
      <th>grade</th>
      <th>sqft_above</th>
      <th>sqft_basement</th>
      <th>yr_built</th>
      <th>yr_renovated</th>
      <th>lat</th>
      <th>long</th>
      <th>sqft_living15</th>
      <th>sqft_lot15</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.015513</td>
      <td>0.010544</td>
      <td>0.009661</td>
      <td>0.001599</td>
      <td>0.008530</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.015509</td>
      <td>0.012167</td>
      <td>0.005916</td>
      <td>0.019444</td>
      <td>0.013285</td>
      <td>0.0</td>
      <td>0.013491</td>
      <td>-0.013465</td>
      <td>0.009001</td>
      <td>0.002020</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.019391</td>
      <td>0.015062</td>
      <td>0.013537</td>
      <td>0.002023</td>
      <td>0.017059</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.011632</td>
      <td>0.013905</td>
      <td>0.015616</td>
      <td>0.000000</td>
      <td>0.013612</td>
      <td>0.0</td>
      <td>0.013385</td>
      <td>-0.013447</td>
      <td>0.014402</td>
      <td>0.002841</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.015513</td>
      <td>0.010544</td>
      <td>0.013895</td>
      <td>0.001605</td>
      <td>0.012794</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.015509</td>
      <td>0.012167</td>
      <td>0.010388</td>
      <td>0.020979</td>
      <td>0.013162</td>
      <td>0.0</td>
      <td>0.013485</td>
      <td>-0.013468</td>
      <td>0.009387</td>
      <td>0.002028</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.011635</td>
      <td>0.006025</td>
      <td>0.009363</td>
      <td>0.000732</td>
      <td>0.017059</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.011632</td>
      <td>0.012167</td>
      <td>0.010800</td>
      <td>0.000000</td>
      <td>0.013114</td>
      <td>0.0</td>
      <td>0.013474</td>
      <td>-0.013468</td>
      <td>0.010159</td>
      <td>0.001071</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.015513</td>
      <td>0.018075</td>
      <td>0.011032</td>
      <td>0.003203</td>
      <td>0.017059</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.011632</td>
      <td>0.013905</td>
      <td>0.012727</td>
      <td>0.000000</td>
      <td>0.013585</td>
      <td>0.0</td>
      <td>0.013435</td>
      <td>-0.013444</td>
      <td>0.014595</td>
      <td>0.003465</td>
    </tr>
  </tbody>
</table>
</div>



Observemos como nuestros datos ahora estan normalizados en los mismos rangos.

Una vez normalizados los datos, procedamos a aplicar el algoritmo KNN utlizando sklearn.


```python
#Primero, declaramos una variable donde estara la instancia de la clase KNR.
knn = KNeighborsRegressor()
#Segundo, llamamos el metodo fit, pasando como parametro nuestra variable independiente asi como la independiente.
knn.fit(tr_normalized,train.price)
```




    KNeighborsRegressor(algorithm='auto', leaf_size=30, metric='minkowski',
                        metric_params=None, n_jobs=None, n_neighbors=5, p=2,
                        weights='uniform')



Y asi de sencillo. Con sklearn es todo lo que necesitamos para implementar el algortimo. Ahora bien, donde elegimos la cantidad de elementos K? En sklearn, es posible pasar la cantidad K vecinos mediante el parametro n_neighbors.
Mencionamos tambien que la metrica para computar la distancia entre el punto query y los K vecinos es la distancia euclideana. Sklearn por usa esta metrica por defecto, es posible configurar otro tipo de metrica, como mikwonski o manhattan, pero no haremos eso. 

Para realizar predicciones llamamos el metodo predict().


```python
knn.predict(test_normalized)
```




    array([878000., 418520., 381190., ..., 290760., 572880., 274670.])



Como podemos evaluar el modelo? En este caso, usaremos una funcion de costos usando la suma de cuadrados de los residuos.

Nos es posible utilizar sklearn para realizar este calculo, sin embargo, mejor implementemos nuestra propia funcion.


```python
def rss(y_pred, y_true): return ((y_pred - y_true) ** 2).sum()
```

Ahora podemos llamar nuestra funcion rss pasando como parametro las predicciones y los valores reales. 


```python
rss(knn.predict(test_normalized), test.price)
```




    -59601295253197.266



Y listo. Eso fue todo lo necesario para aplicar el algoritmo utilizando sklearn. Ahora bien no podemos decir que sklearn nos dara el mejor modelo sin afinarlo, en ocaciones la configuracion por defecto suele funcionar mejor segun sea el problema, pero podemos llevarlo un poco mas lejos.

Podemos utilizar tecnicas como validacion cruzada, busqueda por grid, etc; pero esta vez podemos utilizar la funcion de costos para elegir la mejor K segun sea esta la que nos de el error minimo en el conjunto de prueba; no utilizaremos la clase sklearn mas y procedamos a implementar la nosotros.


Primero que nada, necesitamos normalizar los datos, bien podemos utilizar los datos normalizados anteriormente, pero, implementemos nuestra propia funcion de normalizacion, con numpy es realmente sencillo.


```python
def normalize(data):
  norm = np.linalg.norm(data, axis=0)
  element_wise_norm = data / norm
  return element_wise_norm, norm
```

Nuestra funcion normalize recibe como parametro una variable data. Esta variable data es un matriz, para normalizar los datos nos apoyamos de las funciones de algebra lineal numpy llamando norm. A esta pasamos como parametro la matriz y especificamos que sea atraves de las columnas. 

Luego tenemos que hacerlo por cada elemento dentro de la matriz original y lo dividimos por las normas. Al final regresamos las normas y la matriz normalizada. Esto hace exactamente lo mismo que la funcion normalize de sklearn.

Como siguiente paso normalizemos los datos. Primero que nada creemos la funcion que nos regrese objeto numpy y ademas agregarmos una constante = 1 para que este sea el intercepto.


```python
def transform_numpy(data, features, dep_var):
  data['constant'] = 1
  features = ['constant'] + features
  sub_data = data[features]
  target = data[dep_var]
  return sub_data.values, target.values
```

La funcion transform_numpy recibe tres parametros, data, los features que es un arreglo que contiene los nombre de las columnas para indexar el dataframe y una variable dep_var que se refiere a la variable independiente. 

Al objeto dataframe agregamos una columna *constant* esta columna sera el intercepto. Luego a nuestra lista de features agregamos un elemento constant para que este pueda mapear a la columna recien agregada, como pasos finales indexamos y retornamos objetos numpy llamando la propiedad values.



```python
X, y = transform_numpy(train, feature_list, 'price')
```


```python
X_test, y_test = transform_numpy(test, feature_list, 'price')
X_valid, y_valid = transform_numpy(validation, feature_list, 'price')
```

Procedamos a normalizar.


```python
X_norm, norms = normalize(X)
```


```python
X_test_norm = X_test / norms
```


```python
X_valid_norm = X_valid / norms
```

Hemos hablado que es necesario utilizar la distancia euclideana, definamos nuestra funcion. 


```python
def euclidean_distance(X, y, axis=False):
  if axis:
    return np.sqrt( np.sum( (X - y) ** 2 ,axis=1) )
  else:
    return np.sqrt( np.sum( (X - y) **2))
```

La funcion euclidean distance recibe como parametro la variable independiente y la variable dependiente, como opcional podemos pasar el como operar, ya sea por por renglones o columnas.

Ahora definamos la funcion k_nearest_neighbors. 


```python
def k_nearest_neighbors(k, dataset, query):
  distances  = euclidean_distance(dataset, query, axis=True)
  neighbors = np.argsort(distances)[0:k]
  return neighbors
```

Mediante tres paremetros un K siendo el numero de vecinos, la variable independiente y la variable dependiente calculamos las distancias, las almacenamos y las ordenamos utilizando el motodo argsort() de numpy e indexamos hasta los K vecinos y al final los retornamos.

Con esta funcion podemos obtener los vecinos mas cercanos.

Y como podemos realizar predicciones? Las predicciones son dadas como el promedio del valor de los K vecinos dado un punto.


```python
def predict_price(k, dataset, dep_var, query):
  distances = euclidean_distance(dataset, query, axis=True)
  neighbors = np.argsort(distances)[0:k]
  predicted_mean  = dep_var.iloc[neighbors].mean()
  return predicted_mean
```

Utilizemos la funcion predict_price para predecir los primeros 10 puntos del conjunto de pruebas con una K de valor 10.


```python
preds = dict()
for i in range(10):
  preds[i]=  predict_price(10, X_norm, train.price, X_test_norm[i]) 
```


```python
prices_df = pd.DataFrame([preds]).T
prices_df.columns = ['price']
prices_df.sort_values(by='price')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>6</th>
      <td>350032.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>430200.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>431860.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>457235.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>460595.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>484000.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>512800.7</td>
    </tr>
    <tr>
      <th>5</th>
      <td>667420.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>766750.0</td>
    </tr>
    <tr>
      <th>0</th>
      <td>881300.0</td>
    </tr>
  </tbody>
</table>
</div>



Por comodidad utilizemos un diccionaro para guardar los valores de cada prediccion segun dado el punto, para depues luego crear un objeto DataFrame. Si al momento de pasar un diccionar a una instancia de DataFrame este sera necesario parsarlo dentro de una lista y despues vamos a tomar su transpuesta para obtener una columna, dado que no especificamos el nombre de la columna, a este podemos asignarle el nombre con el atributo columns y asignando una lista, finalmente ordenemos de menor a mayor para visualizar los precios.

Procedamos a evaluar el valor de K. Podemos realizar este paso utilizando un ciclo por cada valor y realizar las predicciones, el enfoque tomare sera de apoyarme de una lista y un diccionario. 


```python
preds_non_sk = []
rss_value_non_sk = dict()
for k_value in range(1,16):
  for i in range(X_valid_norm.shape[0]):
    preds_non_sk.append( predict_price(k_value, X_norm, train.price, X_valid_norm[i]) )
  rss_value_non_sk[k_value] = rss(preds_non_sk, validation.price)
  preds_non_sk = []
```


```python
RSS_non_sk = pd.DataFrame([rss_value_non_sk]).T
RSS_non_sk.columns =['residuals']
RSS_non_sk.sort_values(by='residuals')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>residuals</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>8</th>
      <td>6.736168e+13</td>
    </tr>
    <tr>
      <th>7</th>
      <td>6.834197e+13</td>
    </tr>
    <tr>
      <th>9</th>
      <td>6.837273e+13</td>
    </tr>
    <tr>
      <th>6</th>
      <td>6.889954e+13</td>
    </tr>
    <tr>
      <th>12</th>
      <td>6.904997e+13</td>
    </tr>
    <tr>
      <th>10</th>
      <td>6.933505e+13</td>
    </tr>
    <tr>
      <th>11</th>
      <td>6.952386e+13</td>
    </tr>
    <tr>
      <th>5</th>
      <td>6.984652e+13</td>
    </tr>
    <tr>
      <th>13</th>
      <td>7.001125e+13</td>
    </tr>
    <tr>
      <th>14</th>
      <td>7.090870e+13</td>
    </tr>
    <tr>
      <th>15</th>
      <td>7.110693e+13</td>
    </tr>
    <tr>
      <th>4</th>
      <td>7.194672e+13</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7.269210e+13</td>
    </tr>
    <tr>
      <th>2</th>
      <td>8.344507e+13</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1.054538e+14</td>
    </tr>
  </tbody>
</table>
</div>



Igual que en las predicciones creemos un dataframe y ordenemos. 
Al paracer nuestro error mas bajo sucede cuando K = 8. Utilizemos una grafica.


```python
RSS.plot()
```




    <matplotlib.axes._subplots.AxesSubplot at 0x7fdb22f104e0>




![png](/img/posts/output_63_1.png)


Realizemos el mismo procedimiento pero ahora utilizando sklearn. Verifiquemos si obtenemos el mismo resultado para K y que tanta diferencia hay entre las implementaciones de KNN. 


```python
preds_sk = []
rss_value_sk = dict()
for i in range(1,16):
  knn = KNeighborsRegressor(n_neighbors=i)
  knn.fit(X_norm, train.price)
  preds_sk = knn.predict(X_valid_norm)
  rss_value_sk[i] = rss(preds_sk, validation.price)
  preds_sk = []

```


```python
RSS_sk = pd.DataFrame([rss_value]).T
RSS_sk.columns =['residuals']
RSS_sk.sort_values(by='residuals')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>residuals</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>8</th>
      <td>6.737109e+13</td>
    </tr>
    <tr>
      <th>7</th>
      <td>6.834079e+13</td>
    </tr>
    <tr>
      <th>9</th>
      <td>6.837273e+13</td>
    </tr>
    <tr>
      <th>6</th>
      <td>6.889916e+13</td>
    </tr>
    <tr>
      <th>12</th>
      <td>6.905195e+13</td>
    </tr>
    <tr>
      <th>10</th>
      <td>6.933025e+13</td>
    </tr>
    <tr>
      <th>11</th>
      <td>6.952386e+13</td>
    </tr>
    <tr>
      <th>5</th>
      <td>6.979932e+13</td>
    </tr>
    <tr>
      <th>13</th>
      <td>7.001125e+13</td>
    </tr>
    <tr>
      <th>14</th>
      <td>7.091153e+13</td>
    </tr>
    <tr>
      <th>15</th>
      <td>7.110882e+13</td>
    </tr>
    <tr>
      <th>4</th>
      <td>7.193480e+13</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7.270472e+13</td>
    </tr>
    <tr>
      <th>2</th>
      <td>8.344507e+13</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1.054512e+14</td>
    </tr>
  </tbody>
</table>
</div>



De acuerdo a la ordenacion de los datos, obtenemos el mismo valor para K.


```python
fig=plt.figure(figsize=(18,14))
fig.show()
ax=fig.add_subplot(111)
ax.plot(RSS_non_sk,marker='*',label='scratch implementation', markersize=15)
ax.plot(RSS_sk, marker="o", label='sklearn implementation')
plt.legend(loc=1)
```




    <matplotlib.legend.Legend at 0x7fdb21dacac8>




![png](/img/posts/output_68_1.png)


Podemos visualizar como el error mas bajo es 8 para ambas implementaciones. De misma manera podemos visualizar como ambos siguen la misma trayectoria de error.

Conclucion
---- 
Realizamos una implementacion de KNN desde cero con bastante sencillez, claramente no esta lista para produccion o decir que esta a la par de sklearn. Solamente hicimos nuestra propia reproduccion del algoritmo para entender como funciona teoricamente la clase de sklearn y no sea una caja negra.
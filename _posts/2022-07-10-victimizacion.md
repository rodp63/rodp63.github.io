---
layout: post
title: "Victimización de empresas"
subtitle: "Predicción de la victimización de empresas en Perú durante el año 2022"
date: 2022-07-10
background: '/img/posts/victimizacion/victimizacion-cover.jpg'
---

**Curso**: Introducción a Ciencia de Datos (CS351)

**Profesor**: Dr. Erick Gomez Nieto

**Semestre**: 2022–1

La victimización de empresas es un problema latente en la sociedad peruana, día a día
escuchamos en los medios de comunicación noticias de robos y asaltos a establecimientos
comerciales. Si bien la capital Lima tiene un mayor número de incidencias, los demás
departamentos no son ajenos a este problema social. De hecho, un estudio realizado por el
INEI en el año 2018 demostró qué, durante setiembre del 2017 y agosto del 2018, 28 de cada 100
empresas fueron víctimas de al menos un hecho delictivo.

La violencia impacta en el crecimiento económico del país, no sólo afecta directamente a
la población económicamente activa, sino que disminuye los incentivos directos para invertir,
así como los incentivos gubernamentales que se desvían para reforzar políticas de seguridad y
control. Motivado por este contexto, se realizó un estudio de ciencia de datos sobre la 
victimización de empresas peruanas, intentando modelar su comportamiento para poder predecir 
el número de delitos que podrían ocurrir durante el año 2022.

Para abordar este proyecto se elaboró una solución basada en ciencia de datos, desde el
proceso de recolección de datos, hasta el modelamiento temporal de los eventos de
victimización. Sin mayor preámbulo, repasemos cada una de estas etapas.

## Recolección de datos

Se consideró el uso de los datos abiertos que ofrece el INEI, sin embargo, estos ocultaban 
mucha información por cuestiones de privacidad. Entonces, ¿Cómo recolectar la información 
necesaria? La respuesta es clara, hacer _web scraping_ a páginas web de noticias. 
Si bien existen muchos medios, como las redes sociales, los periódicos se 
caracterizan por tener un mayor grado de credibilidad y confianza al ser entidades que
dependen de su buena reputación y aceptación de los lectores. Luego de revisar varios
diarios en linea, se decidió hacer _web scraping_ al sitio web del diario
[Correo](https://diariocorreo.pe/).

El _spider_ se programó en _Python_ haciendo uso del framweork _Scrapy_. El _pipeline_
seguido fue el siguiente:

<img src="/img/posts/victimizacion/scraping_pipeline.png" alt="pipeline" width="95%"/>

* Primero se leen los parámetros de ejecución, las fechas de recolección de noticias van
  desde enero del 2015 a abril del 2022.

* Luego, se mandan de forma concurrente _requests_ al API de Correo preguntando por los
  _posts_ durante ese rango de tiempo.
  
* Una vez obtenidos los _responses_, se realiza un filtro por _keywords_ de victimización,
  e.g. robo, asalto, ladrones, etc. Para realizar el _match_ se eliminan los _stop words_ del
  texto y se aplica _stemming_ para eliminar ambigüedades léxicas.
  
* Ahora que tenemos noticias sobre victimización, debemos realizar un filtro adicional,
  encontrar aquellas que hablen sobre empresas. Para ello, los _requests_ ya no se dirigen
  al API, ahora se envían al sitio web donde se encuentra la redacción completa de la noticia.
  
* El proceso de filtrado es similar al anterior, ahora se buscan _keywords_ relacionadas
  a empresas, e.g. negocio, comercio, joyería, etc. Una vez filtradas las noticias, se guardan
  los campos más importantes usando selectores. Los campos guardados por cada noticia 
  fueron: _url_, _date_, _title_, _summary_, _body_ y _score_. Este último campo indica 
  qué tan seguro es que dicha noticia efectivamente se trate de la victimización a una
  empresa.
  
* Finalmente, todos los _items_ son exportados en formato CSV. Con el método propuesto se
  lograron recopilar 3747 noticias.

## Preparación de datos

En esta etapa del proyecto se añaden campos de
geo-localización a los datos: _latitude_, _longitude_ y _location_. Para ello, se realizaron
los siguientes pasos:

<img src="/img/posts/victimizacion/cleaning_pipeline.png" alt="pipeline" width="95%"/>

* Se lee el archivo CSV obtenido mediante _web scraping_ usando la librería _pandas_, por ende,
  los datos se representan como un _DataFrame_. Adicionalmente, para encontrar la ubicación
  de las noticias, se utilizó la siguiente 
  [base de datos](https://www.datosabiertos.gob.pe/dataset/c%C3%B3digo-de-ubicaci%C3%B3n-geogr%C3%A1fica-en-el-per%C3%BA-instituto-nacional-de-estad%C3%ADstica-e-inform%C3%A1tica)
  que contiene la información de todos los distritos de Perú.

* Una vez cargadas las noticias, se hace la limpieza de su contenido. Se eliminan 
  caracteres especiales, acentos, y el texto se convierte a minúsculas.

* Para encontrar la ubicación dentro del texto de las noticias, se tuvo que hacer búsquedas 
  exactas de los distritos, provincias y departamentos en determinado orden. Las _url_
  de algunas noticias tienen presente el departamento, reduciendo el campo de búsqueda.
  Se tomaron otros criterios para verificar la ubicación, por ejemplo, si el nombre de
  un distrito está presente en la noticia y es diferente al nombre de la provincia, entonces
  tiene mayor relevancia. Otro criterio es la ubicación dentro de la noticia, si el lugar
  está presente en el título tiene mayor relevancia que un lugar dentro del cuerpo.
  Los resultados fueron muy alentadores, con pocos falsos positivos, aquí un ejemplo del campo 
  resultante:
  
  <img src="/img/posts/victimizacion/df.png" alt="freq" width="95%"/>
  
* Una vez encontrada la locación, se pasó dicho valor al geo-localizador de _geopy_ para
  encontrar la longitud y latitud de las noticias.
  
  ```python
  from geopy.geocoders import Nominatim
  
  geolocator = Nominatim()
  
  def find_coordinates(row):
    loc = geolocator.geocode(row["location"])
    return loc.latitude, loc.longitude

  data[["latitude", "longitude"]] = data.apply(
    lambda row: find_coordinates(row), axis=1, result_type="expand"
  )
  ```
  
* Finalmente, se guardan los datos limpios y geo-localizados, logrando obtener 3670 noticias.
  Aquellas que no obtuvieron una ubicación fueron eliminadas de la base de datos.

## Visualización y exploración de datos

Durante esta etapa se extraen _insights_ de los datos, la primera visualización realizada
fue la frecuencia de crímenes por departamento, como era de esperar, las regiones con
mayor población presentan un mayor número de ocurrencias.

<center>
	<img src="/img/posts/victimizacion/dep_freq.png" alt="freq" width="80%"/>
</center>

Sin embargo, estas cifras no reflejan el nivel de inseguridad de cada departamento. El
_ratio_ de victimización por cantidad de habitantes presenta una mejor visualización de
las incidencias criminales. Para ello, se consiguió una 
[base de datos](https://www.datosabiertos.gob.pe/dataset/poblaci%C3%B3n-peru)
con la población de cada distrito del Perú, haciendo la división correspondiente, obtenemos 
los siguientes valores.

<center>
	<img src="/img/posts/victimizacion/dep_rate.png" alt="freq" width="80%"/>
</center>

Algunos departamentos mantienen sus posiciones, y algunos cambian considerablemente.
Gracias al gráfico de barras, podemos determinar que Tumbes, Tacna, Moquegua, Piura y 
Lambayeque son las cinco regiones cuyas empresas sufren de un mayor grado de victimización
por número de habitantes durante los últimos 7 años.

Otro gráfico interesante es el número de hechos delictivos por mes. La frecuencia de crimen
por unidad de tiempo puede apreciarse en la siguiente visualización.

<center>
	<img src="/img/posts/victimizacion/monthly.png" alt="freq" width="80%"/>
</center>

Estos datos indican cierta estacionalidad anual en los datos, además, los meses de
abril, julio y noviembre presentan los picos más altos. Un hecho interesante es la 
reducción considerable de crimen tras la declaración de cuarentena en marzo del 2020.
No obstante, durante el año 2021 y lo que va del 2022 los valores retoman su 
comportamiento pre-pandemia.

Los _wordclouds_ y _bag of words_ son modelos que representan la frecuencia de palabras
de forma visual y numérica respectivamente. El modelo de _wordcloud_ toma en cuenta
las ocurrencias en todo el _corpus_, sin embargo, en _bag of words_ se considera la
ocurrencia por documento, ponderando la relevancia de cada palabra en el _corpus_. 
Podemos apreciar ambos modelos a continuación, al tratarse de noticias sobre victimización
a empresas, es natural que las palabras mas comunes sean "soles", "dinero", "tienda" o 
"empresa".
	
<img src="/img/posts/victimizacion/freq_words.png" alt="wordcloud" width="98%"/>

De la mano con la frecuencia de las palabras, se puede aplicar el modelo de
_Latent Dirichlet Allocation_ para encontrar los tópicos más comunes dentro del 
_corpus_. Básicamente, un tópico representa el tema de un grupo de noticias, dividiendo
el _corpus_ en _n_ grupos. En esta ocasión, se calcularon 5 tópicos (grupos) y las 5 
palabras que representan a cada tópico.

```python
count_vectorizer = CountVectorizer()
count_data = count_vectorizer.fit_transform(data["normalized_text"])

number_topics = 5
number_words = 5

lda = LatentDirichletAllocation(n_components=number_topics)
lda.fit(count_data)

print_topics(lda, count_vectorizer, number_words)

# Topic 0: comisaría, agentes, lugar, policial, mujer
# Topic 1: banda, penal, criminal, investigación, fiscalía
# Topic 2: dinero, arma, fuego, soles, delincuente
# Topic 3: tienda, agentes, celulares, comercial, robar
# Topic 4: extorsionadores, empresa, extorsión, soles, trujillo
```

Por último, se realizó una visualización geográfica utilizando _folium_ y los datos de
latitud y longitud obtenidos durante la preparación de datos. Se construyó un 
_heatmap_ haciendo uso de los _plugins_ de _folium_.
La visualización refleja las cifras encontradas por departamento. Donde las ciudades
mas importantes tienen un mayor número de marcadores. Puede acceder a la 
[visualización interactiva](/resources/heatmap.html) y descubrir los lugares con altos
niveles de victimización de empresas y las noticias relacionadas.

## Modelamiento

El objetivo del proyecto es la predicción de la cantidad de crímenes que podrían ocurrir
en nuestro país durante los meses de mayo del 2022 a diciembre del 2022.
Ya que se trata de una serie temporal, se decidió utilizar el modelo ARIMA.

Para ajustar mejor el modelo, se consideró el uso de una variable exógena que determine el
comportamiento de la serie. Para ello, se entrenó el modelo con dos bases de datos: el
[PBI](https://estadisticas.bcrp.gob.pe/estadisticas/series/mensuales/resultados/PN01770AM/html) 
mensual y la
[tasa de desempleo](https://estadisticas.bcrp.gob.pe/estadisticas/series/mensuales/resultados/PN38063GM/html)
mensual. Se eligieron dichas variables debido a que, si la tasa de desempleo aumenta, es
muy probable que la tasa de crimen también lo haga. De igual manera, si el PBI disminuye,
la tasa de crimen podría aumentar.

El modelo ARIMA requiere de la definición de tres parámetros:
* orden: (p, d, q).
* variable exógena: desempleo o PBI.
* orden estacionario: (P, D, Q, m).

Para encontrar los valores óptimos se ejecutó el método de _Cross Validation_ y las
gráficas de diferenciación, auto-correlación y auto-correlación parcial.
Una vez encontrados dichos valores, se entrenó el modelo con los datos de entrenamiento.

```python
# Mejores valores obtenidos por Cross Validation
best_order = (2,1,0)
best_exog = "unemployment"
best_seasonal_order = (1,0,0,12)

# Construcción del modelo
best_model = ARIMA(
  train_data["count"],
  exog=train_data[best_exog],
  order=best_order,
  seasonal_order=best_seasonal_order
)
best_model_fit = best_model.fit()
```

Una vez ajustado el modelo, se pudo realizar el _forecasting_ de los meses siguientes.
Los datos de prueba son los primeros 4 meses del 2022. Si graficamos los datos actuales
y los valores conseguidos por el modelo, observamos que la coincidencia es alta.

<img src="/img/posts/victimizacion/modelling.png" alt="model" width="95%"/>

Se calculó el error cuadrado medio luego de aplicar el modelo a los datos de prueba.
Dicho valor debe acercarse a cero y nos indica qué tan bien se pudo modelar el comportamiento
temporal de la serie.

```python
total_score = mean_squared_error(
  final_df["count"], final_df["predicted"]
)
print(total_score)
# 22.26480087584455
```

### Elaborando predicciones

Ahora que tenemos el modelo que configura el número de crímenes a empresas en Perú,
podemos predecir cuántos crímenes podrían ocurrir en los próximos meses. Para ello, 
necesitamos predecir primero la tasa de desempleo con los datos actuales, ya que dichos
valores representan la variable exógena del modelo principal.

Se utilizó también ARIMA para modelar el comportamiento de la tasa de desempleo,
utilizando las gráficas de auto-correlación y diferenciación, se definieron los valores
de los parámetros y se construyó el siguiente modelo.

```python
# Construcción del modelo
unemployment_model = ARIMA(
  unemployment_serie, order=(1,1,1), seasonal_order=(1,1,0,12)
)
unemployment_model_fit = unemployment_model.fit()

# Forecasting
fc = unemployment_model_fit.forecast(steps=8, alpha=0.05)
```

Aplicando el método _forecast_, se obtuvieron los siguientes valores que predicen
la tasa de desempleo hasta el mes de diciembre del 2022.

<img src="/img/posts/victimizacion/unemployment.png" alt="unemployment" width="93%"/>

Ahora que contamos con los valores de desempleo, podemos usarlos como variables exógenas
del modelo de victimización. Aplicando el método de _forecast_ y márgenes de error,
obtenemos la siguiente visualización final.

<img src="/img/posts/victimizacion/prediction.png" alt="prediction" width="95%"/>

Lo cual nos indica qué, a nivel nacional, durante los próximos meses ocurrirán
la siguiente cantidad de eventos de victimización a empresas.

```python
final_model_fit.forecast(8, exog=fc).round()
# ----------------------
# | Date        | Freq |
# ----------------------
# | 2022-05-01  | 55.0 |
# | 2022-06-01  | 57.0 |
# | 2022-07-01  | 60.0 |
# | 2022-08-01  | 55.0 |
# | 2022-09-01  | 57.0 |
# | 2022-10-01  | 62.0 |
# | 2022-11-01  | 62.0 |
# | 2022-12-01  | 63.0 |
# ----------------------
```

Podemos observar que las cifras van en aumento, superando lamentablemente los números de 
los últimos 4 años. Si bien existen muchos factores que afectan el grado de crimen a 
las empresas peruanas, la tasa de desempleo ha demostrado ser un buen indicador del
aumento o disminución de la victimización. Explicar el porqué de los valores de desempleo
es un tema muy interesante que involucra conocimientos de economía, política y el análisis
de más fuentes de datos. Sin embargo, dicho abordaje no forma parte del alcance de este 
proyecto.

## Conclusiones y trabajos futuros

Gracias a la metodología aplicada, se pudo predecir exitosamente el número de crímenes
a empresas que podrían ocurrir en nuestro país durante el año 2022, cumpliendo el objetivo
del proyecto de ciencia de datos. En cuanto a los _insights_, los más interesantes fueron 
los siguientes:

* La determinación de las regiones con un mayor número de eventos de victimización: Si bien
  Lima, Piura y La Libertad son los departamentos con el mayor número de incidencias, las 
  regiones con el mayor _ratio_ de crimen por número de pobladores son Tumbes, Tacna y
  Moquegua.

* Los tópicos más comunes de las noticias recopiladas: Las noticias pueden clasificarse
  dentro de 5 grandes grupos representados por 5 palabras. Además, las palabras que
  resaltan se interceptan con las más relevantes de los modelos _bag of words_ y _wordcloud_.
  De entre todos estos modelos, destacan las palabras "dinero", "comisaria", "banda", 
  "tienda" y "extorsión".
  
* Mapa de calor de la geo-localización de las noticias: La elaboración de dicho mapa
  interactivo nos permitió apreciar la distribución geográfica de los lugares donde se
  presentan mas delitos contra empresas. Verificando que las zonas urbanas y céntricas son 
  las más peligrosas para las empresas y negocios.

En cuanto a las etapas del proyecto, el uso de _web scraping_ permitió la recopilación
rápida y flexible de datos. Por otra parte, fue adecuado trabajar con _python_ y _pandas_
por el volumen de los datos y la sencillez de las consultas. Además, existen muchas
librerías de ciencia de datos y estadística que ayudaron en el abordaje del proyecto de 
forma eficiente.

Como trabajos futuros, podrían probarse otros modelos de _forecasting_ para estudiar la
predicción del número de eventos de victimización a empresas, por ejemplo, modelos de
_machine learning_ supervisados y redes neuronales. En cuanto a la visualización, la
elaboración de un _choropleth map_ podría ser de gran ayuda para observar la frecuencia de
crímenes por departamentos, e inclusive por provincias.

## Referencias

* Gómez, E. (2022) _Course Notebooks and Slides_. Universidad Católica San Pablo: Arequipa, 
  Perú.
  
* Lao, R (2018) _A Beginner’s Guide to the Data Science Pipeline_ [online] disponible en:
  [_https://towardsdatascience.com/a-beginners-guide-to-the-data-science-pipeline-a4904b2d8ad3_](https://towardsdatascience.com/a-beginners-guide-to-the-data-science-pipeline-a4904b2d8ad3)
  
* Prabhakaran, S. (2021) _ARIMA Model – Complete Guide to Time Series Forecasting in Python_
  [online] disponible en:
  [https://www.machinelearningplus.com/time-series/arima-model-time-series-forecasting-python/](https://www.machinelearningplus.com/time-series/arima-model-time-series-forecasting-python/)
  
* Scrapy Developers (2022) _Scrapy 2.6 documentation_ [online] disponible en:
  [ https://docs.scrapy.org/en/latest_](https://docs.scrapy.org/en/latest/)
  
* Pandas Development Team (2022) _Pandas documentation_ [online] disponible en: 
  [_https://pandas.pydata.org/docs_](https://pandas.pydata.org/docs)
  
* Scikit-learn developers (2022) _User Guide_ [online] disponible en:
  [_https://scikit-learn.org/stable/user_guide.html_](https://scikit-learn.org/stable/user_guide.html)
  
* Perktold, J.; Seabold, S.; Taylor, J.; statsmodels developers (2022) _User Guide_ [online]
  disponible en:
  [_https://www.statsmodels.org/stable/user-guide.html_](https://www.statsmodels.org/stable/user-guide.html)

---
layout: post
title: "Victimización de empresas peruanas"
subtitle: "Predicción de la victimización de empresas en Perú durante el año 2022"
date: 2020-07-08
background: '/img/posts/victimizacion/victimizacion-cover.jpg'
---

**Curso**: Introducción a Ciencia de Datos (CS351)

**Profesor**: Dr. Erick Gomez Nieto

**Semestre**: 2022–1

La victimización de empresas es un problema latente en la sociedad peruana, día a día
escuchamos en los medios de comunicación noticias de robos y asaltos a establecimientos
comerciales. Si bien la capital Lima tiene un mayor número de incidencias, los demás
departamentos no son ajenos a este problema social. De hecho, un estudio realizado por el
INEI en año 2018 demostró qué, durante setiembre del 2017 y agosto del 2018, 28 de cada 100
empresas fueron víctimas de al menos un hecho delictivo. 

La violencia compromete el crecimiento económico del país, no sólo afecta directamente a
la población económicamente activa, sino que disminuye los incentivos directos para invertir,
así como los incentivos gubernamentales que se desvían para reforzar políticas de seguridad y
control. Motivado por este contexto, se realizó un estudio de ciencia de datos sobre la 
victimización de empresas peruanas, intentando modelar su comportamiento para poder predecir 
el numero de delitos que podrían ocurrir durante el año 2022.

Para abordar este proyecto se elaboro una solución basada en ciencia de datos, desde el
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
  e.g. robo, asalto, ladrones. Para realizar el _match_ se eliminan los _stop words_ del
  texto y se aplica _stemming_ para eliminar ambigüedades léxicas.
  
* Ahora que tenemos noticias sobre victimización, debemos realizar un filtro adicional,
  encontrar aquellas que hablen sobre empresas. Para ello, los _requests_ ya no se dirigen
  al API, ahora se envían al sitio web donde se encuentra la redacción completa de la noticia.
  
* El proceso de filtrado es similar al anterior, ahora se buscan _keywords_ relacionadas
  a empresas, e.g. negocio, comercio, joyería. Una vez filtradas las noticias, se guardan
  los campos más importantes usando selectores. Los campos guardados por cada noticia 
  fueron: _url_, _date_, _title_, _summary_, _body_ y _score_. Este último campo indica 
  qué tan seguro es   que dicha noticia efectivamente se trate de la victimización a una
  empresa.
  
* Finalmente, todos los _items_ son exportados en formato CSV. Con el método propuesto se
  lograron recopilar 3747 noticias.

## Preparación de datos

En esta etapa del proyecto se añaden campos de
geo-localización a los datos: _latitude_, _longitude_ y _location_. Para ello, se realizaron
los siguientes pasos:

<img src="/img/posts/victimizacion/cleaning_pipeline.png" alt="pipeline" width="95%"/>

* Se lee el archivo CSV obtenido mediante _web scraping_ usando la librería _pandas_, los
  datos se representan como un _DataFrame_. Adicionalmente, para encontrar la ubicación
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
  Los resultados fueron muy positivos y precisos, aquí un ejemplo del campo resultante:
  
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
  
* Finalmente se guardan los datos limpios y geo-localizados, logrando obtener 3670 noticias,
  aquellas que no obtuvieron una ubicación fueron eliminadas de la base de datos.

## Visualización y exploración de datos

Durante esta etapa se extraen _insights_ de los datos, la primera visualización realizada
fue la frecuencia de crímenes por departamento, como era de esperar, las regiones con
mayor población presentan un mayor número de ocurrencias.

<center>
	<img src="/img/posts/victimizacion/dep_freq.png" alt="freq" width="80%"/>
</center>

Sin embargo, estas cifras no reflejan el nivel de inseguridad de cada departamento. El
_ratio_ de victimización por cantidad de habitantes presenta una mejor visualización de
las incidencias criminales. Para ello, se consiguió una base de datos con la población
de cada región del Perú, haciendo la división correspondiente, obtenemos los siguientes
valores.

<center>
	<img src="/img/posts/victimizacion/dep_rate.png" alt="freq" width="80%"/>
</center>

Algunos departamentos mantienen sus posiciones, y algunos cambian considerablemente.
Gracias al gráfico de barras, podemos determinar que Tumbes, Tacna, Moquegua, Piura y 
Lambayeque son las cinco regiones cuyas empresas sufren de un mayor nivel de victimización
por número de habitantes durante los últimos 7 años.

Otro gráfico interesante es el número de hechos delictivos por mes. La frecuencia de crimen
por unidad de tiempo puede apreciarse en la siguiente visualización.

<center>
	<img src="/img/posts/victimizacion/monthly.png" alt="freq" width="80%"/>
</center>

Estos datos indican cierta estacionalidad anual en los datos, además, los meses de
Abril, Julio y Noviembre presentan los picos más altos. Un hecho interesante es la 
reducción considerable de crimen tras la declaración de cuarentena en marzo del 2020.
No obstante, durante el año 2021 y lo que va del 2022 los valores han ido volviendo
a su comportamiento pre-pandemia.

Los _wordclouds_ y _bag of words_ son modelos que representan la frecuencia de palabras
de forma visual y numérica respectivamente. El modelo de _wordcloud_ toma en cuenta
las ocurrencias en todo el _corpus_, sin embargo, en _bag of words_ se considera la
ocurrencia por documento, ponderando la relevancia de cada palabra en el _corpus_. 
Podemos apreciar ambos modelos a continuación, al tratarse de noticias sobre victimización
a empresas, es natural que las palabras mas comunes sean "soles", "dinero", "tienda" o 
"empresa".
	
<img src="/img/posts/victimizacion/freq_words.png" alt="wordcloud" width="98%"/>

Por último, se realizó una visualización geográfica utilizando _folium_ y los datos de
latitud y longitud obtenidos durante la preparación de datos. Se construyó un 
[_heatmap_](/resources/heatmap.html) haciendo uso de los _plugins_ de _folium_. 
La visualización refleja las cifras encontradas por departamento. Donde las ciudades
mas importantes tienen un mayor número de marcadores. Puede acceder a la 
[visualización interactiva](/resources/heatmap.html) y descubrir los lugares con altos
niveles de victimización de empresas.


## Modelamiento

<img src="/img/posts/victimizacion/modelling.png" alt="model" width="95%"/>

### Elaborando predicciones

<img src="/img/posts/victimizacion/prediction.png" alt="prediction" width="95%"/>


## Conclusiones y trabajos futuros



## Referencias


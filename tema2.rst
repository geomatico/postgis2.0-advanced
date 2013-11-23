.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


*****

.. note:: Los ejercicios propuestos en este tema han sido adaptados del libro `PostGIS CookBook <http://www.packtpub.com/postgis-to-store-organize-manipulate-analyze-spatial-data-cookbook/book>`_. En dicho libro se encuentran las soluciones a los ejercicios originales, así como otros ejercicios más complejos propuestos y resueltos. Recomendamos el uso de este libro como referencia para el aprendizaje de técnicas avanzadas con |PGIS|


Trabajando con datos vectoriales
********************************

Una base de datos ordinaria proporciona funciones para manipular los datos en una consulta. Estas funciones incluyen la concatenación de cadenas, operaciones matemáticas o la extración de información de las fechas. Una base de datos espacial debe proporcionar un completo juego de funciones para poder realizar análisis con los objetos espaciales: analizar la composición del objeto, determinar su relación espacial con otros objetos, transformarlo, etc. |PGIS| es capaz de realizar estas operaciones de análisis. Además, es actualmente la única base de datos espacial que permite mezclar **datos de tipo raster y vectorial de manera transparente para hacer análisis**. 

En este capítulo, no obstante, nos centraremos en el apartado vectorial de |PGIS|.


OGC y el Simple Feature Model
=============================
La OGC o Open Geospatial Consortium, define unas normas que serán utilizadas para la definición posterior de las geometrías. Estas son la SFA y la SQL/MM. Según esta última, las geometrías se definirán en función del siguiente esquema de herencia:

	.. image:: _images/ogc_sfs.png
	
Dentro de este esquema se definen tres tipos diferentes de geometrías:

	* **Geometrías abstractas**, que sirven para modelar el comportamiento de las geometrías que heredan de ellas. 
	* **Geometrías básicas**, son las principales de |PGIS|, soportadas desde los inicios de este y que soportan un análisis espacial completo.
	* **Geometrías circulares**, geometrías que soportan tramos circulares

Dimensión de una geometría	
--------------------------
El concepto de dimensión se explica de una manera sencilla mediante el uso de algunos ejemplos:

	* una entidad de tipo punto, tendrá dimensión 0
	* una de tipo linea, tendrá dimensión 1
	* una de tipo superficie, tendrá una dimensión igual a 2.
	
En |PGIS| utilizando una función especial podremos obtener el valor de esta dimensión. Si se trata de una colección de geometrías, el valor que se obtendrá será el de la dimensión de mayor valor de la colección.

Interior, contorno y exterior de las geometrías
-----------------------------------------------

Las definiciones las encontraremos en la norma. A continuación se indican los valores para las geometrías básicas.

	+---------------------+---------------------------+--------------------------------+
	|  **Tipos de         |      **Interior**         |         **Contorno**           |                            
	|  geometría**        |                           |                                |
	+---------------------+---------------------------+--------------------------------+
	|  ST_Point           | El propio punto o puntos  | Vacio                          |
	|                     |                           |                                |
	+---------------------+---------------------------+--------------------------------+
	|  ST_Linestring      | Puntos que permanecen     | Dos puntos finales             |
	|                     | cuando contorno se elimina|                                |
	+---------------------+---------------------------+--------------------------------+
	|ST_MultiLinestring   | Idem                      |Puntos de contorno de un nº     |
	|                     |                           |impar de elementos              |
	+---------------------+---------------------------+--------------------------------+
	|ST_Polygon           | Puntos del interior de    | Conjunto de anillos exterior   |
	|                     | los anillos               | e interior (Rings)             |
	+---------------------+---------------------------+--------------------------------+
	|ST_Multipolygon      | Idem                      | Conjunto de anillos exterior   |
	|                     |                           | e interior (Rings)             |
	+---------------------+---------------------------+--------------------------------+


WKT y WKB
=========
WKT es el acrónimo en inglés de ``Well Known Text``, que se puede definir como una codificación o sintaxis diseñada específicamente para describir objetos espaciales expresados de forma vectorial. Los objetos que es capaz de describir son: puntos, multipuntos, líneas, multilíneas, polígonos, multipolígonos, colecciones de geometría y puntos en 3 y 4 dimensiones. Su especificación ha sido promovida por un organismo internacional, el Open Geospatial Consortium, siendo su sintaxis muy fácil de utilizar, de forma que es muy generalizado su uso en la industria geoinformática. De hecho, WKT es la base de otros formatos más conocidos como el KML utilizado en Google Maps y Google Earth.

Muchas de las bases de datos espaciales, y en especial Postgresql, utiliza esta codificación cuando se carga la extensión PostGIS. Existe una variante de este lenguaje, pero expresada de forma binaria, denominada WKB (Well Know Binary), también utilizada por estos gestores espaciales, pero con la ventaja de que al ser compilada en forma binaria la velocidad de proceso es muy elevada.

A efectos prácticos la sintaxis WKT consta de una descripción de los vértices que componen la geometría. Para que esta forma de especificar las geometrías tengan sentido deben de acompañarse de una indicación de la referencia espacial o proyección cartográfica utilizada en dicho vector.

Ejemplos de sintaxis::

	Punto: POINT(30 50)
	Línea: LINESTRING(1 1, 5 5, 10 10, 20 20)
	Multilínea: LINESTRING( (1 1, 5 5, 10 10, 20 20),(20 30, 10 15, 40 5) )
	Polígono simple: POLYGON ((0 0, 10 0, 10 10, 0 0))
	Varios polígono en una sola geometría (multipolígono): POLYGON ( (0 0, 10 0, 10 10, 0 10, 0 0),( 20 20, 20 40, 40 40, 40 20, 20 20) )
	Geometrías de distinto tipo en un sólo elemento: GEOMETRYCOLLECTION(POINT(4 6),LINESTRING(4 6,7 10))
	Punto vacío: POINT EMPTY
	Multipolígono vacío: MULTIPOLYGON EMPTY
	
WKB acrónimo de ``Well Known Binary`` es la variante de este lenguaje, pero expresada de forma binaria, también utilizada por los gestores espaciales, pero con la ventaja de que al ser compilada en forma binaria la velocidad de proceso es muy elevada.

Tipos de datos espaciales
=========================
Una base de datos ordinaria tiene cadenas, fechas y números. Una base de datos
añade tipos adicionales para georreferenciar los objetos almacenados. Estos
tipos espaciales abstraen y encapsulan estructuras tales como el contorno y
la dimensión.

De forma simplificada, tenemos los siguientes tipos de datos espaciales:

 +----------------------------------+---------------------------------------+
 |    **Tipo de geometria**         |           **WKT**                     |
 +----------------------------------+---------------------------------------+
 |       POINT                      |   "POINT(0 0)"                        |
 +----------------------------------+---------------------------------------+
 |       LINESTRING                 |   "LINESTRING(0 0, 1 1, 2 2, 3 4)"    |
 +----------------------------------+---------------------------------------+
 |       POLYGON                    |   "POLYGON(0 0, 0 1, 1 1, 0 0)"       |
 +----------------------------------+---------------------------------------+
 |       MULTIPOINT                 |   "MULTIPOINT(0 0, 1 1, 2 2)"         |
 +----------------------------------+---------------------------------------+
 |       MULTILINESTRING            |"MULTILINESTRING ((10 10, 2 2, 10 40), |
 |                                  |(40 40, 30 30, 40 20, 30 10))"         |
 +----------------------------------+---------------------------------------+
 |       MULTIPOLYGON               |"MULTIPOLYGON (((3 2, 0 0, 5 4, 3 2))" |
 +----------------------------------+---------------------------------------+
 |       GEOMETRY COLLECTION        |"GEOMETRYCOLLECTION(                   |
 |                                  |      POINT(4 6),LINESTRING(4 6,7 10))"|
 +----------------------------------+---------------------------------------+


Relaciones espaciales
=====================

|PGIS| contiene un gran número de métodos encargados de comprobar relaciones espaciales. Estos métodos lo que hacen es verificar el cumplimiento de determinados predicados geográficos entre dos geometrías distintas. Los predicados geográficos toman dos geometrías como argumento, y devuelven un valor booleano que indica si ambas geometrías cumplen o no una determinada relación espacial. Las principales relaciones espaciales contempladas son equals, disjoint, intersects, touches, crosses, within, contains, overlaps.

	.. image:: _images/mas_predicados_espaciales.png
		:scale: 50 %
		
Figura: Ejemplos de predicados espaciales. Fuente: wikipedia. http://en.wikipedia.org/wiki/File:TopologicSpatialRelarions2.png

	.. image:: _images/touches.png
		:scale: 50 %

Figura: Ejemplos de la relación “Touch” (toca). Fuente: “OpenGIS® Implementation Standard for Geographic information - Simple feature access - Part 1: Common architecture”

	.. image:: _images/crosses.png
		:scale: 50 %

Figura: Ejemplos de la relación “Crosses” (cruza). Fuente: “OpenGIS® Implementation Standard for Geographic information - Simple feature access - Part 1: Common architecture”

	.. image:: _images/within.png	
		:scale: 50 %
	
Figura: Ejemplos de la relación “Within” (contenido en). Fuente: “OpenGIS® Implementation Standard for Geographic information - Simple feature access - Part 1: Common architecture”

	.. image:: _images/overlaps.png
		:scale: 50 %

Figura: Ejemplos de la relación “Overlaps” (solapa). Fuente: “OpenGIS® Implementation Standard for Geographic information - Simple feature access - Part 1: Common architecture”

Los principales métodos de la clase Geometry para chequear predicados espaciales entra la geometría en cuestión y otra proporcionada como parámetro son:

	* **Equals (A, B)**: Las geometrías son iguales desde un punto de vista topológico
	* **Disjoint (A, B)**: No tienen ningún punto en común, las geometrías son disjuntas
	* **Intersects (A, B)**:Tienen por lo menos un punto en común. Es el inverso de Disjoint
	* **Touches (A, B)**: Las geometrías tendrán por lo menos un punto en común del contorno, pero no puntos interiores
	* **Crosses (A, B)**: Comparten parte, pero no todos los puntos interiores, y la dimensión de la intersección es menor que la dimensión de al menos una de las geometrías
	* **Contains (A, B)**: Ningún punto de B está en el exterior de A. Al menos un punto del interior de B está en el interior de A
	* **Within (A, B)**: Es el inverso de Contains. Within(B, A) = Contains (A, B)
	* **Overlaps (A, B)**: Las geometrías comparten parte pero no todos los puntos y la intersección tiene la misma dimensión que las geometrías.
	* **Covers (A, B)**: Ningún punto de B está en el exterior de A. B está contenido en A.
	* **CoveredBy (A, B)**: Es el inverso de Covers. CoveredBy(A, B) = Covers(B, A)

Ejemplos
--------


Ejemplo 1
^^^^^^^^^

Obtengamos el registro completo de la vista ``lonlat_test_points`` que coincide con un valor de geometría determinado::
	
	select ogc_fid from toponimo where ST_Equals(wkb_geometry, '0101000020E66400000CFF1B8EF6500E415E2844FB1E8D4F41');

El resultado es 1.


Ejemplo 2
^^^^^^^^^

Veamos los polígonos de la tabla ``barrios_de_bogota`` que intersectan con el polígono con gid = 16::

	SELECT gid 
	FROM barrios_de_bogota 
	WHERE ST_Intersects(geom, (select geom from barrios_de_bogota where gid = 16))
	AND gid != 16
	
El resultado es::

 	gid 
	-----
   	8
  	15

Ejemplo 3
^^^^^^^^^

¿Cuál es el código postal del barrio en el que se encuentra el *COLEGIO PUBLICO PEDRO I*?::

	SELECT b.cod_postal from codigo_postal b, toponimo p
	WHERE ST_Contains(b.geom, p.wkb_geometry) and
	p.texto = 'COLEGIO PUBLICO PEDRO I'

El resultado es 41410


Ejemplo 4
^^^^^^^^^

Los nombres de los barrios por los que cruza el rio Bogotá

::

	# SELECT b.name 
	FROM barrios_de_bogota b JOIN waterways w 
	ON ST_Crosses(b.geom, w.geom)
	WHERE w.name = 'Rio Bogotá'

::

      	name      
	----------------
 	Bosa
 	Ciudad Kennedy
 	Fontibón
 	Engativá
 	Suba


Cualquier función que permita crear relaciones TRUE/FALSE entre dos tablas puede ser usada para manejar un JOIN espacial, pero comunmente las más usadas son:

	* ST_Intersects
	* ST_Contains
	* ST_DWithin


La última función se utiliza en cálculo de distancias. Algo que veremos con más detenimiento en el siguiente apartado


Cálculo de distancias y transformación de coordenadas
=====================================================

El cálculo de distancias es algo en apariencia trivial que no lo es tanto cuando pensamos en los sistemas de referencia y las proyecciones, algo que vimos en un apartado anterior. 

Por ejemplo, intentemos calcular la distancia existente entre Sevilla y Los Ángeles. Lo haremos usando la base de datos ``natural_earth2``. Lo primero que se nos ocurre es::
	
	select st_distance(o.the_geom, d.the_geom) as distance
	from ne_10m_populated_places o, ne_10m_populated_places d 
	where o.name = 'Seville' and d.name = 'Los Angeles' and d.sov_a3 = 'USA'

El resultado devuelto es **112.253818729785**. Es una cantidad en grados. ¿Cómo obtenemos la distancia en metros?

La primera opción sería transformar la geometría a un sistema de coordenadas que utilice los metros como unidad de medida. Por ejemplo, el *900913*. La consulta nos quedaría::
	
	select st_distance(st_transform(o.the_geom, 900913), st_transform(d.the_geom, 900913))/1000 as distance
	from ne_10m_populated_places o, ne_10m_populated_places d 
	where o.name = 'Seville' and d.name = 'Los Angeles' and d.sov_a3 = 'USA'

Que nos devuelve **12499.0249953461** km. El problema es que la distancia entre Sevilla y Los Ángeles es de 9428.38 km, como podemos ver `aquí <http://es.distance.to/Los-Angeles/Sevilla>`_. ¿Qué es lo que está sucediendo? 

La respuesta se deja como ejercicio propuesto para el lector. Lo veremos en el apartado de ejercicios.

A continuación, vamos a ver el uso de la función ``ST_DWithin``, que permite realizar búsquedas más rápidamente, limitando el radio de búsqueda. Busquemos el nombre de los establecimientos a una distancia máxima de 2 km de la tienda *Bogotanisimo.com*, en Bogotá::

	SELECT name
	FROM points
	WHERE 
		name is not null and
		name != 'Bogotanisimo.com' and
		ST_DWithin(
		     ST_Transform(geom, 21818),
		     (SELECT ST_Transform(geom, 21818)
			FROM points
			WHERE name='Bogotanisimo.com'),
		     2000
		   );
	
El resultado es ::

          name          
	------------------------
 	panaderia Los Hornitos


JOIN y funciones agregadas
==========================

El uso de las funciones espaciales de PostGIS en unión con las funciones de agregación de |PGSQL| nos da la posibilidad de realizar análisis espaciales de datos agregados. Una característica muy potente y con diversas utilidades. Veamos unos ejemplos.

Ejemplo 1
---------

Veamos un ejemplo sencillo: El numero de escuelas que hay en cada uno de los barrios de Bogota::

	#select b.name, count(p.type) as hospitals from barrios_de_bogota b join
	points p on st_contains(b.geom, p.geom) where p.type = 'hospital' 
	group by b.name order by hospitals desc

::

	name      | schools 
  ----------------+---------
   Suba           |       8
   Usaquén        |       5
   Los Mártires   |       3
   Teusaquillo    |       3
   Antonio Nariño |       3
   Tunjuelito     |       2
   Ciudad Kennedy |       2
   Engativá       |       1
   Fontibón       |       1
   Santa Fé       |       1
   Barrios Unidos |       1
   Ciudad Bolívar |       1
 


1. La clausula JOIN crea una tabla virtual que incluye los datos de los barrios y de los puntos de interés
2. WHERE filtra la tabla virtual solo para las columnas en las que el punto de interés es un hospital
3. Las filas resultantes son agrupadas por el nombre del barrio y rellenadas con la función de agregación count().


Veamos otro ejemplo. La estimación de datos censales, usando como criterio la distancia entre elementos espaciales.

Tomemos como base los datos vectoriales de los barrios de Bogotá y los datos vectoriales de vías de ferrocarril (tablas *barrios_de_bogota* y *railways*, respectivamente). Fijémonos en una línea de ferrocarril que cruza 3 barrios (Fontibón, Puente Aranda, Los Mártires)


	.. image:: _images/railways_and_neighborhoods.png 
		:scale: 50%

En la imagen, se han coloreado los polígonos de los barrios, de manera que los colores más claros suponen menos población.

Construyamos ahora un buffer de 1km alrededor de dicha línea de ferrocarril. Es de esperar que las personas que usen la línea sean las que vivan a una distancia razonable. Para ello, creamos una nueva tabla con el buffer::

	#CREATE TABLE railway_buffer as 
	SELECT 
		1 as gid, 
		ST_Transform(ST_Buffer(
			(SELECT ST_Transform(geom, 21818) FROM railways WHERE gid = 2), 1000, 'endcap=round join=round'), 4326) as geom;


Hemos usado la función *ST_Transform* para pasar los datos a un sistema de coordenadas proyectadas que use el metro como unidad de medida, y así poder especificar 1000m. Otra forma habría sido calcular cuántos grados suponen un metro en esa longitud, y usar ese número como parámetro para crear el buffer (más información en http://en.wikipedia.org/wiki/Decimal_degrees). 

Al superponer dicho buffer sobre la línea, el resultado es éste:

	.. image:: _images/railway_buffer.png
		:scale: 50%

Como se observa, hay 4 barrios que intersectan con ese buffer. Los tres anteriormente mencionados y Teusaquillo. 

Una primera aproximación para saber la población potencial que usará el ferrocarril sería simplemente sumar las poblaciones de los barrios que el buffer intersecta. Para ello, usamos la siguiente consulta espacial::

	# SELECT SUM(b.population) as pop 
	FROM barrios_de_bogota b JOIN railway_buffer r 
	ON ST_Intersects(b.geom, r.geom)

Esta primera aproximación nos da un resultado de **819892** personas. 

No obstante, mirando la forma de los barrios, podemos apreciar que estamos sobre-estimando la población, si utilizamos la de cada barrio completo. De igual forma, si contáramos solo los barrios cuyos centroides intersectan el buffer, probablemente infraestimaríamos el resultado.

La solución pasa por realizar una **estimación proporcional**. Algo que veremos en los ejercicios.


Ejemplo 2
---------

La función `ST_Polygonize <http://postgis.net/docs/manual-2.0/ST_Polygonize.html>`_ es una función agregada muy útil. Crea polígonos a partir de un conjunto de geometrías de entrada. El problema es que genera como salida un objeto de tipo ``GeometryCollection``, incompatible con la mayoría de programas de terceros. En la página de la documentación se sugiere utilizarla en conjunción con `ST_Dump <http://postgis.net/docs/manual-2.0/ST_Dump.html>`_ para extraer cada uno de los polígonos de la colección. Veamos cómo.

Primero, creamos la siguiente función PL/pgSQL::
	
	CREATE OR REPLACE FUNCTION polygonize_to_multi (geometry) RETURNS geometry AS $$
	
	WITH polygonized AS (
		SELECT ST_Polygonize($1) AS the_geom),
	dumped AS (
		SELECT (ST_Dump(the_geom)).geom AS the_geom FROM polygonized)

	SELECT ST_Multi(ST_Collect(the_geom)) FROM dumped;

	$$ LANGUAGE SQL;

Y con ella, vamos a crear una tabla que contenga las formas simuladas de los edificios a partir de puntos que representan los portales de Sevilla. La creación de la tabla consistirá en la creación de buffers 5 metros alrededor de los portales y quedarnos con el anillo exterior::
	
	CREATE TABLE portal_buildings_buffer AS
	WITH portal_query AS
		(SELECT ST_ExteriorRing(ST_SimplifyPreserveTopology((ST_Dump(ST_Union(ST_Buffer(wkb_geometry, 5)))).geom, 10)) AS the_geom 
		FROM portales_recortado) SELECT polygonize_to_multi(the_geom) AS the_geom from portal_query;

El aspecto de la tabla  en QGIS es el siguiente:

	.. image:: _images/ej2_edificios_desde_portales_qgis1.png
		:scale: 50%

	.. image:: _images/ej2_edificios_desde_portales_qgis2.png
		:scale: 50%


Validación de geometrías
========================

Una operación común cuando se trabaja con datos vectoriales es validar que dichos datos cumplen ciertas condiciones que los hacen óptimos para realizar análisis espacial sobre los mismos. O de otra forma, que cumplen ciertas condiciones topológicas.

Los puntos y las líneas son objetos muy sencillos. Intuitivamente, podemos afirmar que no hay manera de que sean *topológicamente inválidos*. Pero un polígono es un objeto más complejo, y debería cumplir ciertas condiciones. Y debe cumplirlas porque muchos algoritmos espaciales son capaces de ejecutarse rápidamente gracias a que asumen una consistencias de los datos de entrada. Si tuviéramos que forzar a que esos algoritmos revisaran las entradas, serían mucho más lentos.

Veamos un ejemplo de porqué esto es importante. Supongamos que tenemos este polígono sencillo::

	# POLYGON((0 0, 0 1, 2 1, 2 2, 1 2, 1 0, 0 0));

Gráficamente:

	.. image:: _images/poligono_invalido.png
		:scale: 50 %

Podemos ver el límite exterior de esta figura como un símbolo de *infinito* cuadrado. O sea, que tiene un *lazo* en el medio (una intersección consigo mismo). Si quisiéramos calcular el área de esta figura, podemos ver intuitivamente que tiene 2 unidades de área (si hablamos de metros, serían 2 metros cuadrados).

Veamos qué piensa PostGIS del área de esta figura::

	# SELECT ST_Area(ST_GeometryFromText('POLYGON((0 0, 0 1, 1 1, 2 1, 2 2, 1 2, 1 1, 1 0, 0 0))'));

El resultado será::

	# st_area
	 ---------
       	0

¿Qué es lo que ha sucedido aquí?

El algoritmo de cálculo de áreas de PostGIS (muy rápido) asume que los anillos no van a intersectar consigo mismos. Un anillo que cumpla las condiciones adecuadas para el análisis espacial, debe tener el área que encierra **siempre** en el mismo lado. Sin embargo, en la imagen mostrada, el anillo tiene, en una parte, el área encerrada en el lado izquierdo. Y en la otra, el área está encerrada en el lado derecho. Esto causa que las áreas calculadas para cada parte del polígono tengan valores opuestos (1 y -1) y se anulen entre si.

Este ejemplo es muy sencillo, porque podemos ver rápidamente que el polígono es inválido, al contener una intersección consigo mismo (algo que ESRI permite en un SHP, pero PostGIS no, porque implementa SFSQL: http://www.opengeospatial.org/standards/sfs). Pero, ¿qué sucede si tenemos millones de polígonos? Necesitamos una manera de detectar si son válidos o inválidos. Afortunadamente, PostGIS tiene una función para esto: **ST_IsValid**, que devuelve TRUE o FALSE::

	# SELECT ST_IsValid(ST_GeometryFromText('POLYGON((0 0, 0 1, 1 1, 2 1, 2 2, 1 2, 1 1, 1 0, 0 0))'))

Devuelve::

	# st_isvalid
	 ------------
 		 f

Incluso tenemos una función que nos dice la razón por la que una geometría es inválida::

	# SELECT ST_IsValidReason(ST_GeometryFromText('POLYGON((0 0, 0 1, 1 1, 2 1, 2 2, 1 2, 1 1, 1 0, 0 0))'));

Que devuelve::

	# st_isvalidreason
	------------------------
 	Self-intersection[1 1]


Ejercicios
==========

A continuación, se proponen unos ejercicios para que el alumno resuelva con el apoyo del docente.

Ejercicio 1: Join espacial para mezclar los campos de dos tablas
----------------------------------------------------------------

Añadir a la tabla ``toponimo`` un campo adicional que contenga el código postal, obtenido de la tabla ``codigo_postal``. Evitar el uso de consultas anidadas mediante la clausula ``WITH``.



Ejercicio 2: Añadir otro campo más
----------------------------------

Añadir a la tabla ``codigo_postal`` un campo adicional que contenga el nombre del munipio para cada polígono, obtenido de la tabla que se proporciona como base::

	create table centroides_sevilla as select c.* 
	from centroides_territorios_etrs89 c, codigo_postal cp 
	where st_contains(cp.geom, c.wkb_geometry)


Ejercicio 3: Distancias
-----------------------

Investigar porqué el cálculo de la distancia entre Sevilla y Los Ángeles es erróneo, y modificar la consulta para que devuelva el valor correcto.

.. note:: Pista: Recordar el apartado de sistemas de referencia. ¿Qué problema presenta el sistema de referencia en la que se encuentran los datos (4326)?


Ejercicio 4: Mejora del cálculo de distancias
---------------------------------------------

En este ejercicio vamos a intentar obtener los 10 lugares más cercanos al punto de interés que representa la catedral en la tabla de topónimos (gid = 373)

Una posible consulta que nos daría el resultado deseado es::
	
	with searchpoint as (
		select wkb_geometry from toponimo where ogc_fid = 373
	) select st_distance(searchpoint.wkb_geometry, toponimo.wkb_geometry) as distance
	from searchpoint, toponimo order by st_distance(searchpoint.wkb_geometry, toponimo.wkb_geometry) limit 10;

El problema de esta consulta es que no es escalable. Si la tabla tuviera millones de registros, sería muy lenta. La podemos mejorar limitando la búsqueda a un radio::
	
	with searchpoint as (
	select wkb_geometry from toponimo where ogc_fid = 373
	) select st_distance(searchpoint.wkb_geometry, toponimo.wkb_geometry) as distance
	from searchpoint, toponimo WHERE ST_DWithin(searchpoint.wkb_geometry, toponimo.wkb_geometry, 1000)
	ORDER BY ST_Distance(searchpoint.wkb_geometry, toponimo.wkb_geometry) LIMIT 10;

Esto funciona siempre y cuando la longitud de la ventana sea adecuada (por ej: no llega a buscar 1000 registros, porque no hay tantos dentro de la ventana. La aproximación anterior busca sin límite). Si no somos capaces de saber la longitud de la ventana, no es un gran método.

Investigar qué operadores proporciona |PGIS| para mejorar este caso de uso (búsqueda de vecinos más cercanos)

.. note:: Hay una buena introducción al problema en `http://boundlessgeo.com/2011/09/indexed-nearest-neighbour-search-in-postgis/`_


Ejercicio 5: Estimación propocional
------------------------------------

En uno de los ejemplos, intentamos calcular el número de potenciales usuarios de una vía de ferrocarril basándonos en la población censada en los barrios por donde pasaba. No obstante, mirando la forma de los barrios, podemos apreciar que estamos sobre-estimando la población, si utilizamos la de cada barrio completo. De igual forma, si contáramos solo los barrios cuyos centroides intersectan el buffer, probablemente infraestimaríamos el resultado.

En lugar de esto, podemos asumir que la población estará distribuida de manera más o menos homogénea (esto no deja de ser una aproximación, pero más precisa que lo que tenemos hasta ahora). De manera que, si el 50% del polígono que representa a un barrio está dentro del área de influencia (1 km alrededor de la vía), podemos aceptar que el 50% de la población de ese barrio serán potenciales usuarios del ferrocarril. Sumando estas cantidades para todos los barrios involucrados, obtendremos una estimación algo más precisa. Habremos realizado una suma proporcional.

Para realizar esta operación, vamos a construir una función en PL/pgSQL. Esta función la podremos llamar en una query, igual que cualquier función espacial de PostGIS::
	
	CREATE OR REPLACE FUNCTION public.proportional_sum(geometry, geometry, numeric)
	RETURNS numeric AS

	$BODY$

	SELECT $3 * areacalc FROM
	(SELECT (ST_Area(ST_Intersection($1, $2))/ST_Area($2))::numeric AS areacalc) AS areac;

	$BODY$
	LANGUAGE sql VOLATILE

Modificar la consulta utilizada en el ejemplo para que, introduciendo el uso de la función ``proporcional_sum``, obtengamos una estimación más precisa de cuántas personas podrían usar potencialmente la vía de ferrocarril. Recordemos que la consulta era::
	
	SELECT SUM(b.population) as pop
	FROM barrios_de_bogota b JOIN railway_buffer r
	ON ST_Intersects(b.geom, r.geom)



Ejercicio 6: Simplificación de geometrías
-----------------------------------------

Mediante el uso de ``ST_Union`` y agregación, crear una versión simplificada de la tabla ``barrios_de_bogota``


Ejercicio 7: Arreglando geometrías
----------------------------------

Comprobar si la tabla ``TM_WORLD_BORDERS`` contiene geometrías inválidas. Si es así, arreglarlas mediante el uso de `ST_MakeValid <http://postgis.net/docs/manual-2.0/ST_MakeValid.html>`_ 


Ejercicio 8: Trabajando con trazas GPS
---------------------------------------

Utilizando la tabla de puntos con las trazas GPS cargadas en el primer tema, vamos a construir una tabla que contenga una línea que los une a todos::
	
	select ST_MakeLine(the_geom) AS the_geom,
		trip_date::date,
		MIN(trip_time) as start_time	
		MAX(trip_time) as end_time
	INTO line_tracks
	FROM (
		SELECT the_geom,
			"time"::date as trip_date,
			"time" as trip_time
		FROM gps_track_points
		ORDER BY trip_time
	) AS foo GROUP BY trip_date;

	CREATE INDEX gps_track_points_geom_idx ON gps_track_points USING gist(the_geom);
	CREATE INDEX line_tracks_idx ON line_tracks USING gist(the_geom);
	CREATE INDEX lim_adm_esp_idx ON lim_adm_esp USING gist(geom);

Calcular la longitud total recorrida a partir de la tabla anterior.

 

 

.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


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


JOIN y GROUP BY
===============

El uso de las relaciones espaciales junto con funciones de agregacion, como **group by**, permite operaciones muy poderosas con nuestros datos. Veamos un ejemplo sencillo: El numero de escuelas que hay en cada uno de los barrios de Bogota::

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



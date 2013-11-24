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

.. note:: Los materiales teóricos de este capítulo están basados en parte en el `Taller de Routing impartido por María Arias de Reyna <http://delawen.github.io/Taller-Routing>`_ en las Jornadas de SIG Libre de Girona. En cuanto a los materiales prácticos, parte han sido adaptados del libro `PostGIS CookBook <http://www.packtpub.com/postgis-to-store-organize-manipulate-analyze-spatial-data-cookbook/book>`_. En dicho libro se encuentran las soluciones a los ejercicios originales, así como otros ejercicios más complejos propuestos y resueltos. Recomendamos el uso de este libro como referencia para el aprendizaje de técnicas avanzadas con |PGIS|. Otra parte ha sido extraída del `Workshop de pgRouting impartido en FOSS4G2013 <http://workshop.pgrouting.org/>`_ 



Cálculo de rutas
****************


|PGIS|, además de permitirnos operar con información raster y vectorial, puede cumplir otras funciones. Una de ellas, es el cálculo de rutas. O de otra forma, responder a la pregunta: **¿cuál es el camino más corto para ir de A a B?**

Ya existen soluciones establecidas que dan respuesta a esta pregunta, como las implantadas por Google Maps o Bing Maps. Son herramientas gratuitas, pero todas ellas son propiedad de empresas privadas, y creadas con fines comerciales. 

Como alternativa a estas herramientas, hay diferentes esfuerzos para proveer una solución de ruteo basada en software libre...

.. todo:: Terminar introducción (hablar de todos los proyectos) 


Introducción
============


Cálculo de rutas con pgRouting
==============================

A continuación, veremos cómo implementar el cálculo de rutas usando la extensión de |PGIS| pgRouting. Usaremos para ello los datos 

Para los impacientes
--------------------

Vamos a crear una tabla simple y activar los parámetros necesarios para hacer un cálculo de ruta usando el algoritmo básico (algoritmo de Dijkstra). Esta tabla almacenará las aristas de una sencilla red topológica que construiremos en un paso posterior::
	
	CREATE TABLE routing_simple
	(
  		gid serial NOT NULL,
  		cost double precision,
  		the_geom geometry(LineString,4326),
  		CONSTRAINT routing_simple_pkey PRIMARY KEY (gid )
	)

Insertamos unos pocos datos simples::
	
	INSERT INTO routing_simple(gid, the_geom) values
		(1, st_setsrid(st_geomfromtext('LINESTRING(0 0, 1 1)'),4326)),
		(2, st_setsrid(st_geomfromtext('LINESTRING(1 1, 1 2)'),4326)),
		(3, st_setsrid(st_geomfromtext('LINESTRING(1 1, 2 2)'),4326)),
		(4, st_setsrid(st_geomfromtext('LINESTRING(1 2, 2 2)'),4326));

Actualizamos el coste a la longitud de la geometría::
	
	UPDATE routing_simple set cost = st_length(the_geom)


Establecemos las relaciones origen y destino::
	
	ALTER TABLE routing_simple ADD COLUMN "source" integer;
	ALTER TABLE routing_simple ADD COLUMN "target" integer;

Creamos una topología basada en nuestra tabla::
	
	SELECT pgr_createTopology('routing_simple', 0.00001, 'the_geom', 'gid');

.. warning:: El segundo parámetro es la tolerancia. Depende de la proyección de los datos. Suele ser en grados o metros

Como resultado, se nos creará una tabla de vértices. Éste es el aspecto que tiene nuestra red (los nodos están etiquetados con un número más grande):

	.. image::  _images/ej4_pgrouting_simple1.png

.. note:: La función ``pgr_createTopology`` asume que la tabla a partir de la cuál se va a crear la topología contiene los campos **source** y **target**. Para más información, visitar `http://docs.pgrouting.org/dev/src/common/doc/functions/create_topology.html`_

.. note:: En nuestro caso, la tabla es muy pequeña. Pero es buena idea añadir índices en las columnas *source* y *target*::
	
	CREATE INDEX source_idx ON routing_simple("source");
	CREATE INDEX target_idx ON routing_simple("target");

Y ya podríamos comenzar con las consultas::
	


Algoritmo de Dijkstra
---------------------

Como ya hemos comentado en la introducción, el algoritmo básico de cálculo de rutas es el de Dijkstra. Vamos a ver cómo aplicarlo a nuestra sencilla topología. Calculemos la ruta más corta entre el nodo 1 y el nodo 4::

	SELECT pgr_dijkstra('
                SELECT gid::int as id, source::int, target::int,
                        cost::float8 as cost FROM routing_simple',
                1, 4, false, false
        );

El resultado es el siguiente::
	
	pgr_dijkstra       
	-------------------------
 	(0,1,1,1.4142135623731)
 	(1,2,3,1.4142135623731)
 	(2,4,-1,0)

Anidando la consulta anterior, podemos ver más claramente lo que significan cada uno de los parámetros de las tuplas devueltas::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_dijkstra('
                SELECT gid::int as id, source::int, target::int,
                        cost::float8 as cost FROM routing_simple',
                1, 4, false, false
        );

El resultado es el siguiente::

	seq | node | edge |      cost       
   -----+------+------+--------------------
   	  0 |    1 |    1 | 1.4142135623731
   	  1 |    2 |    3 | 1.4142135623731
   	  2 |    4 |   -1 |               0


Siguiendo la tabla de nodos, podemos ver la secuencia calculada. La columna de coste nos dice el coste de cada salto.

La sintáxis utilizada puede parecer poco intuitiva en un primer momento. En general, calcular una ruta con pgRouting se hace siguiendo este esquema::

	select pgr_<algorithm>(<SQL para las aristas>, <nodo inicial>, <nodo final>, <opciones adicionales>)

El algoritmo a utilizar puede ser cualquiera de los listados `aquí <http://docs.pgrouting.org/2.0/en/src/index.html#routing-functions>`_ 

El primer parámetro es una cláusula *SELECT* destinada a obtener, de la tabla de aristas, aquellas involucradas en la ruta. Dicha consulta debe *extraer* de la tabla de aristas una o más de las siguientes columnas::
	
	id:	int4
	source:	int4
	target:	int4
	cost:	float8
	reverse_cost:	float8
	x:	float8
	y:	float8
	x1:	float8
	y1:	float8
	x2:	float8
	y2:	float8


Como mínimo, serán necesarios *source*, *target* y *cost*. Según los campos que contenga la tabla, el SQL de aristas puede ser::
	
	SELECT source, target, cost FROM edge_table;
	SELECT id, source, target, cost FROM edge_table;
	SELECT id, source, target, cost, x1, y1, x2, y2 ,reverse_cost FROM edge_table


Si los campos de la tabla de aristas tienen diferentes nombres a los mostrados, se pueden renombrar mediante el uso de *as*::
	
	SELECT gid as id, src as source, target, cost FROM othertable;

.. seealso:: Para más información sobre los parámetros de la consulta *SELECT*, consultar `este enlace <http://docs.pgrouting.org/2.0/en/doc/src/tutorial/custom_query.html#custom-query>`_


Los campos de *<origen>* y *<destino>* son simplemente los identificadores de los nodos origen y destino de la ruta

.. note:: El algoritmo de Dijkstra no requiere que origen y destino tengan asociada información geográfica (un SRS)

En cuanto a las opciones adicionales, son dos campos booleanos:
	* *directed*: Si le asignamos ``true`` significa que el grafo que representa nuestra red topológica es un `grafo dirigido <http://en.wikipedia.org/wiki/Directed_graph>`_
	* *has_rcost*: Si le asignamos ``true`` la columna ``reverse_cost`` del conjunto de filas obtenido como resultado será usado como coste para el camino opuesto de la arista en la que se encuentra.

El resultado de la consulta es, como ya se ha adelantado, un conjunto de filas. Cada fila es una tupla que incluye los siguientes campos::
	
	SELECT id, source, target, cost [,reverse_cost] FROM edge_table 

Un coste de -1 indica una arista que no se puede seguir. El campo ``reverse_cost`` solo tiene sentido si los parámetros ``directed`` y ``has_rcost`` son true.

.. todo:: Revisar esto último. No lo acabo de ver claro. Además, no logro hacerlo funcionar pasando true en esos parámetros.





Algoritmo A*
------------


Ejercicios
==========

Veamos unos ejercicios sobre routing



Ejercicio 1
-----------

Para este ejercicio, vamos a usar una sencilla red topológica ya creada. Para ello, basta con que carguemos los datos encontrados en el fichero *vectorial/pgrouting/sample.sql* de nuestra carpeta de datos::

	$ psql -d pgrouting-workshop -f vectorial/pgrouting/sample.sql

Vamos a hacer nuestros vértices visibles como geometrías, de manera que podamos tener más claro lo que estamos haciendo en *idioma PostGIS*::
	
	ALTER TABLE vertex_table ADD COLUMN the_geom geometry(Point,0);
	UPDATE vertex_table SET the_geom = ST_MakePoint(x,y)

El aspecto de la red creada es el siguiente:

	.. image:: _images/trsp-test-image.png

La siguiente captura está hecha con QGIS, etiquetando los vértices (tabla *vertex_table*) y las aristas (tabla *edge_table*)
	
	.. image:: _images/ej4_pgrouting_topo_simple.png

.. note:: En esta segunda captura, lo que se etiqueta en las aristas es el coste. En la primera captura, era el id de la arista

Calcular el camino más corto del nodo 13 al nodo 4. Transformar el resultado en geometrías de tipo LineString, y crear con ellas una tabla para poder visualizarla en PostGIS.


Ejercicio 2
-----------

Repetir el ejercicio 1 con el algoritmo A*. Asegurarse de que la tabla de aristas cumple las condiciones necesarias para poder ejecutar el algoritmo






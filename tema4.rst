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



Cálculo de rutas con pgRouting
******************************

Veremos a continuación en profundidad la extensión de |PGIS| que nos permite realizar cálculo de rutas en grafos.

Introducción
============

|PGIS|, además de permitirnos operar con información raster y vectorial, puede cumplir otras funciones. Una de ellas, es el cálculo de rutas. O de otra forma, responder a la pregunta: **¿cuál es el camino más corto para ir de A a B?**

Ya existen soluciones establecidas que dan respuesta a esta pregunta, como las implantadas por Google Maps o Bing Maps. Son herramientas gratuitas, pero todas ellas son propiedad de empresas privadas, y creadas con fines comerciales. 

Como alternativa a estas herramientas, hay diferentes esfuerzos para proveer una solución de ruteo basada en software libre:
	* **pgRouting**: extensión de PostGIS para dotar a la base de datos PostgreSQL de capacidad de cálculo de rutas
	* **OSRM**: Open Source Routing Machine. Implementación en C++ de un motor de cálculo de rutas. Es un proyecto de Software Libre
	* **OpenLS**: Open Location Service. Estandar OGC para servicios basados en localización. El cálculo de rutas es una de sus funcionalidades. Está pensado para usarse como SaaS (Software as a Service), aislándose de la implementación concreta. Funciona a través de una API XML. Una implementación de este estandar la proporciona Emergya, con *GoFleetLS*, un producto también Open Source implementado en Java. Actualmente (2013), lleva un par de años sin ser actualizado.

.. seealso:: Toda la información del proyecto OSRM está disponible `aquí <http://project-osrm.org/>`_
.. seealso:: El estandar OpenLS puede ser consultado `aquí <http://www.opengeospatial.org/standards/ols>`_
.. seealso:: La implementación de OpenLS proporcionada por Emergya (GoFleetLS), puede ser descargada de `aquí <https://github.com/Emergya/GoFleetLSServer>`_

En el presente capítulo, nos centraremos en el uso de pgRouting, por ser la única herramienta actualmente disponible para realizar el cálculo de rutas utilizando |PGIS| como soporte.

Es importante destacar, no obstante, que pgRouting proporciona una serie de algoritmos para ser ejecutados sobre una red topológica de nodos y aristas que construye a partir de nuestros datos. En otras palabras, **nos proporciona *ladrillos* para construir un sistema de ruteo, pero no el sistema en si mismo**. Por ejemplo, pgRouting **no** es capaz de:
	* Trabajar con diferentes fuentes de datos y solucionar todos sus problemas topológicos para construir una red bien formada. Solo es capaz de transformar una tabla de líneas en una red topológica, asignando vértices en cada intersección. Es posible *afinar* ciertas cosas mediante el paso de parámetros a algunas funciones, pero el trabajo de preparar los datos corre del lado del usuario.
	* Realizar cálculos de rutas arbitrarios entre cualquier punto de mi red con cualquier otro. Para calcular una ruta entre dos puntos, debe haber nodos válidos de la red en esos puntos
	* Calcular automáticamente diferentes perfiles de transporte (caminando, bicicleta, autobús, coche, tren, etc). Pero nos proporciona la flexibilidad para inventar nosotros el sistema de hacerlo

Es decir, pgRouting no es una caja negra mágica que convierte datos caóticos en una red topológica y nos proporciona todo lo necesario para proprocionar una servicio de ruteo al estilo de Google Maps o Bing Maps. Es la base sobre la que construir un sistema semejante. Al igual que |PGIS| en si mismo no es una herramienta comparable a ArcGIS, es una base para poder comenzar a construirlo.



Cálculo de rutas con pgRouting
==============================

A continuación, veremos cómo implementar el cálculo de rutas usando la extensión de |PGIS| pgRouting. La máquina virtual que estamos usando (OSGeo Live 7) incluye ya una base de datos con todas las tablas necesarias para poder ejecutar las consultas de cálculo de rutas. Dicha base de datos se llama ``pgrouting``. Si el alumno solo está interesado en la realización de consultas para calcular rutas, puede saltar directamente al apartado `Algoritmos básicos de pgRouting`_. En los apartados previos a ese, veremos como construir desde cero una base de datos utilizable por pgRouting. Recomendamos, en cualquier caso, que los ejercicios que requieran crear o manipular tablas se hagan en una base de datos nueva, diferente a la ya existente ``pgrouting``.


Para los impacientes
--------------------

Veamos para empezar, una manera de comenzar a trabajar rápidamente. En los siguientes apartados, nos pararemos en las bases teóricas de cada parte

Lo básico
^^^^^^^^

Creamos una tabla simple y activamos los parámetros necesarios para hacer un cálculo de ruta usando el algoritmo básico: el **algoritmo de Dijkstra**. 

Trabajaremos sobre la base de datos ``pgrouting-workshop``, creada en el primer tema del este curso. En dicha base de datos ya hay una tabla llamada *ways* preparada para ejecutar consultas de routing, pero crearemos otra tabla sencilla nosotros mismos, para ver claramente los pasos. 

Ahora creemos la primera tabla. Esta tabla almacenará las aristas de una sencilla red topológica que construiremos en un paso posterior::
	
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


Actualizamos el coste de cada arista a la longitud de la geometría::
	
	UPDATE routing_simple set cost = st_length(the_geom)

Y éste es el aspecto que tiene nuestra red simple:

	.. image::  _images/ej4_pgrouting_simple1.png


Vamos a terminar de generar lo necesario para poder calcular rutas en la red. Establecemos las relaciones origen y destino::
	
	ALTER TABLE routing_simple ADD COLUMN "source" integer;
	ALTER TABLE routing_simple ADD COLUMN "target" integer;

Creamos una topología basada en nuestra tabla::
	
	SELECT pgr_createTopology('routing_simple', 0.00001, 'the_geom', 'gid');


Como resultado, se nos creará una tabla de vértices de nuestra topología.

.. note:: La función ``pgr_createTopology`` asume que la tabla a partir de la cuál se va a crear la topología contiene los campos **source** y **target**. Para más información, visitar `http://docs.pgrouting.org/dev/src/common/doc/functions/create_topology.html`_

.. note:: En nuestro caso, la tabla es muy pequeña. Pero es buena idea añadir índices en las columnas *source* y *target*

::
	
	CREATE INDEX source_idx ON routing_simple("source");
	CREATE INDEX target_idx ON routing_simple("target");

Y ya podríamos comenzar con las consultas.


Veremos a continuación los dos algoritmos más conocidos para el cálculo del camino más corto entre dos puntos: Dijkstra y A*.


Cálculo del camino más corto con el algoritmo de Dijkstra
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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


Cálculo del camino más corto con el algoritmo A*
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Veamos ahora el otro algoritmo más popular de cálculo del camino más corto. Nuestra sencilla tabla requiere que le añadamos algunos campos más::

	ALTER TABLE routing_simple ADD COLUMN x1 double precision;
	ALTER TABLE routing_simple ADD COLUMN y1 double precision;
	ALTER TABLE routing_simple ADD COLUMN x2 double precision;
	ALTER TABLE routing_simple ADD COLUMN y2 double precision;

Ahora, actualizaremos esos campos con el valor que requieren::
	
	UPDATE routing_simple SET x1 = ST_x(ST_PointN(the_geom, 1));
	UPDATE routing_simple SET y1 = ST_y(ST_PointN(the_geom, 1));
	UPDATE routing_simple SET x2 = ST_x(ST_PointN(the_geom, ST_NumPoints(the_geom)));
	UPDATE routing_simple SET y2 = ST_y(ST_PointN(the_geom, ST_NumPoints(the_geom)));	


Ya estamos listos para ejecutar la consulta::
	
	SELECT pgr_astar('
                SELECT gid::int as id, source::int, target::int,
                        cost::float8 as cost, x1, x2, y1, y2 FROM routing_simple',
                1, 4, false, false
        );


Como resultado, obtenemos lo mismo que con el algoritmo de Dijkstra::

	pgr_dijkstra       
	-------------------------
 	(0,1,1,1.4142135623731)
 	(1,2,3,1.4142135623731)
 	(2,4,-1,0)


Extrayendo los campos del resultado por separado::

	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_dijkstra('
                SELECT gid::int as id, source::int, target::int,
                        cost::float8 as cost, x1, x2, y1, y2 FROM routing_simple',
                1, 4, false, false
        );


El resultado es, como en el caso del algoritmo de Dijkstra::


	seq | node | edge |      cost       
   -----+------+------+--------------------
   	  0 |    1 |    1 | 1.4142135623731
   	  1 |    2 |    3 | 1.4142135623731
   	  2 |    4 |   -1 |               0




Veremos con más detenimiento en los siguientes apartados cada uno de los pasos que hemos dado.


Creación de una topología
-------------------------

Del sencillo ejemplo anterior extraigamos una idea: **es necesario crear una topología para poder ejecutar pgRouting**. Los datos de |OSM| son especialmente interesantes para trabajar con pgRouting, por su estructura, pero no siempre vamos a poder o querer trabajar con datos de |OSM|. Tal vez queramos construir una topología a partir de nuestros datos. Veremos cómo hacerlo

.. warning:: Hay que tener en cuenta que, para crear una topología con la que pgRouting pueda calcular rutas, nuestros datos han de cumplir las condiciones mínimas para que pgRouting trabaje con ellos, como vimos en el apartado anterior. Debe haber al menos una tabla de aristas con los campos ``source``, ``target``, de tipo integer o big int. Estos campos serán rellenados.

.. seealso:: `Documentación de la función pgr_CreateTopology <http://docs.pgrouting.org/dev/src/common/doc/functions/create_topology.html>`_

Los datos con los que trabajaremos están en la base de datos ``pgrouting``. Concretamente, la tabla ``ways`` (obtenido a partir del fichero ``sampledata_notopo.sql``, del que hablamos en el primer tema de este curso). Su aspecto es éste::

	Table "public.ways"
  	  Column  |           Type            | Modifiers
	----------+---------------------------+-----------
 	 gid      | bigint                    |
 	 class_id | integer                   | not null
 	 length   | double precision          |
 	 name     | character(200)            |
 	 osm_id   | bigint                    |
 	 the_geom | geometry(LineString,4326) |
	 Indexes:
    	 "ways_gid_idx" UNIQUE, btree (gid)
    	 "geom_idx" gist (the_geom)

Eso serían los campos típicos para poder identificar una calle: un id, un nombre, una longitud y una geometría para poder ser mostrada en cualquier visor de |PGIS|. Pero aun no tenemos información topológica. 

Para empezar, tendremos que crear los campos ``source`` y ``target`` en nuestra tabla::
	
	ALTER TABLE ways ADD COLUMN "source" integer;
	ALTER TABLE ways ADD COLUMN "target" integer;

Lo siguiente sería llamar a la función ``pgr_createTopology``, para crear una topología a partir de la tabla de aristas::
	
	SELECT pgr_createTopology('ways', 0.00001, 'the_geom', 'gid');

.. warning:: El segundo parámetro es la tolerancia. Depende de la proyección de los datos. Suele ser en grados o metros

Añadimos índices a los campos ``source`` y ``target``::
	
	CREATE INDEX source_idx ON ways("source");
	CREATE INDEX target_idx ON ways("target");

Con estas operaciones, ya tenemos nuestra topología creada. Lo que se ha hecho ha sido:
	* Crear una tabla con los vértices de nuestra red (*ways_vertices_pgr*)
	* Actualizar la tabla de aristas (*ways*) con el origen y el destino de cada una de ellas, referenciando la tabla de vértices


Algoritmos básicos de pgRouting
===============================

Como ya hemos visto en el apartado anterior, hay dos algoritmos básicos para obtener el camino más corto entre dos puntos usando pgRouting:
	* Dijkstra
	* A*

Veamos a continuación cómo funcionan estos algoritmos


Algoritmo de Dijkstra:
----------------------

Como ya vimos en un apartado anterior, el algoritmo básico para obtener rutas es el de Dijstra. Su funcionamiento se entiende bien en la siguiente imagen (obtenida de la presentación de María Arias de Reina):

	.. image:: _images/Dijksta_Anim.gif
 
El algoritmo avanza nodo a nodo por el camino que implique menor coste. Matemáticamente, está demostrado que siempre encuentra la ruta de menor coste entre dos nodos cualesquiera de la red.

Su mayor problema es que tarda mucho. Su orden de crecimiento es O(|E| + |V| log(|V|)). Eso significa que, en una ciudad de unos 30.000 viales, como puede ser Sevilla, si se tarda un milisegundo en procesar cada vial y otro en procesar cada cruce, el cálculo de una ruta que atraviese la ciudad tardaría:

	O(|30.000| + |30.000| log(|30.000|))~ 300.000 ms ~ 5 minutos

Es por eso que se idearon otros algoritmos más rápidos para su aplicación práctica.

Con respecto a la ejecución del algoritmo, como ya se vio en el primer apartado, bastaría con::
	
	SELECT seq, id1 AS node, id2 AS edge, cost FROM pgr_dijkstra('
                SELECT gid AS id,
                         source::integer,
                         target::integer,
                         length::double precision AS cost
                        FROM ways',
                30, 60, false, false);


Esa consulta nos daría el camino más corto entre los nodos 30 y 60, considerando el grafo como no dirigido y sin tener en cuenta el coste inverso de cada arista.

Caso de que quisiéramos poder calcular el coste inverso, necesitaríamos añadir un campo a la tabla *ways*::
	
	ALTER TABLE ways ADD COLUMN reverse_cost double precision;
	UPDATE ways SET reverse_cost = length;

Veremos un ejemplo de uso en los ejercicios.

La sintáxis utilizada para llamar al algoritmo puede parecer poco intuitiva en un primer momento. En general, calcular una ruta con pgRouting se hace siguiendo este esquema::

	select pgr_<algorithm>(<SQL para las aristas>, <nodo inicial>, <nodo final>, <opciones adicionales>)

El algoritmo a utilizar puede ser cualquiera de los listados `aquí <http://docs.pgrouting.org/2.0/en/src/index.html#routing-functions>`_ 

El primer parámetro es una cláusula *SELECT* destinada a obtener, de la tabla de aristas, aquellas involucradas en la ruta. Dicha consulta debe *extraer* de la tabla de aristas algunas de las siguientes columnas::
	
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
	SELECT id, source, target, cost, x1, y1, x2, y2, reverse_cost FROM edge_table


Si los campos de la tabla de aristas tienen diferentes nombres a los mostrados, se pueden renombrar mediante el uso de *as*::
	
	SELECT gid as id, src as source, target, cost FROM othertable;

.. seealso:: Para más información sobre los parámetros de la consulta *SELECT*, consultar `este enlace <http://docs.pgrouting.org/2.0/en/doc/src/tutorial/custom_query.html#custom-query>`_


Los campos de *<origen>* y *<destino>* son simplemente los identificadores de los nodos origen y destino de la ruta

.. note:: El algoritmo de Dijkstra no requiere que origen y destino tengan asociada información geográfica (un SRS). Veremos el algoritmo con más detenimiento en el apartado de `Algoritmos básicos de pgRouting`.

En cuanto a las opciones adicionales, son dos campos booleanos:
	* *directed*: Si le asignamos ``true`` significa que el grafo que representa nuestra red topológica es un `grafo dirigido <http://en.wikipedia.org/wiki/Directed_graph>`_
	* *has_rcost*: Si le asignamos ``true`` la columna ``reverse_cost`` del conjunto de filas obtenido como resultado será usado como coste para el camino opuesto de la arista en la que se encuentra.

El resultado de la consulta es, como ya se ha adelantado, un conjunto de filas. Cada fila es una tupla que incluye los siguientes campos::
	
	SELECT id, source, target, cost [,reverse_cost] FROM edge_table 

Un coste de -1 indica una arista que no se puede seguir. El campo ``reverse_cost`` solo tiene sentido si los parámetros ``directed`` y ``has_rcost`` son true. Veremos ejemplos del uso de estos parámetros en los ejercicios.


Algoritmo Dijkstra mejorado: múltiples destinos
-----------------------------------------------

Si queremos calcular el camino óptimo a varios destinos a la vez, podemos usar el algoritmo **kDijkstra**. Permite especificar varios destinos en la misma consulta. 

Por ejemplo, si queremos calcular el coste de los caminos entre el nodo 10 y los nodos 60, 70, 80::
	
	SELECT seq, id1 AS source, id2 AS target, cost FROM pgr_kdijkstraCost('
                SELECT gid AS id,
                         source::integer,
                         target::integer,
                         length::double precision AS cost
                        FROM ways',
                10, array[60,70,80], false, false);

Esa consulta nos calculará el camino más corto a los 3 nodos. 

Si estamos interesados en los caminos en si, podemos llamar a **pgr_kdijkstraPath**::
	
	SELECT seq, id1 AS path, id2 AS edge, cost FROM pgr_kdijkstraPath('
                SELECT gid AS id,
                         source::integer,
                         target::integer,
                         length::double precision AS cost
                        FROM ways',
                10, array[60,70,80], false, false);


Algoritmo A*
------------

El otro algoritmo más común para cálculo de rutas con pgRouting es A*. Este algoritmo añade información geográfica al origen y destino de cada arista de la red. De esta manera, el algoritmo seleccionará, en cada paso, el nodo que esté más cerca del destino. Usará para ello una heurística. El funcionamiento se entiende bien con esta gráfica (también obtenido del taller de María Arias de Reyna):
	
	.. image:: _images/Astar_progress_animation.gif

Como se puede observar, A* es mucho más rápido encontrando el camino óptimo:
	
	.. image:: _images/Dijksta_Anim.gif

	.. image:: _images/Astar_progress_animation.gif

Este funcionamiento requiere que creemos unos cuantos campos adicionales en nuestra tabla de aristas::
	
	ALTER TABLE ways ADD COLUMN x1 double precision;
	ALTER TABLE ways ADD COLUMN y1 double precision;
	ALTER TABLE ways ADD COLUMN x2 double precision;
	ALTER TABLE ways ADD COLUMN y2 double precision;

	UPDATE ways SET x1 = ST_x(ST_PointN(the_geom, 1));
	UPDATE ways SET y1 = ST_y(ST_PointN(the_geom, 1));

	UPDATE ways SET x2 = ST_x(ST_PointN(the_geom, ST_NumPoints(the_geom)));
	UPDATE ways SET y2 = ST_y(ST_PointN(the_geom, ST_NumPoints(the_geom)));

De esta forma, sabremos cuál es el vértice más cercano al destino en cada paso.


La consulta a ejecutar sería la misma que para Dijkstra, cambiando únicamente el nombre del algoritmo, y añadiendo los nuevos campos en la tabla de aristas::
	
	SELECT seq, id1 AS node, id2 AS edge, cost FROM pgr_astar('
                SELECT gid AS id,
                         source::integer,
                         target::integer,
                         length::double precision AS cost,
                         x1, y1, x2, y2
                        FROM ways',
                30, 60, false, false);

El mundo real
=============

¿Cómo trabaja pgRouting en un entorno no idealizado, con problemas reales? Veremos un poco más al respecto a continuación

Problemas que nos podemos encontrar en el mundo real
----------------------------------------------------

En un entorno real, no solo tendremos aristas y nodos. Nos podemos encontrar con dificultades como:
	* Diferentes tipos de vías, cada una con su velocidad
	* Diferentes tipos de vehículos
	* Tráfico
	* Señales y giros no permitidos
	* Obras, accidentes, etc

pgRouting nos permite tener en cuenta todos estos problemas
	
	.. image:: _images/pgrouting_basic.png

¿Semáforos y/o cruces?

	.. image:: _images/pgrouting_cruces.png


¿Calles de una sola dirección?

	.. image:: _images/pgrouting_one_dir.png


¿Diferentes significados del concepto *coste*?


	.. image:: _images/pgrouting_cuesta.png


¿Restricciones en los giros?

	Hasta la versión 2.0 de pgRouting, el algoritmo que tenía en cuenta esto, a través de una tabla auxiliar, era el algortimo *shooting star*. Desde la versión 2.0, se usa el algoritmo *TRSP (Turn Restriction Shortest Path)*, a través de la función ``pgr_trsp``.

En el algoritmo TRSP se especifican las restricciones en giros a través de una consulta SQL que restringe los caminos a tomar. Dicha consulta SQL debe devolver un resultado con varias filas que tengan el siguiente formato::
	
	SELECT to_cost, target_id, via_path FROM restrictions

Cada fila del resultado significa: "Si vienes a través del camino indicado por *via* (una lista de ids de aristas separada por comas), solo puedes pasar por la arista *target_id* pagando el coste *to_cost*".

Esta lógica se puede usar para implementar restricciones en los giros. Veamos un par de ejemplos, usando la red topológica construída con `estos datos <http://docs.pgrouting.org/2.0/en/doc/src/developer/sampledata.html#sampledata>`_.


Crearemos una tabla auxiliar para almacenar las restricciones en los giros::
	
	CREATE TABLE restrictions (
    	rid serial,
    	to_cost double precision,
    	target_id integer,
    	via_path text
	);

	INSERT INTO restrictions VALUES (1, 1000, 11, '8,4,1');

La restricción de giro significa: "Si vienes a través de las aristas 1, 4, 8, solo puedes pasar a través de la arista 11 pagando un costo de 1000".

Veamos ahora cómo sería el camino mínimo sin restricciones de giro, usando trsp. Queremos ir del nodo 1 al 11::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_trsp(
                'SELECT id, source, target, cost FROM edge_table',
                1, 11, false, false
        );

El resultado es::

	 seq | node | edge | cost
	-----+------+------+------
	   0 |    1 |    1 |    1
	   1 |    2 |    4 |    1
       2 |    5 |    8 |    1
       3 |    6 |   11 |    1
       4 |   11 |   -1 |    0

Es decir, que el camino encontrado es 1 - 2 - 5 - 6 - 11. 

Veamos lo que sucede si aplicamos la restricción de giro que impedirá ir del nodo 6 al 11 sin pagar un coste de 1000::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_trsp(
                'SELECT id, source, target, cost FROM edge_table',
                1, 11, false, false,
                'SELECT to_cost, target_id, via_path FROM restrictions'
        );


El resultado es::
	
	 seq | node | edge | cost
	-----+------+------+------
	   0 |    1 |    1 |    1
	   1 |    2 |    4 |    1
	   2 |    5 |   10 |    1
	   3 |   10 |   12 |    1
	   4 |   11 |   -1 |    0

Como se puede apreciar, el camino elegido ha sido 1 - 2 - 5 - 10 - 11. El algoritmo ha preferido ir del nodo 5 al 10, en previsión del sobrecoste que le iba a suponer el otro camino.
 

.. seealso:: La función pgr_trsp se puede consultar `aquí <http://docs.pgrouting.org/dev/src/trsp/doc/index.html>`_. Y las razones para abandonar *shooting_star*, `aquí <http://docs.pgrouting.org/dev/doc/src/developer/discontinued.html#shooting-star>`_. Pero el ejemplo de consulta no es muy afortunado.


.. note:: Las imágenes de este apartado se han obtenido de http://www.slideshare.net/kastl/foss4-g2011-pgrouting


Ejemplos: soluciones a problemas comunes
----------------------------------------

Es posible que surjan ciertas dudas a la hora de empezar a trabajar con pgRouting. La herramienta proporciona unos pocos algoritmos y funciones básicas, pero el trabajo *duro* aun es responsabilidad del usuario. El principio que sigue pgRouting es: aquí tienes estas herramientas para **detectar** errores. Arreglarlos, es responsabilidad del usuario.

Veamos algunas preguntas surgidas mientras se trabaja con pgRouting:


Problemas durante la creación de la topología
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Para crear una topología, la función ``pgr_createTopology`` espera una tabla de aristas (objetos ``LINESTRING``, en terminología de |PGIS|), como ya hemos mencionado. Es importante destacar que el concepto *arista* implica que **existen nodos al principio y al final de cada arista**. Si la estructura de nuestra tabla no es así, **la topología creada contendrá errores**. 

Para asegurarnos de que existen nodos al principio y final de cada arista, existen ciertas funciones que nos ayudan. Estas funciones son:

	* `pgr_analyzeGraph <http://docs.pgrouting.org/2.0/en/src/common/doc/functions/analyze_graph.html#pgr-analyze-graph>`_
	* `pgr_analyzeOneWay <http://docs.pgrouting.org/2.0/en/src/common/doc/functions/analyze_oneway.html#pgr-analyze-oneway>`_
	* `pgr_nodeNetwork <http://docs.pgrouting.org/2.0/en/src/common/doc/functions/analyze_oneway.html#pgr-analyze-oneway>`_

Veremos cómo nos ayudan. Supongamos que partimos únicamente de una tabla que contiene datos de tipo ``LINESTRING`` o ``MULTILINESTRING``. Entre esos elementos, es posible que existan intersecciones. Hasta que esas intersecciones no sean *registradas* como nodos, **no podremos considerar nuestra tabla como una tabla de aristas válida para crear una topología**. Necesitamos *segmentar* nuestras líneas, para que ``pgr_createTopology`` pueda crear un vértice en el punto donde se conectan.

Como ejemplo, vamos a crear una topología con errores, y ver cómo pgRouting puede ayudarnos a solucionarlos. Vamos a usar una nueva base de datos. Podemos crear una nueva a tal efecto, e instalar las extensiones necesarias con ``CREATE EXTENSION postgis`` y ``CREATE EXTENSION pgrouting``. Una vez hecho, ejecutamos las siguientes sentencias SQL::
	
	CREATE TABLE edge_table (
   		id serial,
    	dir character varying,
    	source integer,
    	target integer,
    	cost double precision,
    	reverse_cost double precision,
   		x1 double precision,
    	y1 double precision,
    	x2 double precision,
    	y2 double precision,
    	the_geom geometry
	);

	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  2,0,   2,1);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES (-1, 1,  2,1,   3,1);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES (-1, 1,  3,1,   4,1);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  2,1,   2,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1,-1,  3,1,   3,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  0,2,   1,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  1,2,   2,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  2,2,   3,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  3,2,   4,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  2,2,   2,3);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1,-1,  3,2,   3,3);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1,-1,  2,3,   3,3);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1,-1,  3,3,   4,3);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  2,3,   2,4);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  4,2,   4,3);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  4,1,   4,2);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  0.5,3.5,  1.999999999999,3.5);
	INSERT INTO edge_table (cost,reverse_cost,x1,y1,x2,y2) VALUES ( 1, 1,  3.5,2.3,  3.5,4);


	UPDATE edge_table SET the_geom = st_makeline(st_point(x1,y1),st_point(x2,y2)),
                      dir = CASE WHEN (cost>0 and reverse_cost>0) THEN 'B'   -- both ways
                                 WHEN (cost>0 and reverse_cost<0) THEN 'FT'  -- direction of the LINESSTRING
                                 WHEN (cost<0 and reverse_cost>0) THEN 'TF'  -- reverse direction of the LINESTRING
                                 ELSE '' END;                                -- unknown


De esa forma, hemos creado una tabla de aristas que contiene dos errores en las dos últimas:
	* La penúltima (arista 17), está aislada porque no termina en la coordenada 2, sino en 1.999999999999. Si no creamos la red con la tolerancia adecuada, ambos puntos serán considerados diferentes
	* La última arista está totalmente aislada. A pesar de que cruza la arista 13, no se encuentra segmentada, de manera que esa intersección no está contemplada.

En la imagen podemos ver el aspecto de nuestra red. Los nodos están representados por círculos azules. Fijémonos en las siguientes particularidades:
	* Los nodos 16 y 17 no son alcanzables (arista aislada) 
	* En función de la tolerancia usada para crear la red, el nodo 15 podrá ser alcanzado desde el 14 o no.
	* Hay algunas aristas de sentido único. Se marcan en la imagen. Las no marcadas, son de doble sentido.
	* El coste de todos los caminos permitidos es 1

	.. image:: _images/before_node_net1.png

Para poner de manifiesto los problemas existentes, creemos una topología usando ``pgr_createTopology``, y tratemos de usarla para crear rutas. Después, usemos las funciones de análisis::

	SELECT pgr_createTopology('edge_table', 0.001);

Tras la ejecución de ``pgr_createTopology``, vemos que la tabla *edge_table_vertices_pgr* ha sido creada, y los campos *source* y *target* de nuestra tabla de ariastas han sido rellenados. Ejecutemos algunas consultas de búsqueda de caminos mínimos:


Camino del nodo 14 al 13::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_astar(
                'SELECT id, source, target, cost, reverse_cost, x1, y1, x2, y2 FROM edge_table',
                14, 13, true, true
        );

Devuelve un error, porque no existe un camino entre ambos nodos. Recordemos que la arista que parte del nodo 14 no está realmente conectada con el nodo 15 (y por ende, con el resto de la red), por un problema de tolerancia

Camino del nodo 5 al 17::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_astar(
                'SELECT id, source, target, cost, reverse_cost, x1, y1, x2, y2 FROM edge_table',
                5, 17, true, true
        );

También nos devuelve un error, porque la arista que llega al nodo 17 está aislada de la red, a excepción del nodo 16. Hay una intersección que debería estar contemplada y no lo está.

¿Cómo nos daríamos cuenta de estos problemas? Mediante el uso de ``pgr_analyzeGraph``. Usemos la función **con el mismo valor de tolerancia que hemos usado al crear la topología**::
	
	SELECT pgr_analyzegraph('edge_table', 0.001);

Nos devolverá un informe como el siguiente::
	
	NOTICE:  PROCESSING:
	NOTICE:  pgr_analyzeGraph('edge_table',0.001,'the_geom','id','source','target','true')
	NOTICE:  Performing checks, pelase wait...
	NOTICE:  Analyzing for dead ends. Please wait...
	NOTICE:  Analyzing for gaps. Please wait...
	NOTICE:  Analyzing for isolated edges. Please wait...
	NOTICE:  Analyzing for ring geometries. Please wait...
	NOTICE:  Analyzing for intersections. Please wait...
	NOTICE:              ANALYSIS RESULTS FOR SELECTED EDGES:
	NOTICE:                    Isolated segments: 2
	NOTICE:                            Dead ends: 7
	NOTICE:  Potential gaps found near dead ends: 1
	NOTICE:               Intersections detected: 1
	NOTICE:                      Ring geometries: 0

	Total query runtime: 93 ms.
	1 row retrieved.

Vemos que tenemos dos segmentos aislados (los que unen los vértices 14 - 15 y 16 - 17), 7 vértices marcados como *dead end* (los vértices finales: 1, 7, 13, 14, 15, 16 y 17), una intersección detectada (entre las aristas 13 y 18) y un agujero potencial (el que hace que el vértice 15 no esté realmente conectado con el resto de la red) 

Para entender mejor estos mensajes, podemos echar un vistazo en la tabla de vértices, *edge_table_vertices_pgr*. Hay dos campos especialmente interesantes, **porque son actualizados cuando ejecutamos *pgr_analyzeGraph*** :
	* *cnt*: Para cada vértice, cuenta el número de aristas que referencian este vértice
	* *chk*: Especifica que puede haber un problema con este vértice

Por tanto, para detectar un segmento aislado del resto, habría que encontrar una arista cuyos dos extremos fueran *dead ends*. Lo podemos obtener con la siguiente consulta::
	
	SELECT a.id, a.source, a.target, st_astext(a.the_geom)
    FROM edge_table a, edge_table_vertices_pgr b, edge_table_vertices_pgr c
    WHERE a.source=b.id AND b.cnt=1 AND a.target=c.id AND c.cnt=1;
	
El resultado es el siguiente::
	
	 id | source | target |          st_astext
	----+--------+--------+--------------------------------------
 	 17 |     14 |     15 | LINESTRING(0.5 3.5,1.99999999999 3.5)
 	 18 |     16 |     17 | LINESTRING(3.5 2.3,3.5 4)

Tras ir a la red, vemos que, efectivamente, ha detectado los dos segmentos aislados. 


El conflicto que había con esta red es que no están identificados correctamente todos los nodos. Para ayudarnos a ello, existe la función ``pgr_nodeNetwork``, que realiza exactamente esa labor. Se encargará de detectar las intersecciones sin nodos y añadirlos. Veamos cómo invocarla::
	
	SELECT pgr_nodeNetwork('edge_table', 0.001);

La llamada a esta función generará una tabla llamada *edge_table_noded*, conteniendo los campos imprescindibles de la tabla *edge_table* (*id, source, target, the_geom*) y, además, **dos campos más**:
	
	* *old_id*: identificador de la arista original en la tabla *edge_table*
	* *sub_id*: número del segmento generado con respecto a la arista original

Si obtenemos esos campos::
	
	SELECT old_id,sub_id, st_astext(the_geom) FROM edge_table_noded ORDER BY old_id,sub_id;

Podremos ver que las aristas 13, 14 y 18 han sido segmentadas, y que ahora cuentan con dos segmentos cada una.

Hecho eso, podemos crear nuestra topología en la nueva tabla segmentada::
	
	SELECT pgr_createTopology('edge_table_noded', 0.001);

Si analizamos la nueva tabla, vemos que han desaparecido los errores topológicos::
	
	SELECT pgr_analyzegraph('edge_table_noded', 0.001);

	NOTICE:  PROCESSING:
	NOTICE:  pgr_analyzeGraph('edge_table_noded',0.001,'the_geom','id','source','target','true')
	NOTICE:  Performing checks, pelase wait...
	NOTICE:  Analyzing for dead ends. Please wait...
	NOTICE:  Analyzing for gaps. Please wait...
	NOTICE:  Analyzing for isolated edges. Please wait...
	NOTICE:  Analyzing for ring geometries. Please wait...
	NOTICE:  Analyzing for intersections. Please wait...
	NOTICE:              ANALYSIS RESULTS FOR SELECTED EDGES:
	NOTICE:                    Isolated segments: 0
	NOTICE:                            Dead ends: 6
	NOTICE:  Potential gaps found near dead ends: 0
	NOTICE:               Intersections detected: 0
	NOTICE:                      Ring geometries: 0

	Total query runtime: 93 ms.
	1 row retrieved.


Podemos ahora incluir los nuevos segmentos en nuestra tabla original. Al haberse creado nuevos vértices y aristas, **los ids de algunos vértices habrán cambiado**. Vamos a proceder a en dos pasos:

	* Añadimos una columna *old_id* en *edge_table* para mantener un registro de los ids de la tabla original
	* Insertar solo los segmentos nuevos. Es decir, aquellos en los cuales max(sub_id) > 1

::
	
	alter table edge_table drop column if exists old_id;
	alter table edge_table add column old_id integer;
	insert into edge_table (old_id,dir,cost,reverse_cost,the_geom)
        (with
        segmented as (select old_id,count(*) as i from edge_table_noded group by old_id)
        select  segments.old_id,dir,cost,reverse_cost,segments.the_geom
                from edge_table as edges join edge_table_noded as segments on (edges.id = segments.old_id)
                where edges.id in (select old_id from segmented where i>1) );

Necesitaremos actualizar también los campos x1, x2, y1, y2 de los nuevos segmentos añadidos, puesto que ``pgr_nodeNetwork`` no lo hace::

	UPDATE edge_table SET x1 = ST_x(ST_PointN(the_geom, 1));
	UPDATE edge_table SET y1 = ST_y(ST_PointN(the_geom, 1));

	UPDATE edge_table SET x2 = ST_x(ST_PointN(the_geom, ST_NumPoints(the_geom)));
	UPDATE edge_table SET y2 = ST_y(ST_PointN(the_geom, ST_NumPoints(the_geom)));


Posteriomente, recreamos la topología::
	
	SELECT pgr_createTopology('edge_table', 0.001);

Nuestra nueva topología queda como vemos en la imagen

	.. image:: _images/pgrouting_nueva_red.png


Ya estaríamos listos para ejecutar consultas otra vez. Las dos consultas que anteriormente dieron un error ahora obtienen el resultado correcto::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_astar(
                'SELECT id, source, target, cost, reverse_cost, x1, y1, x2, y2 FROM edge_table',
                12, 10, true, true
        );

Da como resultado::
	
	 seq | node | edge | cost
	-----+------+------+------
   	   0 |   12 |   17 |    1
   	   1 |   13 |   22 |    1
   	   2 |   10 |   -1 |    0


Y la consulta::

	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_astar(
                'SELECT id, source, target, cost, reverse_cost, x1, y1, x2, y2 FROM edge_table',
                5, 15, true, true
        );

Nos devuelve::
	
	 seq | node | edge | cost
	-----+------+------+------
   	   0 |    5 |    8 |    1
   	   1 |    6 |   11 |    1
  	   2 |    7 |   19 |    1
   	   3 |   18 |   24 |    1
   	   4 |   15 |   -1 |    0


Ambos resultados esperados.

De todas formas, aun tenemos dos cosas a tener en cuenta:

	* No sabemos los costes de los nuevos segmentos. Realmente, le hemos asignado a cada nuevo segmento el mismo coste que tenía el segmento original. Podemos modificarlo a mano y asignar el coste que deseemos.
	* Las aristas que han sido segmentadas, aun existen en nuestra topología.

La solución al segundo punto es algo que tenemos que pensar con cuidado. Por ejemplo, si queremos la ruta entre los vértices 7 y 9::
	
	SELECT seq, id1 AS node, id2 AS edge, cost
        FROM pgr_astar(
                'SELECT id, source, target, cost, reverse_cost, x1, y1, x2, y2 FROM edge_table',
                7, 9, true, true
        );

Vemos que se devuelve la ruta directa entre los nodos 7 y 9, sin pasar por el nuevo nodo creado, el 18::
	
	 seq | node | edge | cost
	-----+------+------+------
   	   0 |    7 |   13 |    1
   	   1 |    9 |   -1 |    0


Eso es porque pgrouting, como ya mencionamos, solo es un conjunto de *ladrillos* para construir un sistema de cálculo de rutas. No asume nada con respecto a los datos, salvo lo que nosotros le indiquemos a través de sentencias SQL. En este caso concreto, podrían darse dos situaciones:

	* Queremos simular el hecho de tener intersecciones a diferentes niveles. Por ejemplo, puede que el nodo 18 exista en la ruta pero la antigua conexión directa entre los nodos 7 y 9 siga existiendo como *by-pass*, exclusivamente para ir de 7 a 9.
	* Queremos eliminar las aristas que han sido segmentadas. Bastaría con que ejecutáramos una sentencia ``DELETE`` por cada una de las antiguas aristas. Al estar ya construida la nueva topología, existirán 2 segmentos donde antes había 1, y podemos eliminarlo con la seguridad de que no vamos a crear una desconexión. Si no queremos borrar aristas, también podemos asignarle a esa ruta un coste virtualmente infinito, o negativo. De esta forma, sería siempre ignorada.


Por último, dedicaremos unas palabras al uso de la función ``pgr_analyzeOneWay``. 

Esta función básicamente detecta nodos *sumidero* (entran N aristas, pero no sale ninguna) y nodos *fuente* (salen N aristas pero no entra ninguna). Nodos de este tipo no deberían existir en nuestra red, porque estarían aislados. 

Para detectarlos, ``pgr_analyzeOneWay`` exige que se haya creado una topología previamente. Es decir:
	* Que tengamos una tabla de aristas con los campos *source* y *target* correctamente rellenados
	* Que exista una tabla de vértices conectada con la tabla de aristas.

La función añade dos campos adicionales a la tabla de vértices: *ein* y *eout*. Son campos para almacenar el número de aristas que entran y salen de cada vértice.

Entre los parámetros de la función, vemos que hay cuatro vectores de elementos de tipo texto. Estos cuatro vectores representan las reglas definidas para las aristas. Estas reglas son algo realmente arbitrario, y es responsabilidad del usuario que sean coherentes con la realidad de la red topológica. 

Por *regla*, entendamos una codificación que nos dice si una arista es de sentido único (y en qué dirección) o de doble sentido. Esta codificación la especificamos como a nosotros nos venga en gana. La convención común es añadir una columna de tipo texto a nuestra tabla de aristas y codificar sentido y direccionalidad con identificadores cortos como:
	* 'B': Significa que la arista es bidireccional
	* 'FT': Significa que la arista es de sentido único, en dirección del nodo origen al nodo destino
	* 'TF': Significa que la arista es de sentido único, en dirección del nodo destino al nodo origen
	* '': No hay datos. En este caso, el algoritmo se comporta como si la arista fuera bidireccional
	* NULL: Campo nulo. Cómo debe reaccionar el algoritmo ante este tipo de campos es posible especificarlo mediante un parámetro booleano de la función. Si ese parámetro vale 'true', la arista es tratada como de doble sentido. No queda claro qué sucede si el valor de ese argumento es *false*. En cualquier caso, por defecto vale *true*.

El algoritmo espera que el campo de la tabla que contiene la codificación se llame *oneway*. En caso contrario, se especifica su nombre mediante otro de los parámetros. Lo mismo ocurre con los campos de la tabla de aristas que guardan origen y destino de cada una de las aristas. Si no se llaman *source* y *target*, se debe especificar su nombre por defecto.

Sabiendo esto, podríamos lanzar una consulta ``pgr_analyzeOneWay`` sobre nuestros datos de ejemplo::
	
	SELECT pgr_analyzeOneway('edge_table',
		ARRAY['', 'B', 'TF'],
		ARRAY['', 'B', 'FT'],
		ARRAY['', 'B', 'FT'],
		ARRAY['', 'B', 'TF'],
		oneway:='dir');

Hemos tenido en cuenta el nombre del campo que guarda el sentido y direccionalidad de cada arista (*dir*). 

Lanzado el análisis, podríamos ya obtener un listado de los posibles nodos aislados::
	
	SELECT * FROM edge_table_vertices_pgr WHERE ein=0 OR eout=0;

Y cuáles son las aristas conectadas con esos nodos::
	
	SELECT gid FROM edge_table a, edge_table_vertices_pgr b WHERE a.source=b.id AND ein=0 OR eout=0
	UNION
	SELECT gid FROM edge_table a, edge_table_vertices_pgr b WHERE a.target=b.id AND ein=0 OR eout=0;


Cálculo de rutas desde un punto aleatorio a otro
------------------------------------------------

Realmente, esto no es algo que se pueda hacer de manera sencilla. Para poder calcular una ruta entre dos puntos, han de existir vértices entre esos puntos. Añadir nuevos vértices de manera artificial requeriría reconstruir la topología. 

Desde el punto de vista matemático, yo no me puedo *inventar* nodos donde no los hay. Tal vez en ese punto haya un camino restringido que impide que pueda ser nodo de ninguna ruta. 

Si estamos razonablemente seguros de que empezar y terminar una ruta entre dos puntos aleatorios dados es *seguro*, sería posible completar la funcionalidad de los algoritmos de cálculo de rutas mediante el uso de ciertas funciones de |PGIS|, como `ST_LineLocatePoint <http://postgis.net/docs/ST_Line_Locate_Point.html>`_ o `ST_LineSubstring <http://postgis.net/docs/ST_Line_Substring.html>`_. Se podría plantear como *wrapper* sobre las funciones existentes de cálculo de rutas.



Cálculo de rutas con puntos intermedios
--------------------------------------- 

En ocasiones, querremos calcular la mejor ruta de un punto A hasta un punto B pasando a través de una lista de nodos (planteamiento clásico del problema del viajero). En pgrouting tenemos la función `pgr_trsp <http://docs.pgrouting.org/2.0/en/src/tsp/doc/index.html#pgr-tsp>`_ para ello. Tiene dos modos de funcionamiento, en función de qué le pasemos como argumentos:

	* Si le pasamos como primer argumento una consulta SQL que nos devuelva una lista de vértices con sus coordenadas X,Y, el algoritmo calcula la distancia euclídea entre esos puntos y la utiliza como base. Es poco preciso, pero muy rápido
	* Si le pasamos una matriz de distancias reales calculadas entre los nodos, la precisión aumenta, pero el rendimiento es peor.

Está demostrado que la precisión en el primer caso es ligeramente peor, y perfectamente asumible en el caso de que estemos calculando distancias entre ciudades, por ejemplo. Aun así, si queremos obtener más precisión, para obtener la matriz de distancias mínimas entre nodos, pgrouting nos propociona la función `pgr_makeDistanceMatrix <http://docs.pgrouting.org/2.0/en/src/tsp/doc/index.html#pgr-tsp>`_


Restricciones de giros
----------------------

.. todo:: Acabar esta sección. Revisar https://github.com/pgRouting/pgrouting/wiki/Turn-Restricted-Shortest-Path-(TRSP)
.. todo:: Aquí lo explica, pero bastante mal: http://docs.pgrouting.org/2.0/en/src/trsp/doc/index.html#trsp

Trabajando con datos |OSM|
==========================

Veremos a continuación una breve introducción al trabajo con datos de |OSM| en pgRouting.


Introducción
-------------

La heramienta pgRouting permite la carga de datos de |OSM|, y gracias a la herramienta ``osm2pgrouting``, podemos tener una topología completa basada en los datos de |OSM|, lista para realizar cálculos de rutas.

La información que obtenemos al descargar datos de |OSM|, no obstante, es más compleja de lo que necesitamos. Solo vamos a usar un subconjunto de todos los datos disponibles. Para ello, utilizamos un fichero de configuración que le pasamos a ``osm2pgrouting``. El fichero ha de estar en formato XML, y tiene este aspecto::

	<?xml version="1.0" encoding="UTF-8"?>
	<configuration>
  		<type name="highway" id="1">
    		<class name="motorway" id="101" />
    		<class name="motorway_link" id="102" />
    		<class name="motorway_junction" id="103" />
    		...
    		<class name="road" id="100" />
  		</type>    
  			<type name="junction" id="4">
    		<class name="roundabout" id="401" />
  		</type>  
	</configuration> 

Al instalar ``osm2pgrouting``, un fichero de configuración por defecto es instalado en */usr/share/osm2pgrouting/mapconfig.xml*.

La instrucción para cargar datos de |OSM| en pgRouting es como sigue::
	
	$ osm2pgrouting -file "data/sampledata.osm" \
                          -conf "/usr/share/osm2pgrouting/mapconfig.xml" \
                          -dbname pgrouting-workshop \
                          -user postgres \
                          -clean

Los parámetros vienen explicados `aquí <http://workshop.pgrouting.org/chapters/osm2pgrouting.html>`_ 

Tras la ejecución de la herramienta, veremos que se han creado bastantes más tablas en nuestra base de datos::

	List of relations
 	 Schema |        Name         |   Type   |  Owner
	--------+---------------------+----------+----------
	 public | classes             | table    | postgres
  	 public | geography_columns   | view     | postgres
 	 public | geometry_columns    | view     | postgres
 	 public | nodes               | table    | postgres
 	 public | raster_columns      | view     | postgres
 	 public | raster_overviews    | view     | postgres
 	 public | relation_ways       | table    | postgres
 	 public | relations           | table    | postgres
 	 public | spatial_ref_sys     | table    | postgres
 	 public | types               | table    | postgres
 	 public | vertices_tmp        | table    | postgres
 	 public | vertices_tmp_id_seq | sequence | postgres
 	 public | way_tag             | table    | postgres
 	 public | ways                | table    | postgres
	(14 rows)


Estas tablas adicionales darán más poder a pgRouting. Veremos en los siguientes apartados algunos ejemplos


.. seealso:: `Documentación sobre osm2pgrouting <http://pgrouting.org/docs/tools/osm2pgrouting.html>`_  


Manejando diferentes tipos de vías
---------------------------------------

.. warning:: En este apartado usaremos una base de datos similar a la que se puede crear importando datos de |OSM|, como hemos visto en el apartado anterior. En la máquina virtual, la base de datos ``pgrouting`` ya contiene todas las tablas necesarias. Si el alumno desea crear una base de datos nueva, puede usar el fichero ``sampledata_routing.sql``, como parte de los datos de ejemplo para pgRouting que se descargaron en el primer tema del curso.


Algo que sucede en el mundo real es que tenemos diferentes clases de carreteras, y el coste no es el mismo en todas. En la base de datos más compleja que utilizamos en este apartado, hay dos tablas que implementan esta complejidad: *types* y *classes*. Para implementar el hecho de que no todas las vías son iguales y el coste puede variar de una a otra, ejecutamos estas sentencias::
	
	UPDATE classes SET cost=1 ;
	UPDATE classes SET cost=2.0 WHERE name IN ('pedestrian','steps','footway');
	UPDATE classes SET cost=1.5 WHERE name IN ('cicleway','living_street','path');
	UPDATE classes SET cost=0.8 WHERE name IN ('secondary','tertiary');
	UPDATE classes SET cost=0.6 WHERE name IN ('primary','primary_link');
	UPDATE classes SET cost=0.4 WHERE name IN ('trunk','trunk_link');
	UPDATE classes SET cost=0.3 WHERE name IN ('motorway','motorway_junction','motorway_link');

Añadamos índices::
	
	CREATE INDEX ways_class_idx ON ways (class_id);
	CREATE INDEX classes_idx ON classes (id);

La idea de estas tablas es implementar un *factor de coste*, que multiplique al coste de cada enlace, que puede ser la longitud del mismo::
	
	SELECT seq, id1 AS node, id2 AS edge, cost FROM pgr_dijkstra('
                SELECT gid AS id,
                         source::integer,
                         target::integer,
                         length * c.cost AS cost
                        FROM ways, classes c
                        WHERE class_id = c.id',
                30, 60, false, false);



Otro problema del mundo real es encontrarnos caminos con acceso restringido. Esto lo podemos modelar estableciendo temporalmente un coste virtualmente infinito para esos caminos. Por ejemplo, podemos *cerrar* autopistas::
	
	UPDATE classes SET cost=100000 WHERE name LIKE 'motorway%

O podemos usar cláusulas web para evitar cierto tipo de caminos::
	
	SELECT seq, id1 AS node, id2 AS edge, cost FROM pgr_dijkstra('
                SELECT gid AS id,
                         source::integer,
                         target::integer,
                         length * c.cost AS cost
                        FROM ways, classes c
                        WHERE class_id = c.id AND class_id != 111',
                30, 60, false, false);



Lo importante de todas estas operaciones es que **no necesitamos reconstruir nuestra topología, solo variar ciertos parámetros en tiempo real**. El límite a lo que podemos hacer, solo nos lo impone |PGIS|.


.. seealso:: En el `wiki de pgrouting <https://github.com/pgRouting/pgrouting/wiki>`_ hay información acerca de algoritmos avanzados, como costes dependientes del tiempo o transporte multimodal


Cálculo de rutas con OSRM
=========================

Podemos acceder a una demo del producto en `este enlace <http://map.project-osrm.org/>`_

Podemos instalar nuestra propia versión::

	$ git clone git://github.com/DennisOSRM/Project-OSRM.git

Con eso insalaríamos el motor de cálculo de rutas. Si queremos una interfaz web::

	$ git clone git://github.com/DennisOSRM/Project-OSRM-Web.git




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

Calcular el camino más corto del nodo 4 al nodo 1, asumiendo que el grafo es no dirigido. Transformar el resultado en geometrías de tipo LineString, y crear con ellas una tabla para poder visualizarla en PostGIS.


Ejercicio 2
-----------

Repetir el ejercicio 1 con el algoritmo A*. Asegurarse de que la tabla de aristas cumple las condiciones necesarias para poder ejecutar el algoritmo.


Ejercicio 3
-----------

Repetir las dos búsquedas anteriores, pero asumiendo que el grafo es dirigido. Comparar los resultados.


Ejercicio 4
-----------

Usando los datos de |OSM|, realizar cálculos con Dijkstra, A*, Driving Distance y TRSP

.. note:: ¿Qué es driving distance? Revisar `http://docs.pgrouting.org/dev/src/driving_distance/doc/index.html`_



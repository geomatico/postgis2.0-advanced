.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


****************
.. note:: Los materiales teóricos de este capítulo están basados en parte en el `Taller de Routing <> impartido por María Arias de Reyna <http://delawen.github.io/Taller-Routing>`_ en las Jornadas de SIG Libre de Girona. En cuanto a parte de los materiales prácticos, han sido adaptados del libro `PostGIS CookBook <http://www.packtpub.com/postgis-to-store-organize-manipulate-analyze-spatial-data-cookbook/book>`_. En dicho libro se encuentran las soluciones a los ejercicios originales, así como otros ejercicios más complejos propuestos y resueltos. Recomendamos el uso de este libro como referencia para el aprendizaje de técnicas avanzadas con |PGIS|



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

A continuación, veremos cómo implementar el cálculo de rutas usando la extensión de |PGIS| pgRouting.

Lo básico
---------

Vamos a crear una tabla simple y activar los parámetros necesarios para hacer un cálculo de ruta usando el algoritmo básico (algoritmo de Dijkstra)::
	
	CREATE TABLE routing_simple
	(
  		gid serial NOT NULL,
  		length double precision,
  		the_geom geometry(LineString,4326),
  		CONSTRAINT routing_simple_pkey PRIMARY KEY (gid )
	)

Insertamos unos pocos datos simples::
	
	INSERT INTO routing_simple(gid, the_geom) values
		(1, st_setsrid(st_geomfromtext('LINESTRING(0 0, 1 1)'),4326)),
		(2, st_setsrid(st_geomfromtext('LINESTRING(1 1, 1 2)'),4326)),
		(3, st_setsrid(st_geomfromtext('LINESTRING(1 1, 2 2)'),4326)),
		(4, st_setsrid(st_geomfromtext('LINESTRING(1 2, 2 2)'),4326));



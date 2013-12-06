.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


Consejos de optimización
*************************

Para enteder los parámetros de optimización de |PGIS|, recomendamos dos libros básicos:
	* High Performance PostgreSQL 9
	* PostGIS in Action

El primero de ellos contiene todo lo necesario para optimizar una base de datos basada en |PGSQL| 9. Su autor es uno de los desarrolladores de |PGSQL|, y cubre desde un nivel básico hasta avanzado.

El segundo se puede considerar como el *A, B, C* en lo que se refiere a |PGIS|. La edición actual cubre |PGIS| 1.5.2. A lo largo de 2014 se pondrá a la venta la siguiente edición, que también cubrirá |PGIS| 2.0. 

Ambos libros se encuentran referenciados en el capítulo de Bibliografía.

En cualquier caso, listamos a continuación una serie de consejos prácticos para optimizar una base de datos |PGIS|. Estos consejos, y otros adicionales, pueden ser consultados en la documentación oficial de |PGIS| o en el libro *PostGIS in Action*

Optimizando acceso a tablas pequeñas con geometrías grandes
===========================================================

Las tablas pequeñas con geometrías grandes tienen un problema derivado de la arquitectura de |PGSQL|. En |PGSQL|, existe una técnica especial de almacenamiento llamada *TOAST*. Consiste en almacenar aparte los datos considerados demasiado grandes para caber en las zonas normales de datos. Es el caso, por ejemplo, de geometrías complejas, con muchos elementos. Si tenemos una tabla de pocas filas pero con geometrías de este tipo, la tabla en sí será guardada en una página de datos normal, pero las geometrías serán guardadas en el espacio *TOAST*. Si ahora lanzamos una consulta sobre esa tabla que requiera de una operación espacial sobre todas las columnas, el planificador determinará que el número de filas de la tabla es tan reducido, que es preferible una búsqueda secuencial que una consulta al índice. El problema, por tanto, es que tendremos que leer nuestras geometrías completas de disco, accediendo a la zona *TOAST*. Ante ese problema, se han planteado dos posibles soluciones:

	* Obligar al planificador a acudir al índice, mediante la instrucción ``SET enable_seqscan TO off``. El problema de esta aproximación es que actúa a nivel de conexión, y podría estar limitando el acceso secuencial en otras consultas realizadas en la misma conexión que sí se beneficiarían del mismo.
	* Cachear el BBOX de todas las geometrías en una columna adicional, de manera que el planificador pueda obtener este dato sin necesidad de ir a la zona *TOAST*, mediante la búsqueda sencuencial. Esto funciona en el caso de consultas que hagan uso del BBOX. Veamos un ejemplo.

Si nuestra consulta original era::
	
	SELECT geom_column 
	FROM mytable 
	WHERE geom_column && ST_SetSRID('BOX3D(0 0,1 1)'::box3d,4326);

Añadimos la nueva columna que almacene el BBOX::
	
	SELECT AddGeometryColumn('myschema','mytable','bbox','4326','GEOMETRY','2'); 
	UPDATE mytable SET bbox = ST_Envelope(ST_Force_2d(the_geom));

Y cambiamos la consulta para que use esta nueva columna (el operador ``&&``, de todas formas, actúa sobre el BBOX)::
	
	SELECT geom_column 
	FROM mytable 
	WHERE bbox && ST_SetSRID('BOX3D(0 0,1 1)'::box3d,4326);

Por supuesto, esto implica mantener esta columna sincronizada con la columna geométrica. Se podría hacer con triggers.


Agrupar columnas siguiendo el mismo criterio que el índice creado sobre las mismas
==================================================================================

Si nuestras tablas son básicamente de lectura y existe un índice creado sobre la columna geométrica, podemos usar la `instrucción CLUSTER de PostgreSQL <http://www.postgresql.org/docs/9.1/static/sql-cluster.html>`_ para ordenar físicamente nuestras filas en disco en el mismo orden que se encuentren en el índice. Podemos encontrarnos con problemas si alguna de nuestras columnas geométricas tiene el valor NULL, porque |PGSQL| no permite esta particularidad a la hora de aplicar la instrucción *CLUSTER* sobre índices GiST. Podemos solucionarlo añadiendo una restricción a nuestra tabla::
	
	ALTER TABLE my_table ALTER COLUMN the_geom SET not null; 


Evitar conversiones de coordenadas
==================================

Si tenemos geometrías en 3D pero siempre accedemos a las mismas a través de funciones como ``ST_AsText`` o ``ST_AsBinary``, estas funciones siempre nos van a devolver geometrías en 2D. Para ello, hacen una conversión a 2D mediante el uso de ``ST_Force_2D``. Podemos ahorrar este paso si actualizamos directamente nuestras geometrías a 2D::
	
	UPDATE mytable SET the_geom = ST_Force_2d(the_geom); 


Arreglar o simplificar geometrías
=================================

Podemos asegurarnos de no tener geometrías inválidas mediante el uso de la función ``ST_MakeValid``, de |PGIS| 2.0. Alternativamente, podemos usar alguna técnica de simplificación de geometrías, como el uso de las funciones ``ST_Simplify`` o ``ST_SimplifyTopology``.

.. warning:: Evitar la simplificación de geometrías basadas en datum WGS84. Las técnicas de simplificación trabajan sobre geometría plana. Si las aplicamos sobre geometrías que manejan geometría sobre un esferoide, se producirán resultados inesperados. Es aconsejable, si es el caso de nuestras geometrías, transformarlas primero a una proyección que respete las distancias.


Optimizar nuestras funciones
============================

Desde |PGSQL| 8.3, podemos usar los parámetros *row* y *cost*, que son aplicables a funciones, como parte de su definición. 

Con *cost* hacemos una estimación de cómo creemos que será de costosa nuestra función. Es útil en relación al coste de otras funciones. El planificador ejecutará las funciones que usemos en una consulta en orden de coste, en la medida de lo posible. Es recomendable poner un coste alto en funciones internas, destinadas a ser llamadas por otras funcines, y no por el usuario. Antes de |PGIS| 1.5, este parámetro no se usaba.

Con *row* hacemos una estimación de cuántas filas creemos que va a devolver nuestra función. Es útil solamente en funciones que devuelven filas. También se le pasa esta estimación al planificador.

También podemos jugar con los parámetros *IMMUTABLE*, *STABLE* y *VOLATILE*, a la hora de definir las funciones. Si no especificamos nada, nuestras funciones serán *VOLATILE*.

Una función *IMMUTABLE* es aquella cuyo resultado es constante en el tiempo siempre que no se cambien los parámetros de entrada. En este tipo de funciones, el planificador sabe que va a poder cachear el resultado. 

Un función *STABLE* es aquella que mantiene el resultado con los mismos parámetros solo durante el ciclo de vida de la consulta. No se puede cachear su resultado, porque es posible que, la siguiente vez que sea llamada con los mismos parámetros, alguna de las tablas a las que hace referencia haya cambiado de manera que la función obtenga resultados diferentes.

Por último, una función es *VOLATILE* si su resultado puede variar en cualquier momento aunque no cambien los parámetros de entrada. Un ejemplo sería la función *random()*. Si marcáramos esa función como *IMMUTABLE*, por ejemplo, iría más rápido, pero siempre devolvería lo mismo.


Uso de parámetros de configuración
==================================

Algunos parámetros de configuración de |PGSQL| pueden sernos útiles trabajando con |PGIS|. Estos parámetros pueden ser modificados en el fichero *postgresql.conf*

	* *constraint_exclusion*: Por defecto vale *partition*, y es como mejor funcionará. Esto implica que |PGSQL| solo comprobará si sobran tablas a la hora de hacer consultas en aquellas que usen tablas con herencia. Para versiones de |PGSQL| inferiores a 8.4, esto debería estar a *on*.

	* *shared_buffers*: Cantidad de memoria que |PGSQL| usa para memoria compartida. Se recomienda ponerlo a 1/3 ó 3/4 de la RAM.

	* *work_mem*: Memoria extra usada para consultas complejas. Por defecto se ajusta a 1 MB. Para consultas complejas, puede ajustarse a más del tamaño de la RAM. Si las tablas son pequeñas o las consultas simples, puede bajarse.


.. seealso:: `Presentación de Kevin Neufeld <http://2007.foss4g.org/presentations/view.php?abstract_id=117>`_ en el congreso FOSS4G 2007, sobre trucos de optimización para PostGIS



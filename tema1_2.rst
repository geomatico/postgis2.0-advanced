.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


Buenas prácticas trabajando con PostGIS
**************************************************

En este capítulo veremos algunas buenas prácticas a la hora de trabajar con |PGIS|. Seguiremos los siguientes pasos:
    * Introducción
	* Herencia de tablas
	* La importancia del SRID
	* Indexación espacial
	* Rules y triggers

Finalmente, veremos una serie de ejercicios prácticos, que servirán para fijar los conocimientos adquiridos.


Introducción
============

A la hora de diseñar una base de datos, deberemos enfrentarnos a conceptos como qué elementos queremos modelar, qué tipo de consultas van a hacerse y cuáles deberían hacerse a mayor velocidad, qué tipo de análisis va a soportar, etc. Si nuestra base de datos ha de tener soporte espacial, hay otra serie de preguntas que deberemos hacernos: qué tipo de datos espaciales vamos a almacenar (puntos, líneas, polígonos, datos raster...), con qué precisión debemos almacenarlos, qué herramientas GIS se van a usar para interactuar con la base de datos, etc.

Un mal diseño de una base de datos espacial puede llevar a tener una herramienta inutilizable, puesto que los elementos que se están almacenando no son meros números o textos. Ni siquiera objetos binarios *bobos*. Son elementos muy complejos. En el caso de |PGIS|, estamos hablando de elementos que siguen el estándar `SQL/MM <http://en.wikipedia.org/wiki/Simple_Features>`_.

Es por eso que es conveniente seguir una serie de buenas prácticas a la hora de diseñar una base de datos espacial con |PGIS|. En este curso, vamos a seguir los consejos del libro `PostGIS in Action <http://www.manning.com/obe2/>`_, que consideramos como una de las *Biblias* acerca de |PGIS|.


Herencia de tablas
==================

En el mencionado libro *PostGIS in Action* se plantean tres conceptos de diseño que consideramos especialmente interesantes a la hora de diseñar columnas geográficas en una base de datos espacial. Estos tres conceptos son **heterogeneidad**, **homogeneidad** y **herencia**. Vamos a estudiar el tercero de ellos, la herencia, porque engloba los dos anteriores

La posibilidad de usar herencia con tablas es, hasta el momento, una cualidad única de |PGSQL|. Es el mismo concepto de la orientación a objetos, pero traído a las bases de datos. La idea es que una tabla de |PGSQL| puede heredar su estructura de otra tabla *padre*. Se heredan los campos y las restricciones, pero no las claves primarias ni foráneas. 

Las tablas hijas pueden añadir campos adicionales a los definidos en la tabla padre. Estos campos solo se utilizarán cuando consultemos explícitamente la tabla hija. De hecho, la tabla padre puede no almacenar datos en absoluto, y que todos sean almacenado en las tablas hijas. En este sentido, la tabla padre se comportaría como una *clase abstracta*, en terminología de orientación a objetos. Hay que destacar también que |PGSQL| permite la herencia múltiple, sin límite en cuanto al número de tablas de las que puede heredar una dada. 

Por último, mencionar que |PGSQL| permite *desheredar* una tabla, y volver a declarar la herencia posteriormente. Esto es útil en casos en los que estamos cargando datos en una tabla hija y queremos evitar que consultas hechas contra la tabla padre puedan *caer* en la tabla hija que está siendo actualizada. 

El mecanismo de poder añadir y quitar la herencia implica que, realmente, una tabla puede *adoptar* como hija a cualquier otra en cualquier momento. 

.. note:: Es interesante conocer el concepto de *constraint exclusion*. Una opción de configuración de |PGSQL| que suele usarse junto con la herencia. Mediante esta opción, le decimos a |PGSQL| si queremos que compruebe si es posible evitar alguna de las tablas presentes en una consulta que implique herencia o union. El parámetro puede tener 3 posibles valores: *On*, *Off* y *Partition* (éste último introducido en |PGSQL| 8.4). Un valor de *on* hace que |PGSQL| verifique siempre si se puede evitar alguna tabla en una consulta, incluso aunque no implique herencia o union. *off* elimina la comprobación. En cuanto a *Partition*, hace la comprobación solo si la consulta implica herencia o union. 


Veamos un ejemplo de herencia de tablas, extraído del libro *PostGIS in Action*. El ejemplo pretende modelar una ciudad como París, usando para ello la herencia de tablas. Nosotros cambiaremos la ciudad por Sevilla. El diseño está pensado para almacenar los datos presentes en |OSM| simplificados, y permitir cruzar datos con nuestras propias tablas.

En primer lugar, creamos una tabla para almacenar todas las geometrías de |OSM|, independientemente de su tipo::
	
	CREATE TABLE sevilla_all_osm_geometries(
	gid serial NOT NULL,
        osm_id integer, 
	geom geometry,
        cod_post character varying(5), 
        tags hstore,

        CONSTRAINT sevilla_all_osm_geometries_pk PRIMARY KEY (gid),
        CONSTRAINT enforce_dims_geom CHECK (st_ndims(geom) = 2),
        CONSTRAINT enforce_srid_geom CHECK (st_srid(geom) = 4326)
	);

Y la rellenamos, sin incluir todos los campos de |OSM|. Usamos la tabla *codigo_postal* para incluir también el código postal de cada zona::
	
	INSERT INTO sevilla_all_osm_geometries(osm_id, geom, cod_post, tags)
	SELECT o.osm_id, ST_Intersection(o.geom, a.geom) As geom, a.cod_postal, o.tags
	FROM
		(SELECT osm_id, way As geom, tags FROM planet_osm_line) AS O 
		INNER JOIN 
		(select cod_postal, st_transform(geom, 4326) as geom FROM codigo_postal) AS A 
		ON (ST_Intersects(o.geom, a.geom));


Ahora vamos a crear una clase padre::
	
	CREATE TABLE sevilla_osm(gid SERIAL PRIMARY KEY, osm_id integer, cod_post character varying(5), 
	feature_name varchar(200), feature_type varchar(50), geom geometry);

	ALTER TABLE sevilla_osm ADD CONSTRAINT enforce_dims_geom CHECK (st_ndims(geom) = 2);
	ALTER TABLE sevilla_osm ADD CONSTRAINT enforce_srid_geom CHECK (st_srid(geom) = 4326);

Creamos una clave primaria en la clase padre a pesar de que ésta no vaya a almacenar ningún dato. Lo hacemos porque se considera buena práctica añadida claves primarias a todas las tablas. Algunos programas de terceros, de hecho, lo van a requerir.

Ahora creamos la clase hija que almacenará **solo las líneas**::
	
	CREATE TABLE sevilla_osm_lines(tags hstore,
    	CONSTRAINT sevilla_osm_lines_pk PRIMARY KEY (gid)) INHERITS (sevilla_osm);

Se deja como ejercicio para el alumno rellenar esta clase hija con los datos necesarios de la tabla *sevilla_all_osm_geometries*



La importancia del SRID
=======================

Qué es un SRS
-------------

A la hora de trabajar con datos geográficos, hemos de entender que estos carecen de sentido sin saber qué parte del mundo representan y cómo la representan. En otras palabras, tenemos que conocer el **sistema de referencia espacial** (SRS, por sus siglas en inglés) de nuestros datos para saber dónde está realmente cada punto representado. Es un concepto de vital importancia, porque normalmente, el usuario de un software GIS quiere poder obtener datos de diversas fuentes y superponerlos de manera que estos coincidan. Esto solo puede suceder si todos los datos de todas las fuentes utilizan el mismo sistema de referencia espacial.

En la actualidad, la manera más sencilla de especificar y obtener el SRS de unos datos es a través de los índices EPSG. Estos índices permiten expresar la complejidad subyacente en un SRS con un simple número que indexa una tabla. Si bien es cierto que no todas las fuentes de datos han de tener forzosamente un sistema de referencia espacial indexado por EPSG (sobre todo si son antiguos). Por eso hemos de entender cuáles son los componente fundamentales de un SRS:

	* Un **elipsoide de referencia**: Un elemento matemático construído alrededor del geoide que es realmente la Tierra. Es lo que el SRS tomará como representación del planeta, de manera que ha de cumplir una serie de restricciones para poder ser considerado *suficientemente bueno* como representación. Esto significa desviarse lo menos posible de la forma real del geoide. En la práctica, lo que sucede es que el elipsoide suele ser una representación bastante buena del geoide que recubre **solo en una zona**, perdiendo precisión en el resto del mundo. Es por eso que se suelen definir diferentes tipos de elipsoides, por países o continentes. Aunque actualmente, el elipsoide de referencia más utilizado es el WGS84. Es el que usa el sistema GPS.
	* Un **datum**: Una manera de *enganchar* el elipsoide a una zona concreta de la Tierra. Estrictamente, es un conjunto de valores que definen dónde va realmente cada punto del elipsoide. En España, se suelen usar el datum ED50 y, más actualmente, el ETRS89.
	* Un **sistema de coordenadas**: Sirve para identificar los puntos en nuestro elipsoide de referencia. El más conocido es el sistema de coordenadas geográficas (longitud, latitud).

Por verlo de una manera burda, podríamos construir un sistema de referencia espacial en 3 pasos:
	* Eligiendo un elipsoide de referencia, para modelar la forma de la Tierra.
	* Eligiendo un datum, para saber cómo colocar ese esferoide sobre el geoide.
	* Eligiendo un sistema de coordenadas, para saber cómo ubicar los puntos sobre el esferoide. Por ejemplo: cojamos ambos polos de nuestro elipsoide, y dibujemos rayas verticales que vayan de uno a otro: tendríamos meridianos. Ahora encontremos el ecuador y dibujemos círculos horizontales que vayan hacia los polos. Ya tendríamos los paralelos.


Con estos tres elementos, ya tendríamos suficiente para ubicar cualquier punto sobre la Tierra. Pero todavía no podemos representar nuestros datos en un plano, y disfrutar así de la mayor sencillez de la geometría Euclídea: el área de un cuadrado es su lado al cuadrado y las distancias pueden medirse con el Teorema de Pitágoras. Además, la mayor parte de funciones de PostGIS trabajan sobre un plano cartesiano (a excepción del tipo de datos *geography*).

Para transformar una esfera en un plano, y poder trabajar con geometría Euclídea, usamos las *proyecciones*: conjunto de reglas matemáticas para representar un objeto tridimensional en dos dimensiones. Las proyecciones tienen que lidiar con cuatro características fundamentales de los SRS: medidas, formas, direcciones y áreas. Las que son especialmente buenas conservando una o varias de estas cualidades, suelen fallar en el resto. Hay diferentes tipos de proyecciones en función de diversas características que no vamos a analizar, por quedar fuera del enfoque de este curso. Basta con saber que, entre las más utilizadas, están:

	* Proyección Mercator: Mantienen formas y direcciones, pero son malas para medidas y áreas (la distorsión cerca de los polos es muy grande). Los mapas web popularizaron una variante de esta proyección, denominada "Google Mercator" o "Web Mercator". Actualmente, ya tiene su propio identificador EPSG (3785), aunque aun es posible encontrarla con su anterior id, 900913 (*Google* escrito con números).
	* Proyección Mercator Transversa (UTM): Mantienen medidas, direcciones y formas, pero cubren áreas relativamente pequeñas, de 6 grados de longitud. Hacen falta 60 para cubrir todo el planeta.
	* Grids nacionales: Suelen ser variantes de UTM adaptadas para cubrir una región o país concreto. Suelen ser razonablemente buenas con las medidas y cubren todo el área necesitada, pero pueden fallar manteniendo las formas.

Como ejercicio, vamos a ver la diferencia que hay entre dos sistemas de referencia proyectados (23030 y 25830, usados en España) y uno no proyectado. Basta con entrar en las siguientes urls y elegir *Human Readable OGC-WKT*. Comentar las diferencias.

	* EPSG:23030 (proyectado): `http://spatialreference.org/ref/epsg/23030/ <http://spatialreference.org/ref/epsg/23030/>`_
	* EPSG:25830 (proyectado): `http://spatialreference.org/ref/epsg/25830/ <http://spatialreference.org/ref/epsg/25830/>`_
	* EPSG:4326 (no proyectado): `http://spatialreference.org/ref/epsg/4326/ <http://spatialreference.org/ref/epsg/4326/>`_

En |PGIS| existe una tabla que guarda los SRS. Se llama `spatial_ref_sys`. Comentar su contenido.


Qué SRS elijo para mis datos
----------------------------

A la hora de almacenar datos en |PGIS|, hay una enorme variedad de SRS, y es complicado encontrar uno que se ajuste a todas las necesidades. La respuesta corta es "depende de lo que quieras representar y lo que te interese conservar". Como consejos genéricos, podría decirse que:
	* Si queremos representar un único país o un estado de un país grande: suele ser buena idea elegir un SRS que use un grid nacional.
	* Si queremos representar un área grande, o incluso el mundo entero: en función de si nos interesa mucho poder realizar mediciones y representar los datos en un mapa o no, podríamos:
		* Utilizar Mercator: si solo queremos representar datos, es lo ideal
		* Utilizar WGS84: cubre el mundo entero, y lo utilizan sistemas de navegación (GPS). A cambio, es malo con medidas y formas (problemas para representarlo en un mapa directamente)
		* Utilizar UTM: podremos tomar medidas y se mantienen las formas, pero si queremos cubrir el mundo entero, deberemos manejar cerca de 60 SRS diferentes.
		* Utilizar el tipo de datos geography: podemos almacenar nuestros datos en WGS84 y, al mismo tiempo, tomar medidas de manera casi tan precisa como UTM (salvo para zonas muy pequeñas). Como problemas, destacar que la cantidad de funciones disponibles aun no es tan grande como para los datos de tipo *geometry* y los cálculos son más lentos.

Como resumen, si lo que queremos es almacenar datos del mundo entero y, aun así, mantener las medidas y formas lo suficientemente decentes como para mostrar un mapa y poder tomar medidas sobre él, lo más razonable parece utilizar UTM. En términos de almacenamiento en base de datos, hay diferentes opciones que podríamos elegir:
	* Almacenar todos nuestros datos en EPSG:4326 y realizar transformaciones *on-the-fly* al SRS destino cuando sea necesario
	* Lo anterior, pero actualizando vistas en lugar de transformaciones *on-the-fly*
	* Mantener una tabla por cada región UTM y usar herencia.

Hay diferentes filosofías al respecto, y realmente no se puede considerar ninguna como buena o mala. La experiencia y los casos de uso son los que nos deben guiar a la hora de elegir un SRS para nuestros datos.



Indexación Espacial
===================

Introducción
------------

La indexación espacial es una de las funcionalidades importantes de las bases de datos espaciales. Los indices consiguen que las búsquedas espaciales en un gran número de datos sean eficientes. Sin idenxación, la búsqueda se realizaria de manera secuencial teniendo que buscar en todos los registros de la base de datos. La indexación organiza los datos en una estructura de arbol que es recorrida rapidamente en la búsqueda de un registro.

Como funcionan los índices espaciales
-------------------------------------

Las base de datos estándar crean un arbol jerarquico basados en los valores de las columnas. Los indice espaciales funcionan de una manera diferente, los índices no son capaces de indexar las geometrías, e indexarán las cajas (box) de las geometrías.

	.. image:: _images/boundingbox.png
	
La caja (box) es el rectángulo definido por las máximas y mínimas coordenadas x e y de una geometría.		

	.. image:: _images/bbox.png

En la figura se puede observar que solo la linea intersecta a la estrella amarilla, mientras que si utilizamos los índices comprobaremos que la caja amarilla es intersectada por dos figuras la caja roja y la azul. El camino eficiente para responder correctamente a la pregunta **¿qué elemento intersecta la estrella amarilla?** es primero responder a la pregunta **¿qué cajas intersectan la caja amarilla?** usando el índice (consulta rápida) y luego calcular exactamente **¿quien intersecta a la estrella amarilla?** sobre el resultado de la consulta de las cajas.

Creación de indices espaciales
------------------------------

La síntaxis será la siguiente::

	CREATE INDEX [Nombre_del_indice] ON [Nombre_de_tabla] USING GIST ([campo_de_geometria]);
	
Esta operación puede requerir bastante tiempo en tablas de gran tamaño. 
	
Uso de índices espaciales
-------------------------

La mayor parte de las funciones en |PGIS| (ST_Contains, ST_Intersects, ST_DWithin, etc) incluyen un filtrado por indice automáticamente.

Para hacer que una función utilice el índice, hay que hacer uso del operador **&&**. Para las geometrías, el operador **&&** significa "la caja que toca (touch) o superpone (overlap)" de la misma manera que para un número el operador **=** significa "valores iguales"

ANALYZE y VACUUM 
----------------
El planificador de |PGSQL| se encarga de mantener estadísticas sobre la distribución de los datos de cada columna de la tabla indexada. Por defecto |PGSQL| ejecuta la estadísticas regularmente. Si hay algún cambio grande en la estructura de las tablas, es recomendable ejecutar un ``ANALYZE`` manualmente para actualizar estas estadísticas. Este comando obliga a |PGSQL| a recorrer los datos de las tablas con columnas indexadas y actualizar sus estadísticas internas.

No solo con crear el índice y actualizar las estadísticas obtendremos una manera eficiente de manejar nuestras tablas. La operación  ``VACUUM`` debe ser realizada siempre que un indice sea creado o después de un gran número de UPDATEs, INSERTs o DELETEs. El comando ``VACUUM`` obliga a |PGSQL| a utilizar el espacio no usado en la tabla que dejan las actualizaciones y los borrados de elementos.

Hacer un ``VACUUM`` es crítico para la eficiencia de la base de datos. |PGSQL| dispone de la opción ``Autovacuum``. De esta manera |PGSQL| realizará VACUUMs y ANALYZEs de manera periodica en función de la actividad que haya en la tabla:: 

	VACUUM ANALYZE [Nombre_tabla]
	VACUUM ANALYZE [Nombre_tabla] ([Nombre_columna])
	
Esta orden actualiza las estadísticas y elimina los datos borrados que se encontraban marcados como eliminados.


Rules y Triggers
================

|PGSQL| posee mecanismos para manejar el procesamiento condicional cuando se encuentra con los cuatro comandos básicos de SQL: ``INSERT``, ``UPDATE``, ``SELECT`` Y ``DELETE``. Estos mecanismos son las reglas de re-escritura (*rules*) y los disparadores (*triggers*). Vamos a verlos.

Rules
-----

Una *rule* no es más que una serie de directrices para transformar automáticamente una sentencia SQL en otra. El ejemplo clásico de aplicación de las *rules* son las vistas, o *views*. Una vista no es más que una o varias reglas de re-escritura unidas. Por ejemplo, cuando escribimos::
	
	CREATE VIEW mi_vista AS SELECT * FROM mi_tabla

Si luego queremos extraer información de la vista de esta forma::
	
	SELECT * FROM mi_vista

Esto es automáticamente re-escrito como::

	SELECT * FROM (SELECT * FROM mi_vista) AS mi_vista

Por supuesto, con una vista podemos hacer operaciones más complejas que un simple ``SELECT``. Podemos escribir vistas que manipulen datos.


Triggers
--------

Los *triggers* son procesmientos SQL destinados a ejecutarse antes, después o en lugar de una sentencia ``INSERT``, ``UPDATE`` o ``DELETE``. También pueden cancelar la ejecución de cualquiera de esas tres sentencias si no se cumplen una serie de condiciones.

Los *triggers* pueden ejecutarse una vez por cada fila que participa en una sentencia o una vez por cada sentenceia. Los últimos son normalmente usados para labores de logging.

Vamos a ver a continuación un ejemplo sencillo de utilización de vistas. La utilización de triggers se contemplará como ejercicio adicional.


Ejemplo
-------

Supongamos que tenemos una tabla que contiene datos numéricos, como coordenadas de longitud, latitud. Queremos poder representar esa tabla en un mapa, de manera que crearemos una vista que, por cada elemento, genere un objeto geométrico de tipo ``POINT``

Procedemos primero a crear nuestra tabla de ejemplo, y llenarla::
	
	DROP TABLE IF EXISTS lonlat_test CASCADE; 
	CREATE TABLE lonlat_test
	(
		lon numeric,
		lat numeric
	) WITH (OIDS=FALSE);

	ALTER TABLE lonlat_test ADD COLUMN gid serial;
	ALTER TABLE lonlat_test ADD PRIMARY KEY (gid);

	INSERT INTO lonlat_test (lon, lat) VALUES (random() * 360 - 180, random() * 180 - 90);
	INSERT INTO lonlat_test (lon, lat) VALUES (random() * 360 - 180, random() * 180 - 90);
	INSERT INTO lonlat_test (lon, lat) VALUES (random() * 360 - 180, random() * 180 - 90);
	INSERT INTO lonlat_test (lon, lat) VALUES (random() * 360 - 180, random() * 180 - 90);
	INSERT INTO lonlat_test (lon, lat) VALUES (random() * 360 - 180, random() * 180 - 90);
	INSERT INTO lonlat_test (lon, lat) VALUES (random() * 360 - 180, random() * 180 - 90);

Ahora, creamos una vista para generar los puntos a partir de estos valores::
	
	DROP VIEW IF EXISTS lonlat_test_points;
	CREATE VIEW lonlat_test_points AS
		SELECT lon, lat, ST_SetSRID(ST_MakePoint(lon,lat), 4326) as point FROM lonlat_test;

Ya tenemos una vista creada que contiene nuestros puntos como elementos geométricos.


Ejercicios
==========

Veamos a continuación unos ejercicios

Ejercicio 1
-----------

Rellenar la tabla *sevilla_osm_lines* creada en el apartado de herencia con solo aquellas geometrías de la tabla *sevilla_all_osm_geometries* que sean de tipo *LineString*.

.. note:: Para rellenar los campos *feature_name* y *feature_type*, se pueden usar, respectivamente, *tags->'name' y *COALESCE(tags->'tourism', tags->'railway', 'other')::varchar(50) As feature_type*, respectivamente


Ejercicio 2
-----------

Construir un trigger para mantener actualizado el campo geométrico de una tabla. Comenzaremos añadiendo una columna geométrica a nuestra tabla *lonlat_test* (por tanto, la vista *lonlat_test_points* ya no sería necesaria)::
	
	SELECT AddGeometryColumn ('public','lonlat_test','geom', 4326,'POINT',2);

Completar el código del presente trigger para poder actualizar la columna geométrica de la tabla después de cada inserción y actualización::
	
	CREATE OR REPLACE FUNCTION lonlat_test_pop_geom()
		RETURNS TRIGGER AS $popgeom$

	BEGIN
		-- Insertar el código del trigger

		RETURN NEW;
	END;

	$popgeom$ LANGUAGE plpgsql;



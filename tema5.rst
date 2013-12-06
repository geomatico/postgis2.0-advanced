.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


|PGIS| Topology
***************

Veremos a continuación el uso de la topología en |PGIS|, y cómo es un concepto importante para el aseguramiento de la calidad de los datos geométricos.

Introducción
============

El disponer de unos datos geográficos de calidad es de clave importancia si se quiere que dichos datos sean utilizados para diversos propósitos y durante periodos de tiempo prolongados. Sobre todo, desde que los datos geográficos han cobrado especial importancia de cara al usuario de a pie, no solo el especialista GIS. El valor de negocio de nuestros datos, así como la función informativa que pueden cumplir en el contexto de un sistema de información accesible de manera universal, ha crecido enormemente en los últimos tiempos. Es necesario contar con técnicas para asegurarnos de que dichos datos mantienen unos estándares de calidad y accesibilidad.

Podríamos dividir los parámetros que definen la calidad de nuestros datos en dos grandes grupos:

Parámetros de calidad interna: **¿Cuánto se parecen los datos a la realidad?**

	.. image:: _images/calidad_interna.png
		:scale: 50%


Parámetros de calidad externa: **¿Cuánto se acercan estos datos a lo que el usuario necesita?**

	.. image:: _images/calidad_externa.png
		:scale: 50%


.. note:: Capturas realizadas a partir de la presentación *GIS data quality*, parte del curso *GIS, Photogrammetry and Remote Sensing*, de Lakehead University, disponible `aquí <http://flash.lakeheadu.ca/~forspatial/4270/GIS_data_quality.pdf>`_ 
	

Hay una serie de técnicas y buenas prácticas que nos ayudan a asegurar la calidad de nuestros datos. Para algunas de estas técnicas puede resultarnos de utilidad disponer de una base de datos espacial con |PGIS|. Enumeramos a continuación una serie de buenas prácticas a seguir cuando trabajamos con datos espaciales (y en particular con |PGIS|) si queremos asegurar lo más posible la calidad de los datos geográficos.

	* Uso de una base de datos espacial: Formatos como |SHP| son utilizados normalmente para exportación e intercambio de datos, pero el uso de una base de datos espacial nos permite tener centralizados tanto nuestros datos como las reglas de negocio para su análisis y manipulación. Algunas de estas reglas pueden impedir el mal uso de nuestros datos, como la definición de restricciones en nuestras tablas. Otras aseguran la conectividad y relación entre todos nuestros datos, como la utilización de claves foráneas. Por último, existen una serie de ventajas del almacenamiento en base de datos sobre el almacenamiento de ficheros en disco que no se abordarán, por exceder de los contenidos del presente curso.
	* Creación de un modelo de datos adecuado: Una base de datos espacial es el receptáculo de nuestros datos y las reglas para manipularlos y analizarlos, pero es necesario que primero creemos una representación adecuada de la realidad y la transformemos en un modelo de datos. Características propias de |PGSQL|, como la herencia de tablas, pueden ayudarnos con la creación de un modelo de datos adecuado.
	* Controlar la precisión y exactitud de nuestros datos: Oracle nos permite establecer un parámetro general para manejar la tolerancia en las medidas de nuestras geometrías. |PGIS| no permite algo parecido. Utiliza la máxima tolerancia que le da GEOS, la librería que maneja las geometrías por debajo. Esta tolerancia está definida a base de datos de tipo *double* para almacenar las coordenadas. Funciones como `ST_Snap <http://postgis.net/docs/manual-2.0/ST_Snap.html>`_ o `ST_SnapToGrid <http://postgis.net/docs/manual-2.0/ST_SnapToGrid.html>`_, en la parte vectorial, y `ST_SnapToGrid <http://postgis.net/docs/manual-2.0/RT_ST_SnapToGrid.html>`_, en la parte raster, nos pueden ayudar a alinear nuestros datos con unos criterios concretos de precisión y exactitud.
	* Definición de cuáles van a ser los metadátos asociados a cada capa. |PGIS| usa un esquema en el que se almacenan de manera inherente solo los metadatos relativos a la geometría o raster que almacene una tabla (*extent*, geolocalización, tamaño de píxel, SRID, etc). Pero se pueden crear tantas columnas adicionales en nuestras tablas como consideremos necesario. Libertad total.
	* Definir claves primarias y únicas para todas nuestras tablas.
	* Evitar, en la medida de lo posible, la existencia de campos nulos mediante el uso de ``NOT NULL`` cuando creamos nuestro modelo de datos.
	* Establecer restricciones automáticas a la hora de cargar nuestros datos:
		* A través de la nueva sintaxis typmod presente en |PGIS|, que permite establecer el tipo y SRID de nuestros campos geométricos desde el momento de la creación (ver la presentación de novedades de |PGIS| 2.0 en el congreso FOSS4G 2011: [`PDF <http://www.postgis.us/downloads/FOSS4G2011PostGIS20NewStuff.pdf>`_])
		* Como alternativa a lo anterior, mediante el uso de la función `AddGeometryColumn <http://postgis.net/docs/manual-2.0/AddGeometryColumn.html>`_ (aunque ya no es necesario en |PGIS| 2.0)
		* Estableciendo el flag ``-C`` a la hora de cargar nuestros datos de tipo raster con `raster2pgsql <http://postgis.net/docs/manual-2.0/using_raster.xml.html#RT_Raster_Loader>`_
	* Estableciendo restricciones para los datos una vez cargados en nuestras tablas. En `este enlace <http://www.spatialdbadvisor.com/postgis_tips_tricks/127/how-to-apply-spatial-constraints-to-postgis-tables>`_ hay algunos ejemplos de cómo establecer restricciones en nuestras tablas geométricas para asegurar la calidad de los datos.
	* Usando un modelo topológico para nuestros datos: el definir nuestros datos en términos de vértices, aristas y caras, permite asegurarnos de que nuestra información espacial es topológicamente íntegra (cada intersección es un nodo de la red, las aristas son compartidas) y ahorrar espacio de almacenamiento (cada arista y cada nodo son almacenados una sola vez, de manera que los objetos de nuestro modelo son creados mediante composición de elementos existentes). Además, las relaciones espaciales se vuelven explícitas (en todo momento sabemos si una arista o nodo son compartidos por varios objetos, y conocemos siempre las caras izquierda y derecha de cualquier arista).


La diferencia entre precisión y exactitud, se entiende muy bien en esta imagen (obtenida también de la presentación *GIS data quality*, parte del curso *GIS, Photogrammetry and Remote Sensing*, de Lakehead University, disponible `aquí <http://flash.lakeheadu.ca/~forspatial/4270/GIS_data_quality.pdf>`_)

	.. image:: _images/precision_vs_exactitud.png
		:scale: 30%

El último punto mencionado, el uso de topología, tiene importancia suficiente como para dedicarle un apartado independiente. El soporte de topología en |PGIS| existe desde la versión 1.3, pero con la versión 2.0 se hizo *oficial* y ya se permite su instalación como extensión de |PGIS|, mediante la orden ``create extension postgis_topology``.

.. seealso:: `Documentación oficial sobre PostGIS topology <http://postgis.net/docs/manual-2.0/Topology.html>`_

.. seealso:: `Presentación sobre las novedades de la extensión *topology* en |PGIS| 2.0 <http://strk.keybit.net/projects/postgis/Paris2011_TopologyWithPostGIS_2_0.pdf>`_ 


La extensión Topology en |PGIS|
================================

Como ya hemos dicho, |PGIS| permite la creación de objetos topológicos. En concreto, implementa el `modelo ISO SQL/MM <http://www.doc-live.com/h2-topo-geo>`_. La implementación consiste en un modelo topológico (nodos, aristas, caras, relaciones) y una serie de funciones básicas para operar sobre ellos. Adicionalmente, |PGIS| proporciona un nuevo tipo de dato llamado *TopoGeometry*. Este tipo de dato se comporta de manera similar al tipo *Geometry*, con la excepción de que está definido haciendo referencia a componentes topológicos básicos (nodos, aristas, caras) **compartidos** entre todos los elementos de la topología. 


Algo interesante es que podemos crear objetos topológicos a partir de geometrías existentes en |PGIS|. Para ello, solo tenemos que seguir unos sencillos pasos:

	* Crear un nuevo esquema para almacenar nuestra topología. Lo hacemos con la función `CreateTopology <http://postgis.net/docs/manual-2.0/CreateTopology.html>`_. Eso creará una nueva entrada en la tabla *topology.topology* (donde se registran las topologías que creamos)
	* Añadir una nueva capa a nuestra topología (el equivalente a añadir una nueva capa geométrica en |PGIS|). Lo hacemos con la función `AddTopoGeometryColumn <http://postgis.net/docs/manual-2.0/AddTopoGeometryColumn.html>`_. Eso creará una nueva entrada en la tabla *topology.layer* (donde se añaden las tablas que registramos como nuevas capas con información topológica pertenecientes a cualquier esquema topológico creado)
	* Rellenar la nueva capa con datos de una geometría ya existente. Lo hacemos con la función `toTopoGeom <http://postgis.net/docs/manual-2.0/toTopoGeom.html>`_, asignando valores a nuestra nueva capa mediante una sentecia ``update`` que vaya transformando todos las filas de una tabla que contenga una geometría a nuestra nueva tabla topológica. 

Con eso tendriamos un nuevo esquema creado en nuestra base de datos, con el nombre que le hayamos especificado a llamar a *CreateTopology*.En dicho esquema, habrá 4 tablas: *edge*, *face*, *node* y *relation*

Si lo que queremos es crear una capa topológica añadiendo manualmente nodos y aristas, podemos hacerlo con funciones como `ST_AddIsoNode <http://postgis.net/docs/manual-2.0/ST_AddIsoNode.html>_`,  `ST_AddEdgeNewFaces <http://postgis.net/docs/manual-2.0/ST_AddEdgeNewFaces.html>`_, `ST_AddEdgeModFace <http://postgis.net/docs/manual-2.0/ST_AddEdgeModFace.html>`_ o `ST_ModEdgeSplit <http://postgis.net/docs/manual-2.0/ST_ModEdgeSplit.html>`_. Posteriormente, crearíamos las columnas topológicas a partir de esos elementos mediante la función `CreateTopoGeom <http://postgis.net/docs/manual-2.0/CreateTopoGeom.html>`_. Dicha función exige que haya una columna topológica existente sobre la que crear la topología. La crearíamos con *AddTopoGeometryColumn* previamente. 

Alternativamente, a la creación de elementos de manera manual y la posterior creación de una columna topológica usando esos elementos, podríamos rellenar una topología vacía con una colección de elementos geométricos, mediante la función `ST_CreateTopoGeo <http://postgis.net/docs/manual-2.0/ST_CreateTopoGeo.html>`_. En este caso, también sería necesaria la existencia de una columna topológica, que crearíamos mediante *AddTopoGeometryColumn*.

Para entender bien las maneras de crear una topología, además de las funciones mencionadas y enlazadas en el apartado anterior, podemos consultar la `presentación de Sandro Santilli sobre PostGIS Topology en una conferencia en París, en el año 2011 <http://strk.keybit.net/projects/postgis/Paris2011_TopologyWithPostGIS_2_0.pdf>`_. Se muestran ejemplos muy sencillos de creación de topologías.



Caso de uso de Topology: simplificación de geometrías
=====================================================

Una de las utilidades que puede presentar la extensión *topology* de |PGIS| es la simplificación de geometrías. Vamos a ver un ejemplo concreto, tratado por `Sandro Santilli en su blog <http://strk.keybit.net/blog/2012/04/13/simplifying-a-map-layer-using-postgis-topology/>`_

En primer lugar, tendremos que instalar la extensión |PGIS| topology en nuestra base de datos::
	
	CREATE EXTENSION postgis_topology;

Eso creará un nuevo esquema *topology* en nuestra base de datos. Con las dos tablas de metadatos mencionadas en el apartado anterior: *topology* y *layer*.

Vamos a cargar un fichero |SHP| con los municipios de Francia. El fichero se llama *DEPARTEMENT.shp*, y lo podemos encontrar `aquí <http://www.geotests.net/cours/sigma/webmapping/donnees_pg/>`_. **OJO**: El enlace de descarga original mostrado en el blog de Sandro Santilli no funciona. Lo cargamos en nuestra base de datos::

	$ shp2pgsql -I -s 2154 DEPARTEMENT.SHP france_dept > DEPARTEMENT.sql
	$ psql -d workshop_sevilla -f DEPARTEMENT.sql

Éste es el aspecto que tiene en QGIS:

	.. image:: _images/france_qgis.png
		:scale: 50%

Ahora vamos a:
	* Crear una topología nueva
	* Añadir una capa de tipo *MULTIPOLYGON* a esa topología
	* Llenar nuestra capa con las geometrías de la capa recién cargada

Lo haremos con las siguientes consultas::
	
	SELECT CreateTopology('france_dept_topo', find_srid('public', 'france_dept', 'geom'));
	SELECT AddTopoGeometryColumn('france_dept_topo', 'public', 'france_dept', 'topogeom', 'MULTIPOLYGON');
	UPDATE france_dept SET topogeom = toTopoGeom(geom, 'france_dept_topo', 1);

Veremos que se ha creado un nuevo esquema llamado *france_dept_topo*, y dentro hay 4 tablas: *edge_data*, *face*, *node*, *relation*. 

Ahora procederemos a simplificar esa geometría. Podríamos hacerlo usando la función `ST_ChangeEdgeGeom <http://postgis.net/docs/manual-2.0/ST_ChangeEdgeGeom.html>`_, como sigue::
	
	SELECT ST_ChangeEdgeGeom('france_dept_topo', edge_id, ST_Simplify(geom, 10000))
	FROM france_dept_topo.edge;
 
Esta función hace uso de `ST_Simplify <http://postgis.net/docs/manual-2.0/ST_Simplify.html>`_ para simplificar una geometría siguiendo el `algoritmo de Douglas-Peucker <http://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm>`_. El problema es que lanza errores cuando se encuentra con un error topológico. Podríamos usar en su lugar `ST_SimplifyPreserveTopology <http://postgis.net/docs/manual-2.0/ST_SimplifyPreserveTopology.html>`_, pero esta función realiza cálculos bastante complejos en GEOS, y ha de transformar geometrías para su uso en GEOS. El rendimiento es más pobre. Sería necesario un equilibrio entre un algoritmo que preservara la topología al simplificar y, al mismo tiempo, que el proceso fuera lo suficientemente rápido como para que no penalizara el rendimiento de una aplicación que lo use intensivamente. 

La solución que se ha encontrado es implementar el mismo algoritmo de Douglas-Peucker pero haciendo uso del tipo *TopoGeometry* introducido por la extensión *topology*. La función que realiza esto también se llama *ST_Simplify*, pero acepta un objeto topológico en lugar de una geometría. Actualmente, se encuentra `en la versión de desarrollo de PostGIS <http://postgis.net/docs/manual-dev/TP_ST_Simplify.html>`_.

Para el presente ejercicio, no obstante, nos conformaremos con un *wrapper* sobre la función *ST_Topology* que captura la excepción lanzada en caso de error topológico y varía el factor de tolerancia hasta encontrar uno válido. La función ha sido creada por Sandro Santilli::
	
	CREATE OR REPLACE FUNCTION SimplifyEdgeGeom(atopo varchar, anedge int, maxtolerance float8)
	RETURNS float8 AS $$
	DECLARE
	  tol float8;
	  sql varchar;
	BEGIN
	  tol := maxtolerance;
	  LOOP
		sql := 'SELECT topology.ST_ChangeEdgeGeom(' || quote_literal(atopo) || ', ' || anedge
		  || ', ST_Simplify(geom, ' || tol || ')) FROM '
		  || quote_ident(atopo) || '.edge WHERE edge_id = ' || anedge;
		BEGIN
		  RAISE DEBUG 'Running %', sql;
		  EXECUTE sql;
		  RETURN tol;
		EXCEPTION
		 WHEN OTHERS THEN
		  RAISE WARNING 'Simplification of edge % with tolerance % failed: %', anedge, tol, SQLERRM;
		  tol := round( (tol/2.0) * 1e8 ) / 1e8; -- round to get to zero quicker
		  IF tol = 0 THEN RAISE EXCEPTION '%', SQLERRM; END IF;
		END;
	  END LOOP;
	END
	$$ LANGUAGE 'plpgsql' STABLE STRICT;

Ya podríamos ejecutar una consulta de simplificación::
	
	SELECT SimplifyEdgeGeom('france_dept_topo', edge_id, 10000) FROM france_dept_topo.edge;

Y añadir la columna simplificada a nuestra tabla original, para poder visualizar el resultado::
	
	ALTER TABLE france_dept ADD geomsimp GEOMETRY;
	UPDATE france_dept SET geomsimp = topogeom::geometry;

Éste es el aspecto de la tabla simplificada en QGIS:

	.. image:: _images/france_qgis_simp.png
		:scale: 50%


Validación topológica
======================

Ya hemos mencionado que |PGIS| proporciona algunas herramientas para el aseguramiento de la calidad de los datos geográficos, mediante la imposición de ciertas condiciones en las tablas que carguemos en nuestra base de datos. Si usamos la extensión *topology*, veremos que las restricciones sobre los datos cargados en nuestras tablas son aun más numerosas. Esto es así porque las relaciones entre los elementos de una topología han de cumplirse de manera estricta (no puede haber intersecciones, solapes, etc).

En otras palabras: el uso de la extensión *topology* va a **obligar** a que las geometrías a partir de las cuales hemos creado nuestros elementos topológicos (si elegimos esa opción) cumplan una serie de restricciones adicionales. Podemos verlo si comprobamos las restricciones impuestas en las 4 tablas creadas al generar una topología (*edge_data*, *face*, *node*, *relation*).

En cualquier caso, tenemos una función que sirve para validar topologías: `ValidateTopology <http://postgis.net/docs/manual-2.0/ValidateTopology.html>`_. Al igual que una serie de funciones de edición de topologías, que nos permitirán crear y eliminar nodos y aristas.

Editar topologías a mano es algo bastante complejo. Tenemos a nuestro alcance otras maneras de *limpiar* topologías inválidas:
	* `Atacando a las geometrías subyacentes <http://trac.osgeo.org/postgis/wiki/UsersWikiCleanPolygons>`_
	* `Convirtiendo nuestros objetos Geometry en objetos TopoGeometry <http://postgis.net/docs/manual-2.0/toTopoGeom.html>`_


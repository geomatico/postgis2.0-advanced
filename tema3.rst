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

Trabajando con datos raster
***************************

Desde la versión 2.0 de |PGIS|, es posible cargar y manipular datos de naturaleza ráster en una base de datos espacial, gracias a |PRAS|: un nuevo conjunto de tipos y funciones que dotan a PostGIS de la posibilidad de manipular datos ráster.

El objetivo de |PRAS| es **la implementación de un tipo de datos RASTER lo más parecido posible al tipo GEOMETRY de** |PGIS| **, y ofrecer un único conjunto de funciones SQL que operen de manera transparente tanto en coberturas vectoriales como en coberturas de tipo ráster.**

Como ya hemos mencionado, |PRAS| es parte oficial de |PGIS| 2.0, de manera que **no es necesario instalar ningún software adicional**. Cuando una base de datos PostgreSQL es activada con |PGIS|, ya es al mismo tiempo también activada con |PRAS|.


Tipo de datos Raster
====================

El aspecto más importante a considerar sobre el nuevo tipo de datos RASTER definido por |PRAS| es **que tiene significado por si mismo**. O de otra forma: **una columna de tipo RASTER es una cobertura raster completa, con metadatos y posiblemente geolocalizada, pese a que pertenezca a una cobertura raster mayor**.

Tradicionalmente, las bases de datos espaciales con soporte para ráster, han permitido cargar y teselar coberturas ráster para operar con ellas. Normalmente, se almacenaban los metadatos de la cobertura completa por un lado (geolocalización, tamaño de píxel, srid, extensión, etc) y las teselas por otro, como simples *chunks* de datos binarios adyacentes. La filosofía tras |PRAS| es diferente. También permite la carga y teselado de coberturas completas, pero cada tesela por separado, **contiene sus propios metadatos y puede ser tratada como un objeto raster individual**. Además de eso, una tabla de |PRAS| cargada con datos pertenecientes a una misma cobertura:

	* Puede tener teselas de diferentes dimensiones (alto, ancho).
	* Puede tener teselas no alineadas con respecto a la misma rejilla.
	* Puede contener *huecos* o teselas que se solapan unas con otras.

Este enfoque hace a |PRAS| una herramienta muy poderosa, aunque también tiene algunos problemas inherentes (como el bajo rendimiento en aplicaciones de visualización de datos ráster en tiempo real). Para saber más sobre |PRAS| el mejor sitio donde acudir es http://trac.osgeo.org/postgis/wiki/WKTRaster

En los siguientes apartados, veremos más sobre |PRAS| y algunas de las operaciones que se pueden realizar con las funciones que proporciona.


Similitudes con el formato VRT de GDAL
======================================

En otros capítulos hemos hablado del `formato VRT de GDAL <http://www.gdal.org/gdal_vrttut.html>`_. Ya hemos mencionado que se trata de un *formato contenedor*, capaz de referenciar diferentes capas raster y presentar cara al exterior una interfaz única, de manera que para cualquier herramienta que lea este formato, **lo que está leyendo es un único raster**. El formato en si maneja toda la complejidad para presentar esta interfaz.

En cierto modo, |PRAS| funciona de la misma manera. En un primer vistazo, puede parecer que una tabla de |PRAS| no es más que un conjunto de teselas. Que por un lado están los metadatos de la capa, y por otro las teselas, que sirven para evitar tener un único objeto que contenga todos los datos (como en el resto de formatos que aceptan teselado). 

No es así.

|PRAS| esconde una sutileza que lo hace más versátil, pero también mucho más complejo de manejar por herramientas de terceros. Es algo que ya se ha dicho, pero que se repite por su importancia: **cada tesela es un raster individual**. Y lo es en el mismo sentido que **cada fuente que VRT maneja de manera transparente es una fuente individual**.

Cuando leemos de VRT, por debajo podemos estar leyendo de una multitud de datos raster diferentes. Y cada dato raster **puede tener su propio tamaño de pixel y georreferenciación**. Del mismo modo, cuando leemos de una tabla |PRAS|, estamos leyendo de una multitud de datos raster (teselas) diferentes.

No en vano, la implementación del driver de |GDAL| para |PRAS|, **hereda directamente del driver de VRT**

Esto, como ya se ha dicho, tiene la ventaja de la versatilidad. Cada tesela (fila) tiene vida por separado, y puede ser leída como raster independiente o como parte de una cobertura mayor (la tabla). Como desventaja, la lectura de datos |PRAS| para su visualización en tiempo real es especialmente conflictiva, debido a que el cálculo de dimensiones de una capa |PRAS| **no es algo trivial**. Las teselas que forman una cobertura pueden tener diferentes tamaños, e incluso montarse unas con otras, o tener *agujeros*. 


Usando GDAL para facilitarnos la carga de datos en |PRAS|
=========================================================

En el tema 1 vimos el uso básico del cargador de |PRAS|, ``raster2pgsql``. También vimos como la librería |GDAL| nos facilitaba la carga de datos vectoriales en |PGIS| mediante el uso de ``ogr2ogr``. Dicha herramienta supera algunas de las limitaciones con las que cuenta el cargador oficial de |PGIS|, permitiendo cargar datos en muchos más formatos vectoriales que |SHP|.

Actualmente, no es posible realizar la misma función que ``ogr2ogr`` pero con capas raster. El driver de |GDAL| para |PRAS| aun no tiene funcionalidad de escritura, de manera que no se pueden usar ``gdal_translate`` o ``gdalwarp`` para cargar directamente datos en |PRAS|. Pero lo que si podemos hacer es utilizar la versatilidad de |GDAL| para preparar nuestros datos raster con anterioridad a la carga de los mismos en |PRAS|. Lo veremos a continuación.


Uniendo capas raster para su carga
----------------------------------

Vamos a ver con un ejemplo práctico como unir varias capas raster y recortar una zona de interés antes de pasárle los datos a ``raster2pgsql`` para que los cargue en la base de datos.

Lo que queremos cargar es una capa raster que contiene datos de temperaturas medias en todo el continente europeo en el mes de Noviembre de 2010. Los datos los descargamos de `la web de worldclim <http://www.worldclim.org/tiles.php?Zone=15>`_. Como podemos observar, España está dividida entre dos teselas: la 15 y la 16.

Para este ejemplo hemos descargado las capas correspondientes a las teselas 15 y 16, y extraído solo la correspondiente al mes de Noviembre en ambos casos. Como resultado, tenemos dos ficheros GeoTIFF, que cubren la totalidad de Europa. Lo que queremos es recortar, de esos dos ficheros, únicamente la zona de España. Y utilizando solo las herramientas de línea de comandos proporcionadas por |GDAL|.

En la captura, hemos cargado las dos capas en QGIS, coloreándolas de manera diferente, y hecho zoom a la zona de España. Vemos que una parte queda fuera de la primera capa, y entra en la segunda. Los ficheros que representan ambas capas son *alt_15.tif* y *alt_16.tif*


	.. image:: _images/ej3_tiffs_temperatura_qgis1.png
		:scale: 50 %


El procedimiento que vamos a realizar pasa por construir un raster virtual en `formato VRT <http://www.gdal.org/gdal_vrttut.html>`_, recortar una porción del raster resultante y cargar esa porción con ``raster2pgsql``. 

Primero, construimos el VRT, en el mismo directorio donde tengamos los datos::
	
	$ cd /path/to/data
	$ gdalbuildvrt tmean11.vrt tmean11_15.tif tmean11_16.tif

Ahora, mediante ``gdal_translate``, recorgamos la zona que nos interesa (las coordenadas han sido obtenidas con QGIS, y su obtención se propone como ejercicio en el tema 4)::

	$ gdal_translate -projwin -9.82594936709 43.9746835443 4.67088607595 35.914556962 tmean11.vrt tmean11_spain.tif

El fichero resultado, *tmean_spain.tif*, puede verse cargado en QGIS:

	.. image:: _images/ej3_tiffs_temperatura_qgis2.png
		:scale: 50 %

Ya podemos cargar nuestra imagen, mucho más reducida, mediante ``raster2pgsql``, como vimos en el tema 1::
	
	$ raster2pgsql -I -C -F -t 36x36 -M -s 4326 tmean11_spain.tif > tmean11_spain.sql
	$ psql -d workshop_sevilla -f tmean11_spain.sql


Lidiando con formatos conflictivos
----------------------------------

|GDAL| es muy versátil, y capaz de lidiar con formatos gráficos propietarios, tales como `ECW <http://www.gdal.org/frmt_ecw.html>`_ o `MrSID <http://www.gdal.org/frmt_mrsid.html>`_. Para trabajar con ellos, necesita acceso a librerías de terceros. La librería disponible para el formato ECW solo permite lectura en su versión gratuita. Los fuentes se pueden descargar desde `aquí <https://api.opensuse.org/public/source/home:jluce2:GEO/libecwj/libecwj2-3.3.tar.bz2>`_.

En algunos de los ejemplos, se han utilizado imágenes del PNOA (Plan Nacional de Ortofotografía aérea. Más información `aquí <http://www.ign.es/PNOA/>`_. Dichas imágenes están almacenadas en formato ECW, y |GDAL| no es capaz de leerlo por defecto. Es necesario compilar la librería anterior y recompilar GDAL con soporte para la misma, mediante el uso del flag ``--with-ecw``. Hecho eso, seremos capaces de transformar desde el formato ECW a GeoTIFF, y poder trabajar con las imágenes sin problemas de incompatibilidades. Nuestro fichero ECW se llama PNOA_MA_OF_ETRS89_HU30_h50_0984.ecw, y vamos a transformarlo a formato GeoTIFF y reducir su tamaño, para evitar que ocupe demasiado ::

	$ gdal_translate -outsize 10% 10% PNOA_MA_OF_ETRS89_HU30_h50_0984.ecw PNOA_MA_OF_ETRS89_HU30_h50_0984_reduced.tif

Y ya podemos cargar el raster normalmente::
	
	$ raster2pgsql -I -C -F -t 53x23 -M -s 25830 PNOA_MA_OF_ETRS89_HU30_h50_0984_reduced.tif pnoa_sevilla > pnoa_sevilla.sql
	$ psql -d workshop_sevilla -f pnoa_sevilla.sql


.. note:: Incluso con la librería compilada con soporte para ECW, pueden existir problemas con el formato. Por ejemplo, en ocasiones |GDAL| no es capaz de decodificar la cabecera del ECW para obtener los metadatos. Recomendamos el uso de la variable de entorno ``GDAL_DEBUG=ecw`` mientras trabajamos con las herramientas de |GDAL|, para poder obtener información extra de depuración que nos de los datos requeridos.


|PRAS| Overviews
================

Las *overviews* no son más que versiones de un raster a menor resolución, útiles para su visualización a diferentes niveles de zoom, desde un visor cliente. Construir un conjunto de *overviews* se conoce popularmente como *construir pirámides*, aunque `ArcGIS 10 considera estos dos conceptos como diferentes <http://blogs.esri.com/esri/arcgis/2011/04/06/pyramids-and-overviews-or-pyramids-or-overviews/>`_.

En |PRAS| hay dos maneras de construir *overviews*:
	* Durante la carga de los datos, utilizando el flag ``-l`` de ``raster2pgsql``. 
	* Una vez los datos ya están cargados, mediante la utilización de la función `ST_Rescale <http://postgis.net/docs/manual-2.0/RT_ST_Rescale.html>`_

Cuando se utiliza el primer método, veremos que se crea *una tabla diferente por cada overview que queramos añadir*. Cada tabla *overview* tendrá un nombre con el formato *o_<overview_factor>_<table>*, donde <overview_factor> es el factor de escala, un entero, y <table> es el nombre de la tabla original

Como ejemplo, vamos a ver cómo crear overviews de una capa TIFF durante su carga. Usaremos los ficheros *bio11_15.tif* y *bio11_16.tif* (construiremos primero un VRT)::
	
	$ gdalbuildvrt bio11.vrt bio11_15.tif bio11_16.tif

Recortamos España, como hicimos con la capa de temperaturas::

	$ gdal_translate -projwin -9.82594936709 43.9746835443 4.67088607595 35.914556962 bio11.vrt bio11_spain.tif

Y cargamos la capa **especificando que se creen 3 overviews**::
	
	$ raster2pgsql -I -C -F -t 36x36 -M -s 4326 -l 2,4,8 bio11_spain.tif > bio11_spain.sql
	$ psql -d workshop_sevilla -f bio11_spain.sql

El otro método de obtención de overviews se propondrá como ejercicio para el alumno


Obteniendo metadatos y estadísticas de capas raster
===================================================

Una vez tenemos nuestras imágenes cargadas, vamos a realizar algunas operaciones básicas de obtención de información. Si bien lo haremos con las funciones de |PRAS|, no olvidemos que podemos acceder a nuestros datos **como si accediéramos a cualquier formato gráfico soportado por GDAL**. Por ejemplo, para obtener metadatos y estadísticas, podemos usar *gdalinfo*::

	$ gdalinfo -mm -stats PG:"dbname=workshop_sevilla mode=2"

Ésta es la información que obtendremos::
	
	Driver: PostGISRaster/PostGIS Raster driver
	Files: none associated
	Size is 1219, 782
	Coordinate System is:
	PROJCS["ETRS89 / UTM zone 30N",
    	GEOGCS["ETRS89",
        	DATUM["European_Terrestrial_Reference_System_1989",
            	SPHEROID["GRS 1980",6378137,298.257222101,
                	AUTHORITY["EPSG","7019"]],
            	TOWGS84[0,0,0,0,0,0,0],
            	AUTHORITY["EPSG","6258"]],
        	PRIMEM["Greenwich",0,
           		AUTHORITY["EPSG","8901"]],
        	UNIT["degree",0.0174532925199433,
            	AUTHORITY["EPSG","9122"]],
        	AUTHORITY["EPSG","4258"]],
    	UNIT["metre",1,
        	AUTHORITY["EPSG","9001"]],
    	PROJECTION["Transverse_Mercator"],
    	PARAMETER["latitude_of_origin",0],
    	PARAMETER["central_meridian",-3],
    	PARAMETER["scale_factor",0.9996],
    	PARAMETER["false_easting",500000],
    	PARAMETER["false_northing",0],
    	AUTHORITY["EPSG","25830"],
    	AXIS["Easting",EAST],
    	AXIS["Northing",NORTH]]
	Origin = (217540.000000000000000,4155170.000000000000000)
	Pixel Size = (25.008278145695400,-25.000000000000000)
	Corner Coordinates:
	Upper Left  (  217540.000, 4155170.000) (  6d11'42.67"W, 37d30' 1.01"N)
	Lower Left  (  217540.000, 4135620.000) (  6d11'15.77"W, 37d19'27.61"N)
	Upper Right (  248025.091, 4155170.000) (  5d51' 2.72"W, 37d30'32.78"N)
	Lower Right (  248025.091, 4135620.000) (  5d50'38.70"W, 37d19'59.17"N)
	Center      (  232782.546, 4145395.000) (  6d 1'10.01"W, 37d25' 0.59"N)
	Band 1 Block=53x23 Type=Byte, ColorInterp=Red
    	Computed Min/Max=0.000,255.000
  	Minimum=0.000, Maximum=255.000, Mean=130.923, StdDev=45.081
  	Metadata:
    	STATISTICS_MAXIMUM=255
    	STATISTICS_MEAN=130.92284460241
    	STATISTICS_MINIMUM=0
    	STATISTICS_STDDEV=45.081433973161
	Band 2 Block=53x23 Type=Byte, ColorInterp=Green
    	Computed Min/Max=0.000,255.000
  	Minimum=0.000, Maximum=255.000, Mean=125.201, StdDev=39.143
 	Metadata:
    	STATISTICS_MAXIMUM=255
    	STATISTICS_MEAN=125.20149739105
    	STATISTICS_MINIMUM=0
    	STATISTICS_STDDEV=39.142589242722
	Band 3 Block=53x23 Type=Byte, ColorInterp=Blue
    	Computed Min/Max=0.000,255.000
  	Minimum=0.000, Maximum=255.000, Mean=109.540, StdDev=33.657
  	Metadata:
    	STATISTICS_MAXIMUM=255
    	STATISTICS_MEAN=109.53984755439
    	STATISTICS_MINIMUM=0
    	STATISTICS_STDDEV=33.65724324696
	 
.. note:: Para evitar tener que especificar la cadena completa de conexión con |PRAS|, hemos definido las variables de entorno PGHOST, PGPORT, PGUSER y PGPASSWORD con valores adecuados.

No obstante, veremos como obtener metadatos y estadísticas con funciones de |PRAS|, como ya se ha dicho


Obtención de metadatos
----------------------

Podemos obtener los metadatos de una tabla |PRAS| mediante una consulta al catálogo *raster_columns*

El catálogo *raster_columns* se mantiente actualizado automáticamente con los cambios de las tablas que contiene. Las entradas y salidas del catálogo se controlan mediantes las funciones **AddRasterConstraints** y **DropRasterConstraints**. Para más información, consultar http://postgis.net/docs/manual-2.0/using_raster.xml.html#RT_Raster_Columns

Para consultar los metadatos de una tabla mediante el catálogo *raster_columns* hacemos::


	SELECT
		r_table_name,
		r_raster_column,
		srid,
		scale_x,
		scale_y,
		blocksize_x,
		blocksize_y,
		same_alignment,
		regular_blocking,
		num_bands,
		pixel_types,
		nodata_values,
		out_db,
		ST_AsText(extent) AS extent
	FROM raster_columns WHERE r_table_name = 'pnoa_sevilla';

Como salida obtendremos una fila conteniendo los metadatos de la tabla.

También podemos obtener metadatos mediante las funciones *ST_MetaData* y *ST_BandMetaData*, pero hemos de tener en cuenta que estas funciones **operan sobre una sola columna** mientras que la consulta a *raster_columns* **obtiene los datos de la tabla completa**. En el caso de que el ráster cargado en |PRAS| sea teselado, lo más normal, posiblemente no nos interese obtener los metadatos de cada una de las teselas, sino de la cobertura completa.

Veamos un ejemplo. Para obtener los metadatos de la banda 1 de la tabla *tmean11_spain*::

	SELECT rid,(ST_BandMetadata(rast, 1)).* FROM tmean11_spain;

Como salida, veremos varias filas. No olvidemos que, **al ser nuestro raster teselado, cada fila es una tesela**

Veamos ahora cómo obtener estadísticas

Obtención de estadísticas
-------------------------

Para obtener estadísticas básicas de la banda 1 de *tmean11_spain*::

	WITH stats AS (
        SELECT
                (ST_SummaryStats(rast, 1)).*
        FROM tmean11_spain
        WHERE rid = 8
	)
	SELECT
        count,
        sum,
        round(mean::numeric, 2) AS mean,
        round(stddev::numeric, 2) AS stddev,
        min,
        max
	FROM stats;

El resultado es::

	# count |  sum   |  mean  |  stddev | min | max
 	 -------+--------+--------+---------+-----+-----
   	   248  | 28087  | 113.25 |    5.53 | 98  | 120

Si observamos los valores obtenidos, vemos que son números exageradamente altos para representar temperaturas en grados centígrados. Lo que sucede es que estos valores están escalados por 100. Más información `aquí <http://www.prism.oregonstate.edu/docs/meta/temp_realtime_monthly.htm>`_. 

Se propondrá como ejercicio para el alumno el generar una nueva banda en el ráster conteniendo los valores con su escala real.


.. seealso:: Para ver más detalles del formato |PRAS| y sus diferencias con el formato de Oracle GeoRaster, se puede consultar `esta presentación <http://es.scribd.com/doc/83774246/FOSS4G-2010-Presentation-PostGIS-Raster-an-Open-Source-alternative-to-Oracle-Georaster>`_ realizada en el congreso FOSS4G en 2010.


MapAlgebra sobre capas |PRAS|
=============================

El uso de MapAlgebra permite realizar operaciones algebraicas sobre capas raster. La lógica que subyace tras esta funcionalidad es aplicar una operación algebráica a todos los píxeles de una banda y generar una nueva banda como resultado.

En el apartado anterior, vimos como los valores de temperaturas de la capa ráster estaban escalados por 100. Vamos a cambiar todos estos valores usando una expresión de MapAlgebra. Para ello, añadiremos una nueva banda con los valores cambiados::
	
	UPDATE tmean11_spain SET
		rast = ST_AddBand(
                rast,
                ST_MapAlgebraExpr(rast, 1, '32BF', '[rast] / 100.'),
                1
        );

.. note:: La familia de funciones ``ST_MapAlgebra`` devuelven *un nuevo objeto de tipo raster con una sola banda*. Si queremos modificar el raster original, debemos añadir esta banda del nuevo raster como banda adicional de nuestro raster original

.. warning:: Si se está usando |PGIS| 2.1 en lugar de 2.0, la función ``ST_MapAlgebraExpr`` pasa a ser ``ST_MapAlgebra``

En la llamada a MapAlgebra, hemos especificado que la banda de salida tendrá un tamaño de píxel de 32BF y un valor NODATA de -9999. Con la expresión *[rast] / 100*, convertimos cada valor de píxel a su valor previo al escalado.

Tras ejecutar esa consulta, el resultado es éste::
	
	ERROR:  new row for relation "tmean11_spain" violates check constraint "enforce_out_db_rast"
	********** Error **********
	ERROR: new row for relation "tmean11_spain" violates check constraint "enforce_out_db_rast"
	SQL state: 23514

Como vemos, la consulta no ha funcionado. El problema es que, cuando cargamos esta capa ráster usando raster2pgsql, especificamos el flag **-C**. Este flag activa una serie de restricciones en nuestra tabla, para garantizar que todas las columnas de tipo RASTER tienen los mismos atributos (más información en http://postgis.net/docs/manual-2.0/RT_AddRasterConstraints.html).

El mensaje de error nos dice que hemos violado una de esas restricciones. Concretamente la restricción de *out-db*. A primera vista, puede parecer extraño, porque nosotros no estamos especificando que la nueva banda sea de tipo *out-db*. El problema es que esta restricción solo funciona con una banda, y si se intenta añadir una segunda banda a un ráster que ya tiene una, la restricción lo hace fallar.

La solución a nuestro problema pasa por:

	1. Eliminar las restricciones de la tabla mediante *DropRasterConstraints*
	2. Volver a ejecutar la consulta
	3. Volver a activar las restricciones (**OJO: Es una operación costosa en datos raster muy grandes**)


Las consultas a ejecutar son las siguientes::
	
	SELECT DropRasterConstraints('tmean11_spain', 'rast'::name);
	UPDATE tmean11_spain SET rast = ST_AddBand(rast, ST_MapAlgebraExpr(rast, 1, '32BF', '[rast] / 100.'),1);
	SELECT AddRasterConstraints('tmean11_spain', 'rast'::name);

Y el resultado es::
	
	# droprasterconstraints
	-----------------------
	t
	
	# Query returned successfully: 1323 rows affected, 11246 ms execution time.
	
	# addrasterconstraints
	----------------------
	t

La comprobación de la nueva banda añadida con los resultados correctos se deja como ejercicio para el alumno, desde la sección de ejercicios de este mismo capítulo.



Combinando raster y geometría
=============================

Una de las cualidades más poderosas de |PRAS| es la posibilidad de permitir la operación entre datos raster y vectoriales **de manera transparente**. En algunas funciones, el orden en que pasemos los parámetros de entrada (raster y vector) definirá en qué espacio trabajará la función (raster o vectorial) y en qué formato devolverá el resultado. Aunque actualmente **no todas las operaciones son capaces de operar en ambos espacios**. Para obtener una visión más completa, se recomienda visitar la `referencia de funciones de PostGIS Raster <http://postgis.net/docs/manual-2.0/RT_reference.html>`_, constatemente actualizado.



Ejercicios
==========

Veamos a continuación la mejor manera de entender cómo funciona |PRAS|, con ejemplos prácticos:



Ejercicio 1
-----------

¿Cuál ha sido la temperatura media del mes de Noviembre en los municipios de Sevilla?

.. note:: Pistas: usar la tabla *codigo_postal* (vectorial) y la tabla *tmean11_sevilla* (raster). Recordad que la tabla vectorial tiene un srid diferente al de la tabla raster. Transformar el srid de la tabla vectorial al de la tabla de tipo raster.


Ejercicio 2
-----------

Crear una nueva capa |PRAS| resultante de recortar la capa raster *pnoa_sevilla*, usando para ello la capa vectorial *codigo_postal*. Concretamente, el municipio de *Salteras*, dentro de dicha capa.

.. note:: Pista: Se puede usar como polígono de corte el siguiente::

	with clip_polygon as (
		select geom from codigo_postal where nombre_municipio = 'Salteras' and gid = 120
	)

Exportar la capa resultante como fichero TIFF usando ``gdal_translate``


Ejercicio 3
-----------

A partir de la capa creada en el ejercicio anterior, generar una nueva capa con una resolución que sea el 25% de la resolución original. Exportar dicha capa con ``gdal_translate`` como fichero TIFF y compararlo con el anterior.

.. note:: Pista: Para obtener el tamaño de píxel de la capa creada, suponiendo que hemos llamado a dicha capa *pnoa_sevilla_clip*, se puede usar esta tabla temporal::
	
	WITH meta AS (
        SELECT
			(ST_Metadata(rast)).*
        FROM pnoa_sevilla_clip
	)


Ejercicio 4
-----------

Dentro de las funciones de poligonización de |PRAS|, hay dos con un comportamiento diferente: ``ST_DumpAsPolygons`` y ``ST_PixelAsPolygons``. Vamos a ver sus diferencias con un ejemplo. Partamos de esta consulta::
	
	create table geometry_from_raster1 as
	(WITH geoms AS (
        SELECT
                ST_DumpAsPolygons(
                        ST_Union(
                                ST_Clip(t.rast, 2, ST_Transform(cp.geom, 4326), TRUE)
                        ),
                        1
                ) AS gv
        FROM tmean11_spain t
        JOIN codigo_postal cp
                ON ST_Intersects(t.rast, ST_Transform(cp.geom, 4326))
        WHERE t.rid = 944
	)
	SELECT
        (gv).val,
        (gv).geom AS geom
	FROM geoms)

Que devuelve 40 resultados.

Cambiar la consulta para que utilice la función `ST_PixelAsPolygons <http://postgis.net/docs/manual-2.0/RT_ST_PixelAsPolygons.html>`_ . Comparar el número de resultados y cargar ambas capas en QGIS para comprobar las diferencias.


Ejercicio 5
-----------

Mediante el uso de la función `ST_AsRaster <http://postgis.net/docs/manual-2.0/RT_ST_AsRaster.html>`_ convertir la tabla *codigo_postal* en una cobertura raster. Dicha cobertura raster tendrá:
	* pixel size = 100
	* 4 bandas 8BUI
	* NODATA = 0
	* Color de pixel definido por RGBA a elección del alumno, con el formato (r, g, b, a)

Exportar la capa resultante como fichero TIFF usando ``gdal_translate``


Ejercicio 6
-----------

Para este ejercicio vamos a cargar un fichero DEM en formato TIFF. Lo construiremos con gdalbuildvrt, igual que hicimos con el mapa de temperaturas. Los comandos a ejecutar son estos::

	$ gdalbuildvrt amean11.vrt amean11_15.tif amean11_16.tif

	$ gdal_translate -projwin -9.82594936709 43.9746835443 4.67088607595 35.914556962 amean11.vrt amean11_spain.tif

	$ raster2pgsql -I -C -F -t 36x36 -M -s 4326 amean11_spain.tif > amean_spain.sql

	$ psql -d workshop_sevilla -f amean_spain.sql

Con esto disponemos de un DEM de toda España. 

A partir de este raster, construimos otro que simplemente sea una unión de todas las teselas que cubran la zona representada por la tabla *codigo_postal*, dejando un pequeño margen alrededor (así no tenemos que lidiar con el DEM de toda España)::

	WITH r AS (
        SELECT
                ST_Transform(ST_Union(t.rast), 25830) AS rast
        FROM amean_spain t
        JOIN codigo_postal cp
                ON ST_DWithin(ST_Transform(t.rast::geometry, 25830), cp.geom, 1000)
	)

Usando el anterior raster, crear otro raster que almacene la pendiente de la superficie representada, y recortar solo la parte cubierta por la tabla *codigo_postal* 

.. note:: Pista: Utilizar la función `ST_Slope <http://postgis.net/docs/manual-2.0/RT_ST_Slope.html>`_ 

.. seealso:: Para saber más sobre el concepto de *slope*: `<http://webhelp.esri.com/arcgisdesktop/9.3/index.cfm?TopicName=Calculating%20slope>`_ 








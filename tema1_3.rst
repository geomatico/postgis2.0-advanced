.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


Importación y exportación de datos
**********************************
En este tema nos introduciremos en el uso de herramientas de importación/exportación de datos hasta/desde |PGIS|. Veremos 4 tipos de herramientas:
	* Herramientas |PGSQL| y |PGIS|: ``shp2pgsql``, ``pgsql2shp``, |PGA|, ``psql``
	* Herramientas |GDAL|: ``ogr2ogr``, ``gdal_translate``, ``gdalwarp`` 
	* Plugin SPIT de QGIS
	* Cargador de datos |OSM|: ``osm2pgsql``
	* Cargar datos no espaciales en |PGIS| (y convertirlos en datos espaciales)

Finalmente, veremos una serie de ejercicios prácticos, que servirán para fijar los conocimientos adquiridos.

Herramientas |PGSQL| y |PGIS|
=============================

|PGSQL| y |PGIS| propocionan herramientas para importación y exportación de datos vectoriales y raster.

Importación de datos vectoriales
--------------------------------

El cargador ``shp2pgsql`` convierte archivos |SHP| en SQL preparado para la inserción en la base de datos. Se utiliza desde la linea de comandos, aunque existe una versión con interfaz gráfica para el sistema operativo Windows. Se puede acceder a la ayuda de la herramienta mediante::

	$ shp2pgsql -?
	
Para el uso de la herramienta::

	$ shp2pgsql [<opciones>] <ruta_shapefile> [<esquema>.]<tabla>

.. warning:: El cargador ``shp2pgsql`` solo acepta datos de entrada en formato |SHP|
	
Podemos ver una explicación detallada de las opciones disponibles en la `documentación en línea del cargador vectorial <http://postgis.net/docs/manual-2.0/using_postgis_dbmanagement.html#shp2pgsql_usage>`_
	
Una vez generado el fichero SQL, puede ser cargado en |PGSQL| mediante la herramienta ``psql`` de línea de comandos o a través de `pgAdmin III <http://www.pgadmin.org/>`_, una herramienta gráfica de administración y desarrollo para Windows, Mac y |LX| 

A continuación, veremos un ejemplo de las opciones más típicas usadas a la hora de generar un fichero SQL a partir de un fichero |SHP| con ``shp2pgsql``::

    $ shp2pgsql -s 4258:25830 -I -W LATIN1 mi_fichero.shp > mifichero.sql
    
    
A destacar algunos detalles:
    * Mediante el uso de la sintaxis -s <srid_origen>:<srid_destino>, **forzamos una proyección del SRID origen al destino**. De esta forma, los datos se almacenarán en PostGIS con el srid de destino. Si solo especificamos un srid, **se asigna el mismo a los datos de destino, sin realizar la reproyección**:
    * Con la opción `-W LATIN1` especificamos que nuestros datos de origen (fichero dbf) utilizan la codificación LATIN1. Cuando usamos esta opción, todos los atributos del fichero dbf son convertidos de la codificación especificada a UTF8. El SQL generado contendrá el comando ``SET CLIENT_ENCODING to UTF8``, de manera que el servidor será capaz de convertir desde UTF8 al sistema de codificación que esté usando internamente. 
    * Es recomendable crear un índice espacial sobre la columna de tipo geométrica, si vamos a realizar consultas sobre nuestra tabla que involucren a dicha columna. Lo indicamos con la opción ``-I``
    
.. seealso:: Una `breve introducción a los sistema de codificación y los juegos de caracteres en Python <http://es.scribd.com/doc/159584080/Python-y-los-encodings>`_
    
.. seealso:: `Lo mínimo que cualquier desarrollador debe saber sobre sistemas de codificación y juegos de caracteres <http://www.joelonsoftware.com/articles/Unicode.html>`_, por Joel Spolsky

Una vez tenemos nuestro fichero SQL, podemos insertarlo en la base de datos con esta instrucción::

	psql -d <base_de_datos> -f mifichero.sql

.. note:: A partir de ahora, asumimos que el lector ha configurado el acceso a la base de datos como se especifica en el capítulo de introducción. De esta forma, evita tenerque estar proporcionando constantemente datos como usuario, contraseña o dirección donde escucha las conexiones |PGSQL|

De esta forma ya tenemos una nueva tabla en nuestra base de datos.



Importación de datos raster
---------------------------

Paralelamente al cargador oficial de |PGIS| para datos vectoriales, la versión 2.0 de la librería incluye también un cargador para datos ráster. Se trata de ``raster2pgsql``. Al igual que con los datos vectoriales, esta herramienta transforma datos raster en sentencias SQL, listas para ser insertadas en |PGIS|.

Podemos consultar la ayuda de la herramienta mediante::
	
	$ raster2pgsql -?

Y su uso básico es::

	$ raster2pgsql [<opciones>] <ruta_raster> [<esquema>.]<tabla>

Podemos ver una explicación detallada de las opciones disponibles en la `documentación en línea del cargador raster <http://postgis.net/docs/manual-2.0/using_raster.xml.html#RT_Raster_Loader>`_
	
Una vez generado el fichero SQL, puede ser cargado en |PGSQL| mediante la herramienta ``psql`` de línea de comandos o a través de `pgAdmin III <http://www.pgadmin.org/>`_.

.. note:: El cargador ``raster2pgsql`` delega en la librería |GDAL| para realizar la lectura y transformación de datos raster en sentencias SQL. 

A continuación, veremos un ejemplo de las opciones más típicas usadas a la hora de generar un fichero SQL a partir de un fichero |SHP| con ``raster2pgsql``::

	$ raster2pgsql -I -C -F -t 36x36 -M -s 4326 fichero_raster.tif > fichero_raster.sql

Detalles a destacar:
	* El flag *-C* fuerza a aplicar una serie de restricciones sobre los datos raster a cargar, para así asegurarnos de que es correctamente registrada en la vista `raster_columns`. Veremos este concepto en más profundidad en el tema de `PostGIS Raster`.
	* Al igual que con `shp2pgsql`, el flag `-I` impone la creación de un índice sobre la columna de tipo raster.
	* El flag `-F` añade a la tabla raster un campo con el nombre del fichero original. Esto es útil en el caso de que queramos cargar varios ficheros raster en una misma tabla y queramos identificar qué datos vienen de qué fichero. Es importante tener en cuenta que, caso de cargar varios ficheros raster en la misma tabla, **todos han de tener el mismo SRID**
	* El flag `-t <ancho>x<alto>` especifica un tamaño de tesela para nuestro raster. Cada tesela generada será una columna de un registro de la tabla. Veremos más en detalle el concepto de *tesela* en el tema de |PRAS|
	* Al contrario que sucedía con `shp2pgsql`, **no es posible especificar una proyección de origen y una de destino con el flag** `-s`. Los datos no serán reproyectados en el momento de la carga. No obstante, es posible reproyectar los datos una vez cargados, mediante la `función ST_Transform <http://postgis.net/docs/manual-2.0/RT_ST_Transform.html>`_. Lo veremos con más detalle en el tema de |PRAS|



Exportación de datos vectoriales
--------------------------------

Para este proceso utilizaremos la herramienta ``pgsql2shp``. Con ella podremos convertir los datos de nuestra base de datos en archivos |SHP|. Igual que para el caso anterior, la herramienta se utilizará desde la linea de comandos::

	$ pgsql2shp [<opciones>] <basedatos> [<esquema>.]<tabla>
	$ pgsql2shp [<opciones>] <basedatos> <consulta>
   
las opciones más utilizadas serán:

	* **-f <nombrearchivo>**  Especifica el nombre del archivo a crear
	* **-h <host>**  Indica el servidor donde realizará la conexión
	* **-p <puerto>**  Permite indicar el puerto de la base de datos
	* **-P <password>**  Contraseña
	* **-u <user>** Usuario
	* **-g <geometry_column>** Columna de geometría que será exportada


A continuación, veremos un ejemplo de exportación de datos vectoriales con ``pgsql2shp``::

	$ pgsql2shp -f mifichero.shp <mi_base_datos> <mi_tabla>

Con la orden anterior, crearíamos un fichero de nombre ``mifichero.shp`` a partir de la tabla ``<mi_tabla>`` existente en la base de datos ``<mi_base_de_datos>``



.. note:: No existe actualmente una herramienta equivalente a ``pgsql2shp``, para exportar datos raster desde la base de datos |PGSQL| (su nombre hipotético sería ``pgsql2raster``). Para exportar datos raster, se usa la librería |GDAL|, como veremos en el siguiente apartado

.. seealso:: Hay más maneras de exportar datos raster desde |PGSQL| sin necesidad de usar GDAL. En la `documentación online de PostGIS Raster <http://postgis.net/docs/manual-2.0/using_raster.xml.html#RT_Raster_Applications>`_ se mencionan algunos. 



Herramientas |GDAL|
===================

|GDAL| es una librería de lectura y escritura de formatos geoespaciales, tanto *raster* con GDAL como *vectorial* con OGR. Se trata de una librería de software libre ampliamente utilizada.


Importación de datos vectoriales
--------------------------------

OGR es capaz de convertir a |PGSQL| todos los formatos que maneja, y será capaz de exportar desde |PGSQL| todos aquellos en los que tiene permitida la escritura. Ejecutando::

	$ ogr2ogr --formats
	
podremos comprobar los formatos que maneja la herramienta. La étiqueta ``write`` nos indica si podemos crear este tipo de formatos. Hemos de tener en cuenta el formato de salida para poder manejar los parametros especiales de cada formato.

En la `página principal de GDAL <http://www.gdal.org/ogr2ogr.html>`_ podremos encontrar un listado de todas las opciones que nos permite manejar el comando. Detallamos a continuación algunas de las principales opciones con respecto al formato de origen:

	* **-select <lista de campos>** lista separada por comas que indica la lista de campos de la capa de origen que se quiere exportar
	* **-where <condición>** consulta a los datos de origen
	* **-sql** posibilidad de insertar una consulta más compleja
	
Otras opciones en referencia al formato de destino:

	* **-f <driver ogr>** formato del fichero de salida
	* **-lco VARIABLE=VALOR** Variables propias del driver de salida
	* **-a_srs <srid>** asigna el SRID especificado a la capa de salida
	* **-t_srs <srid>** Reproyecta la capa de salida según el SRID especificado

En `la página específica del driver de PostgreSQL/PostGIS para GDAL <http://www.gdal.org/ogr/drv_pg.html>`_  se explica cómo especificar una cadena de conexión completa, de manera que accedamos a una tabla concreta de nuestra base de datos. Hay que tener en cuenta que, si se configuró el acceso a la base de datos como se especifica en el apartado de introducción, solo será necesario especificar el nombre de la base de datos como parámetro de la cadena de conexión

Es importante destacar que, mientras los cargadores de |PGIS| generan un archivo SQL que debe ser posteriormente insertado en la base de datos, **ogr2ogr carga directamente los ficheros de origen en una tabla de PostgreSQL**, de manera que no es necesario realizar ningún paso posterior.

Adicionalmente, mientras que los cargadores de |PGIS| trabajan únicamente con el formato |SHP|, **ogr2ogr es capaz de reconocer muchos más formatos**. Basta con ejecutar, desde una línea de comandos::

	$ ogr2ogr --formats

Para ver todos los formatos soportados por |GDAL|.

Al igual que ``shp2pgsql``, **también es posible reproyectar datos con** ``ogr2ogr``. Se consigue mediante el parámetro ``-t_srs <srid_destino>``.

.. warning:: Si bien ``shp2pgsql`` acepta únicamente el identificador numérico del SRID, las herramientas de |GDAL| requieren la sintaxis ``epsg:<srid>``. 


Un ejemplo de carga de datos vectoriales en |PGIS| usando ``ogr2ogr``::
	
	$ ogr2ogr -f PostgreSQL -t_srs epsg:25830 pg:dbname=<mi_base_datos> mi_fichero.kml

En el ejemplo anterior, cabe destacar:
	* El flag ``-t_srs`` que, como ya se ha mencionado, fuerza la reproyección de los datos de entrada al srid proporcionado.
	* La construcción de una cadena de conexión con |PGSQL| requiere, como mínimo, que se especifique el nombre de la base de datos, siguiendo la sintaxis ``PG:dbname=<base_datos>``
	* Como ya se ha visto, ``ogr2ogr`` es capaz de cargar datos en diversos formatos vectoriales, no únicamente |SHP|. En el ejemplo, cargamos un fichero `KML <http://en.wikipedia.org/wiki/Keyhole_Markup_Language>`_ 


.. note:: Actualmente, no es posible cargar datos en PostGIS con la herramienta |GDAL|. De hecho **la única manera de cargar datos raster en PostGIS Raster es mediante el cargador oficial raster2pgsql**



Exportación de datos vectoriales
--------------------------------

Al igual que ``ogr2ogr`` permite cargar datos vectoriales de cualquier formato aceptado en |PGSQL|, es posible el paso opuesto: exportar datos desde |PGSQL| a cualquier formato vectorial aceptado. Únicamente tenemos que especificar como fichero de origen una cadena de conexión de |PGSQL|, y como destino, el fichero vectorial deseado. El formato se especifica con el flag *-f*.

Un ejemplo de exportación de una tabla de PostgreSQL a formato `TAB de MapInfo <http://www.gdal.org/ogr/drv_mitab.html>`_::

	$ ogr2ogr -f "Mapinfo File" mi_tabla.tab PG:"dbname<mi_base_datos>" mi_tabla

La orden anterior vuelca la tabla <mi_tabla> a disco en formato TAB de Mapinfo. No realiza ningún cambio de proyección, de manera que el fichero .tab tendrá la misma proyección que la tabla original  


.. note:: Las comillas para el nombre del formato de salida o la cadena de conexión son opcionales, salvo que haya que lidiar con espacios en blanco.

.. seealso:: En la `página de documentación del driver de PostgreSQL/PostGIS <http://www.gdal.org/ogr/drv_pg.html>`_ hay más detalles acerca de cómo interactúa OGR con |PGIS|


Exportación de datos raster
---------------------------

Actualmente, la única manera *sencilla* de exportar datos desde |PRAS|  a cualquier formato gráfico aceptado por |GDAL| es a través de las herramientas ``gdal_translate`` y ``gdalwarp``. 

La primera herramienta, ``gdal_translate``, funciona de manera análoga a ``ogr2ogr``, permitiendo pasar del formato |PRAS| a cualquier formato gráfico, especificando como cadena de origen una conexión a la base de datos. La herramienta ``gdalwarp`` permite, adicionalmente, cambiar la proyección de los datos.

Aunque el formato de la cadena de conexión con |PRAS| es muy parecido al formato de la cadena de conexión con |PGIS| (ver `Exportación de datos vectoriales`), hay algunas diferencias importantes. Concretamente:
	* En la cadena de conexión con |PRAS| es necesario especificar la tabla sobre las que operar mediante el parámetro ``table=<nombre_tabla>``, mientras que la cadena de conexión de |PGIS| no incluye esta información, siendo un parámetro separado.
	* La cadena de conexión de |PGIS| incluye el parámetro ``mode=<modo>``, que puede tomar los valores 1 (considera cada fila de la tabla un raster separado) y 2 (considera toda la tabla como una cobertura raster completa). Por defecto toma el valor 1, así que si queremos leer nuestra tabla como un solo raster, hemos de especificar explícitamente ``mode=2`` 
	* Es posible especificar un grupo de filas de la tabla que queremos exportar, de manera que lo que exportamos es una porción del raster, no el raster completo. Para ello, además del parámetro ``mode=2``, podemos añadir un nuevo parámetro a la cadena, con la forma ``where=<sql_where>``, donde ``<sql_where>`` representa cualquier expresión aceptada por |PGSQL| como clausula *where* de una consulta.

Veamos unos ejemplos, para apreciar más claramente estas diferencias

La siguiente instrucción vuelca una tabla de |PRAS| a un fichero en formato PNG en disco::

	$ gdal_translate -of PNG PG:"dbname=<mi_base_datos> mode=2" mi_fichero.png

Esta instrucción vuelca  una tabla de |PRAS| a un fichero en formato TIFF en disco (si no especificamos formato, es el formato por defecto). Además, reproyecta los datos originales a la `proyección EPSG:23030 <http://spatialreference.org/ref/epsg/23030/>`_::

	$ gdalwarp -t_srs epsg:23030 PG:"dbname=<mi_base_de_datos> mode=2" mi_fichero.tif

Esta instrucción vuelca todas las filas de una tabla con el campo ``rid`` mayor que 165 a formato JPEG::

	$ gdal_translate -of JPEG PG:"dbname=<mi_base_de_datos> table=<mi_tabla> mode=2 where='rid > 165'" mi_fichero.jpg

.. warning:: Es necesario incluir comillas para contener la clausula ``where``

Por último, esta instrucción nos informa de todos los subdatasets que contiene el dataset representado por nuestra tabla, que es una consecuencia directa de usar ``mode=1`` cuando nos referimos a una tabla |PRAS| (recordemos que, si no especificamos parámetro ``mode``, éste es el modo de funcionamiento por defecto)::

	$ gdalinfo PG:"dbname=<mi_base_de_datos> table=<mi_tabla>"


Algunos formatos gráficos pueden actuar como contenedores, conteniendo más de una cobertura raster (*dataset*, en terminología de |GDAL|). En esos casos, es posible acceder por separado a cada una de las coberturas contenidas en el contenedor. |PRAS| es uno de estos formatos. Por ello, salvo que se especifique lo contrario mediante el parámetro ``mode=2``, una tabla de |PRAS| es un contenedor de varias coberturas raster. Cada fila de la tabla es una de estas coberturas.


.. seealso:: En la `documentación sobre el modelo de datos de GDAL <http://www.gdal.org/gdal_datamodel.html>`_ se habla más en profundidad de los formatos que aceptan subdatasets.


Para más información, se pueden consultar la `página de gdal_translate <http://www.gdal.org/gdal_translate.html>`_  y la de `gdalwarp <http://www.gdal.org/gdalwarp.html>`_. Para saber cómo especificar una cadena de conexión con |PRAS|, consultar la `página específica del driver <http://trac.osgeo.org/gdal/wiki/frmts_wtkraster.html>`_

.. warning:: Hay una pequeña inconsistencia en cuanto al orden en el que se pasan los parámetros a las herramientas de la parte raster de |GDAL| y la parte vectorial. Mientras que ``ogr2ogr`` requiere primero el fichero de destino y después el de origen, ``gdal_translate`` y ``gdalwarp`` lo hacen al contrario.


Plugin SPIT de QGIS
====================

Veremos la herramienta de escritorio QGIS en profundidad más adelante. Por ahora, simplemente nos detendremos en la funcionalidad de carga de datos en |PGSQL| mediante el plugin `SPIT <http://www.qgis.org/en/docs/user_manual/plugins/plugins_spit.html>`_

Para instalar el plugin, tendremos que acceder al menú de gestión de plugins de QGIS, en *Plugins*, *Manage plugins*. En la captura se observa dónde se encuentra dicha opción

	.. image:: _images/qgis_gestion_plugins1.png
		:scale: 50%

Una vez accedemos a dicho menú, podemos navegar por la lista de plugins disponibles, como observamos en la siguiente captura

	.. image:: _images/qgis_gestion_plugins2.png
		:scale: 50%

Buscamos el plugin de SPIT, lo seleccionamos, y pulsamos en *OK*. 


	.. image:: _images/qgis_instalar_spit1.png
		:scale: 50%

Con esto ya tendremos disponible el plugin SPIT, listo para cargar datos

	.. image:: _images/qgis_instalar_spit2.png
		:scale: 50%


.. warning:: Al igual que ``shp2pgsql``, SPIT solo es capaz de importar datos de tipo |SHP|



Cargador de datos |OSM|
=========================

Por último, veremos cómo cargar datos de |OSM| En |PGIS|. OpenStreetMaps (abreviado como OSM) es un proyecto colaborativo para crear mapas libres y editables.

Los mapas se crean utilizando información geográfica capturada con dispositivos GPS móviles, ortofotografías y otras fuentes libres. Esta cartografía, tanto las imágenes creadas como los datos vectoriales almacenados en su base de datos, se distribuye bajo licencia abierta Open Database Licence (ODbL).

OSM dispone de un modelo de datos particular que no responde al modelo característico de los SIG. Este está compuesto de:

	* Node
	* Way
	* Relation

a diferencia de las geometrías características como:

	* Punto
	* Linea
	* Poligono
	
una característica particular es la ausencia de polígonos dentro del modelo, estos se realizan mediante la asignación de una relación a una linea cerrada. Esta particularidad no impide que los datos de OSM puedan ser adaptados al modelo de geometrías normal mediante cargadores de datos OSM. A continuación se presentan dos de los más utilizados


Obtener datos de |OSM|
----------------------

Si queremos obtener datos de |OSM| para utilizarlos en nuestras aplicaciones, podemos dirigirnos a `http://www.openstreetmap.org/export <http://www.openstreetmap.org/export>`_. En dicha página, veremos que se nos presenta un mapa y las coordenadas lat, lon de la zona representada, junto con un botón de *Exportar* listo para obtener esos datos. Adicionalmente, se nos permite seleccionar a mano una zona diferente. En la siguiente captura podemos observar estas funcionalidades:

	.. image::  _images/osm_export1.png
		:scale: 50%

Si estamos interesados en una zona diferente a la que aparece en el mapa, podemos lanzar una búsqueda mediante la caja destinada a tal efecto en el lado izquierdo de la pantalla. En la captura se observa:

	.. image::  _images/osm_export2.png

Una vez tenemos nuestra zona de interés seleccionada, podemos exportarla mediante el botón de *Exportar*. Si la zona en cuestión es demasiado grande, se nos redireccionará a una página con enlace a sitios de descarga masiva de datos. Uno de estos sitios es `http://download.geofabrik.de/ <http://download.geofabrik.de/>`_. 

El fichero descargado estará en formato .osm. Para poder importar dicho formato a |PGIS|, utilizaremos el cargador ``osm2pgsql``. Pero antes de eso, vamos a activar en |PGSQL| la extensión *hstore*. Con esta extensión, podremos almacenar en una columna un dato de tipo *clave => valor*. Eso nos permitirá usar etiquetas en las consultas que realicemos. Como por ejemplo::

	$ SELECT way, tags FROM planet_osm_polygon WHERE (tags -> 'landcover') = 'trees';

.. seealso:: Para tener más información, ir a `http://wiki.openstreetmap.org/wiki/Osm2pgsql#hstore <http://wiki.openstreetmap.org/wiki/Osm2pgsql#hstore>`_


Veamos a continuación el uso de la herramienta ``osm2pgsql``


Importación de datos |OSM|
--------------------------
Mediante el uso de este programa podremos incorporar en nuestra base de datos los datos obtenidos desde OSM. Una vez que hemos realizado la importación, aparecerán en nuestra base de datos las tablas que serán el resultado de esta importación:

	* *planet_osm_point*
	* *planet_osm_line*
	* *planet_osm_polygon*
	* *planet_osm_roads*
	
Al disponer el modelo de OSM de cientos de etiquetas, la importación crea en las tablas un gran número de campos de los que la mayoría tendrán valor NULL.

La ejecución se realiza desde la consola::

	$ osm2pgsql [opciones] ruta_fichero.osm otro_fichero.osm
	$ osm2pgsql [opciones] ruta_planet.[gz, bz2]
	
algunas de las opciones se detallan a continuación:

	* *-H* Servidor |PGSQL|
	* *-P <puerto>* Puerto
	* *-U <usuario>* Usuario
	* *-W* pregunta la password del usuario
	* *-d <base_de_datos>* base de datos de destino
	* *-a* añade datos a las tablas importadas anteriormente
	* *-l* almacena las coordenadas en latitud/longitug en lugar de Spherical Mercator
	* *-s* utiliza tablas secundarias para la importación en lugar de hacerlo en memoria
	* *-S <fichero_de_estilos>* ruta al fichero que indica las etiquetas de OSM que se quiere importar
	* *-v* modo verborrea, muestra la salida de las operaciones por consola

El siguiente comando cargaría *mifichero.osm* en |PGIS|. Las tablas generadas, como ya se ha dicho, serían *planet_osm_point*, *planet_osm_line*, *planet_osm_polygon* y *planet_osm_roads*::
	$ osm2pgsql -d <mi_base_datos> --hstore mifichero.osm


Cargar datos no espaciales en |PGIS|
====================================

En ocasiones, queremos trabajar con datos de naturaleza no espacial, agregándoles nosotros esa componente espacial que les falta. Un ejemplo típico son datos tabulados en el que dos de sus columnas son coordenadas de latitud y longitud. Vamos a ver una manera de cargar esos datos en |PGIS| para poder trabajar con ellos, utilizando las posibilidades de |GDAL|.

Los datos de partida que vamos a cargar en |PGIS| son datos en formato CSV. En concreto, el fichero *otros/csv/incendios.csv*, que encontramos en nuestra carpeta de datos. El enlace a la carpeta de datos se encuentra más abajo, en la sección de ejercicios.

.. seealso:: `Más <http://en.wikipedia.org/wiki/Comma-separated_values>`_ sobre el formato CSV

Lo que vamos a hacer es crear un **fichero VRT**, reconocido por |GDAL|, para poder cargar nuestros datos mediante la herramienta ``ogr2ogr``. El formato VRT está basado en XML, y permite crear datasets a partir de otros datasets, únicamente indicando de dónde y cómo se tienen que leer los datos. Para nuestro ejemplo, el fichero VRT a generar contendrá lo siguiente::
	
	<OGRVRTDataSource>
		<OGRVRTLayer name="terremotos">
			<SrcDataSource>terremotos.csv</SrcDataSource>
			<GeometryType>wkbPoint</GeometryType>
			<LayerSRS>EPSG:4326</LayerSRS>
			<GeometryField encoding="PointFromColumns" x="longitude" y="latitude"/>
		</OGRVRTLayer>
	</OGRVRTDataSource>

Guardamos el fichero con el nombre *terremotos.vrt*. Hemos de guardarlo **en el mismo directorio que nuestro fichero terremotos.csv**. 

Los campos del fichero son bastante auto-explicativos, pero se requieren unos mínimos conocimientos sobre el `modelo de datos OGR <http://www.gdal.org/ogr/ogr_arch.html>`_. La línea más importante es::

	<GeometryField encoding="PointFromColumns" x="longitude" y="latitude"/>

Donde se especifica que se creará un campo geométrico de tipo punto a partir de las columnas *longitude* y *latitude* del fichero CSV.

.. seealso:: `Tutorial del formato VRT <http://www.gdal.org/gdal_vrttut.html>`_

Una vez tenemos nuestro fichero VRT, simplemente ejecutamos ``ogr2ogr`` de manera normal, especificando este fichero como origen. Usamos la base de datos *workshop_sevilla*, creada en la introducción::
	
	$ ogr2ogr -a_srs epsg:4326 -f "PostgreSQL" PG:"dbname=workshop_sevilla" terremotos.vrt

Vemos que hemos especificado la opción `-a_srs`. Con este flag simplemente asignamos una proyección a los datos de salida, pero **no se realiza ninguna reproyección**. No es necesario, puesto que ya estamos diciendo en el VRT que se creen los puntos como objetos geométricos con SRID 4326.
	
Una vez cargado el fichero, podemos ver en cualquier visor de escritorio su aspecto. En la captura, vemos el fichero cargado desde QGIS. Veremos más sobre los clientes de escritorio en el último tema.

	.. image::  _images/terremotos_qgis.png

Si bien éste método es muy cómodo para importar ficheros CSV en |PGIS|, no es la única alternativa. Otro camino, algo más largo, es copiar el fichero CSV directamente en |PGSQL| mediante la instrucción *COPY*, generando una tabla no espacial. Posteriomente, añadimos a mano el campo espacial a dicha tabla. 

.. seealso:: La documentación del comando `COPY de PostgreSQL 9 <http://www.postgresql.org/docs/9.1/static/sql-copy.html>`_ 

Ejercicios
==========

A continuación, los ejercicios a realizar:

Ejercicio 1
-----------

Cargar con ``shp2pgsql`` los siguientes datos (todos con encoding ``LATIN1``):

		* *vectorial/shp/Colombia/barrios_de_bogota.shp
		* *vectorial/shp/Colombia/railways.shp
		* *vectorial/shp/Colombia/waterways.shp
		* *vectorial/shp/Colombia/points.shp
		* *vectorial/shp/Sevilla/CODIGO_POSTAL.shp*: Transformándolo a SRID 25830 (primero tenemos que conocer el SRID de origen)
		* *vectorial/shp/Madrid/BCN200_0101S_LIM_ADM.shp*: Transformándolo también a SRID 25830
		* *vectorial/shp/Toledo/BCN200_0101S_LIM_ADM.shp*: En la misma tabla que el fichero anterior (investigar qué parámetros hacen falta para conseguirlo). Transformándolo también a SRID 25830


Ejercicio 2
-----------

Cargar con ``ogr2ogr`` los siguientes datos:

		* *vectorial/shp/Sevilla/TOPONIMO.shp*: Transformándolo a SRID 25830
		* *vectorial/kml/noticias_incendios.kml*: Asignarle (ojo, no es lo mismo que reproyectar) el SRID 4326
		* *vectorial/shp/España/centroides_territorios_etrs89.shp*: Transformar la proyección a SRID 25830
		* *vectorial/shp/TM_WORLD_BORDERS/TM_WORLD_BORDERS.shp*: **OJO**, es posible que sea necesario especificar explícitamente el tipo de geometría para la capa destino, dado que la capa origen mezcla diferentes tipos. Investigar las `opciones de ogr2ogr para conseguirlo <http://www.gdal.org/ogr2ogr.html>`_.


Ejercicio 3
-----------

Cargar el fichero *csv/incendios.csv* mediante el uso del comando *COPY*. Investigar para ello el uso de las opciones *FORMAT* y *DELIMITER* de *COPY*. Tras copiar el fichero, añadir a la tabla un campo entero autoincrementable (pista: *BIGSERIAL*) y un campo geométrico de tipo punto, asignándole a la tabla el SRID 4326 (pista: investigar las funciones `ST_SetSRID <http://postgis.net/docs/manual-2.0/ST_SetSRID.html>`_ y `ST_MakePoint <http://postgis.net/docs/manual-2.0/ST_MakePoint.html>`_). Por último, añadir un índice espacial de tipo GiST a la columna geométrica. 


Ejercicio 4
-----------

Cargar con ``ogr2ogr`` el fichero *vectorial/gpx/traza1.gpx* pero creando previamente la tabla a mano. Para ello, investigar los flags *-append* y *-update* de `ogr2ogr <http://www.gdal.org/ogr2ogr.html>`_. Del fichero GPX, nos van a interesar solo el campo geométrico y los campos *ele* y *time* (pista: investigar el uso del flag *-sql*, y ejecutar una consulta SQL sobre el fichero, obteniendo solo esos dos campos). La tabla donde se cargará el fichero tendrá la siguiente estructura::

		CREATE TABLE gps_track_points
		(
			fid serial NOT NULL,
			the_geom geometry(Point,25830),
			ele double precision,
			"time" timestamp with time zone,
			CONSTRAINT activities_pk PRIMARY KEY (fid)
		);



Ejercicio 5
-----------

Cargar con ``osm2pgsql`` el fichero *vectorial/osm/sevilla.osm*: Asegurarse de que se carga con srid 4326, y no con 900913




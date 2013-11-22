.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


**********************************
Importación y exportación de datos
**********************************
En este tema nos introduciremos en el uso de herramientas de importación/exportación de datos hasta/desde |PGIS|. Veremos 4 tipos de herramientas:
	* Herramientas |PGSQL| y |PGIS|: ``shp2pgsql``, ``pgsql2shp``, |PGA|, ``psql``
	* Herramientas |GDAL|: ``ogr2ogr``, ``gdal_translate``, ``gdalwarp`` 
	* Plugin SPIT de QGIS
	* Cargador de datos |OSM|: ``osm2pgsql``

Finalmente, veremos una serie de ejercicios prácticos, que servirán para fijar los conocimientos adquiridos.

Herramientas |PGSQL| y |PGIS|
==========================================

Carga de datos vectoriales
^^^^^^^^^^^^^^^^^^^^^^^^^^

El cargador ``shp2pgsql`` convierte archivos |SHP| en SQL preparado para la inserción en la base de datos. Se utiliza desde la linea de comandos, aunque existe una versión con interfaz gráfica para el sistema operativo Windows. Se puede acceder a la ayuda de la herramienta mediante::

	$ shp2pgsql -?
	
Para el uso de la herramienta::

	$ shp2pgsql [<opciones>] <ruta_shapefile> [<esquema>.]<tabla>
	
Entre las opciones encontraremos:

	* **-s <srid>**  Asigna el sistema de coordenadas. Por defecto será -1
	* **(-d|a|c|p)**
		* **-d**  Elimina la tabla, la recrea y la llena con los datos del shape
		* **-a**  Llena la tabla con los datos del shape. Debe tener el mismo esquema exactamente
		* **-c**  Crea una nueva tabla y la llena con los datos. opción por defecto.
		* **-p**  Modo preparar, solo crea la tabla
	* **-g <geocolumn>** Especifica el nombre de la columna geometría (usada habitualmente en modo *-a*)
	* **-D** Usa el formato Dump de postgresql
	* **-G** Usa tipo geogrfía, requiere datos de longitud y latitud
	* **-k** Guarda los identificadores en postgresql
	* **-i** Usa int4 para todos los campos integer del dbf
	* **-I** Crea un índice spacial en la columna de la geometría
	* **-S** Genera geometrías simples en vez de geometrías MULTI
	* **-w** Salida en WKT
	* **-W <encoding>** Especifica la codificación de los caracteres. (por defecto : "WINDOWS-1252").
	* **-N <policy>** estrategia de manejo de geometrías NULL (insert*,skip,abort).
	* **-n**  Solo importa el archivo DBF
	* **-?**  Muestra la ayuda

Una vez generado el fichero SQL, puede ser cargado en |PGSQL| mediante la herramienta ``psql`` de línea de comandos o a través de `pgAdmin III <http://www.pgadmin.org/>`_, una herramienta gráfica de administración y desarrollo para Windows, Mac y |LX| 


Carga de datos raster
^^^^^^^^^^^^^^^^^^^^^

Paralelamente al cargador oficial de |PGIS| para datos vectoriales, la versión 2.0 de la librería incluye también un cargador para datos ráster. Se trata de ``raster2pgsql``. KLa ayuda de la herramienta está disponible mediante::
	
	$ raster2pgsql -?

Y su uso básico es::

	$ raster2pgsql [<opciones>] <ruta_raster> [<esquema>.]<tabla>

Entre las opciones encontraremos:

	* **(-d|a|c|p)**
		* **-d**  Elimina la tabla, la recrea y la llena con los datos del raster
		* **-a**  Llena la tabla con los datos del raster. Debe tener el mismo esquema exactamente
		* **-c**  Crea una nueva tabla y la llena con los datos. opción por defecto.
		* **-p**  Modo preparar, solo crea la tabla
	* **Opciones de procesado raster: aplicación de restricciones**
		* **-C**	Aplica restricciones necesarias al raster (srid, tamaño de pixel, etc) para asegurarse de que se registra correctamente en ``raster_columns``. 
		* **-x**	Desactiva la restricción de extensión máxima. Solo se aplica si también se especifica -C
		* **-r**	Establece el parámetro *regular_blocking* a *True*.
	* **Opciones de procesado raster: parámetros opcionales de manipulación de datos de entrada**
		* **-s <SRID>**	Asigna el sistema de coordenadas. Por defecto será -1
		* **-b <BAND>** Índice de banda a extraer (empezando por 1). Si se quieren varias, se pueden separar por comas. Por defecto, se extraen todas las bandas.
		* **-t <TILE_WIDTH>X<TILE_HEIGHT>** Indica que se tesele el raster usando este tamaño de tesela. Cada tesela será una fila de la tabla
		* **-R, --register** Únicamente almacena los metadatos del raster en |PRAS|. El raster en si queda registrado como fichero en disco.
		* **-l <OVERVIEW_FACTOR>** Construye pirámides para el raster de entrada (versiones del mismo a menor escala). Esto consiste en una tabla con el mismo nombre que la original pero precedida por ``o_<OVERVIEW_FACTOR>_<TABLE>``, donde <OVERVIEW_FACTOR> es el factor de escala y <TABLE> el nombre de la tabla original. Se pueden especificar varios factores de escala, separados por comas. El comportamiento de esta opción es similar al de la instrucción ``gdaladdo`` de GDAL.
		* **-N <NODATA>** Valor de NODATA para las bandas que no tengan ya especificado un valor de NODATA.
	* **Parámetros opcionales para manipular objetos en la base de datos**
		* **-q** Escapa los identificadores de |PGSQL|
		* **-f <COLUMN>** Especifica un nombre para la columna de tipo raster. Por defecto es ``rast``
		* **-I** Crea un índice de tipo GiST sobre la columna de tipo raster (o más bien sobre su ``st_convexhull``)
		* **-M** Ejecuta VACUUM ANALYZE sobre la tabla creada
		* **-T <TABLESPACE>** Especifica el espacio donde se creará la tabla. Los índices usarán el espacio por defecto, a menos que se especifique la siguiente opción.
		* **-X <TABLESPACE>** Espacio donde crear los índices.
		* **-Y** Usa sentencias ``COPY`` en vez de ``INSERT``
	* **-e** Ejecuta cada sentencia de manera individual, en lugar de dentro de una transacción


Exportación de datos vectoriales
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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


.. warning:: No existe actualmente una herramienta equivalente a ``pgsql2shp``, para exportar datos raster desde la base de datos |PGSQL| (su nombre hipotético sería ``pgsql2raster``). Para exportar datos raster, se usa la librería |GDAL|, como veremos en el siguiente apartado



Herramientas |GDAL|
==========================================

|GDAL| es una librería de lectura y escritura de formatos geoespaciales, tanto *raster* con GDAL como *vectorial* con OGR. Se trata de una librería de software libre ampliamente utilizada.


Carga de datos vectoriales
^^^^^^^^^^^^^^^^^^^^^^^^^^
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

En `la página específica del driver de PostgreSQL/PostGIS para GDAL <http://www.gdal.org/ogr/drv_pg.html>`_  se explica cómo especificar una cadena de conexión completa, de manera que accedamos a una tabla concreta de nuestra base de datos.

Es importante destacar que, mientras los cargadores de |PGIS| generan un archivo SQL que debe ser posteriormente insertado en la base de datos, **ogr2ogr carga directamente los ficheros de origen en una tabla de PostgreSQL**, de manera que no es necesario realizar ningún paso posterior.

Adicionalmente, mientras que los cargadores de |PGIS| trabajan únicamente con el formato |SHP|, **ogr2ogr es capaz de reconocer muchos más formatos**. Basta con ejecutar, desde una línea de comandos::

	$ ogr2ogr --formats

Para ver todos los formatos soportados por |GDAL|. Lo veremos más adelante, cuando carguemos en |PGIS| un fichero `CSV <http://en.wikipedia.org/wiki/Comma-separated_values>`_ y un fichero `KML <http://en.wikipedia.org/wiki/Keyhole_Markup_Language>`_.

.. warning:: Actualmente, no es posible cargar datos en PostGIS con la herramienta |GDAL|. De hecho **la única manera de cargar datos raster en PostGIS Raster es mediante el cargador oficial raster2pgsql**



Exportación de datos vectoriales
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Al igual que ``ogr2ogr`` permite cargar datos vectoriales de cualquier formato aceptado en |PGSQL|, es posible el paso opuesto: exportar datos desde |PGSQL| a cualquier formato vectorial aceptado.

Únicamente tenemos que especificar como fichero de origen una cadena de conexión de |PGSQL|, y como destino, el fichero vectorial deseado. El formato se especifica con el flag *-f*.


Exportación de datos raster
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Actualmente, la única manera *sencilla* de exportar datos desde |PRAS| a cualquier formato gráfico aceptado por |GDAL| es a través de las herramientas ``gdal_translate`` y ``gdalwarp``. 

La primera herramienta, ``gdal_translate``, funciona de manera análoga a ``ogr2ogr``, permitiendo pasar de cualquier formato gráfico a |PRAS|, especificando como cadena de destino una conexión a la base de datos. La herramienta ``gdalwarp`` permite, adicionalmente, cambiar la proyección de los datos.

Para más información, se pueden consultar la `página de gdal_translate <http://www.gdal.org/gdal_translate.html>`_  y la de `gdalwarp <http://www.gdal.org/gdalwarp.html>`_. Para saber cómo especificar una cadena de conexión con |PRAS|, consultar la `página específica del driver <http://trac.osgeo.org/gdal/wiki/frmts_wtkraster.html>`_


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

osm2pgsql
^^^^^^^^^
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

En caso de no disponer del SRID 900913 en nuestro |PGSQL| podremos utilizar la definición que hay de él en ``osm2pgsql``. Simplemente ejecutaremos el script ``900913.sql``



Ejercicios
==========

	* Crear una base de datos para el workshop, junto con un usuario
	* Importar datos shp con shp2pgsql
	* Importar datos shp con pgAdmin III
	* Importar datos KML con SPIT
	* Importar datos CSV con ogr2ogr (creando tabla y usando VRT)
	* Importar datos OSM con osm2pgsql
	* Exportar datos vectoriales con pgsql2shp
	* Exportar datos raster con gdal_translate
	* Exportar y reproyectar datos raster con gdalwarp

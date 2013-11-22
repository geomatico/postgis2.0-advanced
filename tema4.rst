.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


PostGIS y el escritorio
**********************************
En este tema veremos cómo interactúan los clientes GIS de escritorio más extendidos con una base de datos PostGIS. Concretamente, veremos:
	* `QGIS <http://qgis.org/es/site/>`_
	* `OpenJUMP <http://www.openjump.org/>`_
	* `gvSIG desktop <http://www.gvsig.org/web/>`_ y `gvSIG CE <http://www.gvsigce.org>`_


QGIS
====

El cliente de escritorio QGIS, conocido como Quantum GIS hasta la versión 1.8, está ampliamente difundido y contiene una gran cantidad de funcionalidades. Entre ellas, por supuesto, está la conexión con una base de datos |PGIS|.

Para conectar QGIS con |PGIS| únicamente tendremos que pulsar en el botón azul con forma de base de datos en la barra de botones de QGIS. En la siguiente captura se aprecia el aspecto de dicho botón:

	.. image:: _images/qgis_add_postgis_conn.png

Una vez pulsamos el botón, nos aparecerá la pantalla para realizar una nueva conexión con |PGIS|. Pulsamos en el botón *Nueva*:

	.. image:: _images/qgis_new-postgis_conn.png

Rellenamos los parámetros correspondientes, y pulsamos *Aceptar*:

	.. image:: _images/qgis_new-postgis_conn2.png

Ya tenemos nuestra conexión creda. Podemos ver las tablas existentes en nuestra base de datos. Elegimos la tabla que queremos cargar, y le damos al botón *Añadir*. 

	.. image:: _images/qgis_new-postgis_conn3.png

Eso cargará nuestra tabla en la vista actual:

	.. image:: _images/qgis_pgis_table_loaded.png
		:scale: 50%

El gestor de conexiones también nos permite ejecutar consultas sobre las tablas, si no queremos cargar todos los datos. Basta con pulsar doble click sobre el nombre de cualquiera de las tablas existentes y se nos abrirá el constructor de consultas.

	.. image:: _images/qgis_build_pgis_query.png

En el panel de la izquierda, vemos los campos de nuestra tabla. Al elegir uno de ellos, podemos cargar todos los posibles valores que ese campo tiene en nuestra tabla pulsando el botón *Todos*, debajo del panel derecho. Eso nos ayudará a construir la claúsula *where* en el panel inferior, usando también la barra de botones que tiene justo encima.

En la siguiente captura, vemos la consulta que permitirá cargar solo el polígono con el valor *8* en el campo *gid*
	
	.. image:: _images/qgis_build_pgis_query2.png

Como resultado, vemos cargado únicamente ese polígono:

	.. image:: _images/qgis_build_pgis_query3.png


Además del gestor de conexiones, QGIS dispone de otra herramienta muy útil para interactuar con una base de datos |PGIS|. Se trata del gestor de conexiones, o en inglés, **DB Manager**.

Este plugin es accesible a través del siguiente icono

	.. image:: _images/qgis_dbmanager_icon.png

Pulsando en dicho icono, se abre el gestor. Su funcionamiento es muy parecido al del cliente de escritorio *pgAdmin III*, con la diferencia de que *DB Manager* es capaz de manejar tanto conexiones con bases de datos |PGIS| como con bases de datos SQLite, y que no dispone de tantas funcionalidades como *pgAdmin III*. En la captura, vemos como muestra las tablas de nuestra base de datos:
	
	.. image:: _images/qgis_dbmanager1.png

Al hacer doble clic en una cualquiera de las tablas, se nos carga en el panel de la derecha la información de la misma, dividida en 3 pestañas. En la captura, vemos la primera de ellas, con información general de la tabla.

	.. image:: _images/qgis_dbmanager2.png

La segunda pestaña contiene los datos de la tabla en si.

	.. image:: _images/qgis_dbmanager3.png

Y la última pestaña contiene una previsualización de los datos de la tabla, tal y como se verían en la vista normal

	.. image:: _images/qgis_dbmanager4.png

QGIS es capaz de operar con los datos de la tabla igual que si fueras ficheros vectoriales de cualquier tipo. Por ejemplo, permite seleccionar geometrías de la tabla de manera individual, extraerlas y almacenarlas en formato |SHP|. Para ello, primero hemos de elegir la herramienta de selección de polígonos individuales. Se encuentra en la barra de botones y tiene el aspecto que se muestra en la figura.

	.. image:: _images/qgis_seleccionar_poligono.png

Con dicha herramienta seleccionada, basta con que pulsemos en cualquiera de los polígonos de nuestra capa, y queda seleccionado, como podemos ver en la siguiente captura

	.. image:: _images/qgis_seleccionar_poligono2.png

En el menú *Capa* podemos elegir exportar nuestra selección como capa individual. La opción se resalta en la captura siguiente

	.. image:: _images/qgis_seleccionar_poligono3.png

Tras elegir esa opción, deberemos especificar los parámetros de nuestra capa

	.. image:: _images/qgis_seleccionar_poligono4.png

Por último, podemos exportar nuestro polígono como |SHP| individual y cargarlo de manera independiente

	.. image:: _images/qgis_seleccionar_poligono5.png

Además de extraer información de nuestra capa |PGIS|, también podemos transformarla. En el siguiente ejemplo, extraeremos el borde de cada polígono. y almacenaremos el resultado como geometrías individuales. Para ello, elegimos la opción apropiada del menú

	.. image:: _images/qgis_pol_to_lines.png

Elegimos la capa de entrada y ponemos un nombre a la capa de salida

	.. image:: _images/qgis_pol_to_lines2.png

El resultado es como se ve a continuación

	.. image:: _images/qgis_pol_to_lines3.png



OpenJUMP
========

OpenJUMP es un cliente de escritorio escrito en Java, y con un grado de madurez bastante alto. Al igual que QGIS, permite conexión sencilla con una base de datos |PGIS|. Para ello, solo tenemos que elegir la opción de *Abrir* y seleccionar *Capa de base de datos*, como se ve en la captura

	.. image:: _images/openjump1.png

Pulsando en el pequeño icono que hay a la derecha del primer desplegable, podemos crear una nueva conexión. Lo rellenamos con los datos de nuestra base de datos

	.. image:: _images/openjump2.png

Una vez creada la conexión, ya podemos pulsar en *OK*

	.. image:: _images/openjump3.png

Después de eso, la conexión está lista para funcionar. Se nos mostrará en un desplegable nuestra lista de tablas, y para cada tabla, la geometría que contiene

	.. image:: _images/openjump4.png

Al pulsar en *OK*, ya tenemos nuestra capa cargada

	.. image:: _images/openjump5.png

Al igual que QGIS, OpenJUMP nos permite ejecutar consultas contra nuestra base de datos. La opción es incluso más accesible que en QGIS, en el menú *Archivo*.

	.. image:: _images/openjump_runquery1.png

Pulsando en esa opción, se nos abre una ventana donde podemos escribir una consulta SQL. No tenemos un asistente con botones, como sucedía en QGIS

	.. image:: _images/openjump_runquery2.png

En la consulta, queremos obtener los barrios de Bogotá con más de 50.000 habitantes. Tras ejecutar la consulta, vemos el resultado

	.. image:: _images/openjump_runquery3.png

También como en QGIS, OpenJUMP nos permite seleccionar polígonos individuales o grupos de polígonos. En la captura, se ve el botón de la barra de herramientas a pulsar, y el botón de selección de polígonos dentro de dicha barra de herramientas

	.. image:: _images/openjump_seleccionar_poligono1.png

No solo podemos seleccionar un polígono individual. También se nos permite seleccionar varios usando un rectángulo. En la captura, varios polígonos seleccionados a la vez

	.. image:: _images/openjump_seleccionar_poligono2.png

Con OpenJUMP es igualmente sencillo realizar operaciones sobre las geometrías de la base de datos, igual que si fueran ficheros |SHP|. De hecho, la herramienta Sextante también funciona en OpenJUMP, al igual que existe en QGIS. En la captura, observamos la herramienta de cálculo de centroides en polígonos..

	.. image:: _images/openjump_funciones_geometria1.png

El aspecto de nuestra vista tras ejecutar la herramienta

	.. image:: _images/openjump_funciones_geometria2.png

También tenemos una herramienta de cálculo de distancias, como se puede apreciar

	.. image:: _images/openjump_medida.png


gvSIG y gvSIG CE
================

El cliente de escritorio gvSIG es un proyecto desarrollado por el gobierno local de la Generalitat Valenciana. Actualmente, la última versión estable es la 2.0. Dicha herramienta coexiste con gvSIG CE, un fork de *gvSIG* desarrollado y mantenido por una comunidad. El proyecto gvSIG CE no está relacionado con la asociación gvSIG, ni recibe soporte oficial. 

Ambos proyectos llevan su desarrollo por separado. Aun no existe una versión 1.0 estable de gvSIG CE. Esta versión 1.0 estará integrada con los productos `Sextante <http://www.sextantegis.com/>`_, `GRASS GIS <http://grass.osgeo.org/>`_ y `SAGA <http://www.saga-gis.org/en/index.html>`_

En cualquier caso, tanto *gvSIG* como *gvSIG CE* muestran una incompatibilidad con PostGIS 2.0. En el caso de gvSIG, se produce una excepción al intentar crear una conexión con una base de datos |PGIS|, y ni siquiera llega a mostrar las tablas existentes. En cuanto a gvSIG CE, sí llega a mostrar las tablas, pero lanza un error al intentar cerrar la ventana de conexión.

Parece que la solución al problema pasa por cargar en nuestra base de datos el fichero *legacy.sql* [1]_. Dicho fichero se encuentra, en nuestra máquina, en el directorio */usr/share/postgresql/9.1/contrib/postgis-2.0/*. Podemos instalarlo con::

	$ psql -d workshop_sevilla -f /usr/share/postgresql/9.1/contrib/postgis-2.0/legacy.sql

Hecho eso, gvSIG CE funciona sin problemas. Pero gvSIG sigue lanzando el mismo error. No son descartables problemas de configuración de la máquina local. En cualquier caso, dado el gran parecido entre ambos programas, vamos a utilizar gvSIG CE.

Al abrir gvSIG CE, vemos una interfaz que nos recuerda a OpenJUMP.

	.. image:: _images/gvsigce.png
		:scale: 50%

Lo primero que hemos de hacer es crear una vista
	
	.. image:: _images/gvsigce2.png

Hecho eso, podemos proceder a crear una conexión con la base de datos, como hemos hecho en los casos anteriores. Pulsamos en el botón de la barra de herramientas que nos permite añadir una capa. En la captura, se ve resaltado, junto con la pestaña que nos permitirá crear la conexión en la nueva ventana

	.. image:: _images/gvsigce_adddblayer1.png
		:scale: 50%

Elegimos el driver para realizar nuestra conexión. Al estar gvSIG CE construido en Java, utilizaremos JDBC. 

	.. image:: _images/gvsigce_adddblayer2.png
		:scale: 50%

A continuación, rellenamos los campos con los parámetros de nuestra conexión
	
	.. image:: _images/gvsigce_adddblayer2.png
		:scale: 50%

Lo siguiente que veremos serán las tablas cargadas. Vamos a elegir una

	.. image:: _images/gvsigce_adddblayer4.png
		:scale: 50%

Y así es como luce nuestra tabla cargada en la vista activa

	.. image:: _images/gvsigce_adddblayer5.png
		:scale: 50%

Al igual que con QGIS y OpenJUMP, gvSIG CE tiene acceso a SEXTANTE. De hecho, es un componente básico de su arquitectura. En la captura, vemos el menú de SEXTANTE

	.. image:: _images/gvsigce_sextante1.png
		:scale: 50%

Tenemos una gran cantidad de herramientas para operar sobre polígonos

	.. image:: _images/gvsigce_sextante2.png
		:scale: 50%

Al igual que hicimos en QGIS, podemos obtener las líneas que suponen el borde de nuestros polígonos

	.. image:: _images/gvsigce_sextante3.png
		:scale: 50%

Así queda el resultado

	.. image:: _images/gvsigce_sextante4.png
		:scale: 50%

Otra operación que ya realizamos con QGIS fue la obtención de centroides

	.. image:: _images/gvsigce_sextante5.png
		:scale: 50%

Y así es como queda

	.. image:: _images/gvsigce_sextante5bis.png
		:scale: 50%

Por último, podemos transformar nuestras geometrías en puntos

	.. image:: _images/gvsigce_sextante6.png
		:scale: 50%

Y éste es el aspecto final

	.. image:: _images/gvsigce_sextante7.png
		:scale: 50%

Un poco más cerca

	.. image:: _images/gvsigce_sextante8.png
		:scale: 50%
.. [1] `http://trac.osgeo.org/postgis/ticket/833 <http://trac.osgeo.org/postgis/ticket/833>`_

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
	* `uDig <http://udig.refractions.net/>`_
	* `Kosmo <http://www.opengis.es/>`_ 


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


OpenJUMP
========

gvSIG y gvSIG CE
================

El cliente de escritorio *gvSIG* es un proyecto desarrollado por el gobierno local de la Generalitat Valenciana. Actualmente, la última versión estable es la 2.0. Dicha herramienta coexiste con *gvSIG CE*, un fork de *gvSIG* desarrollado y mantenido por una comunidad. El proyecto *gvSIG CE* no está relacionado con la asociación gvSIG, ni recibe soporte oficial. 

Ambos proyectos llevan su desarrollo por separado. Aun no existe una versión 1.0 estable de *gvSIG CE*. Esta versión 1.0 estará integrada con los productos `Sextante <http://www.sextantegis.com/>`_, `GRASS GIS <http://grass.osgeo.org/>`_ y `SAGA <http://www.saga-gis.org/en/index.html>`_

En cualquier caso, tanto *gvSIG* como *gvSIG CE* muestran una incompatibilidad con *PostGIS 2.0*. En el caso de *gvSIG*, se produce una excepción al intentar crear una conexión con una base de datos |PGIS|, y ni siquiera llega a mostrar las tablas existentes. En cuanto a *gvSIG CE*, sí llega a mostrar las tablas, pero lanza un error al intentar cerrar la ventana de conexión.

Parece que la solución al problema pasa por cargar en nuestra base de datos el fichero *legacy.sql* [1]_. Dicho fichero se encuentra, en nuestra máquina, en el directorio */usr/share/postgresql/9.1/contrib/postgis-2.0/*. Podemos instalarlo con::

	$ psql -d workshop_sevilla -f /usr/share/postgresql/9.1/contrib/postgis-2.0/legacy.sql

Hecho eso, *gvSIG CE* funciona sin problemas. Pero *gvSIG* sigue lanzando el mismo error. No son descartables problemas de configuración de la máquina local. En cualquier caso, dado el gran parecido entre ambos programas, vamos a utilizar *gvSIG CE*.




uDig
====


.. [1] `http://trac.osgeo.org/postgis/ticket/833 <http://trac.osgeo.org/postgis/ticket/833>`_

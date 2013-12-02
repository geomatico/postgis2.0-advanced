.. |PGSQL| replace:: PostgreSQL
.. |PGIS| replace:: PostGIS
.. |PRAS| replace:: PostGIS Raster
.. |GDAL| replace:: GDAL/OGR
.. |OSM| replace:: OpenStreetMaps
.. |SHP| replace:: ESRI Shapefile
.. |SHPs| replace:: ESRI Shapefiles
.. |PGA| replace:: pgAdmin III
.. |LX| replace:: GNU/Linux


Pasos previos para poder seguir el curso
****************************************

Para poder seguir el **Curso avanzado de PostGIS 2.0** será necesario completar una serie de pasos previos. En este capítulo, veremos cuáles son.

En primer lugar, el software básico con el que se va a trabajar es **PostgreSQL 9.1 + PostGIS 2.0 en una máquina basada en XUbuntu 12.04**.

Veremos ahora los pasos mencionados.


Instalar la máquina virtual OSGeo Live 7
========================================== 

Los ejercicios de este curso han sido realizados usando la máquina virtual proporcionada por OSGeo, en su versión 7. Dicha máquina dispone de todo el software necesario instalado y listo para empezar a funcionar, a excepción de unos pocos programas que instalaremos a continuación.

La máquina está basada en **Xubuntu 12.04**. Podemos descargarnos una imagen de disco lista para ser ejecutada en el software de virtualización Virtual Box. `Este enlace <http://live.osgeo.org/en/quickstart/virtualization_quickstart.html>`_ contiene instrucciones detalladas de cómo instalar VirtualBox, descargar la máquina y ponerla a funcionar.

Una vez tengamos la máquina instalada y funcionando, podemos proceder con el siguiente paso


Configurar autenticación peer en |PGSQL| para el usuario *user*
===============================================================

Una vez encendamos la máquina, entraremos automáticamente con el usuario *user*. Para simplificar la configuración del entorno y poder enfocarnos en el aprendizaje de |PGIS|, vamos a realizar todas las operaciones directamente con dicho usuario. Configurando el esquema de autenticación *peer* facilitado por |PGSQL| 9.1, podemos realizar las operaciones normales de conexión con la base de datos sin necesidad de estar constantemente especificando un usuario y una contraseña.

Este método de autenticación funciona obteniendo del propio kernel del sistema operativo el nombre de usuario y utilizándolo como usuario válido de la base de datos. Es un método permitido únicamente para conexiones locales, pero como vamos a realizar todas las conexiones desde la misma máquina donde tenemos instalado |PGSQL|, lo podemos usar sin problemas. Para ello, seguimos los siguientes pasos:

En primer lugar, añadimos esta línea al final del fichero */etc/postgresql/9.1/main/pg_ident.conf*::
	
	workshop_map    user                    postgres

Este mapeo hace que el usuario del sistema operativo *user* sea identificado como el superusuario de la base de datos: *postgres*

Modificar el fichero */etc/postgresql/9.1/main/pg_hba.conf* para que aparezca esta línea::
	local   all             postgres                                peer    map=workshop_sevilla_map

Reiniciar el servidor |PGSQL|::
	$ sudo service postgresql restart

Probar la conexión con el usuario *user*::
	
	$ psql 

Como resultado, debemos ver esto::
	
	psql (9.1.9, server 9.1.10)
	SSL connection (cipher: DHE-RSA-AES256-SHA, bits: 256)
	Type "help" for help.

	user=#

Ya estamos listos para crear una base de datos y empezar a trabajar


Crear base de datos e instalar extensiones
==========================================

Procedemos a crear una base de datos, utilizando *UTF8* como codificación de caracteres. Para ello, desde la línea de comandos, ejecutamos::

	$ createdb --encoding=UTF8 workshop_sevilla

Queda creada la base de datos *workshop_sevilla*. Vamos a proceder a instalar en ella la extensión |PGIS|, que ya incluye |PRAS|. Desde una línea de comandos, ejecutamos::
	
	$ psql -d workshop_sevilla -c "create extension postgis"

Ahora, instalaremos la extensión *hstore*, útil para el almacenamiento de datos de |OSM|, como veremos en el tema 1::
	
	$ sudo apt-get install postgresql-contrib

Ya estamos listos para instalar la extensión *hstore*::

	$ psql -d workshop_sevilla -c "create extension hstore"


Por último, vamos a crear una base de datos específica para el apartado de routing. Para ello, en primer lugar instalamos algunos datos del workshop de pgRouting ofrecido por los propios desarrolladores::
	
	$ sudo add-apt-repository ppa:georepublic/pgrouting-unstable
	$ sudo apt-get update

	$ sudo apt-get install pgrouting-workshop

Copiamos la carpeta de datos a nuestro escritorio, y al root del servidor web de la máquina::

	$ cp -R /usr/share/pgrouting/workshop ~/Desktop/pgrouting-workshop
	$ sudo ln -s ~/Desktop/pgrouting-workshop /var/www/pgrouting-workshop

Finalmente, cargamos los datos proporcionados, para crear una topología con la que poder trabajar::
	
	$ cd ~/Desktop/pgrouting-workshop/
	$ tar -xvzf data.tar.gz

	$ psql -f ~/Desktop/pgrouting-workshop/data/sampledata_notopo.sql

Con esto creamos una nueva base de datos, con nombre ``pgrouting_workshop``, y una serie de tablas que nos resultarán útiles.

.. seealso:: `http://workshop.pgrouting.org/chapters/topology.html#load-network-data`_

.. note:: En OSGeo Live 7 ya existe una base de datos, con nombre ``pgrouting`` que contiene estos datos. Creamos otra diferente para poder ver todo el proceso de instalación (abrir *pgrouting_notopo.sql* para ver con más detenimiento las instrucciones que ejecuta) y dejar la original inalterada. Para comenzar a trabajar con la original, visitar `http://live.osgeo.org/en/quickstart/pgrouting_quickstart.html`_


Instalar software adicional
===========================

Vamos a usar git para instalar algunas cosas, como OSRM. Necesitamos instalarlo, junto con otros paquetes ::
	
	$ sudo apt-get install build-essential git cmake pkg-config libprotoc-dev libprotobuf7 \
	protobuf-compiler libprotobuf-dev libosmpbf-dev libpng12-dev \
	libbz2-dev libstxxl-dev libstxxl-doc libstxxl1 libxml2-dev \
	libzip-dev libboost-all-dev lua5.1 liblua5.1-0-dev libluabind-dev libluajit-5.1-dev



Descargar los datos
===================

Tanto para los ejemplos como para los ejercicios, se han utilizado unos datos que se encuentran disponibles `aquí <https://dl.dropboxusercontent.com/u/6599273/gis_data/taller_sevilla/datos_taller_sevilla.zip>`_ 

Los datos están organizados por tipo y, dentro de esta organización, por formato de fichero. En la siguiente captura se puede apreciar:
	
	.. image::  _images/tree_datos.png

.. note:: Todos los datos han sido obtenidos de fuentes públicas y de libre acceso, o generados manualmente para su uso educativo.


Opcional: Instalar moskitt y pgmodeler
======================================

En el tema 1 se tratarán conceptos de diseño de bases de datos. `Moskitt <http://www.moskitt.org>`_ y `pgmodeler <http://www.pgmodeler.com.br>`_ son dos herramientas que se pueden utilizar para modelar bases de datos relacionales con |PGSQL|. Es por eso que, a pesar de no formar parte del presente curso, incluímos a continuación una pequeña guía de instalación. 

Instalar moskitt
----------------

Moskitt es, en palabra de sus creadores, *"una herramienta CASE LIBRE, basada en Eclipse que está siendo desarrollada por la  Conselleria de Infraestructuras, Territorio y Medio Ambiente"*. Dispone de un plugin GEO, de manera que nos resultará útil. Vamos a ver cómo instalarla.

La instalación de moskitt es muy sencilla. Solo es necesario descargar el binario ejecutable desde `este enlace <http://www.moskitt.org/fileadmin/conselleria/documentacion/Descargas/1.3.10/moskitt_es-1.3.10.v201211081100-linux.gtk.x86.zip>`_. Descomprimimos el fichero zip en una carpeta, y ejecutamos el binario *MOSSKitt_es*.

Una vez arrancado, vamos a instalar el **complemento GEO**. Para ello, nos vamos a *Ayuda, Install new software*. En la captura siguiente se aprecia:
	
	.. image:: _images/instalar_moskitt1.png
		:scale: 50%

En la ventana que se abre, pulsamos el botón *Add*, destacado en la imagen

	.. image:: _images/instalar_moskitt2.png
		:scale: 50%

Este botón nos sirve para añadir un nuevo repositorio. Podemos nombrarlo como queramos, siempre que en la url pongamos **http://download.moskitt.org/moskitt/geo/updates-1.3.8**. Lo vemos en la siguiente captura:

	.. image:: _images/instalar_moskitt3.png
		:scale: 50%

Al introducir la url, nos aparecerán los paquetes disponibles. Marcamos *MOSKitt-Geo Module*, y pulsamos aceptar. Lo vemos en la captura

	.. image:: _images/instalar_moskitt4.png
		:scale: 50%

Tras ello, pulsamos en *Siguiente*, aceptamos la licencia, y se instalará automáticamente. Solo nos queda reiniciar Moskitt.


Instalar pgmodeler
------------------

Al igual que Moskitt, pgmodeler también es una herramienta libre de modelado. Pero en este caso, es específica para |PGSQL|. Para poder usarla, primero tendremos que instalar en nuestra máquina el framework QT5. Lo haremos mediante estos comandos [1]_::

	$ sudo apt-add-repository ppa:ubuntu-sdk-team/ppa
	$ sudo apt-get update
	$ sudo apt-get install qtdeclarative5-dev

Tras ello, estamos listos para descargar los binarios para nuestro sistema operativo desde `http://www.pgmodeler.com.br <http://www.pgmodeler.com.br>`_. 

Para poder ejecutar la herramienta en sistemas |LX|, tendremos que crear un sencillo script bash [2]_, con el siguiente contenido::

	## Inicio script
	#/bin/bash

	# Specify here the full path to the pgmodeler's root directory
	export PGMODELER_ROOT="/path/to/pgmodeler-0.6.1-linux32"

	export PGMODELER_CONF_DIR="$PGMODELER_ROOT/conf"
	export PGMODELER_SCHEMAS_DIR="$PGMODELER_ROOT/schemas"
	export PGMODELER_LANG_DIR="$PGMODELER_ROOT/lang"
	export PGMODELER_TMP_DIR="$PGMODELER_ROOT/tmp"
	export PGMODELER_PLUGINS_DIR="$PGMODELER_ROOT/plugins"
	export PGMODELER_CHANDLER_PATH="$PGMODELER_ROOT/pgmodeler-ch"
	export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:"$PGMODELER_ROOT"
	export PATH=$PATH:$PGMODELER_ROOT

	#Running pgModeler
	pgmodeler
	## Fin script

Únicamente tendremos que modificar la línea::

	export PGMODELER_ROOT="/path/to/pgmodeler-0.6.1-linux32"

Y añadir el path hasta el directorio donde hayamos descomprimido el programa. 

Después de eso, grabamos el script con el nombre *pgmodeler_init.sh* en cualquier lugar de nuestro disco, le damos permisos de ejecución::

	$ chmod +x pgmodeler_init.sh

Y podemos lanzarlo en cualquier momento. Veremos como pgmodeler arranca:
	
	.. image:: _images/pgmodeler.png
		:scale: 50%

Estamos listos para empezar con el curso.


.. [1] `http://askubuntu.com/questions/279421/how-can-i-install-qt-5-x-on-12-04-lts <http://askubuntu.com/questions/279421/how-can-i-install-qt-5-x-on-12-04-lts>`_
.. [2] `http://www.pgmodeler.com.br/wiki/doku.php?id=installation <http://www.pgmodeler.com.br/wiki/doku.php?id=installation>`_ 
	

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
	* Conceptos de diseño: homogeneidad, heterogeneidad, herencia
	* La importancia del SRID
	* Indexación espacial
	* Vistas y triggers
	* Herramientas de diseño de bases de datos espaciales: moskitt y pgmodeler

Finalmente, veremos una serie de ejercicios prácticos, que servirán para fijar los conocimientos adquiridos.


Introducción
============

A la hora de diseñar una base de datos, deberemos enfrentarnos a conceptos como qué elementos queremos modelar, qué tipo de consultas van a hacerse y cuáles deberían hacerse a mayor velocidad, qué tipo de análisis va a soportar, etc. Si nuestra base de datos ha de tener soporte espacial, hay otra serie de preguntas que deberemos hacernos: qué tipo de datos espaciales vamos a almacenar (puntos, líneas, polígonos, datos raster...), con qué precisión debemos almacenarlos, qué herramientas GIS se van a usar para interactuar con la base de datos, etc.

Un mal diseño de una base de datos espacial puede llevar a tener una herramienta inutilizable, puesto que los elementos que se están almacenando no son meros números o textos. Ni siquiera objetos binarios *bobos*. Son elementos muy complejos. En el caso de |PGIS|, estamos hablando de elementos que siguen el estándar `SQL/MM <http://en.wikipedia.org/wiki/Simple_Features>`_.

Es por eso que es conveniente seguir una serie de buenas prácticas a la hora de diseñar una base de datos espacial con |PGIS|. En este curso, vamos a seguir los consejos del libro `PostGIS in Action <http://www.manning.com/obe2/>`_, que consideramos como una de las *Biblias* acerca de |PGIS|.


Conceptos de diseño
====================

En el mencionado libro *PostGIS in Action* se plantean tres conceptos de diseño que consideramos especialmente interesantes a la hora de diseñar columnas geográficas en una base de datos espacial. Estos tres conceptos son **heterogeneidad**, **homogeneidad** y **herencia**. Vamos a verlos más detalladamente por separado

Heterogeneidad
--------------

Esta aproximación al diseño de tablas espaciales implica relajarse en cuanto al tipo de dato que se puede almacenar en una columna geométrica. O de otra forma: permitir que cualquier tipo de geometría sea almacenada en una columna de tipo geométrico de nuestra tabla: puntos, líneas, polígonos, etc. 

El no restringir el tipo de dato geométrico no significa que no tengamos ningún tipo de restricción a la hora de diseñar nuestras tabla. Por ejemplo, es recomendable aplicar una restricción para que todas las columnas de una misma tabla compartan SRID y dimensión de las coordenadas (2D, 3D). Una gran parte de funciones de |PGIS| requieren que estos parámetros sean los mismos en todas las geometrías de entrada.


Ventajas de la heterogeneidad
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Es más sencillo. Una misma tabla puede contener puntos, líneas y polígonos. Si queremos hacer una consulta que pueda devolvernos potencialmente más de un tipo de geometría, podemos buscar en una sola tabla.



..todo:: Terminar la introducción.





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

.. note:: Algunos de los ejercicios de este capítulo han sido adaptados del libro `PostGIS CookBook <http://www.packtpub.com/postgis-to-store-organize-manipulate-analyze-spatial-data-cookbook/book>`_. En dicho libro se encuentran los ejercicios originales, así como otros ejercicios más complejos propuestos y resueltos. Recomendamos el uso de este libro como referencia para el aprendizaje de técnicas avanzadas con |PGIS|.


Programando en |PGIS|
**********************

Veremos a continuación algunas de las opciones disponibles que hay para ampliar la funcionalidad de |PGIS|.


Programando funciones |PGIS| en PL/pgSQL
========================================

Las funcionalidades de |PGIS| pueden ser extendidas sin necesidad de programanr funciones en C que interactúen con el *core* de la extensión. Podemos programar funciones en PL/pgSQL, el lenguaje procedural para |PGSQL|, que permite crear funciones y triggers para realizar procesamientos complejos.


Veremos a continuación algunas funcionalidades programadas con PL/pgSQL


Función para estimación proporcional de censos
----------------------------------------------

En uno de los ejemplos del tema 2, intentamos calcular el número de potenciales usuarios de una vía de ferrocarril basándonos en la población censada en los barrios por donde pasaba. Para ello, usamos la siguiente consulta::
	
	SELECT SUM(b.population) as pop
	FROM barrios_de_bogota b JOIN railway_buffer r
	ON ST_Intersects(b.geom, r.geom)

Que nos daba un resultado de **819892** personas. No obstante, mirando la forma de los barrios, podemos apreciar que estamos sobre-estimando la población, si utilizamos la de cada barrio completo. De igual forma, si contáramos solo los barrios cuyos centroides intersectan el buffer, probablemente infraestimaríamos el resultado.

En lugar de esto, podemos asumir que la población estará distribuida de manera más o menos homogénea (esto no deja de ser una aproximación, pero más precisa que lo que tenemos hasta ahora). De manera que, si el 50% del polígono que representa a un barrio está dentro del área de influencia (1 km alrededor de la vía), podemos aceptar que el 50% de la población de ese barrio serán potenciales usuarios del ferrocarril. Sumando estas cantidades para todos los barrios involucrados, obtendremos una estimación algo más precisa. Habremos realizado una suma proporcional.

Para realizar esta operación, vamos a construir una función en PL/pgSQL. Esta función la podremos llamar en una query, igual que cualquier función espacial de PostGIS::
	
	CREATE OR REPLACE FUNCTION public.proportional_sum(geometry, geometry, numeric)
	RETURNS numeric AS

	$BODY$

	SELECT $3 * areacalc FROM
	(SELECT (ST_Area(ST_Intersection($1, $2))/ST_Area($2))::numeric AS areacalc) AS areac;

	$BODY$
	LANGUAGE sql VOLATILE

Mediante una modificación a nuestra consulta SQL previa, podemos obtener una estimación algo más precisa::
	
	SELECT ROUND(SUM(proportional_sum(a.geom, b.geom, b.population))) FROM
	railway_buffer AS a, barrios_de_bogota as b
	WHERE ST_Intersects(a.geom, b.geom)
	GROUP BY a.gid;

En este caso, el resultado obtenido es **248217**, que parece más razonable.


Función para eliminación de intersecciones internas entre polígonos
-------------------------------------------------------------------

Un problema común con datos vectoriales es que existan intersecciones internas entre polígonos. Eso hace que las consultas destinadas a obtener áreas o perímetros no funcionen como es debido. Un ejemplo de este problema es el fichero |SHP| llamado *barrios_de_bogota*, utilizado en capítulos anteriores.

	.. image::  _images/barrios_de_bogota.png
		:scale: 50 %


Existen diferentes aproximaciones para resolver este problema. En esta sección exploraremos la solución propuesta en el libro *PostGIS CookBook*, que a su vez está basada en una `propuesta de Kevin Neufeld <http://trac.osgeo.org/postgis/wiki/UsersWikiExamplesOverlayTables>`_

El proceso consiste básicamente en reducir los polígonos a sus líneas constituyentes y reconstruirlos después, uniendo las líneas, pero habiendo eliminado las intersecciones internas.

El primer paso, convertir los polígonos en sus líneas constituyentes, puede ser llevado a cabo con una función escrita en PL/pgSQL::
	
	CREATE OR REPLACE FUNCTION polygon_to_line(geometry)
  		RETURNS geometry AS
	$BODY$

    SELECT ST_MakeLine(geom) FROM (
        SELECT (ST_DumpPoints(ST_ExteriorRing(
					(ST_Dump($1)).geom
				))).geom

            ) AS linpoints
	$BODY$
  	LANGUAGE sql VOLATILE

Con esta función podemos convertir nuestros polígonos en líneas. Después, tendremos que *suavizar* las líneas que se solapen y construir los polígonos *suavizados* otra vez::
	
	CREATE TABLE barrios_de_bogota_fixed AS (
    	SELECT (ST_Dump(geom)).geom FROM (
        	SELECT ST_Polygonize(geom) AS geom FROM (
            	SELECT ST_Union(geom) AS geom FROM (
        	SELECT polygon_to_line(geom) AS geom FROM
            	barrios_de_bogota
            	) AS unioned
       	) as polygonized
    	) AS exploded
	);

En esta nueva tabla se puede observar el área que unos polígonos solapaban con otros, como se puede observar en la captura

	.. image::  _images/barrios_de_bogota_alt.png
		:scale: 50 %

Lo realmente interesante de esta nueva capa, es que los solapes que se aprecian **son polígonos nuevos**. Es decir, hemos generado un polígono por cada solape de nuestra capa original. En la siguiente captura, se ven todos los polígonos etiquetados, para que veamos que lo que antes eran solapes, ahora son también polígonos

	.. image::  _images/barrios_de_bogota_fixed.png
		:scale: 50 %

Podemos ver el aspecto de nuestra capa si *quitamos* estos nuevos polígonos provenientes de solapes. Para ello, calculamos los centroides de todos nuestros polígonos originales(*barrios_de_bogota*), y los centroides de la nueva capa generada (*barrios_de_bogota_alt*). Después, creamos una nueva tabla a partir de los polígonos de la tabla *barrios_de_bogota_alt* que contienen los centroides de la tabla *barrios_de_bogota* ::

	create table barrios_de_bogota_with_holes as select b.gid, b.geom from
	barrios_de_bogota_alt b, (select st_centroid(geom) as geom from barrios_de_bogota) p where st_contains(b.geom, p.geom) 

El resultado es así:
	
	.. image::  _images/barrios_de_bogota_holes.png
		:scale: 50 %


.. seealso:: `Documentación de PL/pgSQL <http://www.postgresql.org/docs/9.1/static/plpgsql.html>`_


Servicio de geocoding
---------------------

Vamos a continuación a implementar una función de geocoding, creando previamente una sencilla base de datos de nombres a partir de un CSV descargado de `este enlace <http://download.geonames.org/export/dump/>`_. Concretamente, usaremos los datos de Italia, así que descargaremos el fichero IT.zip y nos quedaremos con IT.txt, fichero en formato CSV contenido dentro del zip.

Cargaremos dicho fichero en la base de datos con la siguiente llamada a ``ogr2ogr``::
	
	$ ogr2ogr -f PostgreSQL -s_srs EPSG:4326 -t_srs EPSG:4326 -lco GEOMETRY_NAME=the_geom -nln geonames PG:"dbname='workshop_sevilla'" CSV:IT.txt -sql 'SELECT name, asciiname FROM IT'

Una vez ejecutada la instrucción, tendremos una nueva tabla en nuestra base de datos, llamada *geonames*. Contiene nombres de municipios italianos junto con puntos geolocalizados representando a dichos municipios. 

Vamos a continuación a crear una función en PL/pgSQL que nos devolverá las ubicaciones más cercanas a un lugar::

	CREATE OR REPLACE FUNCTION Get_Closest_PlaceNames(in_geom geometry, num_results int DEFAULT 5, OUT geom geometry, OUT place_name character varying)
    	RETURNS SETOF RECORD
	AS $$
    BEGIN
        RETURN QUERY
        SELECT the_geom as geom, name as place_name
        FROM geonames
        ORDER BY the_geom <-> ST_Centroid(in_geom) LIMIT num_results;
    END;
	$$ LANGUAGE plpgsql;

La función acepta como argumento una geometría de cualquier tipo, sobre la que se calcula el centroide. También un número de resultados. Si no se especifica este parámetro, por defecto devuelve 5.

Ahora codificaremos una función para buscar las ubicaciones que contengan un determinado texto::
	
	CREATE OR REPLACE FUNCTION Find_PlaceNames(search_string text,
        num_results int DEFAULT 5,
        OUT geom geometry,
        OUT place_name character varying)
    RETURNS SETOF RECORD
	AS $$
    BEGIN
        RETURN QUERY
        SELECT the_geom as geom, name as place_name
        FROM geonames
        WHERE name @@ to_tsquery(search_string)
        LIMIT num_results;
    END;
	$$ LANGUAGE plpgsql;

Al igual que antes, la función acepta un parámetro opcional con el número de resultados deseados. En este caso, no obstante, lo que se le pasa como primer argumento es una cadena de texto, que será buscada entre todos los nombres de ubicaciones presentes en la tabla *geonames*.

Como ejemplo, podemos llamar a nuestras funciones así::
	
	SELECT * FROM Get_Closest_PlaceNames(ST_PointFromText('POINT(13.5 42.19)', 4326), 10);
	
	SELECT * FROM Find_PlaceNames('Rocca', 10);



Programando funciones |PGIS| en Python
=======================================

Para aquellos que prefieran el lenguaje Python, también se pueden desarrollar funciones nuevas con él. Lo primero que tendremos que hacer es instalar el soporte para Python en nuestra base de datos::
	
	$ sudo apt-get install postgresql-server-dev-all python2.7-dev postgresql-plpython-9.1

Después, añadimos la extensión a nuesta base de datos::

	$ psql -d workshop_sevilla -c "create extension plpythonu"

Para evitar problemas de incompatibilidades, trabajaremos en un entorno virtual creado con mkvirtualenvwrapper. `Aquí <http://askubuntu.com/questions/244641/how-to-set-up-and-use-a-virtual-python-environment-in-ubuntu>`_ hay instrucciones acerca de cómo crearlo en Ubuntu 12.04. 

Una vez creado el entorno virtual, entramos en él (suponemos que lo hemos llamado *workshop*)::

	$ workon workshop

Ahora instalamos software necesario para ejecutar los ejemplos::

	$ pip install simplejson
	$ pip install psycopg2
	$ pip install numpy
	$ pip install gdal
	$ pip install geopy
	$ pip install xlrd

Las librerías instaladas habrán de ser añadidas al PATH en el que buscará el intérprete de Python invocado desde |PGSQL|. Para asegurarnos de que el path a dichas librerías es añadido, al principio de cada script Python, escribiremos lo siguiente::

	from sys import path
	path.append('/home/user/.virtualenvs/workshop/lib/python2.7/site-packages')

Asumiendo que el path mostrado es en el que se encuentran las librerías instaladas en el entorno virtual. Realizar la comprobación manual con ls. Si no fuera así, investigar en qué directorio se instalan los entornos virtuales en el sistema en el que se esté trabajando, y acceder a *workshop/lib/python2.7/site-packages* a partir de ese directorio.
	
Y ya estaríamos listos para empezar a desarrollar funciones para |PGIS| en Python


Sumando un rango de números
---------------------------

La primera función que veremos es muy sencilla: suma un rango de números. Dicha función está extraída del libro *PostGIS in Action*::
	
	CREATE OR REPLACE FUNCTION python_addreduce(param_start integer,
	param_end integer)
  	RETURNS integer AS
	$$
 	def add(x,y): return x+y
 	return reduce(add, range(param_start, param_end + 1));
	$$ LANGUAGE 'plpythonu' IMMUTABLE;

Llamarla es igualmente sencillo::
	
	SELECT python_addreduce(1,4);


Cliente de geocoding con geopy
------------------------------

En este apartado vamos a usar `geopy <https://code.google.com/p/geopy/>`_, una herramienta para geocoding con Python. La función a codificar es ésta::
	
	CREATE OR REPLACE FUNCTION Geocode(address text)
        RETURNS geometry(Point,4326)
    AS $$
        from sys import path
        path.append('/home/user/.virtualenvs/workshop/lib/python2.7/site-packages')
        from geopy import geocoders
        g = geocoders.GoogleV3()
        place, (lat, lng) = g.geocode(address)
        plpy.info('Geocoded %s for the address: %s' % (place, address))
        plpy.info('Longitude is %s, Latitude is %s.' % (lng, lat))
        plpy.info("SELECT ST_GeomFromText('POINT(%s %s)', 4326)" % (lng, lat))
        result = plpy.execute("SELECT ST_GeomFromText('POINT(%s %s)', 4326) AS point_geocoded" % (lng, lat))
        geometry = result[0]["point_geocoded"]
        return geometry
    $$ LANGUAGE plpythonu;


Podemos llamarla mediante::

	SELECT ST_AsText(Geocode('Seville, Spain')) as Sevilla



Cargando datos de un fichero XLS
--------------------------------

La siguiente función también está extraída del libro *PostGIS in Action*. Esta función importa filas de un fichero XLS en una tabla de |PGSQL| llamada *imported_places*::
	
	CREATE TYPE place_lon_lat AS (
  		place text, lon float, lat float);

	CREATE TABLE imported_places(place_id serial PRIMARY KEY,
  		place text, geom geometry);

	CREATE OR REPLACE FUNCTION fngetxlspts(param_filename text)
	RETURNS SETOF place_lon_lat AS
	$$
	import xlrd
	book = xlrd.open_workbook(param_filename)
	sh = book.sheet_by_index(0)
	for rx in range(1,sh.nrows):
		yield(sh.cell_value(rowx=rx, colx=0),
			sh.cell_value(rowx=rx, colx=1),
			sh.cell_value(rowx=rx, colx=2)
	) $$
  	LANGUAGE 'plpythonu' VOLATILE;



La llamamos con la siguiente línea::
	
	INSERT INTO imported_places(place, geom)
	SELECT f.place, ST_SetSRID(ST_Point(f.lon,f.lat),4326)
	FROM fngetxlspts('/home/user/Test.xls') AS f;

El fichero *Test.xls* se encuentra en la carpeta *otros/xls* de nuestro fichero de datos.


Creación de diagrama de Voronoi
-------------------------------

Vamos a ver un ejemplo de creación de un diagrama de Voronoi mediante una función procedural para |PGSQL| escrita en Python. La función se encuentra en el fichero *funciones/voronoi_python.sql* de nuestra carpeta de datos. Procedemos a su carga::

	$ psql -d workshop_sevilla -f funciones/voronoi_python.sql

Ahora crearemos unos datos de prueba para mostrar el resultado de ejecutar la función sobre los mismos. En primer lugar, creamos una tabla con puntos repartidos aleatoriamente::

	DROP TABLE IF EXISTS voronoi_test_points;
	CREATE TABLE voronoi_test_points
	(
 		x numeric,
 		y numeric
	)
	WITH (OIDS=FALSE);

	ALTER TABLE public.voronoi_test_points ADD COLUMN gid serial;
	ALTER TABLE public.voronoi_test_points ADD PRIMARY KEY (gid);

	INSERT INTO public.voronoi_test_points (x, y) VALUES (5, 7);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (2, 8);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (10, 4);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (1, 15);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (4, 9);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (8, 3);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (5, 3);
	INSERT INTO public.voronoi_test_points (x, y) VALUES (20, 0);


	SELECT AddGeometryColumn ('public','voronoi_test_points','the_geom',3734,'POINT',2);

	UPDATE voronoi_test_points SET the_geom = ST_SetSRID(ST_MakePoint(x,y), 3734) WHERE the_geom IS NULL;

Hecho eso, vamos a crear una tabla con los polígonos de Voronoi::
	
	CREATE TABLE voronoi_test AS
		SELECT * FROM voronoi('public.voronoi_test_points', 'the_geom') AS (id integer, the_geom geometry);

El resultado es el que se puede apreciar en las imágenes, antes y después de añadir los polígonos de Voronoi:

	.. image:: _images/ej2_voronoi_qgis1.png
		:scale: 50%

	.. image:: _images/ej2_voronoi_qgis2.png
		:scale: 50%


.. seealso:: `Python procedural language <http://www.postgresql.org/docs/9.1/static/plpython-funcs.html>`_

.. seealso:: `Diagramas de Voronoi <http://en.wikipedia.org/wiki/Voronoi_diagram>`_


Obtención de la temperatura actual en la estación meteorológica más cercana a una zona dada
-------------------------------------------------------------------------------------------

Vamos a obtener los datos meteorológicos proporciondos por la `API de OpenWeatherMap <http://openweathermap.org/API>`_. La respuesta estará en formato JSON, que nuestro código procesará para quedarse solo con la parte interesante. La función que deberemos copiar en nuestra base de datos es ésta::
	
	CREATE OR REPLACE FUNCTION public.getweather(lon double precision, lat double precision)
  		RETURNS double precision AS
	$BODY$
    from sys import path
    path.append('/Users/jorgeas80/Dev/virtualenvs/workshop/lib/python2.7/site-packages')
    import urllib2
    import simplejson as json
    data = urllib2.urlopen(
	'http://api.openweathermap.org/data/2.5/weather?lat=%s&lon=%s'
        % (lat, lon))
    js_data = json.load(data)
    if int(js_data['cod']) == 200: # only if cod is 200 we got some effective results
	station_name = js_data['name']
	weather = js_data['weather'][0]['description']
	plpy.info('Station: %s' % station_name)
	plpy.info('%s' % weather)
	if 'main' in js_data:
            if 'temp' in js_data['main']:
                temperature = js_data['main']['temp'] - 273.15 # we want the temperature in Celsius
            else:
                temperature = None
    else:
	plpy.info('No temperature info.')
	temperature = None
    return temperature
	$BODY$
  	LANGUAGE plpythonu VOLATILE
 	COST 100;


	CREATE OR REPLACE FUNCTION GetWeather(geom geometry)
    	RETURNS float
	AS $$
    	BEGIN
        	RETURN GetWeather(ST_X(geom), ST_Y(geom));
    	END;
	$$ LANGUAGE plpgsql;	


Podemos llamar a la función de dos maneras diferentes::

	SELECT GetWeather(ST_GeomFromText('POINT(-5.982330 37.388096)'));	
	SELECT GetWeather(-5.982330, 37.388096);	

La primera forma acepta cualquier tipo de geometría, no solo un punto. Simplemente, obtiene la estación más cercana al centroide del objeto pasado como geometría.


Exportando datos raster
-----------------------

La siguiente función está extraída de la documentación oficial de |PRAS|. Rasteriza una serie de polígonos y los exporta a disco vía |PRAS|::
	
	CREATE OR REPLACE FUNCTION write_file (param_bytes bytea, param_filepath text)
	RETURNS text
	AS $$
	f = open(param_filepath, 'wb+')
	f.write(param_bytes)
	return param_filepath
	$$ LANGUAGE plpythonu;

Podemos llamarla de esta forma::

	SELECT write_file(ST_AsPNG(
	ST_AsRaster(ST_Buffer(ST_Point(1,5),j*5, 'quad_segs=2'),150*j, 150*j, '8BUI',100)),
	 '/home/user/Desktop/slices'|| j || '.png')
	 FROM generate_series(1,5) As j;

La llamada generará 5 ficheros png en el directorio especificado.



Programando con |PGIS|
======================

De igual manera que podemos programar nuevas funcionalidades para |PGIS|, podemos conectarnos con |PGIS| desde nuestro lenguaje de programación favorito. En este curso, nos centraremos en realizar ejemplos con los lenguajes **Python** y **Java**


|PGIS| y psycopg2
-----------------

En este ejemplo conectaremos con |PGIS| a través de psycopg2, un driver de conexión con la base de datos |PGSQL| escrito en Python. 

En primer lugar, descargamos un |SHP| con las ciudades del mundo, e insertamos en una tabla solo aquellas con más de 100.000 habitantes. Lo hacemos con ``ogr2ogr``. Los datos los descargamos de `aquí <http://dds.cr.usgs.gov/pub/data/nationalatlas/citiesx020_nt00007.tar.gz>`_::
	
	ogr2ogr -f PostgreSQL -s_srs EPSG:4269 -t_srs EPSG:4326 -lco GEOMETRY_NAME=the_geom -nln cities PG:"dbname='workshop_sevilla"" -where 'POP_2000 > 100000' citiesx020.shp

En la tabla creada, añadimos un campo para almacenar la temperatura de cada ciudad::
	
	ALTER TABLE cities ADD COLUMN temperature real;


Creamos una tabla para almacenar también datos de estaciones meteorológicas::
	
	CREATE TABLE wstations (
		id bigint NOT NULL,
		the_geom geometry(Point,4326),
		name character varying(48),
		temperature real,
		CONSTRAINT wstations_pk PRIMARY KEY (id )
	);

Ahora, activamos el entorno virtual desde el que ejecutaremos el programa en Python::

	$ workon workshop

Creamos un nuevo fichero con el nombre *get_weather_data.py*, y añadimos lo siguiente::
	
	import urllib2
	import simplejson as json
	import psycopg2

    def GetWeatherData(lon, lat):
        """
        Get the 10 closest weather stations data for a given point.
        """
        # uri to access the JSON openweathermap web service
        uri = (
          'http://api.openweathermap.org/data/2.1/find/station?lat=%s&lon=%s&cnt=10'
          % (lat, lon))
        print 'Fetching weather data: %s' % uri
        try:
            data = urllib2.urlopen(uri)
            js_data = json.load(data)
            return js_data['list']
        except:
            'There was an error getting the weather data.'
            return []

    def AddWeatherStation(station_id, lon, lat, name, temperature):
        """
        Add a weather station to the database, but only if it does not already
        exists.
        """
        curws = conn.cursor()
        curws.execute('SELECT * FROM wstations WHERE id=%s', (station_id,))
        count = curws.rowcount
        if count==0: # we need to add the weather station
            curws.execute(
                """INSERT INTO wstations (id, the_geom, name, temperature)
                VALUES (%s, ST_GeomFromText('POINT(%s %s)', 4326), %s, %s)""",
                (station_id, lon, lat, name, temperature)
            )
            curws.close()
            print 'Added the %s weather station to the database.' % name
            return True
        else: # weather station already in database
            print 'The %s weather station is already in the database.' % name
            return False

    # program starts here
    # get a connection to the database
    conn = psycopg2.connect('dbname=workshop_sevilla')
    # we do not need transaction here, so set the connection to autocommit mode
    conn.set_isolation_level(0)

    # open a cursor to update the table with weather data
    cur = conn.cursor()

    # iterate all of the cities in the cities PostGIS layer, and for each of them
    # grap the actual temperature from the closest weather station, and add the 10
    # closest stations to the city to the wstation PostGIS layer
    cur.execute("""SELECT ogc_fid, name,
        ST_X(the_geom) AS long, ST_Y(the_geom) AS lat FROM cities;""")
    for record in cur:
        ogc_fid = record[0]
        city_name = record[1]
        lon = record[2]
        lat = record[3]
        stations = GetWeatherData(lon, lat)
        print stations
        for station in stations:
            print station
            station_id = station['id']
            name = station['name']
            # for weather data we need to access the 'main' section in the json
            # 'main': {'pressure': 990, 'temp': 272.15, 'humidity': 54}
            if 'main' in station:
                if 'temp' in station['main']:
                    temperature = station['main']['temp']
            else:
                temperature = -9999 # in some case the temperature is not available
            # "coord":{"lat":55.8622,"lon":37.395}
            station_lat = station['coord']['lat']
            station_lon = station['coord']['lon']
            # add the weather station to the database
            AddWeatherStation(station_id, station_lon, station_lat,
                name, temperature)
            # first weather station from the json API response is always the closest
            # to the city, so we are grabbing this temperature and store in the
            # temperature field in the cities PostGIS layer
            if station_id == stations[0]['id']:
                print 'Setting temperature to %s for city %s' % (
                    temperature, city_name)
                cur2 = conn.cursor()
                cur2.execute(
                    'UPDATE cities SET temperature=%s WHERE ogc_fid=%s',
                    (temperature, ogc_fid))
                cur2.close()

    # close cursor, commit and close connection to database
    cur.close()
    #conn.commit() # we shouldn't forget to commit!
    conn.close()


Podemos ejecutar el programa con::
    
    python get_weather_data.py

Veremos en la salida cómo se van añadiendo estaciones meteorológicas a la base de datos y estableciendo temperaturas para ciudades ya existentes. Al final, podremos visualizar el resultado en QGIS
	
    .. image:: _images/usa_wstations.png


|PGIS| y los bindings de |GDAL|
-------------------------------

Crearemos a continuación un servicio de geocoding con los bindings de |GDAL| usando `el servicio de geonames enriquecido con los datos de Wikipedia <http://www.geonames.org/export/wikipedia-webservice.html#wikipediaSearch>`_. Los datos obtenidos los almacenaremos en una tabla. 

Para poder usar el servicio de geonames, es necesario obtener un nombre de usuario desde `http://www.geonames.org/login`_, y activar el uso del servicio web gratuito. Hecho eso, podemos empezar.

En primer lugar, creamos un fichero llamado names.txt, con nombres de ciudades. Una por línea. Algo así::
    
    Seville
    London
    Madrid
    Paris

Después, creamos el fichero *import_names.py*, con este contenido::
    
    import sys
    import urllib2
    import simplejson as json
    from osgeo import ogr, osr

    MAXROWS = 10
    USERNAME = 'USERNAME' #enter your username here

    def CreatePGLayer():
        """
        Create the PostGIS table.
        """
        driver = ogr.GetDriverByName('PostgreSQL')
        srs = osr.SpatialReference()
        srs.ImportFromEPSG(4326)
        pg_ds = ogr.Open(
            "PG:dbname='workshop_sevilla'",
            update = 1 )
        pg_layer = pg_ds.CreateLayer('wikiplaces', srs = srs, geom_type=ogr.wkbPoint,
            options = [
                'DIM=3', # we want to store the elevation value in point z coordinate
                'GEOMETRY_NAME=the_geom',
                'OVERWRITE=YES', # this will drop and recreate the table every time
                'SCHEMA=public',
            ])
        # add the fields
        fd_title = ogr.FieldDefn('title', ogr.OFTString)
        pg_layer.CreateField(fd_title)
        fd_countrycode = ogr.FieldDefn('countrycode', ogr.OFTString)
        pg_layer.CreateField(fd_countrycode)
        fd_feature = ogr.FieldDefn('feature', ogr.OFTString)
        pg_layer.CreateField(fd_feature)
        fd_thumbnail = ogr.FieldDefn('thumbnail', ogr.OFTString)
        pg_layer.CreateField(fd_thumbnail)
        fd_wikipediaurl = ogr.FieldDefn('wikipediaurl', ogr.OFTString)
        pg_layer.CreateField(fd_wikipediaurl)
        return pg_ds, pg_layer

    def AddPlacesToLayer(places):
        """
        Read the places dictionary list and add features in the PostGIS table for each place.
        """
        # iterate every place dictionary in the list
        for place in places:
            lng = place['lng']
            lat = place['lat']
            z = place['elevation'] if 'elevation' in place else 0
            # we generate a point representation in wkt, and create an ogr geometry
            point_wkt = 'POINT(%s %s %s)' % (lng, lat, z)
            point = ogr.CreateGeometryFromWkt(point_wkt)
            # we create a LayerDefn for the feature using the one from the layer
            featureDefn = pg_layer.GetLayerDefn()
            feature = ogr.Feature(featureDefn)
            # now time to assign the geometry and all the other feature's fields,
            # if the keys are contained in the dictionary (not always the GeoNames
            # Wikipedia Fulltext Search contains all of the information)
            feature.SetGeometry(point)
            feature.SetField('title',
                place['title'].encode("utf-8") if 'title' in place else '')
            feature.SetField('countrycode',
                place['countryCode'] if 'countryCode' in place else '')
            feature.SetField('feature',
                place['feature'] if 'feature' in place else '')
            feature.SetField('thumbnail',
                place['thumbnailImg'] if 'thumbnailImg' in place else '')
            feature.SetField('wikipediaurl',
                place['wikipediaUrl'] if 'wikipediaUrl' in place else '')
            # here we create the feature (the INSERT SQL is issued here)
            pg_layer.CreateFeature(feature)
            print 'Created a places titled %s.' % place['title']

    def GetPlaces(placename):
        """
        Get the places list for a given placename.
        """
        # uri to access the JSON GeoNames Wikipedia Fulltext Search web service
        uri = ('http://api.geonames.org/wikipediaSearchJSON?formatted=true&q=%s&maxRows=%s&username=%s&style=full'
                % (placename, MAXROWS, USERNAME))
        data = urllib2.urlopen(uri)
        js_data = json.load(data)
        return js_data['geonames']

    def GetNamesList(filepath):
        """
        Open a file with a given filepath containing place names and return a list.
        """
        f = open(filepath)
        return f.read().splitlines()

    # first we need to create a PostGIS table to contains the places
    # we must keep the PostGIS OGR dataset and layer global, for the reasons
    # described here: http://trac.osgeo.org/gdal/wiki/PythonGotchas
    from osgeo import gdal
    gdal.UseExceptions()
    pg_ds, pg_layer = CreatePGLayer()

    # query geonames for each name and store found places in the table
    names = GetNamesList('names.txt')
    for name in names:
        AddPlacesToLayer(GetPlaces(name))



Lo ejecutamos con::

   $ python import_places.py

Los nombres de *names.txt* serán usados para realizar búsquedas de lugares característicos en geonames, y llenar la tabla *wikiplaces* creada en el proceso.


Programando en Java
===================

Veremos a continuación un ejemplo de sencillo programa escrito en Java, que toma el resultado de una consulta geográfica (creación de un objeto de tipo *GEOMETRY*) y lo vuelca en formato raster, haciendo uso de |PRAS|

Generando ficheros raster con |PRAS|
------------------------------------

En primer lugar, exportamos la siguiente variable de entorno::

    $ export CLASSPATH=.:./postgresql-9.3-1100.jdbc4

Creamos un fichero *Manifest.txt*, con el siguiente contenido::
    
    Class-Path: postgresql-9.3-1100.jdbc4
    Main-Class: SaveQueryImage

Y creamos el fichero *SaveQueryImage.java*::
    
    // Code for SaveQueryImage.java
    import java.sql.Connection;
    import java.sql.SQLException;
    import java.sql.PreparedStatement;
    import java.sql.ResultSet;
    import java.sql.DriverManager;
    import java.io.*;

    public class SaveQueryImage {
        public static void main(String[] argv) {
            System.out.println("Checking if Driver is registered with DriverManager.");
      
          try {
            //java.sql.DriverManager.registerDriver (new org.postgresql.Driver());
            Class.forName("org.postgresql.Driver");
          } 
          catch (ClassNotFoundException cnfe) {
            System.out.println("Couldn't find the driver!");
            cnfe.printStackTrace();
            System.exit(1);
          }
          
          Connection conn = null;
          
          try {
            conn = DriverManager.getConnection("jdbc:postgresql://127.0.0.1:5432/workshop_sevilla","user", "user");
            conn.setAutoCommit(false);

            PreparedStatement sGetImg = conn.prepareStatement(argv[0]);
            
            ResultSet rs = sGetImg.executeQuery();
            
            FileOutputStream fout;
            try
            {
                rs.next();
                /** Output to file name requested by user **/
                fout = new FileOutputStream(new File(argv[1]) );
                fout.write(rs.getBytes(1));
                fout.close();
            }
            catch(Exception e)
            {
                System.out.println("Can't create file");
                e.printStackTrace();
            }
            
            rs.close();
            sGetImg.close();
            conn.close();
          } 
          catch (SQLException se) {
            System.out.println("Couldn't connect: print out a stack trace and exit.");
            se.printStackTrace();
            System.exit(1);
          }   
      }
    }

 
Lo compilamos con::
    
    javac SaveQueryImage.java

Y creamos un fichero jar::
    
    jar cfm SaveQueryImage.jar Manifest.txt *.class

Generamos un raster a partir de una consulta que devuelve un objeto geométrico::
    
    java -jar SaveQueryImage.jar "SELECT ST_AsPNG(ST_AsRaster(ST_Buffer(ST_Point(1,5),10, 'quad_segs=2'),150, 150, '8BUI',100));" "test.png" 


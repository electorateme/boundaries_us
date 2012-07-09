=============
boundaries_us
=============


A full Django deployment for represents-boundaries with definitions for U.S.-specific data files.

This project has confusing provenance. The Chicago Tribune created django-boundaryservice (http://github.com/newsapps/django-boundaryservice), a framework for creating an API around shapefiles. The Open North guys tweaked it in a separate fork (https://github.com/rhymeswithcycle/represent-boundaries). Then I forked the fork to add map tile generation for use with Google Maps API and OpenLayers/OpenStreetMap (https://github.com/tauberer/represent-boundaries) and then wrapped it up inside this project which has actual data and deployment details.

Inside you'll find some ready-to-go data:

* A ready-to-go definition file for loading the 2010 Census's congressional districts data (which are about to go out of date).

* The 2012 redistricting shapefiles collected by the Statistical Reform in Redistricting (SRR) Project (http://www.srrproject.org/resources/redistricting-shapefiles/) and the Brennan Center for Justice at NYU, which in turn were originally created by the 50 states. The ZIP files are included in the repository because they had to be renamed to be consistent and some had to be modified to fix the names of files within the ZIP files.

* A definition file for SRR's 2012 redistricting files which figures out how to handle the slightly different naming conventions used in the shapefiles from each state.

The redistricting layer is currently running here: http://gis.govtrack.us/map/2012-cd/

Installation
------------

Clone this repository.

* Not sure why the externals didn't get cloned... git clone https://github.com/tauberer/represent-boundaries.git externals/represent-boundaries .

Install Django and PostgreSQL, and the other dependencies of GeoDjango (https://docs.djangoproject.com/en/dev/ref/contrib/gis/install/). On Ubuntu::

  sudo apt-get install binutils gdal-bin libproj-dev postgresql-9.1-postgis \
     postgresql-server-dev-9.1 python-psycopg2 python-django

Install the dependencies of represent-boundaries, which are listed in externals/represent-boundaries/README.rst::

  sudo pip install django-appconf django-jsonfield django-tastypie south
  sudo apt-get install python-cairo

Follow GeoDjango's instructions to create a PostGIS spatial database template.
On Debian/Ubuntu, the script is in this git repo::

  sudo -u postgres ./create_template_postgis-debian.sh 

Create the database and the database user::

  sudo -u postgres createdb -T template_postgis boundaries_us
  sudo -u postgres createuser -P boundaries_us

Create a settings_local.py file from the template settings_local.template.py. You'll need to fill in SECRET_KEY. Here's a quick way to generate a new key::

  python -c 'import random; print "".join([random.choice("abcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*(-_=+)") for i in range(50)])'

Set up the Django database tables::

  python manage.py syncdb
  python manage.py migrate
  
Verify that everything worked so far::

  python manage.py runserver
  
And then open http://127.0.0.1:8000/boundary-sets in your web browser. You should see some sane JSON output. Of course there's no actual GIS data loaded yet.

Loading Data
------------

Load 2010 congressional district boundaries::

  cd data/shapefiles/tiger-2010-cd
  wget ftp://ftp.census.gov/geo/tiger/TIGER2011/CD/tl_2011_us_cd112.zip
  cd ../../..
  python manage.py loadshapefiles -c --only 2010-cd

This could take about 10 minutes.
  
(You may get an error "django.db.utils.DatabaseError: invalid byte sequence for encoding "UTF8": 0x00". Postgres 9.1 Django 1.3 do not agree. You can avoid this by editing /etc/postgresql/9.1/main/postgresql.conf and setting standard_conforming_strings = off, and then restart postgresql. See https://code.djangoproject.com/ticket/16778.)

The -c option above computes a coloring scheme for each of the polygons in the shapefile for making nice maps. That process take a while. You can omit -c if you aren't going to be displaying maps.

Now run the server again to test::

  python manage.py runserver

This dataset is loaded as '2010-cd'. Here are some API examples::

  http://127.0.0.1:8000/boundary-sets/2010-cd
  
And you can see the map test page here, which you can adapt to your own needs::

   http://127.0.0.1:8000/map/2010-cd

Caching Maps
------------

You are responsible for caching the map tiles generated for maps. It's best to cache at the level of the HTTP server for this so that cached responses never hit Django.

Not Using Maps?
---------------

You should comment out the boundaries.map_urls entry in urls.py. The map tile generating view is computationally expensive, so you shouldn't expose it if you aren't going to implement caching.


# Setup

These instructions will illustrate the steps required to
successfully render custom Humanitarian OpenStreetMap Team styles.

This guide assumes that a user has postgis installed and running on
a linux environment. Addtionally, osm2pgsql, mapnik2, cascadenik, nik2img are also assumed to be installed. A good guide for getting to this point can
be found at [dbsgeo.com](http://dbsgeo.com/foss4g2010/html/)

## Data Download

This repository contains all required shapefiles, styles, and icons to render the modified osm-bright stylesheet for Haiti. You will be required to download the latest OSM extract for Haiti, provided by [Geofabrik](http://www.geofabrik.de/index.html).

Download the latest.osm.bz2 extract for Haiti
	cd osm
	wget http://labs.geofabrik.de/haiti/latest.osm.bz2

## Loading Into PostGIS

The first step will be to create an empty PostGIS database for import.
We will use psql to accomplish this, although you can also use the PgAdminIII GUI as well.

	# Create the database using the template_postgis database
	psql createdb haiti -T template_postgis

	# If you receive errors about not being able to connect
	# you can try adding in connection parameters
	# the -w flag will prompt you to enter your postgres password
	psql createdb haiti -T template_postgis -h localhost -p 5432 -U postgres -W

Next, use osm2pgsql to import the geofabrik osm extract, 'latest.osm.bz2' into the newly created database, 'haiti'. It is important to note that we are bypassing the normal, 'default.style' file and replacing it with a version specific to collection of data for use in kiosk mapping.

	# Use osm2pgsql to import latest.osm.bz2 into postgis
	# -x flag adds extra attributes, -S flag defines pathway
	# to the custom style sheet
	osm2pgsql -x -S ../kiosk/iom_default.style -d haiti latest.osm.bz2

	# If you revieve connection errors, you can pass in
	# addition connection parameters
	osm2pgsql -x -S ../kiosk/iom_default.style -d haiti -P 5432 -H localhost -U postgres -W latest.osm.bz2

Note that loading in the OSM dataset may take a few minutes to complete. If you are unable to download the geofabrik export, a sample dataset 'sector_4.osm' is provided for use.

## Modify osm-bright.mml PostGIS datasource

Now that the data have been loaded into  PostGIS, you will need to point the cascadenik style file, 'osm-bright.mml' to the your local PostGIS instance.

Start by opening 'osm-bright.mml' in any text editing program. I will be using vim in this example.

	# Navigate to the osm-bright folder
	# and edit osm-bright.mml
	cd ../kiosk/osm-bright/
	vim osm-bright.mml

Begining on line 44, modify any of the connection parameters to match your local PostGIS instance. The likely parameters you will need to change are, 'user', 'password', and 'dbname' if you loaded your data into a database other then 'haiti'.

	 <!-- Database settings -->
 	 <!-- Modify to match your local database settings -->
 	 <!ENTITY db_info '<Parameter name="type">postgis</Parameter>
        	<Parameter name="password">gis</Parameter>
		<Parameter name="host">localhost</Parameter>
        	<Parameter name="port">5432</Parameter>
		<Parameter name="user">postgres</Parameter>
		<Parameter name="dbname">haiti</Parameter>
		<Parameter name="estimate_extent">false</Parameter>
		<Parameter name="extent">-20037508,-19929239,20037508,19929239</Parameter>
		<Parameter name="geometry_field">way</Parameter>
		<Parameter name="srid">900913</Parameter>'>

Save any changes to osm-bright.mml and exit your text editor.

## Adding Custom Fonts to Mapnik

The [osm-bright](https://github.com/developmentseed/mapbox/tree/master/osm-bright/) style developed by AJ Ashton is used as a basemap for the kiosk style. Before data are rendered using this style, custom fonts must be installed and registered with mapnik. Specifically, 'Arial Regular' and 'Arial Bold' are required.

The notes below have been adapted from detailed instructions on the [Mapnik wiki](http://trac.mapnik.org/wiki/UsingCustomFonts).

First, check if the Arial Bold and Arial Regular faces are already registered with Mapnik2. Why Mapnik2 and not Mapnik? because we will be rendering our images using Mapnik2. You can easily use these same steps with Mapnik 0.7.x however, just substitute 'mapnik2' with 'mapnik'.

	# List fonts registered with mapnik2
	python -c "from mapnik2 import FontEngine as e; print '\n'.join(e.instance().face_names())"

The command above will return all fonts registered with mapnik2. If you do not see Arial Regular or Arial Bold, you will need to install them. Arial truetype fonts have been provided in the following directory '/styles/kiosk/osm-bright/fonts'. These fonts have been taken from the msttcorefonts package. If you want to download the fonts from their source, or have access to other Microsoft fonts, read [this tutorial](http://embraceubuntu.com/2005/09/09/installing-microsoft-fonts/).

For the purpose of this tutorial, we will copy the contents of the 'fonts' folder to Mapnik2's FontCollectionPath.

	# Navigate to the fonts folder
	cd fonts
	# Identify where Mapnik2 stores its fonts
	python -c "import mapnik2; print mapnik2.fontscollectionpath"
	# Example output: /usr/local/lib/mapnik2/fonts
	# Copy arial fonts to the directory identified above
	cp * /usr/local/lib/mapnik2/fonts
	# Re-check fonts registered with Mapnik2
	python -c "from mapnik2 import FontEngine as e; print '\n'.join(e.instance().facenames()

The output list should now contain Arial Bold and Arial Regular.

## Convert Cascadenik MML to Mapnik XML

Now that the pre-requisite fonts are installed installed, osm-bright.mml can be converted from the cascadenik format to a mapnik 0.7.x file. A copy of this file will then be converted from mapnik 0.7.x syntax to mapnik2 syntax.

	# Drop back to the osm-bright folder
	cd ../
	# Convert osm-bright.mml to a mapnik 0.7.x XML file
	cascadenik-compile.py osm-bright.mml osm-bright_mapnik07x.xml
	# Upgrade Mapnik XML to Mapnik2 XML
	upgrade_map_xml.py osm-bright_mapnik07x.xml osm-bright_mapnik2.xml

## Render an image of the data using nik2img

	# Use nik2img.py to generate a PNG image from the mapnik2 stylesheet.
	# Note we're setting the center and zoom for Haiti.
	nik2img.py -c -72.252 18.6552 -z 18 osm-bright_mapnik2.xml test_image.png

## Download Support Files from Source

If you are interested in using this style for an area outside of Haiti, or the included shapefiles in the 'shp' folder are not functioning, you can grab them from the source.

Note: See [this](http://dbsgeo.com/foss4g2010/html/getting_stylish.html#getting-stylish) tutorial for full instructions.

	# Navigate to shp folder
	cd path/to/styles/shp

	# Download Them
	wget http://tile.openstreetmap.org/processed_p.tar.bz2
	wget wget http://tile.openstreetmap.org/world_boundaries-spherical.tgz

	# Extract them
	tar xzf world_boundaries-spherical.tgz
	tar xjf processed_p.tar.bz2

	#Index them
	shapeindex processed_p
	shapeindex builtup_area

Now that they have been downloaded, point your cascadenik mml file at them

	# Navigate to osm-bright.mml
	cd ../kiosk/osm-bright/

	# Edit osm-bright.mml
	vim osm-bright.mml

You will need to modify the file path parameters for both the 'land' and 'builtup' layer in osm-bright.mml. These begin on line 74 of the document.

	<!-- == Areas ============================================================ -->
	<!-- If using different shapefiles, modify path here -->
 	<Layer class="land" srs="&srs900913;" status="on">
	   <Datasource>
	      <Parameter name="file">../../shp/processed_p_extract</Parameter>
	      <Parameter name="type">shape</Parameter>
	   </Datasource>
	</Layer>
	<!-- If using different shapefiles, modify path here -->
	<Layer class="builtup" srs="&srsMerc;" status="off">
	   <Datasource>
	      <Parameter name="file">../../shp/builtup_area_extract</Parameter>
	      <Parameter name="type">shape</Parameter>
	   </Datasource>
	</Layer>

## Download Custom Icons

The majority of the icons used in this style are from the SJJB Icon set.

These icons are maintained by Brian Quinian, and many of them are derived from the National Parks Service.

To download the SJJB icon set, visit his page: [http://www.sjjb.co.uk/mapicons/downloads](http://www.sjjb.co.uk/mapicons/downloads)

# Using SQL for GIS analysis with PostGIS

### Requirements
* PostgreSQL
* PostGIS
* Postico
* QGIS 3
* ogr2ogr (for exporting, if applicable)

### Data included
* [Texas schools](https://schoolsdata2-tea-texas.opendata.arcgis.com/datasets/059432fd0dcb4a208974c235e837c94f_0) from Texas Education Agency
* [Texas school districts](http://schoolsdata2-tea-texas.opendata.arcgis.com/datasets/e115fed14c0f4ca5b942dc3323626b1c_0) from Texas Education Agency
* Census Tracts: [Shapefiles](https://www.census.gov/geo/maps-data/data/tiger-line.html) and [ACS data](https://factfinder.census.gov/faces/nav/jsf/pages/index.xhtml) from Census Bureau

## Importing shapefiles into PostGIS

1. Create database and enable PostGIS in your database. From the terminal, run:
```
  $ createdb postgis_intro
  $ psql -d postgis_intro -c "CREATE EXTENSION postgis;"
```

2. Determine your project's SRID (see our glossary).

  The most popular SRID is usually EPSG:4326 (WGS 84). You can also look up your shapefile's SRID [here](http://prj2epsg.org/search) by uploading your `.prj` file (which is where the projection information lives).

  Our data files are all EPSG:4326.

3. Use `shp2pgsql` command to write a SQL file that will contain table and data information for PostgreSQL to upload:
```
$ shp2pgsql -c -s <SRID> -g geom -I <path to shp file>.shp \
 <new postgres table name> > <path to output sql file>.sql
```
So our commands to upload all tables will look like:
```
$ shp2pgsql -c -s 4326 -g geom -I data/texas-schools/texas-schools.shp \
 texas-schools > texas-schools.sql
```
```
$ shp2pgsql -c -s 4326 -g geom -I data/texas-school-districts/texas-school-districts.shp \
 texas-school-districts > texas-school-districts.sql
 ```
 ```
$ shp2pgsql -c -s 4326 -g geom -I data/census-tracts/census-tracts.shp \
 census-tracts > census-tracts.sql
```
Then we'll upload the sql file to the database we created:
```
$ psql -d <database> -f <path to output sql file>.sql
```
Our commands will look like:
```
$ psql -d postgis_intro -f texas-schools.sql
```
```
$ psql -d postgis_intro -f texas-school-districts.sql
```
```
$ psql -d postgis_intro -f census-tracts.sql
```
## Viewing queries in QGIS
1. In the Browser Panel (along left side), right click "PostGIS" and select "New Connection..."
2. Enter connection information and then click "OK".
3. Now from the PostGIS tab in the Browser Panel, you'll be able to expand the PostGIS tab to view tables with geometries from your database. You can double click on the table to view it.
4. In Layers Panel, right click the layer from the table and select "Update SQL Layer..."
5. You can write your PostGIS query in the editor line and once you click the "Update" button, you'll see only the results of the query shown.

## Exporting query results

### Exporting in QGIS
1. Right click layer in Layer Panel. Select "Export" > "Save Features As..."
2. Under "Format" you can chose from shapefile, geojson, CSV, etc. Make sure the CRS is populated with what you want, or you can change it if you'd like.

### Exporting with pgsql2shp
The pgsql2shp command to call from the terminal will look like:
```
$ pgsql2shp -f <path to new shapefile> -g <geometry column in table> <database> "<query>"
```
Here is an example of what ours might look like, if we wanted only Dallas and Forth Worth school districts from the `texas-school-districts` table:
```
$ pgsql2shp -f filter-query.shp -g geom postgis_intro \
 "select * from \"texas-school-districts\" where where name in ('Dallas ISD', 'Fort Worth ISD')"
```

### Exporting with ogr2ogr
This is a good option if you want to go straight from PostGIS to a file other than a shapefile (think GeoJSON for a D3 viz or to load into Mapbox).

Here is a typical command:
```
$ ogr2ogr -f "<filetype>" <path to output file> \
 PG:"host=<host> dbname='<database>'" \
 -sql "<query>";"
```

If we wanted to export the entire `texas-school-districts` table:
```
$ ogr2ogr -f "GeoJSON" tx-school-districts.json \
 PG:"host=localhost dbname='postgis_intro'" \
 -sql "select * from \"texas-school-districts\";"
```

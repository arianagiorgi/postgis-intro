# Using SQL for GIS analysis with PostGIS

### Requirements
* PostgreSQL
* PostGIS
* Postico
* QGIS 3

### Data included
* [Texas schools](https://schoolsdata2-tea-texas.opendata.arcgis.com/datasets/059432fd0dcb4a208974c235e837c94f_0) from Texas Education Agency
* Census Tracts
* Counties

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
$ shp2pgsql -c -s [SRID] -g geom -I [your_shp_file].shp [new_postgres_table_name] > [output_sql_file_name].sql
```
So our command will look like:
```
$ shp2pgsql -c -s 4326 -g geom -I texas-schools.shp texas-schools > texas-schools.sql
```
Then we'll upload the sql file to the database we created:
```
$ psql -d [target_database] -f [output_sql_file_name].sql
```
Our command will look like:
```
$ psql -d postgis_intro -f texas-schools.sql
```

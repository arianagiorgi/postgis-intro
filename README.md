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
 schools > texas-schools.sql
```
```
$ shp2pgsql -c -s 4326 -g geom -I data/texas-school-districts/texas-school-districts.shp \
 districts > texas-school-districts.sql
 ```
 ```
$ shp2pgsql -c -s 4326 -g geom -I data/census-tracts/census-tracts.shp \
 tracts > census-tracts.sql
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

## Spatial queries

PostGIS adds almost 300 new functions and operators to PostgreSQL that can do everything from basic spatial calculations to complex aggregations. It's easy to recognize these new functions because they all use the `ST_` prefix (for spatial type), as you'll see below:

#### Basic calculations

Many functions perform simple calculations on a single geometry or pair of geometries. For example, we can see the area of each of our census tracts using the `ST_Area` function:

```sql
SELECT name, ST_Area(geom)
FROM tracts
ORDER BY id;
```

[`ST_Area`](https://postgis.net/docs/ST_Area.html) and many functions like it that output measurements report them in the coordinate system of the geometries you're using in your query. In our case, that's WGS84, which tells us that the first census tract in our results has an area of _0.046862144283_. That's because our coordinate system is in degrees.

If we had been using, say, [NAD83 / Texas South Central](https://epsg.io/2278), which is in feet, we would've gotten our answer in feet. We can see that in action if we transform our geometries to that coordinate system with [`ST_Transform`](https://postgis.net/docs/ST_Transform.html):

```sql
SELECT name, ST_Area(ST_Transform(geom, 2278))
FROM tracts
ORDER BY id;
```

We now get an answer of _5296730440.84863_ (square feet).

We can also solve this problem using [`Geography`s](https://postgis.net/docs/using_postgis_dbmanagement.html#PostGIS_Geography). Geographies, unlike geometries, operate on spherical coordinates like the latitude and longitude values we're using here. Most common functions, when passed geographies instead of geometries, will return values in meters:

```sql
SELECT name, ST_Area(geom::GEOGRAPHY)
FROM tracts
ORDER BY id;
```

We now get an answer of _491172828.83503_ (square meters).

#### Deriving new geometries

Some functions can calculate new geometries based on your existing ones. For example, we can get the centroid of all of our census tracts using:

```sql
SELECT name, ST_Centroid(geom)
FROM tracts
ORDER BY id;
```

While the original column contains `POLYGON` data, the value returned from [`ST_Centroid`](https://postgis.net/docs/ST_Centroid.html) will be a `POINT`.

Similarly, we can create buffers around our school locations using [`ST_Buffer`](http://www.postgis.net/docs/ST_Buffer.html), which will derive polygons from the points:

```sql
SELECT campname, ST_Buffer(geom, 1)
FROM schools
ORDER BY campname;
```

The size of the buffer (the second function argument) in the example above is in the units of the coordinate system for the geometry. So the above would create a one _degree_ buffer. To buffer by a kilometer, we can cast to a geography just like we did with `ST_Area`:

```sql
SELECT campname, ST_Buffer(geom::GEOGRAPHY, 1000)
FROM schools
ORDER BY campname;
```
## Viewing queries in QGIS
1. In the Browser Panel (along left side), right click "PostGIS" and select "New Connection..."
2. Enter connection information and then click "OK".
3. Now from the PostGIS tab in the Browser Panel, you'll be able to expand the PostGIS tab to view tables with geometries from your database. You can double click on the table to view it.
4. In Layers Panel, right click the layer from the table and select "Update SQL Layer..."
5. You can write your PostGIS query in the editor line and once you click the "Update" button, you'll see only the results of the query shown.

#### Spatial relationships

We really start to see the power of spatial queries when we start to use PostGIS to join and filter out tables based on spatial relationship. For example, we can perform a point-in-polygon query to determine the district that each of our schools is in using the `ST_Contains` function:

```sql
SELECT schools.campname, districts.name
FROM schools
LEFT JOIN counties ON ST_Contains(districts.geom, schools.geom);
```

Combined with a `GROUP BY`, we can now easily count the number of schools in each district:

```sql
SELECT districts.name, COUNT(*) AS num_schools
FROM schools
LEFT JOIN districts ON ST_Contains(districts.geom, schools.geom)
GROUP BY districts.name
ORDER BY num_schools DESC;
```

`ST_Contains` is just one of several functions that can be used to characterize the spatial relationship between two geometries. To determine the correct function to use, you should consult the function's documentation, which contains helpful images that show how the function evaluates.

For example, the [`ST_Contains` documentation](https://postgis.net/docs/ST_Contains.html) says the below geometries would return `TRUE`:

![ST_Contains TRUE](images/st_contains_true.png?raw=true)

And the below would return `FALSE`:

![ST_Contains FALSE](images/st_contains_false.png?raw=true)

We can really see the difference between the spatial functions when we try to use them with two polygon layers - our census tract table and our school districts table. For example, this `ST_Contains` query tells us that there are 2,731 census tracts that are fully contained by our districts:

```sql
SELECT COUNT(*)
FROM tracts
INNER JOIN districts ON ST_Contains(districts.geom, tracts.geom);
```

But using [`ST_Overlaps`](http://postgis.net/docs/ST_Overlaps.html) we can see that there are more than 8,000 times where a tract is partially, but not wholly contained by, a district:

```sql
SELECT COUNT(*)
FROM tracts
INNER JOIN districts ON ST_Overlaps(districts.geom, tracts.geom);
```

And there are 11,000 times, according to [`ST_Intersects`](https://postgis.net/docs/ST_Intersects.html) that a tract "shares any portion of space". This basically combines the results of `ST_Contains` and `ST_Overlaps`:

```sql
SELECT COUNT(*)
FROM tracts
INNER JOIN districts ON ST_Intersects(districts.geom, tracts.geom);
```

Note that there are only about 5,000 census tracts in the state. But the above queries are capturing each pair of overlaps between our districts and census tracts.

_See [here](http://postgis.net/workshops/postgis-intro/spatial_relationships.html) for a complete list of the relationship functions._

#### Aggregations

We can take what we've learned above about spatial relationships and use it to relate data about our census tracts to our school districts. For example, we can take tract-level data and use it to calculate district-level data.

The below adds a column to our `ST_Intersects` query from above that includes the percentage overlap between the district and the tract:

```sql
SELECT
  districts.name,
  tracts.id,
  ST_Area(ST_Intersection(districts.geom, tracts.geom)) / ST_Area(tracts.geom) AS overlap_pct
FROM districts
INNER JOIN tracts ON ST_Intersects(districts.geom, tracts.geom)
```

We can then use allocate that percentage of the under-18-year-olds in the census table to the school district:

```sql
SELECT
  districts.name,
  tracts.id,
  ST_Area(ST_Intersection(districts.geom, tracts.geom)) / ST_Area(tracts.geom) * tracts.age_undr18 AS under_18_pop
FROM districts
INNER JOIN tracts ON ST_Intersects(districts.geom, tracts.geom)
```

And by adding a `GROUP BY` we can get a district-wide estimate of the under-18 population:

```sql
SELECT
  districts.name,
  SUM(ST_Area(ST_Intersection(districts.geom, tracts.geom)) / ST_Area(tracts.geom) * tracts.age_undr18) AS under_18_pop
FROM districts
INNER JOIN tracts ON ST_Intersects(districts.geom, tracts.geom)
GROUP BY district.name
```

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
 "select * from \"districts\" where where name in ('Dallas ISD', 'Fort Worth ISD')"
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
 -sql "select * from \"districts\";"
 ```

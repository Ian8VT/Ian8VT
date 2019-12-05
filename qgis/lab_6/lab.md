## Wetland Drains in Dar es Salaam, Tanzania

### Project Outline
Dar es Salaam, Tanzania, is a highly mapped city on OpenStreetMap. This provides a unique opportunity to conduct a number of different geospatial analyses related to the city. [Ramani Huria](http://ramanihuria.org/), a community-based mapping project in the city that accesses this vast array of geospatial data, draws attention to the presence of damaging seasonal floods in the city which are often compounded by a lack of necessary infrastructure and city planning. Further, there are a large number of structures and settlements in the city that are built on terrain designated as wetland - locations expected to be highly suscpetible to flood disturbances. 

### Lab Summary
In this lab, I measured the length of drains in designated wetlands. Specifically, I compared the length of drains to the footprint area of buildings within the wetlands. I bounded these wetland measurements by subward, the most local political boundary in the city. As such, my final product is a single number which represents how many meters of drain there is per sqaure meter of building within the wetlands of each subward. All final results are a decimal number between 0 and 1. Theoretically, this number is a conceptualization of how well equipped the wetland settlements of that subward are to manage flood events. A number of 1 would mean that for every 1 square meter of building, there is 1 meter of drain in that subward's wetlands. Conversely, a number close to zero means there is very little drain for each square meter of building. Although a final number greater than 1 is mathematically possible, it does not occur in this circumstance. 

### Data Sources
For this lab, I relied upon three input layers. I used [OpenStreetMap](https://www.openstreetmap.org/#map=16/31.9715/35.8944&layers=D) to pull a polygon layer that I gleaned for wetland and building information. I accessed the [Resilience Academy](https://resilienceacademy.ac.tz/) to pull a subward polygon layer and a drain line layer. I uploaded these datasets onto a [PostGIS](https://postgis.net/) database and conducted my SQL analysis on this database via the [QGIS](https://qgis.org/en/site/) interphase. 

### Annotated SQL Analysis Work Flow
#### With Maps of Selected Steps

```

CREATE TABLE dar_polygons AS
SELECT osm_id AS id, building, "natural", st_transform(way,32737) AS way, way_area
FROM planet_osm_polygon
WHERE building IS NOT NULL OR "natural" IS NOT NULL
/* Creates a new table from the input OSM polygon data. Five columns are selected from the input to the 
new table: polygon id; building type; natural feature type; way(in this feature, multipolygon); area of 
that way. The coordinate reference system is simultaneously transformed from the OSM default to the UTM 
projection for Dar es Salaam. */

UPDATE dar_polygons
SET way_area = st_area(way)
/* This ensures that the calculated area of the way_area column is in the correct unit of measurement for 
the new UTM projection, which is meters. */

CREATE TABLE wetlands AS
SELECT id, "natural" AS wetland, way, way_area
FROM dar_polygons
WHERE "natural" = 'wetland'
/* Creates a new table of just polygons that are recorded as wetland. The polygon id, way, and way_area 
of these wetland polygons are also selected for the new table. */

ALTER TABLE dar_polygons ADD COLUMN building_presence INTEGER
/* Adds a new column to the polygon table. This is in preparation for the next step, for which the 
results will be a whole number (either 0 or 1), so I designated that this table will contain data 
in the integer format.  */

UPDATE dar_polygons
SET building_presence = 1
WHERE building IS NOT NULL
/* Inserts a value of 1 in the new column created last step for every polygon mapped as a building 
irregardless of terrain. */

ALTER TABLE dar_polygons ADD COLUMN building_wetland INTEGER
/* Adds a new column to the polygon table in preparation for the next step. Again, the resulting data 
will be in the whole number-integer format. */

UPDATE dar_polygons
SET building_wetland = 1
FROM wetlands
WHERE st_intersects(dar_polygons.way, wetlands.way)
/* Sets a value of 1 in the previously created column for each polygon in the table that intersects 
a wetland irregardless of polygon type. */

CREATE TABLE building_wetlands AS
SELECT id, building, way, way_area
FROM dar_polygons
WHERE building_wetland = 1 AND building_presence = 1
/* Compares the integers from the two recently added columns to creates a new table of only building
polygons that are present in wetlands.  */

```

![building_wetlands](/qgis/lab_6/building_wetlands.png)

This is the map of the building_wetlands table created in the previous SQL step. As you can see, only the buildings that are present within wetlands are included in the output data. In this map, each individual building as its own unique ID.  

```

CREATE VIEW intersection AS
SELECT row_number() OVER () AS id, wetlands.way, subwards.fid, subwards.way AS sub_way,
(st_intersection(wetlands.way,subwards.way))
FROM wetlands 
INNER JOIN subwards ON st_intersects(wetlands.way, subwards.way)
/* This creates a new layer of wetlands bounded by subward boundaries. This is the first appearance 
of the subward polygon layer in the workflow. Each disjointed wetland within a single subward are 
considered indepedent polygons and previously continuous wetland polygons that cross a subward 
boundary are now considered indepdent polygons. Each new feature has a unique id, lists the wetland 
way, the subward id that the wetland is in, and the subward way. */

CREATE TABLE wetland_subward AS
SELECT fid, st_union(intersection.st_intersection) AS wetland_subward
FROM intersection
GROUP BY fid
/* This takes the previously created layer and dissolves all disjointed wetlands within the same 
subward as a single polygon. */

```

![wetland_subward](https://github.com/Ian8VT/Ian8VT.github.io/blob/master/qgis/lab_6/wetland_subward_1.png)

This map portrays the wetland_subward table created in the previous SQL step. In this map, all of the wetland polygons that are within the same subward contain the same unique ID. As an example, I shaded the wetlands of Subward ID 91 to be a a shade of teal. As you can see, even though the components to this wetland are disjointed, they are the same polygon.

```

ALTER TABLE wetland_subward ADD COLUMN area FLOAT
/* Adds a new column in the table in preparation for the next step, the results of which will be 
a decimal number (float data point) */

UPDATE wetland_subward
SET area = st_area(wetland_subward)
/* Calculates the area of all the wetlands within each subward */

ALTER TABLE drains ADD COLUMN length FLOAT
/* Adds a new column in the table in preparation for the next step, the results of which will be 
a float. This is the first appearance of the drain layer in the analysis */

UPDATE drains SET length = st_length(way)
/* Calculates the length of the drains */

ALTER TABLE subwards ADD COLUMN sub_drains INTEGER
/* Adds an integer column for the next step */

UPDATE subwards
SET sub_drains = 1
FROM drains
WHERE st_intersects(drains.way, subwards.way)
/* Sets a value of 1 in the newly created column for subward polygons that intersect with drain 
lines. */

CREATE TABLE subwards_drains AS
SELECT fid, way, sub_drains
FROM subwards
WHERE sub_drains = 1
/* Creates a new table of only subwards that contain mapped drains. */

ALTER TABLE subwards_drains ADD COLUMN wetland INTEGER
/* Adds a new integer column in preparation for the next step */

UPDATE subwards_drains
SET wetland = 1
FROM wetland_subward
WHERE (st_intersects(wetland_subward.wetland_subward, subwards_drains.way)
AND NOT st_touches(wetland_subward.wetland_subward, subwards_drains.way))
/* Sets a value of 1 for subwards that intersect wetlands and not just touch wetlands */

CREATE TABLE subwards_drains_wet AS
SELECT fid, way, wetland
FROM subwards_drains
WHERE wetland = 1
/* Creates a table of subwards that intersect with wetlands. Since the input table was subwards 
that contain drains, this new table is of subwards that contain both drains and wetlands, 
though does not require the drains to be in wetlands. */

```

![subwards_drains_wet](https://github.com/Ian8VT/Ian8VT.github.io/blob/master/qgis/lab_6/subwards_drains_wet.png)

This is a map of the subwards_drains_wet table created in the previous step. The step created a table in which the output only consists of subward polygons that intersect with drain data and contain wetlands. Subwards that do not meet this qualification are excluded from the output and will not be considered in further analysis.

```

:: ERROR Invalid Geometry, Geometry Intersects WITH Self
:: SELECT row_number() OVER () AS id,
:: (st_intersection(building_wetlands.way, wetland_subward.wetland_subward)),
:: building_wetlands.osm_id, building_wetlands.way, building_wetlands.way_area, 
:: wetland_subward.fid, wetland_subward.wetland_subward, wetland_subward.area
:: FROM building_wetlands
:: LEFT JOIN wetland_subward ON st_intersects(building_wetlands.way, 
:: wetland_subward.wetland_subward)

CREATE TABLE wetland_subwards_corrected AS
SELECT fid, area, st_makevalid(wetland_subward) AS way
FROM wetland_subward
/* Edits the layer to correct the simple errors in the geometry features */

CREATE TABLE building_wet_sub AS
SELECT row_number() OVER () AS id,
(st_intersection(building_wetlands.way, wetland_subwards_corrected.way)) AS way,
building_wetlands.osm_id, building_wetlands.way AS building_way, 
building_wetlands.way_area AS building_area, wetland_subwards_corrected.fid, 
wetland_subwards_corrected.way AS wetland_way, 
wetland_subwards_corrected.area AS wetland_area
FROM building_wetlands
LEFT JOIN wetland_subwards_corrected ON st_intersects(building_wetlands.way, 
wetland_subwards_corrected.way)
/* Creates a new table where the wetland buildings and bounded by subwards. Each 
building within each subward has its own unique id */

CREATE TABLE building_area_sub AS
SELECT fid, st_union(building_wet_sub.building_way) AS buildings, 
sum(building_wet_sub.building_area) AS building_area
FROM building_wet_sub
GROUP BY fid
/* Dissolves all wetland buildings by subward so that all wetland buildings within the 
same subward are a single polygon. This step is fine for further analysis, but must be 
redone for a final display deliverable as this output contains all wetland buildings 
within all the subwards of the initial input, not just those subwards with mapped wetland 
and drain data*/

```

![building_area_sub](https://github.com/Ian8VT/Ian8VT.github.io/blob/master/qgis/lab_6/building_area_sub.png)

This is a map of the building_area_sub table created in the previous step. Although this visually appears similar to the building_wetlands table in which I provided a map of previously, the internal data of this table is very different. In this table, all of the buildings which are present within the same subward are now a single multipolygon. As such, the number of building IDs is the same as the number of subwards which contain wetlands. To visualize this, I shaded the building multipolygon of Subward 91 bright green. As you can see, all buildings that are within this subward are the same multipolygon geometry.

```

CREATE TABLE drain_wet AS
SELECT st_multi(st_intersection(a.way,b.wetland_subward)) AS drain_wet, a.fid AS drain_fid, 
a.way AS drain_way, a.length AS drain_length, b.fid AS wet_fid, b.wetland_subward AS wet_way, 
b.area AS wet_area
FROM drains AS a
INNER JOIN wetland_subward AS b
ON (st_intersects(a.way, b.wetland_subward)
AND NOT st_touches (a.way, b.wetland_subward))
/* Creates a table of only the drain line segments that are within wetlands. The drain must 
fully intersect the wetland polygon and not just touch the boundary. */

CREATE TABLE drains_sub AS
SELECT wet_fid, st_union(drain_wet.drain_wet) AS drains_way, sum(drain_length) AS length
FROM drain_wet
GROUP BY wet_fid
/* Creates a new table that dissolves all wetland drains within each subward as a single 
geometry. The length of the dissolved drains within each subward are summed.  */

```

![drains_sub](https://github.com/Ian8VT/Ian8VT.github.io/blob/master/qgis/lab_6/drains_sub.png)

This map is the drains_sub table from the previous SQL step. Each of the drains within the output data are shaded as a bright red. Although there visually appears to be a large number of tiny, disjointed drains, each drain that belongs in the same subward is a single multiline geometry. As such, the number of unique IDs in the attribute table is equivalent to the IDs of subwards which contain wetland and drain data.

```

CREATE TABLE subwards_info AS
SELECT a.fid, a.way, b.length AS drain_length_wetland
FROM subwards_drains_wet AS a LEFT OUTER JOIN drains_sub AS b
ON a.fid=b.wet_fid
WHERE b.length IS NOT NULL
/* In preparation for a final deliverable layer, this step creates a subward layer in 
which the total length of wetland drains is listed for each subward */

CREATE TABLE subwards_info_wetland AS
SELECT a.fid, a.way, a.drain_length_wetland, b.building_area
FROM subwards_info AS a LEFT OUTER JOIN building_area_sub AS b
ON a.fid=b.fid
/* Creates a table that builds upon the previous step to also attach the total area of 
wetland buildings onto a subward layer */

ALTER TABLE subwards_info_wetland ADD COLUMN density_wetland FLOAT;
UPDATE subwards_info_wetland
SET density_wetland = drain_length_wetland/building_area
/* Calculates the length of wetland drain to each square meter of wetland building within 
each subward  */

ALTER TABLE building_wet_sub ADD COLUMN subward INTEGER;
UPDATE building_wet_sub
SET subward = 1 
FROM subwards_info_wetland_meters
WHERE (st_intersects(building_wet_sub.way, subwards_info_wetland_meters.way)
AND NOT st_touches(building_wet_sub.way, subwards_info_wetland_meters.way));
CREATE TABLE buildings AS
SELECT sub_id, st_union(building_way) AS buildings, sum(building_area) AS building_area
FROM building_wet_sub
WHERE subward IS NOT NULL
GROUP BY sub_id
/* Makes a table of buildngs dissolved by subward as a single polygon and only considers 
buildings within subwards that contain final drain density data for wetlands */

CREATE TABLE final_data AS
SELECT id AS subward_id, drain_length_wetland AS wetland_drain_length, 
building_area AS wetland_building_area, drain_density, way
FROM subwards_info_wetland_meters
/* Streamlining final deliverable by only selecting essential columns */

CREATE TABLE final_data_ranked AS
SELECT*, ntile(4)over(ORDER BY drain_density) AS rank
FROM final_data
/* Creates a table with an internal quantile rank of subwards based upon the drain 
density data */

ALTER TABLE wetland_subward ADD COLUMN wetland INTEGER;
UPDATE wetland_subward
SET wetland = 1 
FROM final_data_ranked
WHERE (st_intersects(wetland_subward.way, final_data_ranked.way)
AND NOT st_touches(wetland_subward.way, final_data_ranked.way));
CREATE TABLE wetlands_focused AS
SELECT row_number() OVER () AS id, st_union(way) AS way
FROM wetland_subward
WHERE wetland = 1
GROUP BY wetland
/* In preparation for a final deliverable, creates a table in which all the wetlands that 
intersect and not just touch subwards with final drain data is a single polygon */

CREATE TABLE final_condensed AS
SELECT subward_id, drain_density, rank, way
FROM final_data_ranked
/* Further streamlining of essential columns for a final deliverable */

UPDATE final_condensed 
SET drain_density = round(drain_density::NUMERIC,6)
/* Rounds the drain density data to 6 numerals to further limit data footprint of the 
final deliverable */

```

Here is the [SQL file](https://github.com/Ian8VT/Ian8VT.github.io/blob/master/sqlprocess.sql) for this workflow. 

### Displaying Results
To display the wetland drain density data, I developed a map using [Leaflet](https://leafletjs.com/) software. This map breaks the subwards into four quantile ranks ordered by the density of drain length to building area. Here is the [HTML script](https://github.com/Ian8VT/Ian8VT.github.io/blob/master/dsmmap/dsmmap/index.html) for the interactive Leaflet map.

[map_trial](dsmmap/index.html)

[Return](qgis/qgis.md)
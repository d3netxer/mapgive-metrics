# mapgive-metrics

## Reading this post to generate some metrics

# Querying OpenStreetMap with Amazon Athena
https://aws.amazon.com/blogs/big-data/querying-openstreetmap-with-amazon-athena/#more-2069

What metrics do we want to generate?

### number changesets created

```
SELECT COUNT(*)
FROM changesets
WHERE regexp_like(tags['comment'], '(?i)#mapgive')
```

result: 164,302

### number of distinct mappers

```
SELECT COUNT(DISTINCT uid)
FROM changesets
WHERE regexp_like(tags['comment'], '(?i)#mapgive')
```

result: 7,546

### number of buildings created

```
SELECT COUNT(planet.changeset)
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND planet.tags['building'] IS NOT NULL;
```

result: 1,485,146

note: Use planet.tags['building'] IS NOT NULL vs. planet.tags['building'] IN ('yes');
It returns more results (1,485,146 vs 1,360,720), and is consistent with the queries found below.

### count of highways

```
SELECT COUNT(planet.changeset)
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND regexp_like(planet.tags['highway'], '(?i).')
```

result: 144,859

#### update: We have decided to query from the planet_history file. This will enable to get all edits ever (i.e. edits to buildings that have been subsequently modified). Here is the example of buildings from the planet_history file:

```
select count(planet_history.id)
from planet_history
join changesets on planet_history.changeset = changesets.id
where regexp_like(changesets.tags['comment'], '(?i)#mapgive') and planet_history.type = 'way' and planet_history.tags['building'] is not null
```

Dec 11, 2018 result: 2,930,728

##### The query above only counts modifications to the way itself (i.e. metadata or lists of nodes), if a building vertex has been shifted (node moved), it won't get captured here. Therefore you could create a query that would find all buildings through looking at all the modified nodes first. ex:

```
WITH changesets AS (
  SELECT
    changesets.id,
    min_lat,
    max_lat,
    min_lon,
    max_lon
  FROM changesets
  WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive')
),
nodes_modified AS (
  SELECT
    c.id changeset,
    nodes.id,
    nodes.timestamp
  FROM planet_history nodes
  INNER JOIN changesets c ON nodes.changeset = c.id
  WHERE nodes.type = 'node'
),
buildings AS (
  SELECT
    id,
    nds,
    "timestamp",
    version
  FROM planet_history ways
  WHERE type = 'way'
    AND tags['building'] IS NOT NULL
),
referenced_buildings AS (
  SELECT DISTINCT
    buildings.id,
    nodes.changeset,
    nodes.timestamp
  FROM buildings
  CROSS JOIN UNNEST(nds) WITH ORDINALITY AS t (nd, idx)
  INNER JOIN nodes_modified nodes ON nodes.id = nd.ref
),
final_count AS (
  SELECT  a.*
  FROM    referenced_buildings A
          INNER JOIN
          (
              SELECT id, MIN(timestamp) minDate
              FROM    referenced_buildings
              GROUP BY id
          ) B on a.id = b.id AND
                  a.timestamp = b.minDate
)
SELECT COUNT(*)
FROM final_count
```

Dec 11, 2018 result: 3,033,217

### Kilometers of roads created 

This query returns all of the highways, but how can I display them on GIS software, and how can I calculate the km of roads created?

```
SELECT planet.*, changesets.id
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND regexp_like(planet.tags['highway'], '(?i).')
```

This query will only return the ids of all of the highways with the MapGive hashtag. Then try fetching the highways from overpass.

```
SELECT planet.id
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND regexp_like(planet.tags['highway'], '(?i).')
```
result: 144,859 (1.7 mb file)

In OverPass you can create a line for each highway id you want to query. ex:

```
way(id:4821271,
4823828,
44329138,
176735342,
176738254,
479583192,
479583193,
479583195,
479583196,
479583197
);

/*added by auto repair*/
(._;>;);
/*end of auto repair*/
out;
```

...

so you will have to manupilate the CSV to look like the lines above then paste into an OverPass Turbo query. Using OverPass Turbo I could only query 40,000 ways at a time, so I made 4 seperate queries and exported them as geojson.

The next step is loading the geojson into PostGIS. Use (ogr2ogr)[https://morphocode.com/using-ogr2ogr-convert-data-formats-geojson-postgis-esri-geodatabase-shapefiles/] to do this. Here is an example to create a new database and importing the 1st geojson file:

```
ogr2ogr -f "PostgreSQL" PG:"dbname=mapgive_highways1 user=postgres password=postgres" "1_40k.geojson"
```

Then append the rest of the geojson files:

```
ogr2ogr -f "PostgreSQL" PG:"dbname=mapgive_highways1 user=postgres password=postgres" "40_80k.geojson" -nln ogrgeojson -append
```

Now in PostGIS calculate the total distance of all of your ways:

SELECT SUM(ST_Length(wkb_geometry::geography, true))
FROM public.ogrgeojson

The result will be in meters (53566249.2930868). Convert to km for final result: 53,566 km

### Map of all buildings created

example form American Red Cross with building points:
https://api.mapbox.com/styles/v1/americanredcross/ciyj2djn800222socis30umwr.html?title=true&access_token=pk.eyJ1IjoiYW1lcmljYW5yZWRjcm9zcyIsImEiOiJzdHVRWjA4In0.bnfdwZhKX8tQeMkwY-kknQ#12.37/-1.5454/33.8255

This query works from Seth:

```
WITH buildings AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive')
    AND planet.type = 'way'
    AND planet.tags['building'] IS NOT NULL
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  buildings.id,
  buildings.tags,
  AVG(nodes.lat) lat,
  AVG(nodes.lon) lon
FROM buildings
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (buildings.type, buildings.id, buildings.tags)
```

result: 1,485,146 (88mb file)

something like this can be used to count the total records returned https://stackoverflow.com/questions/5146978/count-number-of-records-returned-by-group-by):

```
WITH buildings AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive')
    AND planet.type = 'way'
    AND planet.tags['building'] IS NOT NULL
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  count(*) RecordsPerGroup,
  COUNT(*) OVER () AS TotalRecords
FROM buildings
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (buildings.type, buildings.id, buildings.tags)
```

result: 1,485,146

I was able to the 88mb building point file and upload it to Carto through the website, then create a visualization: https://tgertin.carto.com/viz/96b70c67-e6c3-4196-81be-d876c6f4a068/public_map

### line chart of edits over time (look into processing csv with python pandas)?

count of MapGive ways and amenity points

```
SELECT COUNT(planet.changeset)
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way') 
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL)
```

result: 1,721,856

dataset MapGive ways and amenity points with an id and timestamp column

```
SELECT planet.id, planet.timestamp
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way') 
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL)
```

size: 63 mb file


### MapGive ways and amenity points with lat, lon, and timestamp column

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL)
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  features.id,
  features.timestamp,
  AVG(nodes.lat) lat,
  AVG(nodes.lon) lon
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.type, features.id, features.timestamp)
```

size: 106 mb file

### Count of MapGive ways and amenity points with lat, lon, and timestamp column

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL)
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  /* each row's RecordsPerGroup number will represent the number of nodes belonging to that feature */
  count(*) RecordsPerGroup,
  /* this counts the total features (or the number of rows) */
  COUNT(*) OVER () AS TotalRecords
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
/* This line below groups the nodes by features */
GROUP BY (features.type, features.id, features.timestamp)
```

count: 1,768,338 (date: July 3, 2017)


### OSMGeoWeek 2017 ways and amenity points with lat, lon, and timestamp column

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2017') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2017') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL)
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  features.id,
  features.timestamp,
  AVG(nodes.lat) lat,
  AVG(nodes.lon) lon
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.type, features.id, features.timestamp)
order by features.timestamp DESC
```

### OSMGeoWeek 2017 count of features

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2017') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2017') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL)
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  count(*) RecordsPerGroup,
  COUNT(*) OVER () AS TotalRecords
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.type, features.id, features.timestamp)
```

### OSMGeoWeek 2017 count of features, between two dates

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (planet.timestamp BETWEEN timestamp '2017-11-12 00:00:00.000' AND timestamp '2017-11-19 00:00:00.000') AND ((regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2017') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2017') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL))
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  count(*) RecordsPerGroup,
  features.timestamp,
  COUNT(*) OVER () AS TotalRecords
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.type, features.id, features.timestamp)
```

result: 292,856


### OSMGeoWeek 2016 count of features, between two dates

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (planet.timestamp BETWEEN timestamp '2017-11-12 00:00:00.000' AND timestamp '2017-11-20 00:00:00.000') AND ((regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2016') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#osmgeoweek2016') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL))
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  count(*) RecordsPerGroup,
  features.timestamp,
  COUNT(*) OVER () AS TotalRecords
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.type, features.id, features.timestamp)
```

result: 110,006

### MapGive count of features, between two dates
```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (planet.timestamp BETWEEN timestamp '2017-12-01 00:00:00.000' AND timestamp '2018-03-01 00:00:00.000') AND ((regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way')
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'node' AND planet.tags['amenity'] IS NOT NULL))
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  /* counts number of nodes per feature */
  count(*) RecordsPerGroup,
  features.timestamp,
  /* counts total number of features */
  COUNT(*) OVER () AS TotalRecords
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.type, features.id, features.timestamp)
```

### MapGive buildings and highways with id, lat, lon, timestamp, and column classifying as either building or highway

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND planet.tags['building'] IS NOT NULL)
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND planet.tags['highway'] IS NOT NULL)
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  features.id,
  features.timestamp,
  CASE WHEN features.tags['building'] IS NOT NULL THEN 'building' 
       WHEN features.tags['highway'] IS NOT NULL THEN 'highway' 
       ELSE NULL END AS building_or_hwy,
  AVG(nodes.lat) lat,
  AVG(nodes.lon) lon
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.id, features.timestamp,features.tags)
```

### MapGive buildings and highways with id, lat, lon, timestamp, user, and column classifying as either building or highway

```
WITH features AS (
  SELECT planet.*
  FROM planet
  JOIN changesets ON planet.changeset = changesets.id
  WHERE (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND planet.tags['building'] IS NOT NULL)
    OR (regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND planet.tags['highway'] IS NOT NULL)
),
nodes_in_bbox AS (
  SELECT *
  FROM planet
  WHERE type = 'node'
)
SELECT
  features.id,
  features.timestamp,
  features.user,
  CASE WHEN features.tags['building'] IS NOT NULL THEN 'building' 
       WHEN features.tags['highway'] IS NOT NULL THEN 'highway' 
       ELSE NULL END AS building_or_hwy,
  AVG(nodes.lat) lat,
  AVG(nodes.lon) lon
FROM features
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (features.id, features.timestamp,features.tags,features.user)
```


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

way(313913835);
way(314205679);
way(314205680);
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

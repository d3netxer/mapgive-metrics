# mapgive-metrics

## Reading this post to generate some metrics

# Querying OpenStreetMap with Amazon Athena
https://aws.amazon.com/blogs/big-data/querying-openstreetmap-with-amazon-athena/#more-2069

What metrics do we want to generate?

- number changesets have been created

```
SELECT COUNT(*)
FROM changesets
WHERE regexp_like(tags['comment'], '(?i)#mapgive')
```

result: 164,302

- number of distinct mappers

```
SELECT COUNT(DISTINCT uid)
FROM changesets
WHERE regexp_like(tags['comment'], '(?i)#mapgive')
```

result: 7,546

- number of buildings created

```
SELECT COUNT(planet.changeset)
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND planet.tags['building'] IN ('yes');
```

result: 1,360,720

- count of highways

```
SELECT COUNT(planet.changeset)
FROM planet
JOIN changesets ON planet.changeset = changesets.id
WHERE regexp_like(changesets.tags['comment'], '(?i)#mapgive') AND planet.type = 'way' AND regexp_like(planet.tags['highway'], '(?i).')
```

result: 144,859

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

- map of buildings created?

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

- map of all ways and nodes created?

- line chart of edits over time (look into processing csv with python pandas)

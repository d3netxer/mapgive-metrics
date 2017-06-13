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

I tried something like this, but it gave an error: Query exhausted resources at this scale factor

```
-- select out nodes and relevant columns
WITH nodes AS (
  SELECT
    type,
    id,
    tags,
    lat,
    lon
  FROM planet
  WHERE type = 'node'
),
-- select out ways and relevant columns
ways AS (
  SELECT
    type,
    id,
    tags,
    nds
  FROM planet
  WHERE type = 'way'
    AND tags['building'] IN ('yes')
),
-- filter nodes to only contain those present within a bounding box
nodes_in_bbox AS (
  SELECT *
  FROM nodes
  WHERE lon BETWEEN -15.0863 AND -7.3651
    AND lat BETWEEN 4.3531 AND 12.6762
)
-- find ways intersecting the bounding box
SELECT
  ways.type,
  ways.id,
  ways.tags,
  AVG(nodes.lat) lat,
  AVG(nodes.lon) lon
FROM ways
CROSS JOIN UNNEST(nds) AS t (nd)
JOIN nodes_in_bbox nodes ON nodes.id = nd.ref
GROUP BY (ways.type, ways.id, ways.tags)
```

- map of all ways and nodes created?

- line chart of edits over time (look into processing csv with python pandas)

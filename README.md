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

- map of buildings created?

example form American Red Cross with building points:
https://api.mapbox.com/styles/v1/americanredcross/ciyj2djn800222socis30umwr.html?title=true&access_token=pk.eyJ1IjoiYW1lcmljYW5yZWRjcm9zcyIsImEiOiJzdHVRWjA4In0.bnfdwZhKX8tQeMkwY-kknQ#12.37/-1.5454/33.8255



- map of all ways and nodes created?

- line chart of edits over time (look into processing csv with python pandas)

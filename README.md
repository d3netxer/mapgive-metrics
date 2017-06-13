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

- map of buildings created?

- map of all ways and nodes created?

- line chart of edits over time

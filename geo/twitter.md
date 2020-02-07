# Let's Analyze Our Twitter Data Set

### Lets look at what data we have that has non-null geo coordindates
```sql
\x

SELECT id, created_at, full_text, user_name, 
  user_location, coordinates
FROM tweets WHERE 
json_typeof(coordinates) <> 'null' 
LIMIT 100;
```

### Now lets get all the tweets with geo data and enrich it by adding a postgis geometry column too 
```sql
DROP TABLE IF EXISTS geotest;
CREATE TABLE geotest AS
SELECT *, 
ST_SetSRID( ST_GeomFromGeoJSON(coordinates::text), 4326) geom 
FROM tweets 
WHERE json_typeof(coordinates) <> 'null';
```

```sql
\x 

SELECT * FROM geotest
LIMIT 3;
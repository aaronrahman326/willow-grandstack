# Data Import

## Parcels

Import county level parcel data (in this case for Gallatin, MT):

Download county level shapefile

```shell
wget ftp://ftp.geoinfo.msl.mt.gov/Data/Spatial/MSDI/Cadastral/Parcels/Gallatin/GallatinOwnerParcel_shp.zip
```

Convert shapefile to geojson, using [GDAL command line tool](https://gdal.org/index.html)

```shell
ogr2ogr gallatin.geojson -f "GeoJSON" -lco id_field=PARCELID -t_srs "EPSG:4326" GallatinOwnerParcel_shp.shp
```

**Import into Neo4j**

Create database uniqueness constraint for `Property` nodes

```cypher
CREATE CONSTRAINT ON (p:Property) ASSERT p.id IS UNIQUE
```

Import `Property` nodes

```cypher
CALL apoc.load.json("file:///gallatin.geojson") YIELD value
FOREACH (feat IN  value.features |
    MERGE (p:Property {id: feat.id})
    SET p += feat.properties
)
```

Create database uniqueness constraint for `City` nodes

```cypher
CREATE CONSTRAINT ON (c:City) ASSERT c.name IS UNIQUE;
```

Add `City` nodes and connect to `Property` nodes

```cypher
MATCH (p:Property) WHERE EXISTS(p.CityStateZ)
WITH p, split(p.CityStateZ,",")[0] AS cityname
MERGE (c:City {name: toUpper(cityname)})
MERGE (p)-[:IN_CITY]->(c)
```

What is the average property value per city?

```cypher
MATCH (c:City)<-[:IN_CITY]-(p:Property)
WITH c.name as city, avg(p.TotalValue) as avgValue
RETURN * ORDER BY avgValue DESC
```

**Working with geospatial data**


Filter out any MultiPolygon geometries, we'll deal with those later

```shell
jq '.features[] | select(.geometry.type=="Polygon") | [.]' gallatin.geojson > filtered.geojson
```

Add polygon property - list of `Point` types

```cypher
CALL apoc.load.json("file:///filtered.geojson") YIELD value
    MATCH (p:Property {id: value.id})
    SET p.polygon = apoc.cypher.runFirstColumnSingle('UNWIND $coords AS coord RETURN COLLECT(Point({latitude: coord[1], longitude: coord[0]}))', {coords: value.geometry.coordinates[0]})
```

Query for property given a latitude and longitude using [spatial algorithms library](https://github.com/neo4j-contrib/spatial-algorithms/)

```cypher
MATCH (p:Property)
WHERE spatial.algo.withinPolygon(Point({latitude:46.88110604, longitude:-113.99735408}), p.polygon)
RETURN p
```

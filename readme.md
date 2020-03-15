## What it is

A map of where you may be able to build ADUs in Ann Arbor

## How to do it

## Data sources

* City boundaries, zoning districts: [City data portal](https://www.a2gov.org/services/data/Pages/default.aspx)
* Parcels: [Washtenaw County](https://gisappsecure.ewashtenaw.org/mapwashtenaw/)

### Vector tiles

Using a protobuf Open Street Map [extract from protomaps](https://protomaps.com/extracts/acd533f7-3ce7-43a6-8547-01a30a756df3)

I used [tilemaker](https://github.com/systemed/tilemaker) to make the basemap tiles

### City boundaries

Load into Postgres:
```
ogr2ogr -f PostgreSQL pg:'user=matth password= host=localhost port=5432 dbname=loveland_geo' -nln ann_arbor -s_srs 'epsg:2253' -t_srs 'epsg:4326' -nlt GEOMETRY -lco precision=NO /Users/matth/Downloads/AA_City_Boundary/AA_City_Boundary.shp
```

Get the concave hull:

```
select ST_AsGeoJSON(ST_ConcaveHull(ST_Collect(wkb_geometry), 0.99), 6) from ann_arbor;
```


### Adding zoning to parcels

Load the shapefile to Postgres:

```
ogr2ogr -f PostgreSQL pg:'user=matth password= host=localhost port=5432 dbname=loveland_geo' -nln mi_washtenaw_zoning -s_srs 'epsg:2253' -t_srs 'epsg:4326' -nlt GEOMETRY -lco precision=NO /Users/matth/Downloads/AA_Zoning_Districts/AA_Zoning_Districts.shp

create index mi_washtenaw_zoning_idx on mi_washtenaw_zoning using gist(wkb_geometry);

update mi_washtenaw p
set zoning = zoningclas
from mi_washtenaw_zoning b
where st_intersects(st_centroid(p.wkb_geometry), b.wkb_geometry);
```

###  Render the parcels to tiles using tippecanoe

First, export the data:

```
ogr2ogr -f GeoJSON mi_washtenaw_parcels.geojson pg:'user=matth password= host=localhost port=5432 dbname=loveland_geo' mi_washtenaw -sql "select * from mi_washtenaw where city='ann-arbor'"
```

Render the parcels with tippecanoe:

```
tippecanoe --output-to-directory=parcel_tiles mi_washtenaw_parcels.geojson --read-parallel --include=address --include=zoning --include=ll_gissqft --no-tile-compression -X -P -g2 -f --coalesce-densest-as-needed --detect-shared-borders --no-tiny-polygon-reduction
```

To copy it over (I render it in a different directory)

```
rm -r ../adus/parcel_tiles/; mv parcel_tiles/ ../adus/
```

## Notes and ackowledgements

https://github.com/mapbox/tippecanoe/issues/311#issuecomment-255442483

Layer name will be mi_washtenaw_parcels

Thanks to [klokantech/mapbox-gl-js-offline-example](https://github.com/klokantech/mapbox-gl-js-offline-example)
for examples

Thanks to [this codepen for mask inspiration](https://jsfiddle.net/kmandov/cr2rav7v)
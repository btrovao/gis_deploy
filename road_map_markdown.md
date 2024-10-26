# Deploying a geographic information systems application

## OBJECTIVES:
- Deploy a GIS application for monitoring fire alerts using OpenStreetMap and Geoserver data
- Characterizing Indigenous Lands (villages, tracks, roads, rivers)

## 0. Configuring PostgreSQL
### 0.1 Enabling Extensions
```sql
CREATE EXTENSION postgis;
CREATE EXTENSION ogr_fdw;
CREATE EXTENSION postgres_fdw;
```
### 0.4 Configuring `hba.conf`
```plaintext
host    all             all             0.0.0.0/0               md5
```
### 0.5 Configuring `postgresql.conf`
[PGTune](https://pgtune.leopard.in.ua/)
### 0.6 Create Foreign Servers
[pgsql-ogr-fdw](https://github.com/pramsey/pgsql-ogr-fdw)
```sql
CREATE SERVER geoserver_funai
    FOREIGN DATA WRAPPER ogr_fdw
    OPTIONS (datasource 'WFS:https://geoserver.funai.gov.br/geoserver/ows?version=1.0.0', format 'WFS', config_options 'GDAL_HTTP_UNSAFESSL=yes');

CREATE SERVER nasa_firms
    FOREIGN DATA WRAPPER ogr_fdw
    OPTIONS (datasource 'WFS:https://firms.modaps.eosdis.nasa.gov/mapserver/wfs/South_America/your_key/?SERVICE=WFS&REQUEST=GetCapabilities&VERSION=1.0.0', format 'WFS', config_options 'GDAL_HTTP_UNSAFESSL=yes');

-- Connect to an external DB (Optional)
CREATE SERVER foreign_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'XXXXX', port '5432', dbname 'postgres');

CREATE USER MAPPING FOR current_user
    SERVER foreign_server
    OPTIONS (user 'remote_user', password 'remote_password');
```

## 1. Downloading Data
### 1.1 OSM Data
[Geofabrik](https://download.geofabrik.de/)
### 1.2 Fire Alerts
[NASA Firms](https://firms.modaps.eosdis.nasa.gov/mapserver/wfs-info/)
### 1.3 Fire Alerts BDQueimadas
- Download CSV [BDQueimadas](https://terrabrasilis.dpi.inpe.br/queimadas/bdqueimadas/#exportar-dados)

## 2. Loading Data
### 2.1 OSM Data
#### 2.1.1 Selecting AOI Extent
```plaintext
-54.01,-13.00,-52.53,-10.60
```
#### 2.1.2 Clipping with `osmconvert`
[osmconvert](https://wiki.openstreetmap.org/wiki/Osmconvert)
```bash
# Linux
osmconvert brazil-latest.pbf -b="-54.01,-13.00,-52.53,-10.60" -o=xingu --drop-author --drop-version

# Windows
.\osmconvert64-0.8.8p brazil-latest.pbf -b="-54.01,-13.00,-52.53,-10.60" -o=xingu --drop-author --drop-version
```
#### 2.1.3 Opening with QGIS
- Drag and drop the output "xingu" into QGIS
- Import to the schema 'osm' in postgres with QGIS database manager tool (create spatial index)

### 2.2 Fire Alerts
#### 2.2.1 Creating Schema
```sql
CREATE SCHEMA fire_alerts;
```
#### 2.2.2 Importing Data from NASA Firms
```sql
IMPORT FOREIGN SCHEMA ogr_all FROM SERVER nasa_firms INTO fire_alerts;
```
#### 2.2.3 Importing Data from BDQueimadas (Brazil)
- Import to QGIS with "Add delimited text layer", explore and adjust the geometry configuration
- Export to Postgres with QGIS database manager tool (create spatial index)

### 2.3 Indigenous Lands
#### 2.3.1 Creating Schema
```sql
CREATE SCHEMA indigenous_lands;
```
#### 2.3.2 Importing Data
```sql
IMPORT FOREIGN SCHEMA ogr_all FROM SERVER geoserver_funai INTO indigenous_lands;
```
#### 2.3.3 Creating Materialized View in the AOI (for large processing tasks)
```sql
CREATE MATERIALIZED VIEW funai_f.mv_xingu_reserve AS
SELECT a.* 
FROM funai_f.funai_tis_poligonais_portarias a
WHERE a."terrai_codigo" IN (6001, 33801, 49801, 55201);
```

### 2.4 (Optional) OSM Data from Other Database
#### 2.4.1 Creating Schema
```sql
CREATE SCHEMA osm_f;
```
#### 2.4.2 Importing Data
```sql
IMPORT FOREIGN SCHEMA osm FROM SERVER bd_win INTO osm_f;
```

## 3. Data Preparation
### 3.1 Loading data through Materialized Views in the AOI
[PostGIS Tips](https://postgis.net/documentation/tips/tip_intersection_faster/)
```sql
CREATE MATERIALIZED VIEW funai_f.mv_xingu_reserve AS
SELECT fid, gml_id, gid, terrai_codigo, terrai_nome, etnia_nome, st_area(the_geom::geography, true)/10000 AS area_ha,           
CASE
    WHEN st_isvalid(st_collectionextract(st_buffer(a.the_geom, 0::double precision), 3)) IS FALSE 
    THEN st_setsrid(st_multi(st_collectionextract(st_makevalid(st_buffer(a.the_geom, 0.0000000::double precision)), 3)), 4674)::geometry(MultiPolygon,4674)
    ELSE st_setsrid(st_multi(st_collectionextract(st_buffer(a.the_geom, 0::double precision), 3)), 4674)::geometry(MultiPolygon,4674) 
END AS the_geom
FROM funai_f.funai_tis_poligonais_portarias a
WHERE a."terrai_codigo" IN (6001, 33801, 49801, 55201);

CREATE MATERIALIZED VIEW fire_alerts.mv_xingu_fire_7days AS
SELECT a.*, b."terrai_codigo"
FROM fire_alerts.fires_modis_7days a 
JOIN funai_f.mv_xingu_reserve b ON a.points && st_transform(b.the_geom, 4326);
```

### 3.2 Creating Spatial Indexes
#### 3.2.1 Indigenous Lands
```sql
CREATE INDEX idx_mv_xingu_reserve_geom ON funai_f.mv_xingu_reserve USING GIST (the_geom);
```
#### 3.2.2 Fire Alerts
```sql
CREATE INDEX idx_mv_xingu_fire_7days_geom ON fire_alerts.mv_xingu_fire_7days USING GIST (geom);
```

## 4.0 Creating Triggers, functions and materialized views
### 4.1 Trigger for Update Timestamp
```sql

--Adding a timestamp
CREATE OR REPLACE FUNCTION osm.update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_timestamp_trigger
BEFORE UPDATE ON osm.xingumultipolygons
FOR EACH ROW
EXECUTE FUNCTION osm.update_timestamp();

-- Correcting geometries 
CREATE OR REPLACE FUNCTION osm.invalid()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
BEGIN
	if not st_isValid(NEW.geom) THEN
	NEW.geom = (ST_multi(ST_buffer((ST_makevalid(NEW.geom)),0))); 
	RETURN NEW;
 else    
      RETURN NEW;
    END IF;

END;
$function$
;

CREATE TRIGGER invalid
BEFORE UPDATE ON osm.xingumultipolygons
FOR EACH ROW
EXECUTE FUNCTION osm.invalid();
```

### 4.2 Creating Function (Calculate Distance to POIs: Roads, Tracks, Rivers, Villages, etc)
```sql

--Finding the closest POIs from the fire alerts points

CREATE OR REPLACE FUNCTION osm_f.find_closest_pois(input_geom geometry)
RETURNS TABLE(result json)
LANGUAGE plpgsql
AS $function$
BEGIN
    -- Ensure the input geometry is valid and transform to EPSG:4326
    input_geom := ST_Transform(ST_MakeValid(input_geom), 4326);
    -- Find the closest POI in OSM data from each geom and return as JSON objects
    RETURN QUERY    
    SELECT              
        json_build_object(
            'type_dist', 'Villages',
            'coordinates', ST_AsGeoJSON(st_shortestline(input_geom, mv.geom))::json->'coordinates',                        
            'distance_km', ST_Distance(input_geom::geography, mv.geom::geography) / 1000
        ) AS result     
    FROM osm.xingupoints mv
    WHERE mv.place IN ('hamlet','isolated_dwelling','village')
    ORDER BY ST_Distance(input_geom::geography, mv.geom::geography) ASC
    LIMIT 3;

    RETURN QUERY    
    SELECT              
        json_build_object(
            'type_dist', 'Roads',
            'coordinates', ST_AsGeoJSON(st_shortestline(input_geom, mv.geom))::json->'coordinates',                        
            'distance_km', ST_Distance(input_geom::geography, mv.geom::geography) / 1000
        ) AS result     
    FROM osm.xingulines mv
    WHERE mv.highway IN ('residential','secondary','service','tertiary','unclassified')
    ORDER BY ST_Distance(input_geom::geography, mv.geom::geography) ASC
    LIMIT 3;

    RETURN QUERY    
    SELECT              
        json_build_object(
            'type_dist', 'Other ways',
            'coordinates', ST_AsGeoJSON(st_shortestline(input_geom, mv.geom))::json->'coordinates',                        
            'distance_km', ST_Distance(input_geom::geography, mv.geom::geography) / 1000
        ) AS result     
    FROM osm.xingulines mv
    WHERE mv.highway IN ('footway','path','track')
    ORDER BY ST_Distance(input_geom::geography, mv.geom::geography) ASC
    LIMIT 3;                
END;
$function$;
```

#### Expected Output
```sql
select osm_f.find_closest_pois(a.points)
from fire_alerts.mv_xingu_fire_7days a
where fid = 1845
order by acq_datetime desc 

{"type_dist" : "Villages", "coordinates" : [[-53.16825,-12.41827],[-52.89841,-11.148044]], "distance_km" : 0}
{"type_dist" : "Villages", "coordinates" : [[-53.16825,-12.41827],[-52.9022289,-11.8684727]], "distance_km" : 0}
{"type_dist" : "Villages", "coordinates" : [[-53.16825,-12.41827],[-52.7357819,-10.803888]], "distance_km" : 0}
{"type_dist" : "Roads", "coordinates" : [[-53.16825,-12.41827],[-52.9452942,-11.4583474]], "distance_km" : 0}
{"type_dist" : "Roads", "coordinates" : [[-53.16825,-12.41827],[-52.840857,-12.0458308]], "distance_km" : 0}
{"type_dist" : "Roads", "coordinates" : [[-53.16825,-12.41827],[-52.7683778,-11.6598491]], "distance_km" : 0}
{"type_dist" : "Other ways", "coordinates" : [[-53.16825,-12.41827],[-52.5643149,-12.8966368]], "distance_km" : 0}
{"type_dist" : "Other ways", "coordinates" : [[-53.16825,-12.41827],[-52.9291141,-12.0447591]], "distance_km" : 0}
{"type_dist" : "Other ways", "coordinates" : [[-53.16825,-12.41827],[-52.5329371,-12.804805]], "distance_km" : 0}
```

### 4.3 Creating materialized view for main results
```sql
CREATE MATERIALIZED VIEW fire_alerts.mv_ind_lands_alerts_7days
TABLESPACE pg_default
AS SELECT row_number() OVER () AS row_number,
    w.terrai_nome,
    w.terrai_codigo,
    w.ind_land_ha,
    count(w.fid) AS alerts_amount,
    array_agg(w.acq_date) AS acq_dates,
    round((st_area(st_union(st_transform(w.geom, 29101)))::integer / 10000)::numeric, 0) AS sob_fire_px_ha,
    round((st_area(st_union(st_transform(w.geom, 29101)))::integer / 10000)::numeric / w.ind_land_ha * 100::numeric, 4) AS perc_fire_px,
    st_collect(w.geom) AS geom_fire
   FROM ( SELECT b.fid,
            b.acq_date,
            round(st_area(st_transform(a.the_geom, 29101))::numeric / 10000::numeric, 0) AS ind_land_ha,
            a.terrai_nome,
            a.terrai_codigo,
                CASE
                    WHEN st_coveredby(st_transform(st_envelope(st_buffer(st_transform(b.points, 29101), 500::double precision)), 4326), st_transform(a.the_geom, 4326)) THEN st_transform(st_envelope(st_buffer(st_transform(b.points, 29101), 500::double precision)), 4326)
                    ELSE st_intersection(st_transform(st_envelope(st_buffer(st_transform(b.points, 29101), 500::double precision)), 4326), st_transform(a.the_geom, 4326))
                END AS geom
           FROM funai_f.mv_xingu_reserve a
             LEFT JOIN fire_alerts.mv_xingu_fire_7days b ON st_intersects(st_transform(a.the_geom, 4326), st_transform(st_envelope(st_buffer(st_transform(b.points, 29101), 500::double precision)), 4326))
          GROUP BY b.fid, b.acq_date, a.terrai_nome, a.the_geom, a.terrai_codigo, b.points) w
  GROUP BY w.terrai_nome, w.terrai_codigo, w.ind_land_ha
WITH DATA;
```
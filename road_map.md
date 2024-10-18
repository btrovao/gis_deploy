
 
OBJECTIVES:
Deploy a GIS application for monitoring fire alerts using OpenStreetMap and Geoserver data
Characterizing Indigenous Lands (villages, tracks, roads, rivers)


/*
0- Configuring postgresql
    0.1 - Enabling Extensions
        0.1.1 - CREATE EXTENSION postgis;
        0.1.2 - CREATE EXTENSION ogr_fdw;
        0.1.3 - CREATE EXTENSION postgres_fdw;
    0.4 - Configuring hba.conf 
        '(host    all             all             0.0.0.0/0               md5)'      
    0.5 - Configuring postgresql.conf 
            '(https://pgtune.leopard.in.ua/)'
    0.6 - Create foreign servers (https://github.com/pramsey/pgsql-ogr-fdw)
        0.6.1 - CREATE SERVER geoserver_funai
        	FOREIGN DATA WRAPPER ogr_fdw
            OPTIONS (datasource 'WFS:https://geoserver.funai.gov.br/geoserver/ows?version=1.0.0', format 'WFS', config_options 'GDAL_HTTP_UNSAFESSL=yes');
        0.6.2 - CREATE SERVER nasa_firms
            FOREIGN DATA WRAPPER ogr_fdw
            OPTIONS (datasource 'WFS:https://firms.modaps.eosdis.nasa.gov/mapserver/wfs/South_America/your_key/?SERVICE=WFS&REQUEST=GetCapabilities&VERSION=1.0.0', format 'WFS', config_options 'GDAL_HTTP_UNSAFESSL=yes');
        0.6.3 - (Connect to an external DB - Optional) CREATE SERVER foreign_server
	        'FOREIGN DATA WRAPPER postgres_fdw
	        OPTIONS (host 'XXXXX', port '5432', dbname 'postgres');'
            'CREATE USER MAPPING FOR current_user
            SERVER foreign_server
            OPTIONS (user 'remote_user', password 'remote_password');'

1- Downloading data
    1.1 - OSM data (https://download.geofabrik.de/)
    1.2 - Fire alerts (https://firms.modaps.eosdis.nasa.gov/mapserver/wfs-info/)
2- Loading data
    2.1 - OSM data
        2.1.1 - Selecting AOI extent 
            '-54.01,-13.00,-52.53,-10.60'
        2.1.2 - Clipping with osmconvert (https://wiki.openstreetmap.org/wiki/Osmconvert)
            (linux) = 'osmconvert brazil-latest.pbf -b="-54.01,-13.00,-52.53,-10.60"  -o=xingu --drop-author --drop-version' 
            (win) = '.\osmconvert64-0.8.8p brazil-latest.pbf -b="-54.01,-13.00,-52.53,-10.60"  -o=xingu --drop-author --drop-version'            
        2.1.3 - Opening with QGIS
            .Drag and drop the output "xingu" into QGIS
            .Import to Postgres with QGIS database manager tool (create spatial index)
    2.2 - Fire alerts      
        2.2.1 - Creating schema
            'CREATE SCHEMA fire_alerts;'
        2.2.2 - Importing data from Nasa Firms
            'IMPORT FOREIGN SCHEMA public FROM SERVER geoserver_funai INTO fire_alerts;'
        2.2.2 - Imporing data from BDQueimadas (Brazil)
            .Download CSV (https://terrabrasilis.dpi.inpe.br/queimadas/bdqueimadas/#exportar-dados)      
            .Import to Qgis with "Add delimited text layer", explore and adjust the geometry configuration
            .Export to Postgres with QGIS database manager tool (create spatial index)
    2.3 - Indigenous Lands        
        2.3.1 - Creating schema
            'CREATE SCHEMA indigenous_lands;'
        2.3.2 - Importing data
            'IMPORT FOREIGN SCHEMA public FROM SERVER geoserver_funai INTO indigenous_lands;'
        2.2.3 - Creating materialized view in the AOI (for large processing tasks)    
            'CREATE MATERIALIZED VIEW funai_f.mv_xingu_reserve AS
            select a.* from funai_f.funai_tis_poligonais_portarias a
            where a."terrai_codigo" in (6001, 33801, 49801, 55201)'
    2.4 - (Optional) OSM data from other database
       2.4.1 - Creating schema
            'CREATE SCHEMA osm_f;'
        2.3.3 - Importing data
            'IMPORT FOREIGN SCHEMA osm FROM SERVER bd_win INTO osm_f;
3- Data preparation
    3.1 - Creating materialized views in the AOI (make large processing tasks faster, https://postgis.net/documentation/tips/tip_intersection_faster/ ) 
            'CREATE MATERIALIZED VIEW funai_f.mv_xingu_reserve AS
                SELECT fid, gml_id, gid, terrai_codigo, terrai_nome, etnia_nome, st_area(the_geom::geography, true)/10000 as area_ha,           
                CASE
	                WHEN st_isvalid(st_collectionextract(st_buffer(a.the_geom, 0::double precision), 3)) IS FALSE 
	                THEN st_setsrid(st_multi(st_collectionextract(st_makevalid(st_buffer(a.the_geom, 0.0000000::double precision)), 3)), 4674)::geometry(MultiPolygon,4674)
	                ELSE st_setsrid(st_multi(st_collectionextract(st_buffer(a.the_geom, 0::double precision), 3)), 4674)::geometry(MultiPolygon,4674) 
                    end as the_geom
                from funai_f.funai_tis_poligonais_portarias a
                where a."terrai_codigo" in (6001, 33801, 49801, 55201);'     
            'CREATE MATERIALIZED VIEW  fire_alerts.mv_xingu_fire_7days AS
                select a.*, b."terrai_codigo"
                from fire_alerts.fires_modis_7days a 
                join funai_f.mv_xingu_reserve b on a.points && st_transform(b.the_geom, 4326);'
    3.2 - Creating spatial indexes
        3.2.1 - Indigenous lands
            'CREATE INDEX idx_mv_xingu_reserve_geom ON funai_f.mv_xingu_reserve USING GIST (the_geom);'         
        3.2.2 - Fire alerts
            'CREATE INDEX idx_mv_xingu_fire_7days_geom ON fire_alerts.mv_xingu_fire_7days USING GIST (geom);'            
    3.3 - Creating triggers
        3.3.1 - Update timestamp
            'CREATE OR REPLACE FUNCTION update_timestamp()
            RETURNS TRIGGER AS $$
            BEGIN
                NEW.updated_at = NOW();
                RETURN NEW;
            END;
            $$ LANGUAGE plpgsql;'
            'CREATE TRIGGER update_timestamp_trigger
            BEFORE UPDATE ON osm_data
            FOR EACH ROW
            EXECUTE FUNCTION update_timestamp();'

    3.3 - Creating function (calculate distance to POIs: Roads, tracks, rivers, villages, etc)    

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
                                FROM osm_f.xingupoints mv
                                where mv.place in ('hamlet','isolated_dwelling','village')
                                ORDER BY ST_Distance(input_geom::geography, mv.geom::geography) ASC
                                LIMIT 3;
                            RETURN QUERY    
                            SELECT              
                                json_build_object(
                                'type_dist', 'Roads',
                                    'coordinates', ST_AsGeoJSON(st_shortestline(input_geom, mv.geom))::json->'coordinates',                        
                                    'distance_km', ST_Distance(input_geom::geography, mv.geom::geography) / 1000
                                ) AS result     
                                FROM osm_f.xingulines mv
                                where mv.highway in ('residential','secondary','service','tertiary','unclassified')
                                ORDER BY ST_Distance(input_geom::geography, mv.geom::geography) ASC
                                LIMIT 3;
                            RETURN QUERY    
                            SELECT              
                                json_build_object(
                                'type_dist', 'Other ways',
                                    'coordinates', ST_AsGeoJSON(st_shortestline(input_geom, mv.geom))::json->'coordinates',                        
                                    'distance_km', ST_Distance(input_geom::geography, mv.geom::geography) / 1000
                                ) AS result     
                                FROM osm_f.xingulines mv
                                where mv.highway in ('footway','path','track')
                                ORDER BY ST_Distance(input_geom::geography, mv.geom::geography) ASC
                                LIMIT 3;                
                    END;
                    $function$;

                   
        

# JFMP Burn Prioritisation
Updating JFMP prioritisation process for new Bushfire Risk Analysis Framework


## Spatial data preparation in ArcGIS

### 1. Filter JFMP shapefile to remove non-burn fuel management
  > Note: previously, we would have filtered to remove CFA burns but the shapefile provided doesn't have an 'agency' field so they're retained here

### 2. Ensure required fields exist and are populated. These fields are:
 * name        - name of the burn
 * treatment_  - coded id of the burn, including district and type
 * category    - type of burn
 * jfmpyrpr    - proposed year/season of treatment (as single 4 digit numeric year)
 > Note: Capitalisation shouldn't matter (but I had better check this!)

### 3. Clip shapefile to latest treatability layer
 > Note: we really should store treatability in Athena and skip this step

### 4. Convert the JFMP shapefiles to 180m grid data - The slow way
   1. Import 180m grid cells from risk2temp_db

      ```sql
      select 
        cellid, x_coord, y_coord, geom_centroid, delwp_district 
      from 
        risk2temp_db.reference_brau.grid_cell_180m
      where
        state = 'Victoria'
      ```

   2. Join JFMP to 180m grid
   3. Export to shapefile
   4. Import shapefile to risk2temp database
      
      Create a new schema to hold the data and results
      ```sql
      create schema if not exists jfmp_2023_prioritisation
      ```
      
      Upload the shapefile using PostGIS Shapefile Import/Export manager
      <img src="https://user-images.githubusercontent.com/100050237/227848065-9e6c8ea4-d36b-4bf6-8c80-e75c971c4e9c.png" width="500" />
      > Note: You can run this tool without installing PostGIS by downloading the latest zip bundle from http://download.osgeo.org/postgis/windows/ and extracting just the /bin/ folder.
      
      

### 4. Convert the JFMP shapefiles to 180m grid data - The fast way
> Note: ensure that these datasets are projected to VicGrid94 prior to carrying out these steps:
   1. Prepare Spatial SQL versions of the JFMP source datasets using the `shp2pgsql` command. 

      ```bash
      shp2pgsql -I -s 3111 -g geom FuelTreatmentsPlannedBurns.shp jfmp_20_21.jfmp_20200903 > jfmp_20200903.sql
      shp2pgsql -I -s 3111 -g geom JFMP_select_treatable.shp jfmp_20_21.jfmp_treatable_20200903 > jfmp_treatable_20200903.sql
      ```

   2. Insert the JFMP source datasets into the database (command will prompt for database password once connected; use the password from above):
      
      ```bash
      psql -h riskanalysis2018.cr1xujdl5nau.ap-southeast-2.rds.amazonaws.com -d sourcedata -U brau -p 1352 -f jfmp_20200903.sql
      psql -h riskanalysis2018.cr1xujdl5nau.ap-southeast-2.rds.amazonaws.com -d sourcedata -U brau -p 1352 -f jfmp_treatable_20200903.sql
      ```
      
   3. Since the datasets have probably been created using ESRI ArcGIS, the geometries have to be checked and probably fixed. Check for geometry problems with:

      ```sql
      select ST_IsValidReason(geom)
      from jfmp_20_21.jfmp_20200903
      where not ST_IsValid(geom);
      ```
      and
      ```sql
      select ST_IsValidReason(geom)
      from jfmp_20_21.jfmp_treatable_20200903
      where not ST_IsValid(geom);
      ```

   4. There will probably be a bunch of Self-Intersections or Ring Self-Intersections because ArcGIS has a different definition of valid geometry to PostgreSQL. Fix up geometry problems with:

      ```sql
      update jfmp_20_21.jfmp_20200903
      set geom = ST_Multi(ST_Buffer(geom, 0.0))
      where not ST_IsValid(geom);

      update jfmp_20_21.jfmp_treatable_20200903
      set geom = ST_Multi(ST_Buffer(geom, 0.0))
      where not ST_IsValid(geom);
      ```
   5. Join the JFMP shapefile with 180m grid cells 
      ```sql
      drop table if exists jfmp_20_21.jfmp_treatable_xy180;
      create table jfmp_20_21.jfmp_treatable_xy180 as (
         select a.cellid,  b.gid, b.name , b.treatment_ as burnnum, b.fop_year, a.delwp_district, c.regionname as op_region , a.geom_polygon
         from reference_brau.grid_cell_180m as a
		        inner join jfmp_20_21.jfmp_treatable_20200903_grouped as b
			         on ST_Intersects(a.geom_polygon, b.geom)
		        left join reference.delwp_districts as c
			         on a.delwp_district = c.dstrctname
         );
      ```
6. Export

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

### 4. Create a new schema on risk2temp_db
   Create a new schema to hold the data and results
   ```sql
   create schema if not exists jfmp_2023_prioritisation
   ```

### 5. Convert the JFMP shapefiles to 180m grid data - The slow way
   1. Import 180m grid cells from risk2temp_db

      ```sql
      select 
        cellid, x_coord, y_coord, geom_centroid, delwp_district 
      from 
        risk2temp_db.reference_brau.grid_cell_180m
      where
        state = 'Victoria'
      ```

   2. Join JFMP to 180m grid and export to a shapefile

      Select grid cells within JFMP polygons
      
      ```arcpy.management.SelectLayerByLocation("180m_grid", "WITHIN", "JFMP_Draft", None, "NEW_SELECTION", "NOT_INVERT")```
      
      Use Spatial Join to add the JFMP details to selected grid cells
      
      ```arcpy.management.```
      
      Export features
      
      ```arcpy.management. ```
   
   3. Import shapefile to risk2temp database

      Upload the shapefile using PostGIS Shapefile Import/Export manager
      <img src="https://user-images.githubusercontent.com/100050237/227848065-9e6c8ea4-d36b-4bf6-8c80-e75c971c4e9c.png" width="500" />
      > Note: You can run this tool without installing PostGIS by downloading the latest zip bundle from http://download.osgeo.org/postgis/windows/ and extracting just the /bin/ folder.
      
      

### 5. Convert the JFMP shapefiles to 180m grid data - The fast way
> Note: ensure that these datasets are projected to VicGrid94 prior to carrying out these steps.

> Note: You can run shp2pgsql without installing PostGIS by downloading the latest zip bundle from http://download.osgeo.org/postgis/windows/ and extracting just the /bin/ folder.
   1. Prepare Spatial SQL versions of the JFMP source datasets using the `shp2pgsql` command. 

      ```bash
      shp2pgsql -I -s 3111 -g geom FuelTreatmentsPlannedBurns.shp jfmp_2023_prioritisation.jfmp_20200903 > jfmp_20200903.sql
      shp2pgsql -I -s 3111 -g geom JFMP_select_treatable.shp jfmp_2023_prioritisation.jfmp_treatable_20200903 > jfmp_treatable_20200903.sql
      ```

   2. Insert the JFMP source datasets into the database (command will prompt for database password once connected; use the password from above):
      
      ```bash
      psql -h riskanalysis2018.cr1xujdl5nau.ap-southeast-2.rds.amazonaws.com -d sourcedata -U brau -p 1352 -f jfmp_20200903.sql
      psql -h riskanalysis2018.cr1xujdl5nau.ap-southeast-2.rds.amazonaws.com -d sourcedata -U brau -p 1352 -f jfmp_treatable_20200903.sql
      ```
      
   3. Since the datasets have probably been created using ESRI ArcGIS, the geometries have to be checked and probably fixed. Check for geometry problems with:

      ```sql
      select ST_IsValidReason(geom)
      from jfmp_2023_prioritisation.jfmp_20200903
      where not ST_IsValid(geom);
      ```
      and
      ```sql
      select ST_IsValidReason(geom)
      from jfmp_2023_prioritisation.jfmp_treatable_20200903
      where not ST_IsValid(geom);
      ```

   4. There will probably be a bunch of Self-Intersections or Ring Self-Intersections because ArcGIS has a different definition of valid geometry to PostgreSQL. Fix up geometry problems with:

      ```sql
      update jfmp_2023_prioritisation.jfmp_20200903
      set geom = ST_Multi(ST_Buffer(geom, 0.0))
      where not ST_IsValid(geom);

      update jfmp_2023_prioritisation.jfmp_treatable_20200903
      set geom = ST_Multi(ST_Buffer(geom, 0.0))
      where not ST_IsValid(geom);
      ```
   5. Join the JFMP shapefile with 180m grid cells 
      ```sql
      drop table if exists jfmp_2023_prioritisation.jfmp_treatable_xy180;
      create table jfmp_2023_prioritisation.jfmp_treatable_xy180 as (
         select a.cellid,  b.gid, b.name , b.treatment_ as burnnum, b.fop_year, a.delwp_district, c.regionname as op_region , a.geom_polygon
         from reference_brau.grid_cell_180m as a
		        inner join jfmp_2023_prioritisation.jfmp_treatable_20200903_grouped as b
			         on ST_Intersects(a.geom_polygon, b.geom)
		        left join reference.delwp_districts as c
			         on a.delwp_district = c.dstrctname
         );
      ```
   
### 5. Convert the JFMP shapefiles to 180m grid data - The new way

   1. Create a new schema in risk2temp_db to hold/store our data
      
      ```sql
      CREATE SCHEMA jfmp_2022_test;
      ```
   
   2. Import JFMP shapefile to risk2temp database

      Upload the shapefile using PostGIS Shapefile Import/Export manager
      <img src="https://user-images.githubusercontent.com/100050237/227848065-9e6c8ea4-d36b-4bf6-8c80-e75c971c4e9c.png" width="500" />
         > Note: You can run this tool without installing PostGIS by downloading the latest zip bundle from http://download.osgeo.org/postgis/windows/ and extracting just the /bin/ folder.
   
   3. Join JFMP shapefile to 180m grid cells
      
      ```sql
      CREATE TABLE jfmp_2022_test.jfmp_xy180 AS
          SELECT
              a.cellid,
              b.name,
              b.treatment_ as burnnum,
              b.JFMPYEARpr as jfmp_year,
              b.CATEGORY as category,
              a.delwp_district,
              a.delwp_region,
              a.treatable,
              a.geom_polygon
          FROM
              reference_brau.grid_cell_180m_treatability as a
          INNER JOIN
              jfmp_2022_test. as b
          ON 
              ST_Within(a.geom_point, b.geom)
          WHERE 
              b.T_TYPE_FMS in ('FUEL REDUCTION', ECOLOGICAL)
      ;
      ```

 
### 5. Process Phoenix data in AWS Athena

   1. Create a new schema in Athena to hold/store our data
      
      ```sql
      CREATE SCHEMA jfmp_2022_test;
      ```
      
   2. Export jfmp_xy180 table from risk2temp to Athena database

   3. Calculate year1 burn scores

```sql
-- summarise the intersections of the JFMP polygons and Phoenix fire areas;
-- allocate burn weighting to each burn based on the number of intersecting treatable cells in the burn divided by the total number of treatable cells in all intersecting burns
create table test_jfmp_2022.weighted_burns as 
with 
    jfmp_year as (select 2023 as year1, 2024 as year2, 2025 as year3),
    allcells_y1_nojfmp_wx01 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb1.cell),-- year 1 nojfmp allcells table wx01
    allcells_y1_nojfmp_wx02 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb2.cell),-- year 1 nojfmp allcells table wx02
    allcells_y1_nojfmp_wx03 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb3.cell),-- year 1 nojfmp allcells table wx03
    allcells_y1_nojfmp_wx04 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb4.cell),-- year 1 nojfmp allcells table wx04
    allcells_y1_nojfmp_wx05 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb5.cell),-- year 1 nojfmp allcells table wx05
    allcells_y1_nojfmp_wx06 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb6.cell),-- year 1 nojfmp allcells table wx06
    allcells_y1_nojfmp_wx07 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb7.cell),-- year 1 nojfmp allcells table wx07
    allcells_y1_nojfmp_wx08 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb8.cell),-- year 1 nojfmp allcells table wx08
    allcells_y1_nojfmp_wx09 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb9.cell),-- year 1 nojfmp allcells table wx09
    allcells_y1_nojfmp_wx10 as (select * from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb10.cell),-- year 1 nojfmp allcells table wx10

    burm_num_cells as (
        select 'wx01' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx01 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx02' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx02 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx03' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx03 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx04' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx04 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx05' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx05 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx06' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx06 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx07' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx07 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx08' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx08 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx09' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx09 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
        union
        select 'wx10' as weather, a.ignitionid, b.name, b.burnnum, b.jfmp_year, count (a.cellid) as num_burnt
        from allcells_y1_nojfmp_wx10 as a inner join jfmp_xy180 as b on a.cellid = b.cellid inner join jfmp_year on b.jfmp_year = jfmp_year.year1
        where coalesce (a.intensity, 0) > 0
        group by a.ignitionid, b.name, b.burnnum, b.jfmp_year
    ),
    ignition_num_cells as (
        select weather, ignitionid, sum (num_burnt) as num_burnt
        from burm_num_cells
        group by weather, ignitionid
    )
(select 
    a.weather, a.ignitionid, a.name, a.burnnum, a.jfmp_year,
    a.num_burnt as burn_cells,
    b.num_burnt as ignition_cells,
    a.num_burnt/b.num_burnt as burn_weight
from burn_num_cells a left join ignition_num_cells b on a.ignitionid = b.ignitionid
);
```


   4. Combine all 10 ignition nojfmp impact tables into a single view

```sql
-- combine all 10 ignition nojfmp impact tables into a single view
create view test_jfmp_2022.impact_y1_nojfmp as (
    select *,       'wx01' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb1.ignition_impact
    union select *, 'wx02' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb2.ignition_impact
    union select *, 'wx03' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb3.ignition_impact
    union select *, 'wx04' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb4.ignition_impact
    union select *, 'wx05' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb5.ignition_impact
    union select *, 'wx06' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb6.ignition_impact
    union select *, 'wx07' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb7.ignition_impact
    union select *, 'wx08' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb8.ignition_impact
    union select *, 'wx09' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb9.ignition_impact
    union select *, 'wx10' as weather from jfmp_2023_2022fh_2km_nojfmp_v2_70deb0b6f5524cb98570dfdc875189eb10.ignition_impact
    );
```

   4. Combine all 10 ignition fulljfmp impact tables into a single view

```sql
-- combine all 10 ignition fulljfmp impact tables into a single view
create view test_jfmp_2022.impact_y1_fulljfmp as (
    select *,       'wx01' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb1.ignition_impact
    union select *, 'wx02' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb2.ignition_impact
    union select *, 'wx03' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb3.ignition_impact
    union select *, 'wx04' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb4.ignition_impact
    union select *, 'wx05' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb5.ignition_impact
    union select *, 'wx06' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb6.ignition_impact
    union select *, 'wx07' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb7.ignition_impact
    union select *, 'wx08' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb8.ignition_impact
    union select *, 'wx09' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb9.ignition_impact
    union select *, 'wx10' as weather from jfmp_2023_2022fh_2km_fulljfmp_v2_ab8e3737f94c409195e27bfd791ccfeb10.ignition_impact
    );
```

-- revise jfmp xy180 field names !! Note: this won't be required in future !!
create table jfmp_xy180 as (
    select cellid,
        ignitionid_jfmp as ignitionid,
        gid,
        name,
        burnnum,
        jfmpyearpr as jfmp_year,
        delwp_dist as delwp_district,
        op_region as delwp_region
    from
        jfmp2022_y1_no_jfmp)

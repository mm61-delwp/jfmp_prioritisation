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
   
   2. Import shapefile to risk2temp database

      Upload the shapefile using PostGIS Shapefile Import/Export manager
      <img src="https://user-images.githubusercontent.com/100050237/227848065-9e6c8ea4-d36b-4bf6-8c80-e75c971c4e9c.png" width="500" />
         > Note: You can run this tool without installing PostGIS by downloading the latest zip bundle from http://download.osgeo.org/postgis/windows/ and extracting just the /bin/ folder.
   
   3. Join JFMP shapefile to 180m grid cells
      
      ```sql
      CREATE TABLE jfmp_2022_test.jfmp_treatable_xy180 AS
          SELECT
              a.cellid,
              b.name,
              b.treatment_ as burnnum,
              b.JFMPYEARpr as jfmp_year,
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

```sql
create table jfmp_20_21.scored_burns_year1 as (

    -- Only update the input tables below; everything else will reference these
    jfmp_year            as (select 2023 as year1, 2024 as year2, 2025 as year3),                       -- translate jfmp_year to year1/2/3
    ignition_grid        as select * from reference_brau.grid_ignition_5km_2016,                        -- ignition grid
    impact_y1_jfmp       as select * from statewide_runs.ignition_jfmp_2020_21_target_130_5km_impact,   -- year1 jfmp impact table
    impact_y1_nojfmp     as select * from statewide_runs.ignition_jfmp_2020_2021_nothing_130_5km_impact,-- year 1 nojfmp impact table
    allcells_y1_nojfmp   as select * from jobid.cell                                                    -- year 1 nojfmp allcells table
    jfmp_treatable_xy180 as select * from jfmp_2022_prioritisation.jfmp_treatable_xy180,
    --jfmp_treatable_grouped as select * from   jfmp_20_21.jfmp_treatable_20200903_grouped,

    -- find (for each burn in year 1 of the JFMP) all bushfire footprint cells that the burn interacts with.
    relevant_bushfire_footprints as (
        select a.*, b.ignitionid
        from allcells_y1_nojfmp as a
        left join jfmp_treatable_xy180 as b -- switched a and b !!!! CHECK THIS !!!!
        on a.cellid = b.cellid
        where jfmp_year in jfmp_year.year1
        and coalesce(a.intensity, 0) > 0
    ),

    -- for each burn, count the number of treatable cells that intersect each simulated bushfire
    num_cells_jfmp_treatable as (
        select name, burnnum, category, jfmp_year, count(cellid) as num_cells, ignitionid
        from relevant_bushfire_footprints
        where jfmp_year in jfmp_year.year1
        and treatable = 1
        group by name, burnnum, category, jfmp_year, ignitionid
    ),

    -- join the ignition summaries back to the full ignition grid, for the year 1 JFMP impact results
    ignition_houseloss_JFMP as (
	select a.ignitionid, coalesce(b.tot_calc_loss, 0) as jfmp_tot_calc_loss
	from ignition_grid as a
        left join impact_y1_jfmp as b
        on a.ignitionid =  b.ignitionid
    ),

    -- join the ignition summaries back to the full ignition grid, for the year 1 "no JFMP" impact results
    ignition_houseloss_no_JFMP as (
	SELECT a.ignitionid, coalesce(b.tot_calc_loss, 0)  as no_jfmp_tot_calc_loss
	from ignition_grid as a
	left join impact_y1_nojfmp as b
	on a.ignitionid =  b.ignitionid
    ),

    -- calculate the difference in losses between the year 1 JFMP impacts and the year 1 "no JFMP" impacts
    -- on a per-ignition basis; thus calculating the loss reduction effect of the JFMP
    loss_diff as (
		select a.ignitionid, b.no_jfmp_tot_calc_loss, a.jfmp_tot_calc_loss, (b.no_jfmp_tot_calc_loss - a.jfmp_tot_calc_loss) as loss_diff
		from ignition_houseloss_JFMP as a
		left join ignition_houseloss_no_JFMP as b
		on a.ignitionid =  b.ignitionid
	),

	-- count the number of cells that were burnt in each jfmp polygon for each ignition
    burn_num_cells as (
		select ignitionid, name, burnnum, category, jfmp_year, count(*) as num_cells
		from jfmp_treatable_xy180
		where jfmp_year in jfmp_year(year1)
        and coalesce(intensity, 0) > 0
		group by ignitionid, name, burnnum, category
	),
    -- count the total number of cells that were burnt by each ignition
    bushfire_num_cells as (
        select ignitionid, count(*) as num_cells
		from relevant_bushfire_footprints
		where jfmp_year in jfmp_year(year1)
        and coalesce(intensity, 0) > 0
		group by ignitionid
    )

	weighted_burns as (
		select  b.name, b.burnnum, b.category, b.jfmp_year, b.ignitionid_jfmp as ignitionid,
		 b.num_cells, l.loss_diff, (b.num_cells::numeric / bnc.num_cells::numeric) * l.loss_diff as score
		from num_cells_jfmp_treatable as b
			join burn_num_cells as bnc on b.name = bnc.name and b.burnnum = bnc.burnnum and b.category = bnc.category, jfmp_year = bnc.jfmp_year
			join loss_diff as l on b.ignitionid_jfmp = l.ignitionid
		order by score desc

		select  b.ignitionid, b.name, b.burnnum, b.category, b.jfmp_year, b.num_cells,
                l.loss_diff, (b.num_cells::numeric / bnc.num_cells::numeric) * l.loss_diff as score
        from
            burn_num_cells as b
            join bushfire_num_cells as f on f.ignitionid = b.ignitionid
            join loss_diff as l on b.ignitionid_jfmp = l.ignitionid
        order by score desc
		),

	scored_burns as (
		select name, burnnum, category, jfmp_year, sum(score) as score
		from weighted_burns
		group by name, burnnum, category, jfmp_year
	),
	max_score as (
		select max(score) as max_score
		from scored_burns
	),
	norm_scored_burns as (
		select name, burnnum, category, jfmp_year, round((s.score/max_score.max_score*100),1) as normalised_score, s.score as raw_score
		from scored_burns as s, max_score
	)
    UP TO HERE!!!
	select  n.normalised_score,
            n.raw_score, n.gid,
            b.name, b.burnnum
	from norm_scored_burns as n
		join jfmp_treatable_grouped as b
		on n.gid= b.gid
	order by n.normalised_score desc
 '''

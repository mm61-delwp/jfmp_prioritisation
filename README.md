# JFMP Burn Prioritisation
Updating JFMP prioritisation process for new Bushfire Risk Analysis Framework


## Spatial data preparation in ArcGIS

### 1. Ensure required fields exist and are populated. These fields are:
 * name        - name of the burn
 * treatment_  - coded id of the burn, including district and type
 * category    - type of burn
 * T_TYPE_FMS  - type of burn
 * jfmpyrpr    - proposed year/season of treatment (as single 4 digit numeric year)
  > Note: Capitalisation shouldn't matter (but I had better check this!)
  > 
  > Note: The field for type of burn keeps changing. Pick one and update the WHERE statement in 2.3 below if required.
   
### 2. Convert the JFMP shapefiles to 180m grid data 

   1. Create a new schema in risk2temp_db to hold/store our data
      
      ```sql
      CREATE SCHEMA jfmp_2022_test;
      ```
   
   2. Import JFMP shapefile to risk2temp database

      Upload the shapefile using PostGIS Shapefile Import/Export manager
      <img src="https://user-images.githubusercontent.com/100050237/227848065-9e6c8ea4-d36b-4bf6-8c80-e75c971c4e9c.png" width="500" />
         > Note: You can run this tool without installing PostGIS by downloading the latest zip bundle from http://download.osgeo.org/postgis/windows/ and extracting just the /bin/ folder.
   
   3. Repair shapefile geometry (because ESRI's idea of valid geometry is different to PostGIS)
   
      ```sql
      UPDATE jfmp_2022_test.jfmp_shapefile SET geom = ST_MakeValid(geom);
      ```
   
   4. Join JFMP shapefile to 180m grid cells
      
      ```sql
      CREATE TABLE jfmp_2022_test.jfmp_xy180 AS
          SELECT
              a.cellid,
              b.name,
              b.treatment_ as burnnum,
              b.JFMPYEARpr as jfmp_year,
              a.delwp_district,
              a.delwp_region,
              a.treatable,
              a.geom_polygon,
              a.geom_centroid
          FROM
              reference_brau.grid_cell_180m_treatability as a
          INNER JOIN
              jfmp_2022_test. as b
          ON 
              ST_Within(a.geom_centroid, b.geom)
          WHERE 
              b.T_TYPE_FMS in ('FUEL REDUCTION', ECOLOGICAL) -- or b.category in ('FUEL REDUCTION', ECOLOGICAL)
      ;
      ```

 
### 3. Process Phoenix data in AWS Athena

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


   4. Combine all 10 nojfmp ignition impact tables into a single view

```sql
-- combine all 10 nojfmp ignition impact tables into a single view
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

   5. Combine all 10 fulljfmp ignition impact tables into a single view

```sql
-- combine all 10 fulljfmp ignition impact tables into a single view
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

   6. Calculate the difference in loss values between nojfmp and fulljfmp from ignition_impact tables

```sql
create view test_jfmp_2022.ignition_houseloss_phx as (
    select nojfmp.ignitionid, nojfmp.weather,
        coalesce(nojfmp.sum_loss_all_int, 0) as nojfmp_tot_calc_loss,
        coalesce(jfmp.sum_loss_all_int, 0) as jfmp_tot_calc_loss,
        coalesce(jfmp.sum_loss_all_int, 0) - coalesce(nojfmp.sum_loss_all_int, 0) as loss_diff -- negative value = reduction in house loss
    from impact_y1_nojfmp nojfmp
    left join impact_y1_fulljfmp jfmp on jfmp.ignitionid = nojfmp.ignitionid and jfmp.weather = nojfmp.weather
    );
```

   7. Calculate the difference in loss values between nojfmp and fulljfmp from bayes net tables

```sql
create view test_jfmp_2022.ignition_houseloss_bn as 
  with 
    bn_y1_nojfmp as (select * from bn_jfmp_2023_2022fh_2km_nojfmp_047883390e9245e9acf81230f193dca0.bn_ignition_summary),
    bn_y1_fulljfmp as (select * from bn_jfmp_2023_2022fh_2km_fulljfmp_4814dab5877a422c8314724550fc508a.bn_ignition_summary)

  (select nojfmp.ignition_id as ignitionid,
        coalesce(nojfmp.phoenix_houseloss, 0) as nojfmp_tot_calc_loss_phx,
        coalesce(jfmp.phoenix_houseloss, 0) as jfmp_tot_calc_loss_phx,
        coalesce(nojfmp.houseloss_mean_res, 0) as nojfmp_tot_calc_loss_bn,
        coalesce(jfmp.houseloss_mean_res, 0) as jfmp_tot_calc_loss_bn,
        coalesce(jfmp.phoenix_houseloss, 0) - coalesce(nojfmp.phoenix_houseloss, 0) as loss_diff_phx, -- negative value = reduction in house loss
        coalesce(jfmp.houseloss_mean_res, 0) - coalesce(nojfmp.houseloss_mean_res, 0) as loss_diff_bn -- negative value = reduction in house loss
    from bn_y1_nojfmp nojfmp
    left join bn_y1_fulljfmp jfmp on jfmp.ignition_id = nojfmp.ignition_id 
    );
```

   8. Make bayes net burn score summary view
    
```sql
create view burn_score_bn as
with 
    weighted_burns_bn as (
        select 
            ignitionid, name, burnnum, jfmp_year, 
            sum(burn_cells) as burn_cells, 
            sum(ignition_cells) as ignition_cells,
            cast(sum(burn_cells) as real)/cast(sum(ignition_cells) as real) as burn_weight
        from weighted_burns
        group by ignitionid, name, burnnum, jfmp_year
    ),
    
    burn_scores as (
        select
            b.name, b.burnnum, b.jfmp_year,
            sum(b.burn_weight * coalesce(hl.loss_diff_bn,0)) as burn_score_bn
        from weighted_burns_bn b
        left join ignition_houseloss_bn hl on hl.ignitionid = b.ignitionid
        group by b.name, b.burnnum, b.jfmp_year
    ),
    
    max_score as (
        select min(burn_score_bn) as max_burn_score_bn from burn_scores -- !! Note: burn scores are change values; negative is a reduction in house loss !!
    )

(select 
    b.name, b.burnnum, b.jfmp_year,
    b.burn_score_bn as burn_score_raw,
    cast(b.burn_score_bn as real)/cast(m.max_burn_score_bn as real) as burn_score_normalised
from burn_scores b, max_score m
order by burn_score_normalised desc
);
```

   9. Make Phoenix burn score summary view

```sql
create view burn_score_phx as
with 
    burn_scores as (
        select b.name, b.burnnum, b.jfmp_year,
            sum(b.burn_weight * coalesce(hl.loss_diff,0)) as burn_score_phx
        from weighted_burns b
        left join ignition_houseloss_phx hl on hl.ignitionid = b.ignitionid
        group by b.name, b.burnnum, b.jfmp_year
    ),
    max_score as (
        select min(burn_score_phx) as max_burn_score_phx from burn_scores -- !! Note: burn scores are change values; negative is a reduction in house loss !!
    )
(select 
    b.name, b.burnnum, b.jfmp_year,
    b.burn_score_phx as burn_score_raw,
    cast(b.burn_score_phx as real)/cast(m.max_burn_score_phx as real) as burn_score_normalised
from burn_scores b, max_score m
order by burn_score_normalised desc
);
```

   10. Combine burn score tables

```sql
create view burn_score_combined as(
    select  a.name, a.burnnum, a.jfmp_year, a.burn_score_raw as score_raw_phx, b.burn_score_raw as score_raw_bn,
            a.burn_score_normalised as score_norm_phx, b.burn_score_normalised as score_norm_bn
    from burn_score_phx a left join burn_score_bn b on a.name = b.name and a.burnnum = b.burnnum and a.jfmp_year = b.jfmp_year
    );
```

# JFMP Burn Prioritisation
Updating JFMP prioritisation process for new Bushfire Risk Analysis Framework


## Spatial data preparation in ArcGIS
1. Filter JFMP shapefile to remove non-burn fuel management
> Note: previously, we would have filtered to remove CFA burns but the shapefile provided doesn't have an 'agency' field so they're retained here

2. Ensure required fields exist and are populated. These fields are:
* name        - name of the burn
* treatment_  - coded id of the burn, including district and type
* category    - type of burn
* jfmpyrpr    - proposed year/season of treatment (as single 4 digit numeric year)
> Note: Capitalisation shouldn't matter (but I had better check this!)

3. Clip shapefile to latest treatability layer

> Note: we really should store treatability in Athena and skip this step

4. Import 180m grid cells from risk2temp_db

```sql
select 
  cellid, x_coord, y_coord, geom_centroid, delwp_district 
from 
  risk2temp_db.reference_brau.grid_cell_180m
where
  state = 'Victoria'
```

5. Join JFMP to 180m grid

6. Export

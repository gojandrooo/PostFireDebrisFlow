# Data preparation notebooks

This directory contains the following notebooks for data preparation, which should be executed in that order:

1. `add_site_ids.ipynb`: Retrieves Staley data and adds site IDs.  The result is stored in the Parquet file `staley16_debrisflow.parquet`.
2. `extract_contributing_region.ipynb`: Computes catchment area and fuel related features for each site.  Reads `staley16_debrisflow.parquet` and writes `staley16_observations_catchment_fuelpars_v3.parquet` as well as 
`staley16_sites_catchment_fuelpars_v3.parquet`.
3. `extract_rock_type.ipynb`: Extracts rock types found within catchment area from geological map, and aggregates to fraction of Igneous, Metamorphic, Sedimentary and Unconsolidated rock types making up each catchment.  Writes `staley16_observations_catchment_fuelpars_rocktype_v3.parquet`.
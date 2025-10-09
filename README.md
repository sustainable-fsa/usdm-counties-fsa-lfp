
<!-- README.md is generated from README.Rmd. Please edit that file -->

[![Static
Badge](https://img.shields.io/badge/Repo-sustainable--fsa%2Fusdm--counties--fsa--lfp-magenta?style=flat)](https://github.com/sustainable-fsa/usdm-counties-fsa-lfp/)
![Last
Update](https://img.shields.io/github/last-commit/sustainable-fsa/usdm-counties-fsa-lfp?style=flat)
![Repo
Size](https://img.shields.io/github/repo-size/sustainable-fsa/usdm-counties-fsa-lfp?style=flat)

This repository provides weekly US Drought Monitor (USDM) data
aggregated to [USDA Farm Service Agency Livestock Forage Program county
boundaries](https://sustainable-fsa.github.io/fsa-lfp-counties). This
dataset facilitates county-level analysis of drought conditions,
supporting research, policy-making, and climate resilience planning.

<a href="https://sustainable-fsa.github.io/usdm-counties-fsa-lfp/manifest.html" target="_blank">📂
View the US Drought Monitor FSA LFP county archive listing here.</a>

Federal law and FSA guidance describes a process by which [county-level
eligibility](https://sustainable-fsa.github.io/fsa-lfp-eligibility/) is
determined by the intersection of county boundaries, the [United States
Drought Monitor weekly drought
assessment](https://sustainable-fsa.github.io/usdm/), and the [normal
grazing
period](https://sustainable-fsa.github.io/fsa-normal-grazing-period/)
for each type of grazing land in each county.

The county data used in this repository were acquired via FOIA request
**2025-FSA-08431-F** by R. Kyle Bocinsky (Montana Climate Office) and
fulfilled on September 8, 2025. The FOIA response, including the
original Esri file geodatabase, is archived at
[`https://sustainable-fsa.github.io/fsa-lfp-counties`](https://sustainable-fsa.github.io/fsa-lfp-counties).

------------------------------------------------------------------------

## 📈 About the US Drought Monitor (USDM)

The US Drought Monitor is a weekly map-based product that synthesizes
multiple drought indicators into a single national assessment. It is
produced by:

- National Drought Mitigation Center (NDMC)
- US Department of Agriculture (USDA)
- National Oceanic and Atmospheric Administration (NOAA)

Each weekly map represents a combination of data analysis and expert
interpretation.

The USDM weekly maps depicting drought conditions are categorized into
six levels:

- **None**: Normal or wet conditions
- **D0**: Abnormally Dry
- **D1**: Moderate Drought
- **D2**: Severe Drought
- **D3**: Extreme Drought
- **D4**: Exceptional Drought

While USDM drought class boundaries are developed without regard to
political boundaries, it is often aggregated by political boundaries to
assist in decision-making and for regulatory purposes. **This repository
focuses on aggregating these data to the county level, enabling more
localized analysis and decision-making.**

> **Note**: This archive is maintained by the Montana Climate Office,
> but all analytical authorship of the USDM drought maps belongs to the
> named USDM authors.

------------------------------------------------------------------------

## 🗂 Directory Structure

- `usdm-counties-fsa-lfp.R`: R script that processes and aggregates
  weekly USDM shapefiles to county boundaries.
- `usdm-counties-fsa-lfp.parquet`: Processed county-level USDM data in a
  single parquet file.
- `data/`: Directory containing processed county-level USDM data.
- `README.Rmd`: This README file, providing an overview and usage
  instructions.

------------------------------------------------------------------------

## Data Sources

- **USDM Polygons**: Weekly `.parquet` files from
  [sustainable-fsa/usdm](https://github.com/sustainable-fsa/usdm)
- **FSA LFP County Boundaries**: Acquired via FOIA request
  [**2025-FSA-08431-F**](ttps://sustainable-fsa.github.io/fsa-lfp-counties).

**USDM Polygons** are reprojected to the same coordinate reference
system as the **FSA LFP County Boundaries**, geometrically validated,
and saved in `.parquet` format.

## Processing Pipeline

The analysis pipeline is fully contained in
[`usdm-counties-fsa-lfp.R`](usdm-counties-fsa-lfp.R) and proceeds as
follows:

1.  **Install and load dependencies**:

- Uses `pak::pak()` to ensure fresh, source-built installs of critical
  geospatial packages.

2.  **Download FSA LFP county boundary data**. -Re-calculate county
    areas and save as
    [`data/fsa-lfp-counties.parquet`](data/fsa-lfp-counties.parquet)

3.  **Download and intersect**:

- For each weekly USDM `.parquet` file:
- Read county and drought geometries
- Perform spatial intersection
- Calculate the percent of each county area affected by each drought
  class
- Tabular output is saved to `data/usdm/USDM_{YYYY-MM-DD}.parquet`

4.  **Output Structure**: Each output file is a non-spatial `.parquet`
    file with the following fields:

- `STATEFP`, `State`, `COUNTYFP`, `County`, `CountyLSAD`
- `usdm_date`: The date of the USDM map
- `usdm_class`: One of `None`, `D0`, `D1`, `D2`, `D3`, `D4`
- `usdm_percent`: Proportion of the county in this drought class (as a
  decimal between 0 and 1)

------------------------------------------------------------------------

## 🛠️ Dependencies

Key R packages used:

- `sf`
- `terra`
- `arrow`
- `tidyverse`
- `curl`

The script installs all required packages using the
[`pak`](https://pak.r-lib.org) package.

------------------------------------------------------------------------

## 📍 Quick Start: Visualize a Weekly County USDM Map in R

This snippet shows how to load a weekly GeoParquet file from the archive
and create a simple drought classification map using `sf` and `ggplot2`.

``` r
# Load required libraries
library(arrow)
library(sf)
library(ggplot2) # For plotting
library(tigris)  # For state boundaries
library(rmapshaper) # For innerlines function

## Get latest USDM data
latest <-
  jsonlite::fromJSON(
    "manifest.json"
  )$path |>
  stringr::str_subset("parquet") |>
  stringr::str_subset("data/usdm") |>
  max()
# e.g., [1] "data/usdm/USDM_2025-05-27.parquet"

date <-
  latest |>
  stringr::str_extract("\\d{4}-\\d{2}-\\d{2}") |>
  lubridate::as_date()

# Get the highest (worst) drought class in each county
usdm <-
  latest |>
  arrow::read_parquet() |>
  dplyr::group_by(STATEFP, COUNTYFP) |>
  dplyr::filter(usdm_class == max(usdm_class))

## Load the fsa-lfp-counties parquet file
counties <- 
  # You can read straight from online
  # sf::read_sf("https://sustainable-fsa.github.io/usdm-counties-fsa-lfp/data/fsa-lfp-counties.parquet") |>
  sf::read_sf("data/fsa-lfp-counties.parquet") |>
  # transform to WGS 84
  sf::st_transform("EPSG:4326") |>
  sf::st_cast("POLYGON", warn = FALSE, do_split = TRUE) |>
  tigris::shift_geometry() |>
  dplyr::group_by(STATEFP, COUNTYFP) |>
  dplyr::summarise(.groups = "drop") |>
  sf::st_cast("MULTIPOLYGON")

usdm_counties <-
  usdm |>
  dplyr::left_join(counties) |>
  sf::st_as_sf()

# Census counties for bounding box
census_counties <-
  tigris::counties(cb = TRUE,
                   resolution = "5m", 
                   year = 2020) |>
  dplyr::filter(!(STATEFP %in% c("60", "66", "69", "78"))) %>%
  # transform to WGS 84
  sf::st_transform("EPSG:4326") |>
  sf::st_cast("POLYGON", warn = FALSE, do_split = TRUE) |>
  tigris::shift_geometry() |>
  dplyr::group_by(STATEFP, COUNTYFP) |>
  dplyr::summarise(.groups = "drop") |>
  sf::st_cast("MULTIPOLYGON")
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |                                                                      |   1%  |                                                                              |=                                                                     |   2%  |                                                                              |==                                                                    |   2%  |                                                                              |==                                                                    |   3%  |                                                                              |===                                                                   |   4%  |                                                                              |===                                                                   |   5%  |                                                                              |====                                                                  |   6%  |                                                                              |=====                                                                 |   7%  |                                                                              |=====                                                                 |   8%  |                                                                              |======                                                                |   8%  |                                                                              |======                                                                |   9%  |                                                                              |=======                                                               |   9%  |                                                                              |=======                                                               |  10%  |                                                                              |=======                                                               |  11%  |                                                                              |========                                                              |  11%  |                                                                              |========                                                              |  12%  |                                                                              |=========                                                             |  12%  |                                                                              |=========                                                             |  13%  |                                                                              |=========                                                             |  14%  |                                                                              |==========                                                            |  14%  |                                                                              |==========                                                            |  15%  |                                                                              |===========                                                           |  15%  |                                                                              |===========                                                           |  16%  |                                                                              |============                                                          |  17%  |                                                                              |============                                                          |  18%  |                                                                              |=============                                                         |  18%  |                                                                              |=============                                                         |  19%  |                                                                              |==============                                                        |  19%  |                                                                              |==============                                                        |  20%  |                                                                              |==============                                                        |  21%  |                                                                              |===============                                                       |  21%  |                                                                              |===============                                                       |  22%  |                                                                              |================                                                      |  22%  |                                                                              |================                                                      |  23%  |                                                                              |=================                                                     |  24%  |                                                                              |==================                                                    |  25%  |                                                                              |==================                                                    |  26%  |                                                                              |===================                                                   |  27%  |                                                                              |===================                                                   |  28%  |                                                                              |====================                                                  |  28%  |                                                                              |====================                                                  |  29%  |                                                                              |=====================                                                 |  30%  |                                                                              |=====================                                                 |  31%  |                                                                              |======================                                                |  31%  |                                                                              |======================                                                |  32%  |                                                                              |=======================                                               |  32%  |                                                                              |=======================                                               |  33%  |                                                                              |========================                                              |  34%  |                                                                              |========================                                              |  35%  |                                                                              |=========================                                             |  35%  |                                                                              |=========================                                             |  36%  |                                                                              |==========================                                            |  37%  |                                                                              |==========================                                            |  38%  |                                                                              |===========================                                           |  38%  |                                                                              |===========================                                           |  39%  |                                                                              |============================                                          |  40%  |                                                                              |=============================                                         |  41%  |                                                                              |=============================                                         |  42%  |                                                                              |==============================                                        |  43%  |                                                                              |===============================                                       |  44%  |                                                                              |===============================                                       |  45%  |                                                                              |================================                                      |  45%  |                                                                              |================================                                      |  46%  |                                                                              |=================================                                     |  47%  |                                                                              |==================================                                    |  48%  |                                                                              |==================================                                    |  49%  |                                                                              |===================================                                   |  50%  |                                                                              |===================================                                   |  51%  |                                                                              |====================================                                  |  51%  |                                                                              |====================================                                  |  52%  |                                                                              |=====================================                                 |  52%  |                                                                              |=====================================                                 |  53%  |                                                                              |======================================                                |  54%  |                                                                              |======================================                                |  55%  |                                                                              |=======================================                               |  55%  |                                                                              |=======================================                               |  56%  |                                                                              |========================================                              |  56%  |                                                                              |========================================                              |  57%  |                                                                              |========================================                              |  58%  |                                                                              |=========================================                             |  58%  |                                                                              |=========================================                             |  59%  |                                                                              |==========================================                            |  59%  |                                                                              |==========================================                            |  60%  |                                                                              |==========================================                            |  61%  |                                                                              |===========================================                           |  61%  |                                                                              |===========================================                           |  62%  |                                                                              |============================================                          |  62%  |                                                                              |============================================                          |  63%  |                                                                              |============================================                          |  64%  |                                                                              |=============================================                         |  64%  |                                                                              |=============================================                         |  65%  |                                                                              |==============================================                        |  65%  |                                                                              |==============================================                        |  66%  |                                                                              |===============================================                       |  66%  |                                                                              |===============================================                       |  67%  |                                                                              |===============================================                       |  68%  |                                                                              |================================================                      |  68%  |                                                                              |================================================                      |  69%  |                                                                              |=================================================                     |  69%  |                                                                              |=================================================                     |  70%  |                                                                              |=================================================                     |  71%  |                                                                              |==================================================                    |  71%  |                                                                              |==================================================                    |  72%  |                                                                              |===================================================                   |  72%  |                                                                              |===================================================                   |  73%  |                                                                              |====================================================                  |  74%  |                                                                              |====================================================                  |  75%  |                                                                              |=====================================================                 |  75%  |                                                                              |=====================================================                 |  76%  |                                                                              |======================================================                |  77%  |                                                                              |======================================================                |  78%  |                                                                              |=======================================================               |  78%  |                                                                              |=======================================================               |  79%  |                                                                              |========================================================              |  79%  |                                                                              |========================================================              |  80%  |                                                                              |=========================================================             |  81%  |                                                                              |=========================================================             |  82%  |                                                                              |==========================================================            |  82%  |                                                                              |==========================================================            |  83%  |                                                                              |===========================================================           |  84%  |                                                                              |===========================================================           |  85%  |                                                                              |============================================================          |  85%  |                                                                              |============================================================          |  86%  |                                                                              |=============================================================         |  87%  |                                                                              |=============================================================         |  88%  |                                                                              |==============================================================        |  88%  |                                                                              |==============================================================        |  89%  |                                                                              |===============================================================       |  90%  |                                                                              |================================================================      |  91%  |                                                                              |================================================================      |  92%  |                                                                              |=================================================================     |  93%  |                                                                              |==================================================================    |  94%  |                                                                              |==================================================================    |  95%  |                                                                              |===================================================================   |  95%  |                                                                              |===================================================================   |  96%  |                                                                              |====================================================================  |  97%  |                                                                              |====================================================================  |  98%  |                                                                              |===================================================================== |  98%  |                                                                              |===================================================================== |  99%  |                                                                              |======================================================================| 100%

``` r
# Plot the map
ggplot(counties) +
  geom_sf(data = sf::st_union(counties),
          fill = "grey80",
          color = NA) +
  geom_sf(data = usdm_counties,
          aes(fill = usdm_class), 
          color = NA) +
  geom_sf(data = rmapshaper::ms_innerlines(counties),
          fill = NA,
          color = "white",
          linewidth = 0.1) +
  geom_sf(data = counties |>
            dplyr::group_by(STATEFP) |>
            dplyr::summarise() |>
            rmapshaper::ms_innerlines(),
          fill = NA,
          color = "white",
          linewidth = 0.2) +
  scale_fill_manual(
    values = c("grey80",
               "#ffff00",
               "#fcd37f",
               "#ffaa00",
               "#e60000",
               "#730000"),
    drop = FALSE,
    name = "Drought\nClass") +
  labs(title = "US Drought Monitor",
       subtitle = format(date, " %B %d, %Y")) +
  coord_sf(
    xlim = sf::st_bbox(census_counties)[c("xmin", "xmax")],
    ylim = sf::st_bbox(census_counties)[c("ymin", "ymax")],
    clip = "off"
  ) +
  theme_void()
```

<img src="./example-1.png" style="display: block; margin: auto;" />

------------------------------------------------------------------------

## 📝 Citation & Attribution

**Citation format** (suggested):

> US Drought Monitor authors and R. Kyle Bocinsky YYYY. *US Drought
> Monitor Weekly Maps Aggregated to FSA LFP County Boundaries*. Data
> processed, curated, and archived by R. Kyle Bocinsky, Montana Climate
> Office. Accessed via GitHub archive, YYYY-MM-DD.
> <https://sustainable-fsa.github.io/usdm-counties-fsa-lfp/>

**Acknowledgments**:

- Map content by USDM authors.
- Data processing, curation, and archival structure by R. Kyle Bocinsky,
  Montana Climate Office, University of Montana.

------------------------------------------------------------------------

## 📄 License

- **Raw USDM data** (NDMC): Public Domain (17 USC § 105)
- **Processed data & scripts**: © R. Kyle Bocinsky, released under
  [CC0](https://creativecommons.org/publicdomain/zero/1.0/) and [MIT
  License](./LICENSE) as applicable

------------------------------------------------------------------------

## ⚠️ Disclaimer

This dataset is archived for research and educational use only. The
National Drought Mitigation Center hosts the US Drought Monitor. Please
visit <https://droughtmonitor.unl.edu>.

------------------------------------------------------------------------

## 👏 Acknowledgment

This project is part of:

**[*Enhancing Sustainable Disaster Relief in FSA
Programs*](https://www.ars.usda.gov/research/project/?accnNo=444612)**  
Supported by USDA OCE/OEEP and USDA Climate Hubs  
Prepared by the [Montana Climate Office](https://climate.umt.edu)

------------------------------------------------------------------------

## 📬 Contact

**R. Kyle Bocinsky**  
Director of Climate Extension  
Montana Climate Office  
📧 <kyle.bocinsky@umontana.edu>  
🌐 <https://climate.umt.edu>

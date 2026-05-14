# GEE Carbon Variable Extraction

Reproducible Google Earth Engine scripts for extracting and resampling annual environmental variables for carbon sequestration and ecosystem productivity studies.

## Research Motivation

Carbon sequestration and ecosystem productivity are affected by vegetation activity, climate, topography, land cover, and soil properties. This project provides a reproducible workflow to prepare grid-aligned environmental variables for subsequent trend analysis and statistical modeling.

## Variables

| Variable | Dataset | Temporal operation | Unit |
|---|---|---|---|
| NDVI | MODIS MOD13A1 | Growing-season mean | Unitless |
| NPP | MODIS MOD17A3HGF | Annual value | g C m⁻² yr⁻¹ |
| Precipitation | ERA5-Land Daily | Annual sum | mm yr⁻¹ |
| Temperature | ERA5-Land Daily | Annual mean | °C |
| Shortwave radiation | ERA5-Land Daily | Annual sum | J m⁻² yr⁻¹ |
| Elevation | SRTM | Static | m |
| Land cover | MODIS MCD12Q1 | Annual class | IGBP class |
| SOC | OpenLandMap | Mean of 0, 10, and 30 cm layers | g kg⁻¹ |

## Workflow

1. Define study area and target year.
2. Load remote sensing and climate datasets from Google Earth Engine.
3. Calculate annual or growing-season variables.
4. Resample all variables to a common WGS84 grid.
5. Export a grid-aligned CSV table for further analysis.

## How to Use

1. Open the script in the Google Earth Engine Code Editor.
2. Replace the study area asset:

```javascript
studyAreaAsset: 'users/your_username/your_study_area_asset'

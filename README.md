# EO Skills for Claude Code

A collection of [Claude Code skills](https://claude.ai/code) for Earth Observation data workflows — downloading satellite data, converting heights, and more.

## Skills

### `swot-download`
Downloads SWOT (Surface Water and Ocean Topography) satellite data via NASA Earthdata using `earthaccess`.

**Products supported:**
- `SWOT_L2_HR_PIXC_D` — Pixel Cloud (~22 m posting, Version D recommended)
- `SWOT_L2_HR_PIXCVec_D` — Pixel Cloud Vectors
- `SWOT_L2_HR_RiverSP_D` — River Single-Pass (reach/node elevation, width, slope)
- `SWOT_L2_HR_LakeSP_D` — Lake Single-Pass (water surface elevation and area)

**Trigger phrases:** "download SWOT data", "get PIXC for \<region\>", "fetch SWOT river data", "download SWOT lake products"

---

### `is2-download`
Downloads ICESat-2 and ArcticDEM data via the `sliderule-tool`.

**Products supported:**
- `atl03x` — raw photons (preferred over legacy `atl03`)
- `atl03x-surface` — ATL06-SR surface-fitted elevations
- `atl06` — land/ice surface elevation per segment
- `atl08` — vegetation height and canopy cover
- `atl24` — coastal/nearshore bathymetry
- `arcticdem` — ArcticDEM elevation sampled along ICESat-2 tracks

**Trigger phrases:** "download ATL06", "get ICESat-2 data for \<region\>", "fetch bathymetry data", "download ATL24", "surface fit photons"

---

### `gee-download`
Downloads satellite imagery from Google Earth Engine using the `gee_downloader` tool. Handles cloud/snow coverage filtering and exports image lists for bulk download.

**Trigger phrases:** "download Sentinel-2 images", "get GEE imagery for \<region\>", "download satellite images from Google Earth Engine"

---

### `ellip-ortho`
Converts heights between ellipsoidal (WGS84/GNSS) and orthometric (above geoid/MSL) reference frames using the EGM2008 geoid model via `pygeodesy`.

**Trigger phrases:** "convert ellipsoidal to orthometric", "h to H conversion", "WGS84 height to MSL", "geoid undulation", "EGM2008 correction"

---

## Requirements

All skills assume a conda environment named `sme-chain` with the following packages:

| Package | Used by |
|---|---|
| `earthaccess` | swot-download |
| `sliderule-tool` | is2-download |
| `pygeodesy` | ellip-ortho |
| `geopandas`, `rasterio`, `xarray`, `h5netcdf` | all |

The EGM2008 geoid file (`egm2008-1.pgm`) is required for `ellip-ortho` and expected at:
```
/home/yan/WorkSpace/dataset/SharedDataset/geoid/egm2008-1.pgm
```

## Installation

Clone this repo into your Claude Code skills directory:

```bash
git clone https://github.com/peterpan83/eo-skills.git ~/.claude/skills/eo-skills
```

Or configure the path in your Claude Code settings.

## Usage

Skills are triggered automatically when you describe your task to Claude Code. You can also invoke them explicitly:

```
/swot-download
/is2-download
```

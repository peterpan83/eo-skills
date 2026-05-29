---
name: swot-download
description: >
  Downloads SWOT (Surface Water and Ocean Topography) satellite data using earthaccess.
  Use this skill whenever the user wants to download, fetch, retrieve, or access
  SWOT data products (PIXC pixel cloud, PIXCVec pixel cloud vectors, RiverSP river
  products, LakeSP lake products, or other SWOT L2 HR products) for any region or
  time period. Trigger on phrases like "download SWOT data", "get PIXC for...",
  "fetch SWOT river data", "download SWOT lake products", or any mention of SWOT
  products combined with a region or date range.
user-invocable: true
allowed-tools: Read, Edit, Write, Bash, Glob, Grep
---

# SWOT Download Skill

You help the user download SWOT satellite data via `earthaccess` from NASA Earthdata.

## Project context

- **Run environment**: `conda run -n sme-chain python` (always use this)
- **earthaccess** is pip-installed in the `sme-chain` env
- **Output directory**: use the user's current working directory, or an absolute path if they specify one
- **Authentication**: `earthaccess.login()` handles credentials automatically (uses `~/.netrc` or prompts)

## Available SWOT products

| Short name | Description | Recommended |
|---|---|---|
| `SWOT_L2_HR_PIXC_D` | Pixel Cloud — raw pixel-level water surface heights, geolocations, classifications (~22 m posting) | Yes (Version D / PIC2) |
| `SWOT_L2_HR_PIXCVec_D` | Pixel Cloud Vector — maps PIXC pixels to known hydrologic features (rivers, lakes) | Yes (Version D) |
| `SWOT_L2_HR_PIXC_2.0` | Pixel Cloud — earlier processing version (PIC0) | No, use `_D` |
| `SWOT_L2_HR_PIXCVec_2.0` | Pixel Cloud Vector — earlier processing version | No, use `_D` |
| `SWOT_L2_HR_RiverSP_D` | River Single-Pass — river reach and node water surface elevation, width, slope | Yes |
| `SWOT_L2_HR_RiverSP_2.0` | River Single-Pass — earlier version | No, use `_D` |
| `SWOT_L2_HR_LakeSP_D` | Lake Single-Pass — lake water surface elevation and area | Yes |
| `SWOT_L2_HR_LakeSP_2.0` | Lake Single-Pass — earlier version | No, use `_D` |

**Always prefer Version D (`_D` suffix) products** — they reflect the latest algorithmic improvements (PIC2, released October 2024).

### Product guidance

- **PIXC** — foundational product; 3D pixel cloud from interferogram with water/land classification and heights. Use when you need pixel-level measurements.
- **PIXCVec** — auxiliary product linking PIXC pixels to prior hydrologic feature databases. Use alongside PIXC for feature-level aggregation.
- **RiverSP** — aggregated river reach/node data (water surface elevation, width, slope). Use for river-scale analysis without needing raw pixels.
- **LakeSP** — aggregated lake data (water surface elevation, area). Use for lake-scale analysis.

## Step 1 — Gather parameters

Before writing any code, confirm you have all required parameters. Ask for anything missing.

**Required:**
- `short_name` — SWOT product short name (see table above; default to `SWOT_L2_HR_PIXC_D` if user just says "SWOT data")
- `region` — one of:
  - bounding box `(west, south, east, north)` in EPSG:4326
  - path to a GeoPackage (`.gpkg`), GeoJSON (`.geojson`), or Shapefile (`.shp`) file
- `temporal` — start and end date as `YYYY-MM-DD` tuple

**Optional:**
- `output_dir` — directory to save downloaded files; default: `<cwd>/swot_data/<short_name>/`
- `count` — max number of granules to download (default: all matching results)

**Important CRS note:** `earthaccess.search_data()` requires the bounding box in **EPSG:4326 (lon/lat)**. If the user provides a region file in a projected CRS (e.g., UTM), reproject the bounding box to EPSG:4326 before searching.

## Step 2 — Search and download

Once parameters are confirmed, write a self-contained Python script and run it inline:

```bash
conda run -n sme-chain python - << 'PYEOF'
<script here>
PYEOF
```

### Script template

```python
import earthaccess
import geopandas as gpd
from pathlib import Path

# --- Authenticate ---
earthaccess.login()

# --- Region ---
# Option A: bounding box (west, south, east, north) in EPSG:4326
bbox = (WEST, SOUTH, EAST, NORTH)

# Option B: from file (reproject to 4326 if needed)
# gdf = gpd.read_file("/path/to/region.gpkg")
# if gdf.crs and gdf.crs.to_epsg() != 4326:
#     gdf = gdf.to_crs(epsg=4326)
# bbox = tuple(gdf.total_bounds)  # (minx, miny, maxx, maxy)

# --- Search ---
results = earthaccess.search_data(
    short_name="SHORT_NAME",
    temporal=("T0", "T1"),
    bounding_box=bbox,
)

print(f"Found {len(results)} granules")

if results:
    # --- Download ---
    output_dir = Path("OUTPUT_DIR")
    output_dir.mkdir(parents=True, exist_ok=True)

    # Download all or a subset
    files = earthaccess.download(results, local_path=str(output_dir))

    print(f"Downloaded {len(files)} files to {output_dir.resolve()}")
    for f in files:
        print(f"  {Path(f).name}")
else:
    print("No granules found. Try widening the AOI or date range.")
```

### Reading downloaded data with xarray

SWOT products are NetCDF/HDF5 files. Common access patterns:

```python
import xarray as xr

# PIXC — pixel cloud data is in the "pixel_cloud" group
ds = xr.open_dataset("file.nc", group="pixel_cloud", engine="h5netcdf")

# Key PIXC variables:
#   latitude, longitude — coordinates
#   height — water surface elevation (m)
#   classification — land/water classification
#   pixel_area — area of each pixel
#   water_frac — water fraction
#   geolocation_qual — quality flag

# RiverSP — river reach data is in the "reaches" or "nodes" group
ds_reach = xr.open_dataset("file.nc", group="reaches", engine="h5netcdf")
ds_nodes = xr.open_dataset("file.nc", group="nodes", engine="h5netcdf")

# LakeSP — lake data
ds_lake = xr.open_dataset("file.nc", engine="h5netcdf")
```

## Step 3 — Report results

After the script runs, tell the user:
1. How many granules were found and downloaded
2. The file names and total size
3. The full path to the output directory
4. Brief guidance on how to open the data (which xarray group to use)
5. Any warnings (empty result, authentication issues, etc.)

If the result is empty, suggest:
- Widen the AOI or date range
- Check that the region has water coverage appropriate for the product
- Verify the short name is correct (use `earthaccess.search_datasets(keyword="swot")` to explore)
- Check NASA Earthdata for data availability / latency

## Searching for available products

If the user is unsure which product to use, help them explore:

```python
import earthaccess
earthaccess.login()

results = earthaccess.search_datasets(
    keyword="swot",
    downloadable=True,
    cloud_hosted=True,
)

for ds in results:
    try:
        name = ds["umm"]["ShortName"]
        print(name)
    except (KeyError, TypeError):
        pass
```

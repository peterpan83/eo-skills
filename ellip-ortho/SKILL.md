---
name: ellip-ortho
description: >
  Converts heights between ellipsoidal (geodetic) and orthometric (above geoid/sea level) reference
  frames using the EGM2008 geoid model via pygeodesy's GeoidKarney.
  Use this skill whenever the user wants to convert ellipsoidal height to orthometric height, or
  orthometric height to ellipsoidal height, or asks about geoid undulation/separation (N).
  Trigger on phrases like "convert ellipsoidal to orthometric", "h to H conversion",
  "geoid height correction", "WGS84 height to MSL", "MSL to ellipsoidal", "EGM2008 correction",
  "geoid undulation", or any question involving the difference between ellipsoidal and
  orthometric/MSL/AMSL heights for a given lat/lon location.
---

# Ellipsoidal ↔ Orthometric Height Conversion

You help the user convert heights between ellipsoidal (h) and orthometric (H) reference frames
using the EGM2008 geoid model.

## Background

- **Ellipsoidal height (h)**: height above the WGS84 ellipsoid — reported by GPS/GNSS
- **Orthometric height (H)**: height above the geoid (mean sea level approximation)
- **Geoid undulation (N)**: the difference; N = h − H, so:
  - h → H: `H = h − N`
  - H → h: `h = H + N`

## Setup

**Python environment**: always use `conda run -n sme-chain python`

**Geoid file**: `/home/yan/WorkSpace/dataset/SharedDataset/geoid/egm2008-1.pgm`  
(EGM2008 1-arcminute grid, ~300 MB; already present — do not re-download)

**Check / install pygeodesy** (it is already installed, but verify defensively):
```bash
conda run -n sme-chain python -c "import pygeodesy" 2>/dev/null \
  || conda run -n sme-chain pip install pygeodesy
```

## Step 1 — Gather inputs

Before writing code, confirm you have:

| Parameter | Notes |
|-----------|-------|
| `lat` | Latitude in decimal degrees (positive = North) |
| `lon` | Longitude in decimal degrees (positive = East) |
| `height` | The height value to convert (in metres) |
| `direction` | `h→H` (ellipsoidal → orthometric) **or** `H→h` (orthometric → ellipsoidal) |

If the direction is ambiguous (user says "convert my GPS height"), default to `h→H` and
confirm with them.

## Step 1b — Deriving a representative lat/lon from a dataset

When the user has a spatial dataset rather than explicit coordinates, compute a mean
lat/lon to use as the geoid query point.

### Single N vs. per-point N

Before choosing an approach, consider the spatial extent:

- **< ~100 km extent** (one image tile, a short track segment): N varies < 1 m — use a
  single mean N applied to all heights. This is fast and accurate enough.
- **> ~100 km extent** (full ICESat-2 pass, wide-swath image, multi-strip mosaic): N can
  vary 10–50 m across the scene. Compute N per-point instead (loop or vectorised).

### ICESat-2 ATL03X parquet

The photon parquet produced by sliderule has `lat` and `lon` columns (decimal degrees).

```python
import numpy as np
import duckdb

parquet = "/path/to/track.parquet"
con = duckdb.connect()

# Pull just the coordinate columns — avoid loading photon heights into memory
df = con.execute("SELECT lat, lon FROM read_parquet(?)", [parquet]).df()

mean_lat = df["lat"].mean()
# Circular mean handles any longitude range, including antimeridian crossings
lon_rad = np.deg2rad(df["lon"].values)
mean_lon = np.rad2deg(np.arctan2(np.sin(lon_rad).mean(), np.cos(lon_rad).mean()))
```

For per-point conversion on a large parquet, apply N inside a DuckDB query or vectorise
with a list comprehension over the loaded DataFrame rather than a Python loop.

### Satellite image (GeoTIFF / raster)

The centre pixel of the image is a good representative point for a single-N correction.
Use `rasterio` (already in `sme-chain`):

```python
import rasterio
from rasterio.crs import CRS
from rasterio.warp import transform_bounds

with rasterio.open("/path/to/image.tif") as src:
    bounds = src.bounds          # in native CRS
    crs = src.crs

# Reproject bounds to WGS84 if needed
if not crs.equals(CRS.from_epsg(4326)):
    left, bottom, right, top = transform_bounds(crs, "EPSG:4326",
                                                 bounds.left, bounds.bottom,
                                                 bounds.right, bounds.top)
else:
    left, bottom, right, top = bounds.left, bounds.bottom, bounds.right, bounds.top

mean_lat = (bottom + top) / 2
# Circular mean for safety (handles imagery near antimeridian)
import numpy as np
lons = np.array([left, right])
mean_lon = np.rad2deg(np.arctan2(
    np.sin(np.deg2rad(lons)).mean(),
    np.cos(np.deg2rad(lons)).mean()
))
```

### General tabular data (CSV / DataFrame with lat/lon columns)

```python
import numpy as np
import pandas as pd

df = pd.read_csv("/path/to/data.csv")   # or pd.read_parquet(...)

mean_lat = df["lat"].mean()             # adjust column names as needed
lon_rad = np.deg2rad(df["lon"].values)
mean_lon = np.rad2deg(np.arctan2(np.sin(lon_rad).mean(), np.cos(lon_rad).mean()))
```

### Vector / GeoDataFrame (GeoJSON, shapefile, etc.)

```python
import numpy as np
import geopandas as gpd

gdf = gpd.read_file("/path/to/file.geojson").to_crs(epsg=4326)
centroid = gdf.geometry.unary_union.centroid
mean_lat, mean_lon = centroid.y, centroid.x
```

### Antimeridian warning

Naive longitude averaging (`lons.mean()`) silently gives a wrong answer when the data
spans the antimeridian (e.g. lons running from 170° → −170°, arithmetic mean ≈ 0°).
Always use the **circular mean** snippet above — it costs nothing and is always correct.

## Step 2 — Run the conversion

Use this pattern:

```python
from pygeodesy import GeoidKarney
from pygeodesy.ellipsoidalKarney import LatLon

PGM = "/home/yan/WorkSpace/dataset/SharedDataset/geoid/egm2008-1.pgm"
geoid = GeoidKarney(PGM)

lat, lon = <lat>, <lon>

# Get geoid undulation N at this location
N = geoid(LatLon(lat, lon))

# h → H  (ellipsoidal to orthometric)
h = <input_height>
H = h - N
print(f"Geoid undulation N = {N:.4f} m")
print(f"Ellipsoidal h     = {h:.4f} m")
print(f"Orthometric H     = {H:.4f} m")

# H → h  (orthometric to ellipsoidal)
# H = <input_height>
# h = H + N
```

Key API notes:
- `geoid(LatLon(lat, lon))` returns geoid undulation N in metres
- `GeoidKarney` loads the whole `.pgm` once; reuse the object for batch conversions
- Latitude range: −90 to +90; longitude: −180 to +180 (or 0–360, both accepted)

## Step 3 — Report results

Always report all three quantities: **N**, **h**, and **H**, with units (metres).
If the user provides multiple points, format as a table.

Example single-point output:
```
Location     : lat=48.8000°, lon=2.3000°
Geoid (N)    : +44.661 m
Ellipsoidal h: 100.000 m  →  Orthometric H: 55.339 m
```

## Edge cases

- **Multiple points from a file** (CSV/parquet/etc.): load with pandas/geopandas, call
  `geoid(LatLon(row.lat, row.lon))` per row (or vectorise with a list comprehension), append
  result columns, and save back.
- **Points near poles or dateline**: pygeodesy handles wraparound; pass `wrap=True` if the
  user's longitudes span the antimeridian.
- **pygeodesy not installed**: run
  `conda run -n sme-chain pip install pygeodesy` then retry.
- **PGM file missing**: the file should be at the path above; if absent, tell the user and
  ask them to confirm the path — do not attempt to download it.

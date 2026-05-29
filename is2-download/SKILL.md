---
name: is2-download
description: >
  Downloads ICESat-2 and ArcticDEM data using the sliderule-tool.
  Use this skill whenever the user wants to download, fetch, retrieve, or access
  ICESat-2 data products (ATL03 photons, ATL06 surface elevation, ATL08 vegetation,
  ATL24 bathymetry, ATL03x-surface fitted photons) or ArcticDEM elevation data
  for any region or time period.
  Trigger on phrases like "download ATL06", "get ICESat-2 data for...", "fetch
  bathymetry data", "download icesat2 ATL24", "get elevation data from sliderule",
  "download atl03x-surface", "surface fit photons", or any mention of ICESat-2
  products combined with a region or date range.
---

# ICESat-2 Download Skill

You help the user download ICESat-2 and ArcticDEM data via the sliderule-tool.

## Project context

- **Run environment**: `conda run -n sme-chain python` (always use this)
- **Entry point**: `sliderule_tool` is pip-installed in the env — import it from anywhere, no `cd` needed
- **Output directory**: use the user's current working directory, or an absolute path if they specify one

## Step 1 — Gather parameters

Before writing any code, confirm you have all required parameters. Ask for anything missing.

**Required:**
- `product` — one of: `atl03x`, `atl03x-surface`, `atl06`, `atl08`, `atl24`, `arcticdem` (`atl03` legacy, prefer `atl03x`)
- `region` — bounding box `(west, south, east, north)` **or** path to a `.geojson` / `.shp` file
- `t0`, `t1` — start and end date as `YYYY-MM-DD`

**Optional (use sensible defaults if not given):**
- `output_path` — absolute path preferred; default: `<cwd>/output/<product>_<YYYYMMDD_HHMMSS>.parquet`
  where `<cwd>` is resolved at runtime via `Path.cwd()` inside the script
- `res` — along-track resolution in metres (default `20.0`) — **not applicable for `atl03x`**
- `len` — segment length in metres (default `40.0`) — **not applicable for `atl03x` or `atl03x-surface`**
- `atl08_class` — for atl06/08 (default `["atl08_ground"]`) — **not applicable for `atl03x` or `atl03x-surface`**
- `cnf` — photon confidence filter for `atl03x-surface` (e.g. `icesat2.CNF_SURFACE_HIGH` or `["atl03_high"]`)
- `fit` — ATL06-SR surface fit params for `atl03x-surface` (default `{}` uses server defaults; can set `{"maxi": …, "H_min_win": …, "sigma_r_max": …}`)
- `atl24_class` — for atl24 (default `["bathymetry", "sea_surface"]`)
- `sampling_radius` — for arcticdem, metres (default `10.0`)

**Product guidance:**
- `atl03x` — **preferred** raw photon endpoint; returns absolute `x_atc`, per-photon `gt`/`spot`/`height`; no segment params needed
- `atl03x-surface` — `atl03x` with ATL06-SR surface fitting; returns per-segment fitted elevations (`h_mean`, `dh_fit_dx`, `w_surface_window_final`, `rms_misfit`, `h_sigma`, etc.); uses `res`/`ats`/`cnt`/`cnf`/`fit` params; calls `sliderule.run("atl03x", parm)` with `fit` included
- `atl03` — legacy segment-based photon subsetter (`atl03sp`); `x_atc` is relative to segment start, requires `segment_dist + x_atc`; avoid for new work
- `atl06` — land/ice surface elevation per segment
- `atl08` — vegetation height and canopy cover
- `atl24` — coastal/nearshore bathymetry; `srt` and `cnf` are set automatically
- `arcticdem` — ArcticDEM elevation sampled along ATL06 tracks

## Step 2 — Generate and run the script

Once parameters are confirmed, write a self-contained Python script and run it inline using the Bash tool.
`sliderule_tool` is installed in the conda env, so no `cd` is required — run from anywhere:

```bash
conda run -n sme-chain python - << 'EOF'
<script here>
EOF
```

If the user specifies an output path, use it as-is (absolute or relative to their current project).
If not, default to a timestamped file inside `output/` under the **current working directory** —
resolved at runtime so the script works regardless of where it is invoked.

### Script template

```python
import logging
from pathlib import Path
from datetime import datetime

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s", datefmt="%H:%M:%S")

from sliderule_tool import SlideRuleTool, RegionOfInterest

# --- Region ---
# bbox:    region = RegionOfInterest.from_bbox(west, south, east, north)
# geojson: region = RegionOfInterest.from_geojson("/absolute/path/to/file.geojson")
# shp:     region = RegionOfInterest.from_shapefile("/absolute/path/to/file.shp")

region = RegionOfInterest.from_bbox(WEST, SOUTH, EAST, NORTH)

# Output: absolute path or cwd-relative default
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
output_path = Path.cwd() / "output" / f"PRODUCT_{timestamp}.parquet"
output_path.parent.mkdir(parents=True, exist_ok=True)

with SlideRuleTool() as tool:
    gdf = tool.fetch(
        dataset="DATASET",
        product="PRODUCT",
        region=region,
        t0="T0",
        t1="T1",
        # product-specific kwargs here
        output_path=output_path,
    )

if gdf.empty:
    print("WARNING: No data returned. Try widening the AOI or date range.")
else:
    print(f"Records  : {len(gdf):,}")
    print(f"Columns  : {gdf.columns.tolist()}")
    print(f"Saved to : {output_path.resolve()}")
    print(gdf.head(3).to_string())
```

### Product-specific kwargs to add inside `tool.fetch()`

| Product | Extra kwargs |
|---|---|
| `atl03x` | *(none — no segment params needed)* |
| `atl03x-surface` | `res=2, ats=10.0, cnt=10, cnf=icesat2.CNF_SURFACE_HIGH, fit={}` |
| `atl03` | `res=10.0, len=40.0, cnt=0` *(legacy, prefer atl03x)* |
| `atl06` | `res=10.0, len=40.0, atl08_class=["atl08_ground"]` |
| `atl08` | `res=10.0, len=40.0, atl08_class=["atl08_canopy","atl08_top_of_canopy"]` |
| `atl24` | `atl24_class=["bathymetry","sea_surface"]` *(srt/cnf auto-set)* |
| `arcticdem` | `dataset="arcticdem"`, import and pass `sampling=ArcticDEMSamplingParams(radius=10.0)` |

For `atl03x-surface`, add this import:
```python
from sliderule import icesat2
# inside fetch():  cnf=icesat2.CNF_SURFACE_HIGH, fit={}
# fit={} uses server defaults; override with {"maxi": 10, "H_min_win": 5.0, "sigma_r_max": 3.0}
```

For `arcticdem`, add this import and kwarg:
```python
from sliderule_tool import ArcticDEMSamplingParams
# inside fetch():  sampling=ArcticDEMSamplingParams(radius=10.0)
```

## Step 3 — Report results

After the script runs, tell the user:
1. How many records were downloaded
2. The column names (briefly explain the key ones)
3. The full path to the saved file
4. Any warnings (empty result, version mismatch, etc.)

If the result is empty, suggest: widen the AOI, extend the date range, or check that the region has coastal/ice coverage appropriate for the product.

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)

# Drone_data_on_banana_plantation

## Overview

`A05_Banana_census.ipynb` converts ML detection outputs (CSVs + GeoTIFF tiles) into a geolocated, boundary-clipped GeoJSON of banana plants, with each vector point classified as either a **Parent** stalk or a **Sucker** offshoot using DSM-derived surface height. This is for a plantation block ideal for exteding to large field sizes.

---

<details>
<summary><strong>Pipeline Position</strong></summary>

```
ML Detection Output (CSVs + GeoTIFFs)
        ↓
  A05_Banana_census.ipynb              ← geolocation, boundary clip, parent/sucker classification
        ↓
  count_geo_output/model_points.geojson
        ↓  (manual QA / visualisation)
  [downstream cleaning / summary notebooks]
```

</details>

---

<details>
<summary><strong>A05_Banana_census.ipynb</strong> — geolocate & classify banana plants</summary>

Converts per-tile ML detection CSVs and paired GeoTIFF image tiles into a single deduplicated, boundary-clipped GeoJSON. Each plant point is then classified as a **Parent** or **Sucker** using the Digital Surface Model (DSM): plants within 1.5 m or specified planting distance of each other are paired, and the shorter plant in each pair is labelled a sucker.

### Inputs

| Parameter | Description | Example |
|---|---|---|
| `in_tiff_path` | Root directory of tiled GeoTIFF image tiles (one sub-folder per block) | `.../count/count_image_tiles/` |
| `in_dsm_path` | Directory of per-block DSM GeoTIFF files | `.../count/DSM_tiles/` |
| `in_csv_path` | Root directory of ML model output CSVs (one sub-folder per block, matched to tiles) | `.../count/count_ML_output/` |
| `in_boundary_path` | Directory of block boundary GeoJSON files | `.../boundary_data/geojson_data/` |

> Image tiles and CSV files must be organised into matching sub-folders named by block or plot (e.g. `block1/`). Within each block sub-folder the filenames must sort into matching pairs. The notebook validates this before processing.

### Outputs

| File | Location | Description |
|---|---|---|
| `model_points.geojson` | `<image_dir>/../count_geo_output/` | Deduplicated, boundary-clipped banana plant points with parent/sucker classification |

Each point includes: `Plant_id`, `block_id`, `lat`, `long`, `elevation_status`, `Surface_value` (weighted mean DSM DN), `status` (`Parent` or `Sucker`), `geometry`.

### Processing Steps

1. **Input validation** — confirms that the number of block sub-folders in the CSV directory matches the image directory, and that filenames within each block match.
2. **CSV–image pairing** — iterates each block sub-folder and sorts files into matched `[image, csv]` pairs.
3. **Geolocation** — reads pixel-space keypoint coordinates (`keypoint_x_center`, `keypoint_y_center`) from each CSV and converts them to geographic coordinates using the paired GeoTIFF's affine transform.
4. **GeoDataFrame creation** — assembles a `GeoDataFrame` per block, assigns `block_id`, and drops duplicate geometries.
5. **Boundary clipping** — for each block, iterates all boundary GeoJSON files and retains the one where ≥ 95 % of points fall inside. Points outside the matched boundary are dropped. Raises `ValueError` if no boundary meets the threshold.
6. **DSM surface extraction** — buffers each point by 0.5 m and, within that buffer, extracts DSM pixel values, computes distances from the buffer centroid, keeps the 50th-percentile closest pixels, and calculates a proximity-weighted mean DN (`Surface_value`).
7. **Parent / sucker classification** — computes pairwise distances between all points in a block; pairs closer than 1.5 m are identified. Within each pair the taller plant (higher `Surface_value`) is labelled **Parent** and the shorter is labelled **Sucker**.
8. **Merge & deduplicate** — all per-block `GeoDataFrame`s are concatenated and duplicate geometries removed.
9. **ID assignment** — sequential `Plant_id` values (1-based string) are assigned to the merged output.
10. **Export** — saved as `model_points.geojson` under `count_geo_output/`.

### Class Methods

**`__init__(in_tiff_path, in_dsm_path, in_csv_path, in_boundary_path)`** — validates that input CSV and image directories exist; stores all paths.

**`image_csv_pair(img_cvs_dir)`** — builds a dictionary of `{block_name: [[img, csv], ...]}` pairs by sorting and zipping files from both parent directories.

**`img_csv_match()`** — asserts that block sub-folder counts and names match between the image and CSV directories.

**`aoi_gdf(in_gdf, in_bdry_path)`** — clips a `GeoDataFrame` to the first boundary file that contains ≥ 95 % of its points.

**`image_values(image, row_geometry)`** — masks a rasterio dataset to a polygon and returns raw DN values, positive DN values, and pixel x/y coordinates.

**`dn_distance(row_geometry, all_x, all_y, dn_val)`** — computes Euclidean distance from each positive-DN pixel to the polygon centroid.

**`percentile_filter(dn_distance, percentile_num, dn_vals)`** — retains only the DN values corresponding to the closest N-th percentile of pixels.

**`mean_dn(central_pxls)`** — computes a proximity-weighted mean DN (inverse weighting, normalised) from the filtered central pixels.

**`pair_distance(pt_gdf)`** — builds a full pairwise distance matrix and returns all point pairs closer than 1.5 m.

**`status(pt_gdf, plant_pairs)`** — labels the shorter plant in each close pair as `Sucker`; all others default to `Parent`.

**`parent_sucker(point_gdf, dsm_file)`** — orchestrates DSM extraction and parent/sucker classification for a single block.

**`generate_geolocators(img_csv_pairs)`** — iterates all blocks, geolocates detections, clips to boundary, and classifies parents and suckers.

**`process()`** — top-level entry point; calls all steps and saves the final merged GeoJSON.

### Usage

```python
input_image_dir    = '.../Project_repo/count/count_image_tiles/'
input_dsm_dir      = '.../Project_repo/count/DSM_tiles/'
input_csv_dir      = '.../Project_repo/count/count_ML_output/'
input_boundary_dir = '.../Project_repo/boundary_data/geojson_data/'

post_processing = Banana_Census(input_image_dir, input_dsm_dir, input_csv_dir, input_boundary_dir)
post_processing.process()
```

### Dependencies

```
rasterio, pandas, geopandas, numpy, scipy, pathlib, re
```

</details>

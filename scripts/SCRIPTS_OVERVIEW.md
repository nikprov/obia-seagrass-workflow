# Scripts overview: rationale, libraries, functionality

> **Disclaimer.** The code is **not published yet**. It is undergoing stability testing, safety hardening for non-expert users and publication preparation. The detailed methodology and the script repository will be released with the standalone publication. This page only summarises the rationale, key libraries and functionality of each tool, in workflow order.
>
> **Status note.** The Nov 2026 pipeline update (split finder v14, optimizer v17, implementator v9, segmenter v3.1) is being validated; real-data runs of the updated versions are pending.

**Conventions (all tools):** interactive CLI with `tqdm` progress; no GDAL/OTB calls from Python (rasterio, geopandas, shapely, scikit-image, scikit-learn instead); versioned file names (`_v3_1`); rasters written as tiled, DEFLATE with PREDICTOR 2, input data type preserved unless stated.

**Workflow in one line:** field data → navigation + imagery → photomosaics → raster stack (VRT) → segmentation → zonal statistics → split → optimise → classify → manual review → statistics → report.

---

## A. Field data and ground truth

### 1. `fifish_log_suite.py` (ROV navigation log repair)
- Reads the FIFISH W6 DVL log (CSV, optional GPX) and repairs known export faults: malformed GPX header, rows missing the last column, non-standard elevation tag.
- Validity report: missing seconds, duplicate or out-of-order timestamps, position-jump outliers, sensor sanity.
- Seabed depth = −(ROV depth + DVL altitude).
- Outputs a full-resolution GPX/CSV and an optional 1 Hz table with circular interpolation of heading; a geometry-based heading is kept as a QA cross-check.
- Libraries: pandas, numpy.

### 2. `gps_merge_interpolate.py` (vessel/USBL track to 1 Hz)
- Reads SL2/SL3 sonar-log files (GPS, heading, speed); merges files chronologically; optional date filter and timezone shift.
- Cubic spline smoothing of lat/lon on user-set keypoints (3–30 s).
- Resamples to a strict 1 s grid: position from the smoothed track, heading and speed from the raw high-density data (circular interpolation for heading).
- Writes GPX with heading/speed extensions, plus CSV.
- Libraries: sonarlight, gpxpy, pandas, scipy.

### 3. `gopro_georef.py` (photo georeferencing and enhancement)
- Standalone steps: EXIF clock offset correction (from a photo of a reference clock); GPS matching to a GPX or 1 Hz CSV track by time interpolation; handling of photos outside the track window (extrapolation or manual coordinates).
- Writes GPS and heading (GPSImgDirection) into EXIF; heading only injected within ±0.5 s of a track point.
- Optional enhancement during export; all other EXIF preserved.
- Libraries: piexif, gpxpy, opencv, pandas.

### 4. `CLAHE_underwater_preprocessor.py` (image enhancement)
- Three optional stages, in fixed order: gray-world colour-cast correction (strength 0–1), CLAHE on the L* channel (CIE-LAB), per-channel histogram stretch (percentile, min-max or mean ± kσ).
- Rationale: balance the colour first so that luminance enhancement is not biased by the cast.
- EXIF/GPS preserved; threaded batch processing.
- Libraries: opencv, numpy, Pillow, piexif.


---

## B. Raster preparation

### 5. `quick_raster_check.py` (nodata diagnostic)
- Reports per-band data type, nodata value and valid-pixel share, to expose layers that do not overlap.
- Runs before building the stack.
- Library: rasterio.

---

## C. Segmentation (suite)

Files: `obia_segmenter.py` (CLI), `obia_segmenter_gui.py` (GUI), `obia_engine.py`, `obia_vrt_io.py`, `obia_vector.py`, `obia_common.py`.

### 6. Segmenter (primitives → graph → clean → polygons)
- Reads the multiband VRT one band at a time (mixed data types are safe); never copies the VRT profile.
- **Domains** (e.g. deep/mid/shallow): defined by support bands, optional zone band range (e.g. depth) and priority; the first matching domain claims a pixel. Each domain has its own robust z-scores (1–99 % clipped), weights, optional log transform and Gaussian smoothing that ignores missing pixels.
- **Stage 1:** tiled, parallel Felzenszwalb over-segmentation into small primitives (scale auto-calibrated by bisection to a target size); primitives never cross tile edges, so no stitching step exists.
- **Stage 2:** global adjacency graph; optional Mumford-Shah hierarchy (higra) for merging; contrast-aware elimination of small regions (soft/hard minimum size, tolerance τ, so small but spectrally distinct patches survive).
- **Output:** uint32 label raster; every valid pixel has exactly one label, so gaps and overlaps cannot exist (verified); polygons derived from it; `--recut` re-cuts in seconds; JSON run record.
- Libraries: rasterio, scikit-image, scipy, numpy, higra (optional), shapely/geopandas.

---

## D. Feature extraction

### 7. `zonal_statistics_calculator.py`
- Computes per-polygon mean, median, std and var for every band of the VRT (and auxiliary layers such as depth, slope, distance), producing the `bN_stat` columns used by the classifiers.
- VRT-safe: band-by-band reads, per-band validity masks, striped polygon reads, thread-local file handles.
- Pixels with nodata are excluded; polygons with no valid pixels receive NaN, which the tree models handle natively.
- Libraries: rasterio, geopandas, numpy.

---

## E. Sampling design and modelling

Shared modules: `obia_splits.py`, `obia_stats.py`, `obia_calibration.py`.

### 8. `optimal_split_finder.py` (train/test split)
- Searches for a spatially defensible split of labelled polygons: random, grid or stripe candidates.
- Checks per candidate: achieved train/test ratio versus the requested one, class guards (every class present in both parts), Clark–Evans nearest-neighbour index of the sample pattern.
- Representativeness index (editable weights); the representative split is locked rather than taking the single best-scoring candidate, to limit the winner's-curse effect.
- Writes train and test GPKG plus `split_metadata_*.json` (includes the spatial block size).

### 9. `universal_optimizer.py` (hyperparameter search)
- Models: Random Forest, Extra Trees, HistGradientBoosting; all NaN-native, no scaler, no imputation.
- Cross-validation: StratifiedGroupKFold on spatial blocks (block size from the split metadata or automatic); folds are redrawn until each class appears in each validation fold.
- Objective (per model): 0.40·balanced accuracy + 0.35·kappa + 0.25·macro-F1.
- Search: full grid, randomised, or successive halving; parallel with joblib, hardware-aware worker cap (RAM, battery); shortcuts that give exact scores (forest sub-ensembles, HGB staged prediction).
- Test-set report: bootstrap 95 % CIs, exact McNemar tests between models, raw-probability diagnostics (ECE, Brier, log-loss). Sample SD (ddof = 1).
- Libraries: scikit-learn, joblib, geopandas, matplotlib/seaborn.

### 10. `universal_implementator.py` (final training and map)
- Reads the optimiser JSON (hyperparameters, HGB early-stopping pinned to False).
- Calibration of all three models, default cross-fitted temperature scaling (class labels unchanged); Platt and isotonic available.
- Ensemble: hard majority vote is primary; soft vote is also written; tie-break strategies (min class, random, max confidence).
- Interactive ecological rules (e.g. slope or depth limits), applied after prediction; corrected polygons get NaN confidence.
- Writes the classified GPKG with per-model class, confidence, agreement, flags and `split_role` (train/test/unlabelled): the map should be judged on the unlabelled polygons.
- Libraries: scikit-learn, scipy, geopandas.

### Shared modules
- `obia_splits.py`: centroid extraction, spatial blocks, Clark–Evans, safe balanced accuracy, grouped CV splits.
- `obia_stats.py`: sample SD, Nadeau–Bengio corrected CI, bootstrap metrics, exact McNemar, Wilson CI.
- `obia_calibration.py`: temperature/Platt/isotonic calibrators, cross-fitting, ECE, Brier, log-loss, reliability plots.

---

## F. Review and reporting

### 11. Manual review (QGIS)
- The classified GPKG is edited by hand in QGIS (corrections stored in a `manual_edits` column).

### 12. `habitat_stats_extractor.py`
- Reads the edited GPKG and the implementator JSON; computes habitat areas, proportions, confidence summaries and accuracy figures.
- Writes a statistics JSON and a Markdown summary for the report.
- Libraries: geopandas, pandas.

### 13. `report_assembler.py` (+ `REPORT_TEMPLATE.docx`)
- Fills a Word template from tokens and conditional flags, using the statistics JSON, `project_config.json` and `pipeline_state.json`.
- Produces the report body and a review checklist of items that still need author input.
- Libraries: python-docx, lxml.

---

## Reading order for a new project (short)
1. Navigation logs → 1 Hz track → geotag frames → mosaics (A).
2. Check rasters → stack as VRT (B).
3. Segment (C) → zonal statistics (D).
4. Split → optimise → implement (E).
5. Review in QGIS → statistics → report (F).

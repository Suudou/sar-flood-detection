# SNAP Preprocessing Pipeline

This directory contains the SAR preprocessing pipeline that converts raw 
Sentinel-1 GRD products into analysis-ready geocoded sigma0 in dB scale, 
ready for change detection in Python.

## Files

- `flood_pipeline.xml` — the working canonical pipeline (8 operators)
- `tests/` — diagnostic graphs built incrementally to debug a silent failure
- `tests/README.md` — narrative of the diagnostic process

## Pipeline overview
Read → Apply-Orbit-File → Calibration → Speckle-Filter →
Terrain-Correction → Subset → LinearToFromdB → Write

Each stage is described below with description for what it does and 
why it must be in this order.

## Operator-by-operator

### 1. Read

Loads the Sentinel-1 `.SAFE` product (manifest + bands + metadata).
The `.SAFE` is a structured directory with calibration LUTs, noise vectors, 
orbit ephemerides, and the actual Intensity/Amplitude rasters per polarization.

### 2. Apply-Orbit-File

Replaces the rough orbit ephemerides packaged with the scene with **precise orbit data** (POEORB), published by ESA ~3 weeks after acquisition.

**Why it matters:** SAR geolocation depends on knowing exactly where the 
satellite was. Without precise orbits, geolocation error is tens of meters; 
with them, it's sub-meter. **Required by Terrain-Correction** — without Apply-Orbit-File, TC has insufficient geometric information and either fails or returns invalid output.

### 3. Calibration

Converts raw digital numbers (DN) to **sigma0** — calibrated radar 
backscatter coefficient in linear scale (typically 0.0001 to ~5.0).

**Why:** Two scenes acquired at different times have different sensor gain 
settings. Without calibration, before/after comparisons are meaningless. 
Calibrated sigma0 is the physical quantity in which water (specular 
reflection) and land (diffuse scattering) are comparable across acquisitions.

**Output bands:** `Sigma0_VV` (linear scale, NOT dB — see step 7).

### 4. Speckle-Filter (Refined Lee, 5×5 window)

Reduces multiplicative speckle noise inherent to coherent SAR imaging.

**Why Refined Lee specifically:** classical filters (Boxcar, Lee 1980) 
smooth speckle but blur edges. Refined Lee (Lee 1981) uses local statistics 
and directional masks to **preserve edges** — critical for our use case 
where flood boundaries are the signal of interest.

**Why before TC, not after:** speckle filters assume the multiplicative 
noise model holds — this requires linear-scale data with original radar 
geometry sampling. Filtering after geocoding can produce artifacts at the 
resampling boundary.

### 5. Terrain-Correction (Range-Doppler, Copernicus 30m DEM)

Projects the image from radar geometry (range × azimuth, slant) to a 
geographic grid (EPSG:4326, lat/lon) using a Digital Elevation Model.

**Why we need it:** SAR measures slant range — a hill appears closer to 
the satellite than its base, so raw SAR images are geometrically distorted 
(foreshortening, layover, shadow). Terrain-Correction inverts this 
distortion using DEM-derived geometry, producing a map-projected raster.

**DEM choice:** Copernicus 30m Global DEM (TanDEM-X based, hosted on ESA 
infrastructure) chosen over SRTM 1Sec HGT after diagnostic testing 
(see `tests/README.md`).

**Pixel spacing: 20m** — matches native S1 IW GRDH resolution. Subsampling 
would lose information; oversampling would create artificial detail.

### 6. Subset

Crops the geocoded raster to the AOI (Galveston Bay area, ~1500 km²).

**Why after TC, not before:** Subset removes parts of the abstract metadata 
that Terrain-Correction requires. Applying Subset before TC was the root 
cause of silent failure in earlier diagnostic pipelines (see `tests/README.md`). 
Placing Subset after TC is correct *and* memory-efficient — SNAP's lazy 
evaluation propagates the AOI bounds backward through the graph, so TC 
only computes the pixels needed for the final crop.

### 7. LinearToFromdB

Converts sigma0 from linear scale to **decibel scale**: 
`sigma0_dB = 10 · log10(sigma0_linear)`.

**Why dB:** sigma0 spans several orders of magnitude (water ~0.001, 
buildings ~3.0). In linear scale, all non-urban pixels look like zero 
in any visualization or threshold. In dB, the range becomes ~−25 dB 
(water) to ~+5 dB (buildings) — uniformly distributed, easy to threshold 
for change detection.

**Why last, not earlier:** Speckle-Filter and Terrain-Correction operate 
on linear-scale sigma0 (assume multiplicative noise / radiometric scaling). 
Converting to dB earlier would break those operators' assumptions.

### 8. Write

Exports the final raster as **GeoTIFF** — the standard format for 
geospatial rasters, readable by Python (rasterio), QGIS, ArcGIS, etc.

**Note:** GeoTIFF discards SAR-specific metadata. This is fine for the 
final output (downstream Python analysis doesn't need orbit vectors), but 
means intermediate results cannot be saved as GeoTIFF and re-fed into 
later SAR operators (a constraint that motivated the diagnostic process).
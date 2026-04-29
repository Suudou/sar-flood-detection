# Diagnostic SNAP Graphs

These pipelines were built incrementally to isolate why the full preprocessing 
chain returned zero rasters.

## Test progression
| File | Stage added | Result | Conclusion |
|---|---|---|---|
| `test1_subset_only.xml` | Read + Subset |  24M valid pixels (raw DN) | Subset extracts AOI correctly |
| `test2_calibrated.xml` | + Calibration |  Sigma0 in linear scale (0.0001-665) | Calibration produces sigma0 |
| `test3_speckle.xml` | + Speckle-Filter (Refined Lee 5×5) |  Smoothed, outliers clipped | Filter preserves data |
| `test4_terrain_srtm.xml` | + TC with SRTM 1Sec HGT |  All zeros | TC failure isolated |
| `test5_terrain_copernicus.xml` | TC with Copernicus 30m |  Still all zeros | Not a DEM issue |

 Rest of the tests where done manualy, above tests were done to help isolate single operators and repair silent errors.

## Note on minimalism

These diagnostic pipelines are intentionally minimal — they omit `Apply-Orbit-File` to isolate single operators. This minimalism turned 
out to be one contributing factor to TC failure. Manual invocation of 
Terrain-Correction in the SNAP GUI revealed the actual error 
(`Input should be a SAR product`), pointing to two issues:

1. Apply-Orbit-File is required by TC (provides orbit state vectors)
2. Subset before TC removes SAR metadata that TC needs

## Working pipeline

The corrected pipeline lives at `../flood_pipeline.xml` and adds 
Apply-Orbit-File + reorders Subset to come after Terrain-Correction.

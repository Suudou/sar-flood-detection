# Data

This project uses Sentinel-1 Synthetic Aperture Radar (SAR) data acquired before 
and after Hurricane Beryl (landfall: 8 July 2024, Texas coast).

## Source data — Sentinel-1 GRD

Both scenes are not committed to this repository (~820 MB each, exceeds 
GitHub limits and would bloat the repo history).

| Scene | Date (UTC) | Role | Product ID |
|---|---|---|---|
| Before | 2024-06-29 00:26 | Pre-event (10 days before landfall) | `S1A_IW_GRDH_1SDV_20240629T002653_20240629T002718_054531_06A2F0_DE05` |
| After | 2024-07-11 00:26 | Post-event (3 days after landfall) | `S1A_IW_GRDH_1SDV_20240711T002652_20240711T002717_054706_06A90A_B225` |

### Acquisition geometry

Both scenes share **identical geometry**, eliminating geometric artifacts in change detection:

- **Mode:** IW (Interferometric Wide swath)
- **Polarization:** Dual VV+VH (`1SDV`)
- **Resolution:** Ground Range Detected, High resolution (~10×10 m, resampled to 20 m)
- **Pass:** Descending
- **Relative orbit:** Same track (143)
- **Acquisition time difference:** ~1 second between before/after acquisitions in 
  their respective orbital cycles, 12 days apart

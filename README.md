 # FloodPrint

**Spatiotemporal verification of NWS Flash Flood Warnings across CONUS for the 2024 warm season.**

FloodPrint matches every Flash Flood Warning (FFW) polygon issued by the National Weather Service against observed Flash Flood Local Storm Reports (LSRs) to measure, at national scale, how often warnings actually verified — and how that skill varies geographically. The headline result is that the national verification rate is far lower than figures commonly cited in the literature, and that the gap is driven largely by **where storm reports get collected**, not by where forecasters get it wrong.

## Key findings

- **National FFW verification rate: ~45.2%** for the 2024 warm season (April–September). This is well below the 70–85% range frequently cited in NWS verification literature, in part because this study uses a strict, observation-grounded definition of a "hit" (an LSR must spatially and temporally coincide with the warning).
- **Strong geographic gradient.** Verification rates range from roughly **0–13% in the desert Southwest** to **77–85% in the Northeast**.
- **The gradient tracks spotter density, not forecast skill.** Regions with sparse reporting networks produce few LSRs, so genuine flash floods often go unreported and warnings appear "unverified." The geography of verification largely mirrors the geography of who is around to file a report. This framing is the central interpretive caveat of the study and should accompany any use of these numbers.

These figures depend directly on the matching choices below (60-minute grace period, 5 km LSR buffer). They are descriptive verification statistics, not a measure of forecaster performance.

## Repository structure

```
FloodPrint/
├── notebooks/
│   ├── LSRs.ipynb            # Fetch + QC Flash Flood LSRs from IEM
│   ├── Warnings.ipynb        # Fetch + QC FFW warning polygons from IEM
│   ├── Join.ipynb            # Spatiotemporal join, verification + false-alarm labels
│   ├── Herbie_Helene.ipynb   # Case study — Hurricane Helene HRRR fields vs. LSRs
│   └── HRRR.ipynb            # HRRR field extraction for the case study
├── figures/                  # Rendered output figures (see below)
├── src/                      # Reserved for refactored modules
├── requirements.txt
├── LICENSE                   # CC BY 4.0
└── README.md
```

> **Data files are not committed.** Raw and processed `.parquet`/`.csv`/GRIB2 files are excluded via `.gitignore`. The notebooks regenerate them from the IEM API and NOAA's HRRR archive. The cleaned datasets are also published on Hugging Face — see [Data](#data).

## Pipeline

The study runs as a sequence of self-contained notebooks. Each reads from and writes to a local `data/` directory consumed by the next.

### Local Storm Reports (`LSRs.ipynb`)
Fetches all Flash Flood LSRs for the 2024 warm season from the Iowa Environmental Mesonet (IEM) API, month by month, then filters to flash-flood type (`TYPECODE == 'F'`) in Python. Applies quality control (CONUS bounding-box filter, timestamp parsing) and writes `lsr_clean_ws2024.parquet`. Also produces a national LSR location map and a kernel-density map of report coverage.

### Warning polygons (`Warnings.ipynb`)
Fetches FFW polygons from the IEM watch/warning archive, again month by month to avoid server timeouts. Filters to Flash Flood + Warning significance + polygon geometry (`PHENOM == 'FF'`, `SIG == 'W'`, `GTYPE == 'P'`), runs QC on issue/expire timestamps, assigns a stable `WARNING_ID`, and writes `ffw_polygons_ws2024.parquet`. Produces an FFW-polygon / LSR overlay map.

### Spatiotemporal join (`Join.ipynb`)
The analytical core. For every warning polygon:
1. **Spatial pre-filter** — query a spatial index on the *full* LSR set by polygon bounding box to get candidate reports.
2. **Temporal filter** — keep candidates whose timestamp falls between the warning's issue time and expiration **+ a 60-minute grace period**.
3. **Verification test** — buffer each surviving LSR by **5 km** and test whether the buffer *intersects* the warning polygon (a buffer-intersects test, not a bare point-in-polygon test). A warning is `verified = True` if at least one buffered LSR intersects it.
4. **False alarm fraction** — union all intersecting LSR buffers, intersect that union with the polygon, and compute the fraction of polygon area left uncovered. Unverified warnings are set to `1.0` (fully unverified) for consistency with the binary label.

Centroids are computed in EPSG:5070 (Albers Equal Area) and reprojected back to lat/lon; month and issue-hour are attached for downstream analysis. Output: `ffw_labeled_ws2024.parquet`. The notebook also renders the WFO verification-rate map, monthly verification trend, and false-alarm-fraction distributions.

### Hurricane Helene case study (`Herbie_Helene.ipynb`, `HRRR.ipynb`)
For selected Helene warnings (Sep 26–30, 2024), downloads HRRR `f01` fields (precipitable water, CAPE, composite reflectivity) via [Herbie](https://herbie.readthedocs.io/) and plots them alongside the warning polygon and the LSRs that fell inside it — a qualitative look at how environmental fields related to verification for one high-impact event.

## Data

| Dataset | Description |
| --- | --- |
| `lsr_clean_ws2024.parquet` | QC'd Flash Flood LSRs, CONUS, Apr–Sep 2024 |
| `ffw_polygons_ws2024.parquet` | QC'd FFW warning polygons with `WARNING_ID` |
| `ffw_labeled_ws2024.parquet` | Warnings joined to LSRs with `verified`, `false_alarm_fraction`, `n_lsr_inside`, `first_lsr_minutes`, centroid coordinates, and month/hour fields |

The cleaned and labeled datasets are published on Hugging Face in the **[FloodPrint collection](https://huggingface.co/collections/harshithkethavath/floodprint)** (CC BY 4.0):

- [`FloodPrint-LSR-WS2024`](https://huggingface.co/datasets/harshithkethavath/FloodPrint-LSR-WS2024) — cleaned Flash Flood LSRs
- [`FloodPrint-FFW-Polygons-WS2024`](https://huggingface.co/datasets/harshithkethavath/FloodPrint-FFW-Polygons-WS2024) — QC'd FFW warning polygons
- [`FloodPrint-FFW-Labeled-WS2024`](https://huggingface.co/datasets/harshithkethavath/FloodPrint-FFW-Labeled-WS2024) — warnings labeled with verification and false-alarm metrics

**Sources:** [Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/) (LSRs and warning polygons) and NOAA's HRRR archive on AWS S3 via Herbie.

## Figures

| File | What it shows |
| --- | --- |
| `lsr_locations.png` | National map of all Flash Flood LSR points |
| `lsr_density.png` | Kernel-density map of LSR reporting coverage (the spotter-density story) |
| `ffw_lsr_overlay.png` | Warning polygons overlaid with LSR points |
| `wfo_verification_rate.png` | Verification rate by NWS Weather Forecast Office |
| `monthly_verification.png` | Verification rate and mean false-alarm fraction by month |
| `false_alarm_fraction.png` | Distribution of false-alarm fraction, verified vs. unverified |
| `helene_hrrr_3panel.png` | Helene case study: HRRR PWAT / CAPE / reflectivity with LSR overlay |

## Installation & usage

Tested with Python 3.10.

```bash
git clone https://github.com/harshithkethavath/FloodPrint.git
cd FloodPrint
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

`cfgrib`/`eccodes` and `cartopy` have system-level dependencies; installing them via conda/conda-forge is often the smoothest path on macOS and Linux.

Run the notebooks in order — `LSRs.ipynb` → `Warnings.ipynb` → `Join.ipynb` — to regenerate the datasets from scratch, then `Herbie_Helene.ipynb` for the case study. Each notebook reads from and writes to a local `data/` directory (created on first run, git-ignored).

## Method notes & reproducibility caveats

A few decisions materially affect the results and are worth flagging for anyone reusing this work:

- **Verification is observation-limited.** An "unverified" warning means no qualifying LSR was found — not that no flash flood occurred. In low-spotter-density regions this systematically depresses the verification rate. Treat regional comparisons accordingly.
- **The 60-minute grace period and 5 km buffer are tunable.** Both were chosen as reasonable defaults consistent with the spatial/temporal uncertainty of LSRs; widening or narrowing them shifts the verification rate.
- **IEM type/WFO filters are unreliable**, so the pipeline fetches all records and filters in Python.
- **GRIB longitudes** from HRRR are stored in 0–360 format and require conversion to ±180 before plotting.
- Equal-area operations (areas, buffers, centroids) use **EPSG:5070**; lat/lon is restored only for display.

## Citation

If you use the data or methodology, please cite:

```bibtex
@misc{floodprint2026,
  title     = {FloodPrint: Spatiotemporal verification of NWS Flash Flood Warnings across CONUS for the 2024 warm season},
  author    = {Harshith Kethavath},
  year      = {2026},
  publisher = {GitHub},
  url       = {https://github.com/harshithkethavath/FloodPrint}
}
```

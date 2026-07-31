# cropwatch

End-to-end agricultural field monitoring pipeline using Sentinel-2 satellite imagery and Google Earth Engine.
 
Produces four outputs per field:
- Vegetation health map (Healthy / Moderate Stress / High Stress)
- Water status map (Adequate / Mild Deficit / Strong Deficit)
- Field uniformity score with spatial hotspot detection
- Automated agronomic report with rainfall context
---
 
## How It Works
 
The pipeline pulls Sentinel-2 Surface Reflectance composites (ESA Copernicus, via GEE) and computes four spectral indices server-side before any local download:
 
| Index | Bands | Purpose |
|---|---|---|
| NDVI | B8, B4 | Vegetation density and photosynthetic activity |
| EVI | B8, B4, B2 | NDVI saturation correction in dense canopy |
| NDWI (Gao 1996) | B8, B11 | Canopy liquid water content |
| NDMI | B8, B12 | Plant and soil moisture stress |
 
Health classification uses a weighted NDVI + EVI score; water classification uses a combined NDWI + NDMI score. The two maps are intentionally independent so the pipeline can distinguish vegetation stress from water stress.
 
Spatial anomaly detection runs Isolation Forest on all four indices simultaneously, then DBSCAN to cluster anomaly pixels into coherent hotspot zones. A 30-day CHIRPS rainfall layer provides environmental context for interpreting whether water deficit reflects drought or soil/irrigation issues.
 
---
 
## Quickstart
 
### Prerequisites
 
- Python 3.9+
- A Google Earth Engine account with an initialized project
- GEE Python API authenticated
```bash
pip install earthengine-api geemap geopandas shapely scikit-learn folium
```
 
### Authentication
 
On first run, uncomment the authentication line in Section 1:
 
```python
ee.Authenticate()  # uncomment on first run only
ee.Initialize(project='your-gee-project-id')
```
 
### Configure Your Field
 
All parameters are in **Section 0** of the notebook. At minimum, set your field coordinates and GEE project:
 
```python
AOI_LAT = 45.15        # Field center latitude
AOI_LON = 11.80        # Field center longitude
AOI_BUFFER_KM = 2.0    # Buffer radius (max ~5.6 km)
DATE_CENTER = '2024-07-15'
GEE_PROJECT = 'your-project-id'
```
 
Then run all cells in order (Sections 0 through 10).
 
---
 
## Repo Structure
 
```
crop-health-classifier/
├── crop_health_classifier_sentinel2.ipynb   # Main pipeline notebook
├── README.md
├── requirements.txt
├── .gitignore
└── sample_outputs/                          # Optional: example maps and report
    ├── health_map.png
    ├── water_map.png
    ├── uniformity_hotspots.png
    └── sample_report.txt
```
 
---
 
## Configuration Reference
 
| Parameter | Default | Description |
|---|---|---|
| `AOI_LAT`, `AOI_LON` | 45.15, 11.80 | Field center (Po Valley, Italy) |
| `AOI_BUFFER_KM` | 2.0 | Buffer radius in km (max ~5.6) |
| `DATE_CENTER` | 2024-07-15 | Analysis date |
| `DATE_WINDOW_DAYS` | 10 | ±days for median composite |
| `CLOUD_THRESHOLD` | 20 | Max cloud cover % per scene |
| `HEALTH_THRESHOLDS` | [0.5, 0.3] | Healthy / Moderate / High Stress boundaries |
| `WATER_THRESHOLDS` | [0.0, -0.3] | Adequate / Mild / Strong Deficit boundaries |
| `RAINFALL_LOOKBACK_DAYS` | 30 | CHIRPS rainfall window |
| `ISOLATION_CONTAMINATION` | 0.05 | Anomaly fraction for Isolation Forest |
| `N_HOTSPOTS` | 5 | Max hotspot zones to report |
 
---
 
## Design Notes
 
**Why four indices instead of one?** NDVI alone saturates in dense canopies and conflates water stress with general vegetation decline. Using NDVI + EVI for health and NDWI + NDMI for water provides two independent signals per dimension, increasing classification confidence.
 
**Why Isolation Forest + DBSCAN for hotspots?** Simple per-index thresholding flags pixels that are low on one signal. Isolation Forest flags pixels that are simultaneously anomalous across all four signals, which is a stronger stress indicator. DBSCAN then groups spatially adjacent anomaly pixels into coherent zones rather than reporting isolated pixels.
 
**Cloud masking approach:** Uses QA60 bitmask (ESA standard). SCL-based masking is noted as a future improvement for finer-grained cloud shadow detection.
 
---
 
## Extending the Pipeline
 
The notebook includes starter code in **Section 11** for:
 
- **Multi-field batch processing** — every function is parameterized by `aoi` and `date_center`, so wrapping in a loop over a field registry CSV requires no refactoring
- **Temporal change detection** — run `load_sentinel2()` across monthly intervals to build NDVI time series and detect early stress trends
- **Deforestation monitoring** — swap NDVI/EVI for NDFI and add NBR; the Isolation Forest anomaly detection transfers directly
- **Additional environmental layers** — SMAP soil moisture, MODIS LST, ESA WorldCover crop type mask
---
 
## Data Sources
 
- **Sentinel-2 Surface Reflectance:** `COPERNICUS/S2_SR_HARMONIZED` via Google Earth Engine
- **Rainfall:** `UCSB-CHG/CHIRPS/DAILY` via Google Earth Engine
---
 
## Requirements
 
See `requirements.txt`. Key dependencies:
 
```
earthengine-api
geemap
numpy
pandas
matplotlib
scikit-learn
folium
geopandas
shapely
```

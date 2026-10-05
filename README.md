# Bangkok Heat-Risk Data Validation

This project validates Bangkok daily maximum and minimum temperature estimates against local AirBKK station observations. The project compares three evaluation arms:

1. **Arm 1 — Raw ERA5:** raw ERA5 values returned by the HeatReady API.
2. **Arm 2 — Global model:** HeatReady global-model corrections that pass the required `base_source` and `delta_c` checks.
3. **Arm 3 — Local Random Forest:** separate local models for Tmax and Tmin trained with ERA5-Land grid values and AirBKK observations.

## Notebooks

Run the notebooks in this order:

1. [`Bangkok data preprocess.ipynb`]
   - Reads hourly AirBKK observations and station metadata.
   - Converts timestamps to `Asia/Bangkok`.
   - Converts temperature values to numeric form.
   - Retains physically plausible values from 15–45°C.
   - Assigns stations to Bangkok districts with a spatial join.
   - Accepts station-days with at least 20 of 24 readings (the Guide's 80% threshold).
   - Aggregates hourly observations into daily Tmax and Tmin ground truth.

2. [`Bangkok Validation.ipynb`]
   - Retrieves historical HeatReady API results.
   - Builds Arm 1 and Arm 2 from the API response.
   - Retrieves ERA5-Land values at district centroids for Arm 3.
   - Splits districts into train and test sets using `seed=42` and the Guide's approximately 80/20 district-level split.
   - Trains separate Random Forest models for Tmax and Tmin using `n_estimators=200` and `random_state=42`.
   - Calculates Guide-style RMSE and a supplementary comparison on the same district-days.
   - Exports row-level results, RMSE, and coverage tables as CSV files.

## Evaluation design

The train/test split is performed by district rather than by row. A district appears in either train or test, never both. With 49 districts containing ground truth, the split produces 40 train districts and 9 test districts.

RMSE is calculated separately for Tmax and Tmin. The reported aggregate RMSE is calculated across matched district-day rows; it is not an average of district-level RMSE values.

Arm 2 is available only when:

- the source row has `data_source=era5`;
- `downscaled` is present;
- `downscaled.base_source=era5_land`; and
- the relevant target has a non-null `delta_c`.

Missing Arm 2 corrections remain missing and are not replaced with zero.

## Python requirements

The notebooks require Python 3 and the following packages:

```text
pandas
numpy
geopandas
requests
scikit-learn
openpyxl
IPython
```

Install missing packages in the same Jupyter kernel used to run the notebooks.

## HeatReady credentials

`Bangkok Validation.ipynb` reads HeatReady credentials from:

```text
API Extraction/heatready_credentials.local.json
```

Create the file locally with this structure:

```json
{
  "username": "YOUR_USERNAME",
  "key": "YOUR_API_KEY"
}
```

Do not place real credentials directly in a notebook cell, output, README, committed JSON file, issue, or pull request.

The current notebooks reference the local credentials file but do not contain the actual username or API key.

## Generated files

The notebooks may create or overwrite these local outputs:

- `AirBKK_daily_tmax_tmin_guide.csv`
- `era5_land_centroids_20260601_20260902.local.json`
- `bangkok_validation_test_rows_20260601_20260902.csv`
- `bangkok_validation_rmse_20260601_20260902.csv`
- `bangkok_validation_district_coverage_20260601_20260902.csv`
- `bangkok_validation_monthly_coverage_20260601_20260902.csv`

The ERA5-Land cache contains grid data retrieved for district centroids and does not contain HeatReady credentials.

## Known coverage limitations

- Ground truth is available for 49 of Bangkok's 50 districts; Bang Kapi has no accepted station-days.
- Suan Luang has 30 accepted days, all in June 2026.
- Arm 1 and Arm 3 are evaluated on 835 district-day pairs across 9 test districts.
- Arm 2 has 577 qualifying corrected pairs across 7 test districts. Its RMSE does not represent all 9 test districts.
- Arm 3 uses ERA5-Land values at district centroids rather than the exact station coordinates.
- The notebook calls the Open-Meteo Historical API directly because the Guide's `src/open_meteo.py` helper is not present in this workspace.

## Licenses

When publishing HeatReady or ERA5-derived results, include the applicable Copernicus/C3S attribution required by the Guide and the relevant data licenses.

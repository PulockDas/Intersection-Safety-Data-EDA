# Project Context — USDOT Intersection Safety Challenge Stage-1B EDA

> **For the AI assistant:** Read this file at the start of every new session.  
> It summarises what has been built, what was discovered from running the notebooks,  
> and what the logical next steps are.

---

## 1. Project Goal

Exploratory Data Analysis (EDA) of the **USDOT Intersection Safety Challenge (ISC) Stage-1B**  
**VALIDATION subset** — understanding dataset structure, parsing every file type correctly,  
and building toward conflict detection / signal-phase alignment analysis.

---

## 2. Dataset

| Property | Value |
|----------|-------|
| Source URL | `https://data.transportation.gov/download/j58q-ymi4/application%2Fx-zip-compressed` |
| Download method | `wget` directly into Colab `/content/isc_stage1b.zip` (~28.6 GB) |
| Extracted to | `/content/isc_stage1b/` (~248 MB after selective extraction) |
| Extraction rule | Skip `.pcap` (112 files, ~18 GB) and `.mp4` (533 files, ~14 GB) |
| Total runs | **41 runs** (all at top level: `Run_1003/`, `Run_1023/`, … `Run_55/`, etc.) |
| No validation subfolder | The entire zip IS the validation set (no `validation/` directory inside) |

---

## 3. Workspace Structure

```
d:\Intersection Safety Data EDA\
├── notebooks/
│   ├── 00_dataset_structure_exploration.ipynb   ← general structure + GT inspection
│   ├── 01_non_gt_file_inspection.ipynb          ← non-GT file deep inspection
│   └── 02_parser_validation_and_data_dictionary.ipynb  ← parser layer + data dictionary
├── outputs/
│   └── tables/
│       └── parser_data_dictionary.csv           ← saved in Colab at /content/outputs/tables/
└── context.md                                   ← THIS FILE
```

All notebooks are **Google Colab-compatible**. Each notebook:
- Downloads the zip via `wget` if not already in `/content/`
- Selectively extracts (skips `.pcap`/`.mp4`) if not already extracted
- Both steps are no-ops if the same Colab session already has the files

---

## 4. Per-Run File Inventory (confirmed from notebook 01)

Every `Run_XXXX/` folder contains:

| File pattern | Present in | Size typical |
|---|---|---|
| `Run_XXXX_GT.csv` | all runs | ~165–207 KB |
| `ISC_Run_XXXX_ISC_all_timing.csv` | all runs | ~2 KB |
| `ISC_Run_XXXX_v2xhub_timing.csv` | runs with V2X hardware | ~276 B |
| `VisualCamera[1-8]_Run_XXXX_frame-timing.csv` | all runs | ~62–108 KB each |
| `ThermalCamera[1-5]_Run_XXXX_frame-timing.csv` | all runs | ~60–72 KB each |
| `V2XHubSensor_Run_XXXX.csv` | runs with V2X hardware | ~1.7–1.8 KB |
| `Radars_Run_XXXX_sensor[1-4].json` | runs with radar | ~665 B – 2.2 MB each |
| `Radars_Run_XXXX_traffic-triggers-output.json` | runs with radar | ~6–9 MB |

Root-level file (not inside any run folder):
- `conflict_no_conflict_labels_all_validation_runs.csv` — columns: `Run ID`, `Conflict_No_Conflict_Label`

---

## 5. Critical Parsing Rules (from notebook 02)

### Files with correct headers (parse normally)
| File | Header columns |
|---|---|
| `Run_XXXX_GT.csv` | `Time`, then `CLASS_xctr/yctr/zctr/xlen/ylen/zlen/xrot/yrot/zrot` |
| `VisualCameraX_Run_XXXX_frame-timing.csv` | `Image_number`, `Timestamp` |
| `ThermalCameraX_Run_XXXX_frame-timing.csv` | `Image_number`, `Timestamp` |

### Files that are HEADERLESS — must use `header=None`

| File | Correct column names | Bug if parsed normally |
|---|---|---|
| `ISC_Run_XXXX_ISC_all_timing.csv` | `sensor_type, sensor_name, ip_address, filename, start_time, end_time, duration` | First data row eaten as header |
| `ISC_Run_XXXX_v2xhub_timing.csv` | Same 7 columns as above | Same bug |
| `V2XHubSensor_Run_XXXX.csv` | `timestamp_ms, active_signal_ids, event_list` | First data row eaten + `[2, 8]` splits into 2 extra cols = 4 raw cols total |

### V2XHubSensor special fix
The value `[2, 8]` is unquoted, so pandas splits on the inner comma giving **4 raw columns**.
Parse with `header=None`, then merge `cols 1..(n-2)` back into `active_signal_ids`:
```python
df_raw = pd.read_csv(path, header=None)
n = df_raw.shape[1]  # will be 4, not 3
df_fix = pd.DataFrame({
    'timestamp_ms'      : df_raw.iloc[:, 0],
    'active_signal_ids' : df_raw.iloc[:, 1:n-1].astype(str).apply(','.join, axis=1).str.strip(),
    'event_list'        : df_raw.iloc[:, n-1],
})
```

---

## 6. Timestamp Formats (all files)

| File / Source | Format | Timezone |
|---|---|---|
| `Run_XXXX_GT.csv` `Time` column | Unix epoch **float** (seconds) | **UTC** |
| Camera frame-timing `Timestamp` | ISC string: `YYYY-MM-DD-HH-MM-SS_microseconds` | **LOCAL time** |
| ISC timing `start_time`/`end_time` | datetime string `YYYY-MM-DD HH:MM:SS.ffffff` | **LOCAL time** |
| `V2XHubSensor` `timestamp_ms` | Unix epoch **milliseconds** (integer) | **UTC** |
| Radar sensor JSON `payload.timestamp` | ISO 8601: `YYYY-MM-DDTHH:MM:SS+00:00` | **UTC** |
| Traffic-triggers JSON `payload.timestamp` | ISO 8601: `YYYY-MM-DDTHH:MM:SS+00:00` | **UTC** |

### 5-hour timezone offset (confirmed empirically)
- GT shows `2023-11-20 19:53` UTC
- Cameras show `2023-11-20 14:53` local  
- Offset = **+5 hours UTC** (site likely in US Eastern timezone, EST = UTC-5)
- ISC timing `start_time` strings also appear in local time

**To align camera timestamps to UTC:** `camera_unix + 5*3600` (verify per run).

### ISC timestamp parser
```python
def parse_isc_ts(ts: str) -> float:
    main, us = str(ts).rsplit('_', 1)
    return pd.to_datetime(main, format='%Y-%m-%d-%H-%M-%S').timestamp() + int(us)/1e6
```

---

## 7. GT File Schema (wide-format)

- **Row unit:** one timestamp (frame), ~10 Hz, Unix epoch float UTC
- **Column scheme:** `{ObjectClass}_{FieldSuffix}` where suffixes are:
  - `xctr, yctr, zctr` — centroid position
  - `xlen, ylen, zlen` — bounding box dimensions  
  - `xrot, yrot, zrot` — rotation angles
- **Multiple instances:** `Passenger_Vehicle_xctr`, `Passenger_Vehicle_xctr.1`, `.2`, etc.
- **Most cells are NaN** — only columns for objects present in that frame are filled
- **To use:** melt each class+slot into long format for analysis

Confirmed object classes (vary by run):
`Passenger_Vehicle`, `VRU_Adult`, `VRU_Child`, `VRU_Adult_Using_Walker`, plus others

---

## 8. Radar JSON Schema

### `Radars_Run_XXXX_sensorN.json`
```
List[Record]
  .topic          str    "sensor-traffic-objects/sensor1" OR "sensor-diagnostics/sensor1"
  .payload
    .timestamp    str    ISO 8601 UTC
    .cycle_time   float  seconds (~0.1 s)
    .reference_name str  "sensor1"
    .objects      List[Object]
        .id               int
        .class            str    "car" | "truck" | "pedestrian" | ...
        .heading          float  degrees
        .speed            float  m/s
        .length           float  m
        .acceleration     float  m/s²
        .position_facing  [x,y,z]  metres from radar origin
        .position_front   [x,y,z]
        .closest_lane     str    "lane10"
        .within_zone      List[str]  ["zoneAI"]
        .tracking_status
            .quality                  float 0–1
            .new_object               bool
            .mileage                  float metres tracked total
            .cycles_since_last_update int
  .qos            str    "AT_LEAST_ONCE"
  .receivedAt     str    local datetime string
  .retain         bool
```
Filter to `topic` containing `"traffic-objects"` — skip `"diagnostics"` records.

### `Radars_Run_XXXX_traffic-triggers-output.json`
```
List[Record]  (~2525 per run)
  .topic    "traffic-triggers-output"
  .payload
    .timestamp       str  ISO 8601 UTC
    .trigger_outputs List[TriggerOutput]
        .traffic_triggers List[Trigger]
            .reference_name      str  "trigger_11_3"
            .associated_lane     str  "lane22"
            .associated_sensor   str  "sensor3"
            .associated_zone     str  "zoneK"
```
Flatten: one row per trigger per timestamp → columns: `timestamp_utc, reference_name, associated_lane, associated_sensor, associated_zone`

---

## 9. V2X / Signal Data

- `V2XHubSensor_Run_XXXX.csv`: 1 row per second; `active_signal_ids` = phase IDs active; `event_list` = phase-change events
- `ISC_Run_XXXX_v2xhub_timing.csv`: sensor metadata rows — `v2xhub_sensors` (BSM/CSV) and `v2xhub_spat` (SPaT/PCAP)
- SPaT PCAP files are **not extracted** (skipped as `.pcap`)
- V2X hardware was **only present in some runs** (not Run_1003)

---

## 10. Conflict Labels

File: `conflict_no_conflict_labels_all_validation_runs.csv`  
Columns: `Run ID`, `Conflict_No_Conflict_Label`  
Values: `"conflict"` or `"no conflict"`

From the 3 inspected runs:
| Run | Label |
|---|---|
| Run_1003 | no conflict |
| Run_1023 | conflict |
| Run_1027 | conflict |

---

## 11. Camera Coverage

Per run: **8 visual cameras** + **5 thermal cameras** = 13 frame-timing CSVs  
Frame counts vary: ~1670–4060 frames per camera per run  
Approx frame rate: **~30 fps** (Visual), consistent thermal  
All 13 files share identical schema: `['Image_number', 'Timestamp']`  
`Image_number` is 0-based index into the corresponding `.mp4` (not extracted)

---

## 12. What Has Been Done

| Notebook | Status | Key outputs |
|---|---|---|
| `00_dataset_structure_exploration.ipynb` | ✅ Complete + run in Colab | Dataset tree, file type counts, GT schema, timestamp ranges, bar chart, timeline plot |
| `01_non_gt_file_inspection.ipynb` | ✅ Complete + run in Colab | Per-file deep inspection, ISC timing columns revealed as headerless, V2X content understood, Radar JSON schema confirmed, cross-file timestamp alignment table |
| `02_parser_validation_and_data_dictionary.ipynb` | ✅ Complete (V2X column-count bug fixed) | Parser strategy registry, wrong-vs-correct parse comparisons, GT wide-format decoder, ISC timing correct parse, V2X 4-column merge fix, full data dictionary, saves `parser_data_dictionary.csv` |

---

## 13. Suggested Next Notebooks

| # | Notebook | Goal |
|---|---|---|
| 03 | `03_gt_eda.ipynb` | GT deep EDA: object class distribution, track lengths, spatial heatmaps, temporal density, conflict vs non-conflict comparison |
| 04 | `04_timestamp_alignment.ipynb` | Convert camera local→UTC, align GT↔camera↔radar↔V2X on a common timeline per run |
| 05 | `05_radar_eda.ipynb` | Flatten radar sensor JSONs, object class/speed/zone distributions, per-sensor coverage |
| 06 | `06_trigger_analysis.ipynb` | Flatten traffic-triggers JSON, zone/lane/sensor activation patterns, conflict run vs non-conflict run comparison |
| 07 | `07_v2x_signal_analysis.ipynb` | Parse V2X signal phase IDs, reconstruct phase state timeline, correlate with GT and triggers |
| 08 | `08_cross_modal_fusion.ipynb` | Join GT + radar + triggers + V2X on aligned timestamps, build a unified per-frame event table |

---

## 14. Coding Conventions Used

- All notebooks are **Google Colab-compatible**
- Download + selective extraction at the top of every notebook (idempotent)
- `DATA_ROOT = '/content/isc_stage1b'` and `MAX_RUNS = 3` in every config cell
- Shared helpers always defined in their own cell: `find_files`, `safe_read_csv`, `parse_isc_ts`, `fmt_size`
- No `.pcap` or `.mp4` files are ever loaded
- Only `pandas`, `numpy`, `pathlib`, `json`, `re`, `matplotlib` used — no extra installs
- Output tables saved to `/content/outputs/tables/` in Colab

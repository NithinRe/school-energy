# Seed Data, Instructions, and Steps

*Data acquisition guide · school year 2024–25*

How to download, build and combine the data behind the School District Energy Analysis Tool, for every public school in the 50 states, DC, the U.S. territories and the Bureau of Indian Education. Each step is one Jupyter notebook in `notebooks/`.

### A · School dataset

**Step 1 · School data:** NCES directory, enrollment, staff, location

↓ adds columns

**Step 2 · Square footage:** Microsoft building footprints

↓ adds columns

**Step 3 · Climate zone:** ASHRAE county table

→ one row per school

### B · Benchmark dataset

**Step 4 · BPD benchmarks:** Site, electric and fuel EUI percentiles by climate zone

→ one row per climate zone × building type

### C · Final datasets

**Step 5 · Consolidate:** Joins A, adds districts, carries B

→ schools · districts · bpd_eui_benchmarks\
(CSV files + one Excel workbook)

| Step | Source | Access | Notebook |
| --- | --- | --- | --- |
| 1 | NCES Common Core of Data + EDGE geocodes | Public download | `01_data_acquisition_ccd_edge.ipynb` |
| 2 | Microsoft US Building Footprints | Downloaded by the notebook | `02_building_footprints_sqft.ipynb` |
| 3 | ASHRAE Standard 169-2020, Addendum a, Table A-4 | Manual extraction · ASHRAE standard | `03_climate_zones.ipynb` |
| 4 | Building Performance Database (BPD) | Manual download · BPD sign-in | `04_bpd_eui_benchmarks.ipynb` |
| 5 | Outputs of steps 1–4 | No download | `05_consolidate_datasets.ipynb` |

## Before you start

### Software

- Python 3.11 with `pandas`, `numpy`, `matplotlib`, `seaborn`
- `geopandas` and `shapely` for step 2
- `openpyxl` for reading and writing Excel files
- The `unzip` command (built into macOS and Linux). Step 1's enrollment file uses a compression format Python cannot open by itself.

Run every notebook from the `notebooks/` folder, in order. Paths are relative to the project root.

### Folder layout

```
data/
├── raw/          downloads (you place these)
│   ├── ccd_2024_25/      step 1
│   ├── edge_2024_25/     step 1
│   ├── climate_zones/    step 3
│   └── bpd/              step 4
├── interim/      outputs of steps 1–4
└── processed/    final datasets (step 5)
```

## Part A · Build the school dataset

Steps 1, 2 and 3 each add columns to the same list of schools. Step 1 creates the list; step 2 adds square footage; step 3 adds the climate zone.

### Step 1 · School data

**Access:** Public download

Four public files from the National Center for Education Statistics describe every public school: who it is, how many students and teachers it has, and where it is. All are for the **2024–25** school year, the latest NCES has published (2025–26 appears in the NCES menu but has no files yet).

#### Download: directory, membership and staff

1. Open [nces.ed.gov/ccd/files.asp](https://nces.ed.gov/ccd/files.asp)
2. Set the three dropdowns: first dropdown **Nonfiscal**, Level **School**, School Year **2024 - 2025**.
3. Under each of **Directory**, **Membership** and **Staff**, click the data file link labelled **“Flat and SAS Files”**. If a notice window opens, continue to the download.
4. Move the three zip files, still zipped, into `data/raw/ccd_2024_25/`.

| Component | File | Size | What it gives each school |
| --- | --- | --- | --- |
| Directory | [ccd_sch_029_2425_w_1a_073025.zip](https://nces.ed.gov/ccd/Data/zip/ccd_sch_029_2425_w_1a_073025.zip) | 13 MB | ID, name, district, address, status, type, level, grades |
| Membership | [ccd_sch_052_2425_l_1a_073025.zip](https://nces.ed.gov/ccd/Data/zip/ccd_sch_052_2425_l_1a_073025.zip) | 213 MB | Total enrollment |
| Staff | [ccd_sch_059_2425_l_1a_073025.zip](https://nces.ed.gov/ccd/Data/zip/ccd_sch_059_2425_l_1a_073025.zip) | 6 MB | Teacher full-time equivalents |

Each component also has a **Companion File**, an Excel workbook that is NCES's official column dictionary. It is useful for reference but not needed to run the notebook.

#### Download: school geocodes

1. Open [nces.ed.gov/programs/edge/Geographic/SchoolLocations](https://nces.ed.gov/programs/edge/Geographic/SchoolLocations)
2. Under **Public Schools**, download **2024–25**: [EDGE_GEOCODE_PUBLICSCH_2425.zip](https://nces.ed.gov/programs/edge/data/EDGE_GEOCODE_PUBLICSCH_2425.zip) (30 MB).
3. Move it into `data/raw/edge_2024_25/`. Field definitions are in the [EDGE technical documentation](https://nces.ed.gov/programs/edge/docs/EDGE_GEOCODE_PUBLIC_TECHDOC.pdf).

#### What the notebook does

- Reads each file with IDs as text, so leading zeros survive (`0100005`), and checks every row count against the total NCES publishes.
- Keeps **every school**, whatever its status, and marks the ones NCES counts as operating (`is_operational`).
- Joins the four files on the 12-character NCES school ID, checking after each join that no school was added or lost.
- Keeps a missing enrollment as missing, with NCES's reason (Missing, Suppressed, Not reported), never as 0.
- Keeps two kinds of state apart: who *reports* the school, and the county its coordinates fall in.

#### Result

- **57** reporting jurisdictions

- **102,178** schools, all statuses

- **100,237** operational schools

- **18,587** districts with at least one school

- **49,343,524** students

- **99.01%** operational schools with enrollment

3,669 schools fail at least one check (3,670 flags; one school fails two). Flagged schools stay in the data; each reason is listed in `school_data_checks_2024_25.csv`.

| Check | Why it matters |
| --- | --- |
| No enrollment number | Square footage is estimated from enrollment (students × 180 sq ft minimum). |
| Reports zero students | The size estimate rests on the footprint alone. About 4 in 5 are alternative, career-technical or special-education schools. |
| Coordinates in another state | The climate zone comes from the county the point falls in; a point at a headquarters gives the wrong zone. |

**Output:** `data/interim/schools_2024_25.csv`, `school_data_checks_2024_25.csv`, `jurisdiction_summary_2024_25.csv`.

### Step 2 · Square footage

**Access:** Downloaded by the notebook

Each operating school's floor area is estimated from the building its map point sits on, using [Microsoft US Building Footprints](https://github.com/microsoft/USBuildingFootprints) (release 2, 129.6 million buildings, one file per state for the 50 states and DC).

#### How to run

1. In section 2 of the notebook, set `RUN_SCOPE = "one_state"` with `STATE_TO_RUN = "NY"` (any state or DC postal code), or `RUN_SCOPE = "all"`.
2. Run all cells. For each state the notebook downloads the zip, loads only buildings within 150 m of schools, matches each school, saves that state's results and **deletes the download** before moving on.
3. If a run stops, run it again: finished states are skipped.

The national run downloads **4.8 GB** (34.4 GB unzipped, read straight from the zip) and takes several hours. New York took 2.5 minutes with about 720 MB of memory. The largest states (California, Texas, Florida) have not been test-run.

#### The rule

```
footprint < students × 180 sq ft  →  use students × 180
no building under the point       →  use students × 180
otherwise                         →  use the footprint
```

Areas are measured in an equal-area map projection, so they are exact in every state. The column `sqft_method` records which rule set each number.

4,810 operational New York schools. Where both datasets found the same building, the area matches the SEED prototype within 1%.

**Output:** `data/interim/school_sqft_2024_25_<STATE or all>.csv`

### Step 3 · Climate zones

**Access:** Manual extraction · ASHRAE standard

Each operating school gets the ASHRAE climate zone of the county it sits in (for example `4A`, `5A`, `6B`). The county table is **not provided** with this project.

#### Prepare the county table

1. Obtain **ANSI/ASHRAE Standard 169-2020, Addendum a (2021)** from ASHRAE. It is copyrighted.
2. Extract **Table A-4, “Climate Zones for United States Counties”**, which covers the 50 states and DC.
3. Save it as `data/raw/climate_zones/county_climate_zones.csv` with three columns: `State` (postal code), `County`, `ClimateZone`.

```
State,County,ClimateZone
AL,Autauga,3A
AL,Baldwin,2A
```

#### What the notebook does

- Matches each school by the state and county it *physically* sits in, so Bureau of Indian Education schools land in their real state.
- Cleans county names on both sides before matching (“Acadia Parish” = “Acadia”, “Saint” = “St”, “De Kalb” = “DeKalb”), but keeps “city” because Virginia, Maryland and Missouri have counties and cities with the same name.
- Handles Alaska's 2019 split of Valdez-Cordova, and gives new county names (such as Connecticut's 2022 planning regions) their state's zone when every county in that state shares one zone.
- Lists every school left without a zone, by county and reason, so the table can be fixed at the source.

**Output:** `data/interim/school_climate_zone_2024_25.csv`

## Part B · Build the benchmark dataset

A separate table, not tied to individual schools: how much energy school buildings use per square foot in each climate zone.

### Step 4 · BPD benchmarks

**Access:** Manual download · BPD sign-in

The [Building Performance Database](https://bpd.lbl.gov) gives the distribution of **Energy Use Intensity** (EUI, kBtu per square foot per year) for school buildings. Benchmarks are taken **by climate zone, not by state**: within one state there are too few school buildings for a stable comparison. The downloads are not provided with this project.

#### Download one zone at a time

1. Open [bpd.lbl.gov](https://bpd.lbl.gov) and sign in.
2. **Building Type**: deselect all, select **Education**, then only **Preschool/Daycare** and **K-12 School**.
3. **Location and Data Source → Climate Zone**: deselect all, then select one zone (e.g. `4A`).
4. In the **Table** card set **Group By: Building Type** and **Analyze By: Site EUI** (the default is Source EUI). Download and rename to `4A_Site_EUI.csv`.
5. Change **Analyze By** to **Electric EUI**, download as `4A_Electric_EUI.csv`; then **Fuel EUI**, download as `4A_Fuel_EUI.csv`. Both are required: site EUI alone does not say how energy splits between electricity and fuel.
6. Repeat for every climate zone and put all files in `data/raw/bpd/`.

#### What the notebook does

- Reads every file, checks the names, checks that the Building Type filter was set (no non-school types with data) and that every zone has all three metrics.
- Keeps every Education category BPD reports, whatever it is called. Earlier downloads split *Elementary or middle school* from *High school*; current downloads group all grades under K-12.
- Builds one table: count, mean, standard deviation, min, 25th, 50th and 75th percentile and max of site, electric and fuel EUI, side by side.

#### Future option

BPD offers an API under a commercial license, which would allow these tables, or building-level peer data, to be pulled automatically.

#### Model-based alternative

A model trained on K-12 school data to estimate a school's site EUI from its building characteristics was developed after the SEED prototype. It was not used in the SEED prototype or in these benchmarks; it can be shown if useful.

**Output:** `data/interim/bpd_eui_benchmarks.csv`

## Part C · Final datasets

### Step 5 · Consolidate

**Access:** No download

Combines the outputs of steps 1–4 into three datasets, written to `data/processed/` as CSV files and as three sheets of `school_energy_datasets_2024_25.xlsx`.

| Dataset | One row per | Built from |
| --- | --- | --- |
| `schools` | school, all statuses | Step 1 + step 2 square footage + step 3 climate zone, joined on `school_id` |
| `districts` | district | The school data: district ID, name, state and school counts (used by the application's district search) |
| `bpd_eui_benchmarks` | climate zone × building type | Step 4, unchanged |

Square footage and climate zone exist only for operating schools. A school outside the step 2 run (for example every school outside New York after a New York-only run) gets a **blank** square footage labelled `outside_square_footage_run`, never 0.

#### How a school is linked to its benchmark

The datasets are not pre-joined to benchmarks; the application makes the link. BPD now groups elementary, middle and high schools into one K-12 category, so **every school is benchmarked as K-12**:

1. Take the school's `climate_zone` (e.g. `5A`).
2. Find the `bpd_eui_benchmarks` row with that zone and the K-12 building type (set `K12_BUILDING_TYPE` in step 5 to BPD's exact name).
3. Read the percentile needed: `site_eui_p50` for a median estimate, or `electric_eui_p50` and `fuel_eui_p50` for the split.

```
estimated annual energy (kBtu) = estimated_sqft × chosen EUI
```

## All call-outs

Known limits, documented rather than corrected. Each is also stated in the notebook where it applies.

1. **Enrollment gaps (step 1).** 992 operating schools have no student count; their reasons come from NCES.
2. **Footprint age (step 2).** Building outlines come from imagery taken years before 2024–25 (mostly 2019–2020 or older). In New York, 3,268 of 4,534 matched buildings have no recorded capture date. New or expanded schools can be missing or too small.
3. **Schools sharing a building (step 2).** Where schools share one point (e.g. a middle and a high school), each receives the whole building's area, so the building is counted more than once. 1,083 of New York's 4,810 operating schools share a point.
4. **The point misses the roof (step 2).** A point on a parking lot or field finds no building, and the estimate falls back to students × 180.
5. **One building per school (step 2).** Gyms, annexes and portables on a campus are not added.
6. **Roof area, not floor area (step 2).** The footprint counts one floor, so multi-storey schools are under-measured.
7. **The 180 sq ft rule sets most estimates (step 2).** 66% of New York's estimates come from enrollment, not a measured building.
8. **Territories (steps 2–3).** No footprints and no ASHRAE county zones for Puerto Rico, the U.S. Virgin Islands, Guam, the Northern Mariana Islands and American Samoa.
9. **One zone per county (step 3).** Valleys and mountains in the same county share one zone; zone boundaries also change between ASHRAE editions.
10. **Self-selected, anonymized benchmarks (step 4).** BPD holds only buildings whose owners reported data, and does not say where in a zone they are.
11. **Small samples (step 4).** Some zone × building type cells hold few buildings; their percentiles are unstable.
12. **One K-12 benchmark (steps 4–5).** Elementary, middle and high schools receive the same benchmark now that BPD groups them.
13. **Use the “Other classroom” category cautiously (step 4).** It is BPD's broadest education category and does not identify the kind of school.
14. **Site EUI, not source EUI (step 4).** Losses in generating and delivering electricity are not included.

Sources: NCES Common Core of Data and EDGE (2024–25); Microsoft US Building Footprints (ODbL); ASHRAE Standard 169-2020 Addendum a; Building Performance Database, Lawrence Berkeley National Laboratory.

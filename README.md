# Housing Affordability Analytical Pipeline

A reproducible Python pipeline that computes the statistical results
reported in the dissertation's analysis of housing affordability across
English Local Authorities (LAs), 2005–2024. It performs raw data
ingestion, validation, aggregation, and inferential statistical testing
from source data through to a set of clean, versioned output tables.
Power BI is a downstream consumer of these outputs only; it performs no
independent statistical computation, so every headline figure and
significance test quoted in the dissertation can be regenerated from
this codebase alone.

## Contents

- [Overview](#overview)
- [Repository structure](#repository-structure)
- [Data sources](#data-sources)
- [Requirements](#requirements)
- [Usage](#usage)
- [Pipeline stages](#pipeline-stages)
- [Outputs](#outputs)
- [Spatial weights matrix](#spatial-weights-matrix)
- [Data quality notes](#data-quality-notes)
- [Reproducibility](#reproducibility)
- [Relationship to the Power BI dashboard](#relationship-to-the-power-bi-dashboard)
- [Limitations and further work](#limitations-and-further-work)

## Overview

The dissertation examines how housing affordability — measured as the
ratio of lower-quartile house prices to lower-quartile earnings — has
evolved across English LAs since 2005, and whether that evolution shows
statistically significant temporal trends, regional disparities, and
spatial clustering. This pipeline implements that analysis as a linear
sequence of small, independently testable modules, each responsible for
one stage of the process, orchestrated by `run_pipeline.py`.

## Repository structure

```
data/                                                     Raw inputs and generated spatial weights
  england_LA_affordability_2002_2025.csv                  ONS LQ affordability ratio + HM Land Registry
                                                            price panel, pre-region-join
  Local_Authority_District_to_Region__December_2024__
  Lookup_in_EN.csv                                        Official ONS LAD-to-region lookup
  la_spatial_weights.csv                                  Real LA-level spatial weights matrix
                                                            (generated locally; see below)

output/                                                   Generated results (created by run_pipeline.py)

raw_ingest.py           Stage 0 — joins the raw panel with the official region lookup;
                        produces fact_affordability_la_year.csv + dim_local_authority.csv
validate.py             Stage 1 — data-quality checks and anomaly detection
aggregate.py            Stage 2 — national / regional summaries, Gini coefficient series
divergence.py           Stage 3 — Price/Earnings Growth Index, Divergence Index
inferential.py          Stages 4-5 — Mann-Kendall trend test, Kruskal-Wallis H test, Moran's I, LISA;
                        both as a 2024 snapshot and as a full 2005-2024 panel-year series
run_pipeline.py         Orchestrator — runs all stages end-to-end, writes output/
build_spatial_weights.py  Standalone script — builds the real LA-level spatial weights matrix
                          from ONS boundary data (requires internet access; run separately)
ingest.py               Loader used only by each module's standalone `python <module>.py` run
requirements.txt        Python dependencies for the core pipeline
```

## Data sources

| Source | Description |
|---|---|
| Office for National Statistics (ONS) | Lower quartile house price to lower quartile earnings affordability ratio, by Local Authority District |
| HM Land Registry | Price Paid Data, aggregated to annual average price by Local Authority District |
| ONS Open Geography Portal | Local Authority District to Region (December 2024) lookup, and LAD boundary geometries for spatial weights |

The merged affordability/price panel and the official region lookup are
the two source files closest to the original ONS/Land Registry downloads
available for this project; `raw_ingest.py` documents the exact
transformation applied to each.

## Requirements

- Python 3.10+
- Core pipeline: `pandas`, `numpy`, `scipy` (see `requirements.txt`)
- Optional (spatial weights generation only): `geopandas`, `libpysal`

```bash
pip install -r requirements.txt
```

## Usage

Run the full pipeline:

```bash
python run_pipeline.py
```

This reads the two raw source files from `data/`, runs all six stages,
and writes every result table to `output/`.

Each stage module can also be run independently against the most recently
generated `output/` tables, for debugging or inspection:

```bash
python validate.py
python aggregate.py
python divergence.py
python inferential.py
```

### Generating the real spatial weights matrix (optional, recommended)

`inferential.py`'s Moran's I / LISA analysis can run against either an
approximate 9-region contiguity matrix (no setup required) or a real
LA-level (n≈294) queen-contiguity matrix built from official ONS boundary
data. The latter requires a one-off, internet-connected step:

```bash
pip install geopandas libpysal
python build_spatial_weights.py
```

This downloads the ONS Local Authority Districts (December 2024) boundary
file, builds a queen-contiguity weights matrix restricted to the LAs in
the study panel, patches any islands with a nearest-centroid fallback (see
[Spatial weights matrix](#spatial-weights-matrix)), and writes
`data/la_spatial_weights.csv`. Once that file exists, `run_pipeline.py`
and `inferential.py` detect and use it automatically — no further changes
are needed.

## Pipeline stages

| Stage | Module | Method | Purpose |
|---|---|---|---|
| 0 | `raw_ingest.py` | Deterministic join on LA name | Builds the harmonised fact/dimension tables from the two raw source files |
| 1 | `validate.py` | Referential integrity and plausibility checks | Flags nulls, out-of-range years, region-join failures, AreaCode mismatches, and implausible ratio values |
| 2 | `aggregate.py` | Descriptive statistics | National and regional medians/means, and a Gini coefficient series measuring cross-LA affordability inequality over time |
| 3 | `divergence.py` | Index construction | Price Growth Index, Earnings Growth Index, and their difference (the Divergence Index), all rebased to a common reference year |
| 4 | `inferential.py` | Non-parametric hypothesis testing and spatial statistics | Mann-Kendall trend test, Kruskal-Wallis H test, global and local Moran's I — 2024 snapshot |
| 5 | `inferential.py` | Same tests repeated per year | Kruskal-Wallis H test and global Moran's I for every year, 2005–2024 |

### Inferential statistics (Stages 4-5)

- **Mann-Kendall trend test** on the national median affordability ratio
  (2005–2024) — tests whether the observed trend is monotonic and
  statistically significant, rather than assumed from visual inspection
  of the time series.
- **Kruskal-Wallis H test** on LA-level ratios across regions — a
  non-parametric analogue of one-way ANOVA, chosen because the ratio
  distribution is right-skewed (a small number of very high-ratio London
  boroughs violate the normality assumption ANOVA requires). Computed
  both as a single 2024 snapshot and as a full panel-year series
  (`kruskal_wallis_all_years`), so results are filterable by year rather
  than fixed to one reference year.
- **Global Moran's I** — tests for statistically significant positive
  spatial autocorrelation in affordability ratios (whether neighbouring
  LAs are more similar to one another than chance would predict), computed
  analytically (randomisation assumption) rather than by simulation. Also
  computed as a full panel-year series (`morans_i_all_years`) at both
  regional and LA level side by side, so any strengthening or weakening
  of spatial clustering over the study period is directly testable rather
  than inferred from a single year.
- **Local Moran's I (LISA)** — decomposes the global statistic into
  per-LA hot-spot (High-High), cold-spot (Low-Low), and spatial outlier
  (High-Low / Low-High) classifications, available when the real LA-level
  spatial weights matrix has been generated. Currently computed for the
  2024 snapshot only.

## Outputs

All outputs are written to `output/` and are the exact files loaded into
the Power BI data model (Get Data → Text/CSV, or a folder query pointed
at `output/`).

| File | Contents |
|---|---|
| `fact_affordability_la_year.csv` | Rebuilt LA-year panel, region-joined from raw sources |
| `dim_local_authority.csv` | LA-to-region lookup, from the official ONS boundary file |
| `validation_summary.csv` | One row per data-quality check |
| `validation_anomalies.csv` | Row-level detail for every flagged anomaly |
| `summary_national_year.csv` | National medians/means, growth indices, Divergence Index, year-on-year % change |
| `summary_region_year.csv` | Same measures, by region and year |
| `summary_inequality_year.csv` | Gini coefficient series and inequality range, by year |
| `inferential_statistics.json` | Mann-Kendall, Kruskal-Wallis, and Moran's I results — 2024 snapshot |
| `lisa_local_morans_i_la_2024.csv` | Local Moran's I (LISA) per LA, 2024 — written only when `data/la_spatial_weights.csv` is present |
| `kruskal_wallis_by_year.csv` | Kruskal-Wallis H test result, one row per year, 2005–2024 |
| `morans_i_by_year.csv` | Global Moran's I (regional and, when available, LA-level), one row per year, 2005–2024 |

## Spatial weights matrix

Moran's I and LISA require a spatial weights matrix defining which
geographic units are "neighbours." Two tiers are supported:

- **Approximate (default, no setup):** a manually specified contiguity
  matrix over England's 9 NUTS1-equivalent regions. Adequate for a
  regional-level demonstration, but n=9 gives limited statistical power
  and is a simplification rather than the authoritative spatial
  relationship.
- **Real (recommended, requires `build_spatial_weights.py`):** an
  authoritative LA-level queen-contiguity matrix (n≈294) built from
  official ONS boundary geometries.

Every Moran's I result carries an explicit `weights_source` field
(`"real (LA-level, supplied)"` or `"approximate (9-region contiguity)"`),
so it is never ambiguous which mode produced a given figure.

Two LAs — Isle of Wight and Isles of Scilly — share no land border with
any other LA in the panel and are therefore queen-contiguity islands.
Each is patched with a single nearest-centroid neighbour (k-nearest-
neighbour, k=1) so that every LA has at least one spatial neighbour and
none are excluded from LISA purely for being an island. (Isles of Scilly
is nonetheless excluded from the 2024 run for an unrelated reason: it has
no recorded affordability ratio for that year in the source panel — see
the `excluded_area_codes` field in the Moran's I result.)

## Data quality notes

`validate.py` performs referential-integrity and plausibility checks on
every pipeline run, and surfaces one issue traced to the raw source data
itself rather than to any processing step in this pipeline:

Barnsley and Sheffield carry different `AreaCode` values in the raw
ONS/Land Registry panel (`E08000038` / `E08000039`) than in the official
ONS region lookup (`E08000016` / `E08000019`, the current correct codes).
Joining on `AreaCode` alone would silently drop these two LAs (48 rows:
2 LAs × 24 years). `raw_ingest.py` therefore joins on LA name instead,
which resolves all rows correctly, and the AreaCode mismatch is preserved
(not silently corrected) in the output tables so `validate.py` can
continue to detect and report it via
`distinct_LAs_with_AreaCode_mismatch_fact_vs_dim`. `inferential.py`'s
LA-level spatial join uses the same LA-name-derived key for consistency.

## Reproducibility

`raw_ingest.py` rebuilds `fact_affordability_la_year.csv` and
`dim_local_authority.csv` directly from the two raw source files, rather
than assuming them as a pre-joined starting point. Both rebuilt tables
were checked cell-by-cell against the previously used versions: zero
numeric or categorical mismatches across all 7,041 fact rows and 296
dimension rows. Every downstream summary table (`summary_national_year.csv`,
`summary_region_year.csv`, `summary_inequality_year.csv`) was similarly
checked cell-by-cell against its prior version before any new computation
was added.

## Relationship to the Power BI dashboard

The Power BI dashboard is a presentation layer over the CSV/JSON outputs
of this pipeline. DAX measures in the dashboard perform aggregation and
interactivity only (e.g. `SELECTEDVALUE`-based filtering, `MAX`/`MIN` over
already-clean data) and are never the source of a headline statistic —
every Gini coefficient, growth index, and significance test is computed
here, in Python, and loaded into the model as a finished value.

## Limitations and further work

- The approximate 9-region contiguity matrix remains available as a
  fallback for environments without internet access to ONS boundary data;
  it should not be treated as a substitute for the LA-level result where
  the latter is available.
- Local Moran's I significance is assessed via a conditional permutation
  test (999 permutations) rather than an analytical variance formula,
  which is standard practice for local indicators but means results carry
  a small amount of Monte Carlo variation between runs (mitigated here
  with a fixed random seed).
- Local Moran's I (LISA) is currently computed for a single reference
  year (2024) only; the global Kruskal-Wallis and Moran's I tests are
  already available as a full 2005–2024 panel-year series
  (`kruskal_wallis_by_year.csv`, `morans_i_by_year.csv`), but extending
  LISA the same way would allow specific hot-spot/cold-spot LAs to be
  tracked over time rather than viewed as a single snapshot.

# 💊 PharmaTrace

> **Dual-source pharmacovigilance intelligence** — cross-referencing FDA adverse event reports with PubMed literature to surface drug safety signals at scale.

![Python](https://img.shields.io/badge/Python-3.9%2B-blue?style=flat-square&logo=python)
![Streamlit](https://img.shields.io/badge/Streamlit-1.x-red?style=flat-square&logo=streamlit)
![openFDA](https://img.shields.io/badge/openFDA-FAERS-005ea2?style=flat-square)
![PubMed](https://img.shields.io/badge/PubMed-API-326599?style=flat-square)

---

## Overview

**PharmaTrace** is an end-to-end pharmacovigilance and literature mining platform that simultaneously queries two independent data sources — the **FDA Adverse Event Reporting System (FAERS)** via the openFDA API and the **PubMed biomedical literature database** — to help researchers, analysts, and drug safety professionals identify and cross-validate drug–disease and drug–adverse effect signals.

Upload a list of drugs and diseases, hit **Analyze**, and PharmaTrace returns:
- How frequently each drug–disease pair appears in scientific literature (PubMed co-occurrence counts)
- Which adverse effects are most reported for each drug in real-world FDA safety data (FAERS MedDRA reaction counts)
- Side-by-side stacked bar visualizations for rapid pattern recognition

---

## Features

- **Dual-source signal detection** — PubMed literature co-occurrence + FAERS spontaneous reporting, analyzed in parallel
- **Flexible CSV inputs** — plug in any list of drugs, diseases, or adverse effect filters; no code changes needed
- **Optional adverse effect filtering** — narrow FAERS results to a curated list of terms of interest
- **Interactive Streamlit dashboard** — two-column layout with live spinners, downloadable dataframes, and publication-quality stacked bar charts
- **One-click example analysis** — pre-loaded oncology dataset (tumor types × immunotherapy drugs) to demo the full pipeline instantly
- **CSV export** — download raw results at every stage for downstream analysis

---

## Architecture

```
PharmaTrace/
├── app/
│   ├── Pharmacovigilance_hub.py   # Streamlit frontend & orchestration
│   ├── literature_mining.py        # Core API logic (PubMed + FAERS)
│   └── example_data/
│       ├── drugs.csv               # Example drug list (immunotherapies)
│       ├── tumors.csv              # Example disease list (tumor types)
│       └── adverse_effects.csv     # Example AE filter list
├── notebooks/
│   ├── open-fda-api.ipynb          # FAERS API exploration & prototyping
│   └── pubmed-api.ipynb            # PubMed API exploration & prototyping
├── data/                           # User data (gitignored)
├── results/                        # Output CSVs (gitignored)
├── requirements.txt
└── README.md
```

### Data Flow

```
User CSVs (drugs, diseases, AEs)
         │
         ▼
┌────────────────────────┐     ┌────────────────────────────┐
│     PubMed API         │     │     openFDA FAERS API       │
│  (pymed — co-occurrence│     │  (REST — MedDRA reaction    │
│   counts per query)    │     │   counts per drug)          │
└──────────┬─────────────┘     └─────────────┬──────────────┘
           │                                 │
           ▼                                 ▼
   pivot_table (wide)              pivot_table (wide)
   top-N filtering                 optional AE filtering
   stacked bar chart               stacked bar chart
           │                                 │
           └──────────┬──────────────────────┘
                      ▼
            Streamlit two-column dashboard
```

---

## Quickstart

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/pharmatrace.git
cd pharmatrace
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Set up your API key

PharmaTrace reads the openFDA API key from Streamlit secrets. Create the secrets file:

```bash
mkdir -p .streamlit
echo 'FAERS_API_KEY = "your_openfda_api_key_here"' > .streamlit/secrets.toml
```

Get a free API key at [https://open.fda.gov/apis/authentication/](https://open.fda.gov/apis/authentication/)  
*(The API works without a key at lower rate limits, but a key is strongly recommended.)*

### 4. Launch the app

```bash
streamlit run app/Pharmacovigilance_hub.py
```

---

## Usage

### Option A — Example Analysis (no setup needed)

Click **Example Analysis** in the sidebar. The app runs the full pipeline on a pre-loaded oncology dataset: a panel of immunotherapy drugs (e.g., ipilimumab) cross-referenced against multiple tumor types, with curated adverse effects as a filter.

### Option B — Custom Analysis

Prepare up to three CSV files and upload them in the sidebar:

| File | Required | Format | Description |
|------|----------|--------|-------------|
| **Diseases CSV** | ✅ Yes | Single column, header required | List of diseases / tumor types to query |
| **Drugs CSV** | ✅ Yes | Single column, header required | List of drug names to query |
| **Adverse Effects CSV** | ❌ Optional | Single column, header required | Filter FAERS results to these AE terms only |

**Template downloads** are available directly in the sidebar once the app loads.

Click **Analyze** — results appear in the two-column dashboard within seconds to minutes depending on list sizes and API rate limits.

---

## Input CSV Format

### drugs.csv
```
drug
ipilimumab
nivolumab
pembrolizumab
```

### diseases.csv (or tumors.csv)
```
tumor
lung cancer
melanoma
colorectal cancer
```

### adverse_effects.csv (optional filter)
```
adverse_effect
sarcopenia
dysphagia
fatigue
```

---

## Notebooks

The `notebooks/` directory contains the full exploratory and prototyping work behind the app:

| Notebook | Description |
|----------|-------------|
| `open-fda-api.ipynb` | Prototypes the FAERS pipeline: single-drug queries, batch queries, data preprocessing, and stacked bar visualization |
| `pubmed-api.ipynb` | Prototypes the PubMed pipeline: co-occurrence counting, iterative search across drug–disease combinations, parallelized fetching with `ThreadPoolExecutor`, and paper-level retrieval |

---

## API Details

### openFDA FAERS
- **Endpoint:** `https://api.fda.gov/drug/event.json`
- **Query:** `patient.drug.medicinalproduct.exact` → counted by `patient.reaction.reactionmeddrapt.exact`
- **Rate limiting:** 1-second delay between requests; API key raises the hourly quota significantly
- **Output fields:** `drug`, `term` (MedDRA reaction), `count`

### PubMed (via `pymed`)
- **Query type:** Boolean `AND` co-occurrence — `"{disease} AND {drug}"`
- **Metric:** `getTotalResultsCount()` — number of indexed papers matching the query
- **Output fields:** `disease`, `drug`, `results`

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Streamlit |
| Data processing | pandas, numpy |
| Visualization | matplotlib, seaborn |
| FAERS integration | requests (REST) |
| PubMed integration | pymed |
| Concurrency | concurrent.futures (ThreadPoolExecutor) |
| Secrets management | Streamlit secrets |

---

## Limitations & Known Issues

- **PubMed rate limits:** The iterative nested search (diseases × drugs) generates many sequential API calls. For large lists (>20 terms each), the PubMed API may return a rate limit error. The app surfaces this gracefully with an info message.
- **FAERS drug name matching:** FAERS uses the exact medicinal product name as reported. Non-standard or brand name variants may miss records — normalize drug names before input.
- **Co-occurrence ≠ causation:** PubMed results count papers mentioning both terms together; this is a signal for research interest, not clinical evidence of a drug–disease relationship.
- **MedDRA term casing:** Adverse effect filtering is case-insensitive, but slight spelling differences between your filter list and MedDRA preferred terms may cause misses.

---

## Roadmap

- [ ] Asynchronous API calls for faster large-batch processing
- [ ] Disproportionality analysis (ROR / PRR) on FAERS data
- [ ] Time-trend analysis of adverse event reporting over years
- [ ] Export visualizations as PNG/PDF
- [ ] Docker containerization for one-command deployment
- [ ] Cloud deployment (Streamlit Community Cloud / Render)
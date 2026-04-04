<p align="center">
  <img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/LiCS_logo.jpg" height="80">
</p>

<h1 align="center">LiCSBAS</h1>
<h3 align="center">InSAR Time Series Analysis with Post-Migration Data Resilience</h3>

<p align="center">
  <a href="https://github.com/comet-licsar/LiCSBAS"><img src="https://img.shields.io/badge/upstream-comet--licsar%2FLiCSBAS-0969da?style=for-the-badge&logo=github" alt="Upstream"></a>
  <a href="https://github.com/bcankara"><img src="https://img.shields.io/badge/maintainer-Dr.%20Burak%20Can%20KARA-2ea44f?style=for-the-badge" alt="Maintainer"></a>
  <a href="https://www.gnu.org/licenses/gpl-3.0"><img src="https://img.shields.io/badge/license-GPLv3-e8b32a?style=for-the-badge" alt="License"></a>
</p>

<p align="center">
  <a href="#-whats-new"><b>What's New</b></a> · 
  <a href="#-installation"><b>Install</b></a> · 
  <a href="#-processing-workflow"><b>Workflow</b></a> · 
  <a href="#-bug-fixes"><b>Bug Fixes</b></a> · 
  <a href="#-changelog"><b>Changelog</b></a> · 
  <a href="#-citations"><b>Cite</b></a>
</p>

---

> **This is a maintained fork of [comet-licsar/LiCSBAS](https://github.com/comet-licsar/LiCSBAS)** that resolves critical data retrieval failures caused by the October 2025 JASMIN storage migration, fixes inversion crashes, and adds an intelligent multi-period batch automation system.

<p align="center">
  <img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/sample_vel.png" height="180">  
  <img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/sample_ts.png" height="180">
</p>

---

## 🚀 What's New

### The Problem: October 2025 LiCSAR Data Migration

In October 2025, the COMET LiCSAR archive on JASMIN was restructured. The original download URLs broke for the vast majority of historical interferograms. After the migration, the server at `gws-access.jasmin.ac.uk/public/nceo_geohazards/` now looks like this:

| Directory | Content |
|-----------|---------|
| `LiCSAR_products/` | Original path — now contains only a **small subset** of recent data (~66 IFG pairs per frame) |
| `LiCSAR_products.public/` | New primary storage — complete dataset (3 000+ pairs), but some IFG directories contain only burst-overlap products (`bovldiff`, `mag_cc`) **without** `unw.tif`/`cc.tif` |
| `LiCSAR_products.future/` | HTML catalogue — does not host files directly but contains `<a href>` links pointing to the actual download URLs on `data.ceda.ac.uk` |

**The upstream LiCSBAS only tries the original URL.** If that times out (as it now does for most data), the download silently fails. This fork solves the problem entirely.

### The Solution: 4-Tier Automatic URL Resolution

Every file request now cascades through four independent resolution tiers. No user configuration is required — it just works.

```
Tier 1 → Original URL (LiCSAR_products/)
         HEAD request, 30 s timeout
         ✓ 200 → download    ✗ → fall through

Tier 2 → Public mirror (LiCSAR_products.public/)
         HEAD request, 30 s timeout
         ✓ 200 → download    ✗ → fall through

Tier 3 → CEDA via XML metadata
         Fetch .tif.xml from .public, parse Atom <link rel="enclosure">
         Extract dap.ceda.ac.uk URL
         ✓ 200 → download    ✗ → fall through

Tier 4 → Future catalogue (LiCSAR_products.future/)
         Fetch HTML page, parse <a href> for target filename
         Resolve to data.ceda.ac.uk or .public
         ✓ 200 → download    ✗ → file unavailable
```

| Data Scenario | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|---------------|--------|--------|--------|--------|
| Recent / not yet migrated | ✅ | — | — | — |
| New post-migration (2025+) | ⏱ timeout | ✅ direct `.tif` | — | — |
| Old pre-migration (≤2023) | ⏱ timeout | no `.tif` (only metadata) | ✅ CEDA | — |
| Burst-overlap only in `.public` | ⏱ timeout | no `unw.tif` | no XML match | ✅ via CEDA link |

### Aggregated IFG Discovery

Instead of reading interferogram listings from a single directory, `fetch_listing()` now scans **all three catalogues** (`original`, `.public`, `.future`) and merges them into a **deduplicated master list**. This guarantees that IFGs scattered across different storage nodes are all discovered and processed.

---

## 🐛 Bug Fixes

### 1. `IndexError` Crash in `LiCSBAS13_sb_inv.py`

**Symptom:** The SBAS inversion step crashes with `IndexError: index out of bounds` during gap-filling when processing certain time series.

**Root Cause:** A stale loop variable (`i`) was used to index into `gap_patch` after the loop had already advanced past the valid range. This occurred specifically when gap patches were present and the inversion iteratively processed spatial tiles.

**Fix:** Replaced the stale index reference with the correct patch-scoped variable, ensuring `gap_patch` access stays within bounds for all tile geometries.

### 2. `bcankara.sh` — ASC/DSC Mode Reset on Period Boundaries

**Symptom:** When processing multiple time periods, the script incorrectly resets to ASC mode at the start of each period, re-running already completed ASC work.

**Root Cause:** The ASC/DSC state variable was reassigned at the top of the period loop without checking log completion state.

**Fix:** Introduced a 3-step progress log system (`ASC` → `DSC` → `DECOMPAS`) that independently tracks each phase per period. Completed phases are skipped automatically on re-run.

### 3. `bcankara.sh` — Decompas Fails on Resume (Directory Not Found)

**Symptom:** When re-running the script after ASC+DSC completed successfully, Decompas crashes because it cannot find `01_ASC_...` and `02_DSC_...` directories.

**Root Cause:** On a successful previous run, those directories were already moved into the period archive folder (`2020-01-01_2020-07-01/`), but Decompas only searched the working root.

**Fix:** Decompas now performs a two-stage directory lookup: first in the working root, then inside the period archive folder. This allows Decompas to run independently even after ASC/DSC phases have been archived.

---

## 📦 Installation

### Prerequisites

- **Python** ≥ 3.8
- **GDAL** ≥ 2.4
- **Git**

### Option 1 — Conda Environment (Recommended)

```bash
# Clone this fork
git clone https://github.com/bcankara/LiCSBAS.git
cd LiCSBAS

# Create and activate the conda environment
conda env create -f environment.yml
conda activate licsbas

# Register LiCSBAS paths
echo "source $(pwd)/bashrc_LiCSBAS.sh" >> ~/.bashrc
source bashrc_LiCSBAS.sh
```

### Option 2 — Miniconda from Scratch

```bash
# Install Miniconda (if not installed)
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

# Clone and set up
git clone https://github.com/bcankara/LiCSBAS.git
cd LiCSBAS
conda env create -f environment.yml
conda activate licsbas
source bashrc_LiCSBAS.sh
```

### Option 3 — pip Install (Into Existing Environment)

```bash
git clone https://github.com/bcankara/LiCSBAS.git
cd LiCSBAS
pip install .
source bashrc_LiCSBAS.sh
```

### Option 4 — Add This Fork as a Remote

If you already have a clone of the upstream repo:

```bash
git remote add bcankara https://github.com/bcankara/LiCSBAS.git
git fetch bcankara
git checkout bcankara/main
```

### Verify Installation

```bash
LiCSBAS_check_install.py
```

Expected output:
```
LiCSBAS install is OK
```

#### Required Python Packages

| Package | Purpose |
|---------|---------|
| `numpy`, `scipy` | Core numerical computing |
| `matplotlib` | Visualization and plotting |
| `h5py` | HDF5 time series data I/O |
| `gdal` | Geospatial raster operations |
| `requests`, `beautifulsoup4` | HTTP downloads and HTML/XML parsing |
| `statsmodels` | Statistical modeling for detrending |
| `networkx` | Interferogram network graph analysis |
| `cmcrameri` | Scientific colour maps |
| `xarray`, `rioxarray` | Multi-dimensional labeled arrays with GeoTIFF |

> 📋 Full list: [`LiCSBAS_requirements.txt`](LiCSBAS_requirements.txt) / [`environment.yml`](environment.yml)

---

## 🔄 Processing Workflow

LiCSBAS processes Sentinel-1 InSAR data through a structured pipeline:

```
 ┌─────────────────────────────────── DATA PREPARATION ──────────────────────────────────┐
 │                                                                                       │
 │  Step 01   Download GeoTIFFs from COMET-LiCS portal (with 4-tier URL fallback)        │
 │  Step 02   Multilooking and format conversion                                         │
 │  Step 03*  Atmospheric correction with GACOS                                          │
 │  Step 04*  Coherence-based masking of unwrapped interferograms                        │
 │  Step 05*  Spatial clipping to area of interest                                       │
 │                                                                                       │
 ├─────────────────────────────── TIME SERIES ANALYSIS ──────────────────────────────────┤
 │                                                                                       │
 │  Step 11   Quality check — identify and remove bad interferograms                     │
 │  Step 12   Loop closure — detect unwrapping errors via phase triangles                │
 │  Step 13   SBAS inversion — compute cumulative displacement time series               │
 │  Step 14   Velocity & standard deviation estimation (Bootstrap)                       │
 │  Step 15   Mask noisy pixels in the time series                                       │
 │  Step 16   Spatio-temporal filtering of the time series                               │
 │                                                                                       │
 ├────────────────────────────────── OUTPUT & EXPORT ─────────────────────────────────────┤
 │                                                                                       │
 │  Step 17   Generate KMZ files for Google Earth                                        │
 │  Step 18   LOS decomposition (ASC + DSC → Up-Down / East-West)                       │
 │                                                                                       │
 └───────────────────────────────────────────────────────────────────────────────────────┘
                                                          * = optional steps
```

> 📖 For detailed parameter descriptions, see the original [LiCSBAS Wiki](https://github.com/yumorishita/LiCSBAS/wiki/2_0_workflow).

---

## 📁 Modified Files

| File | Changes |
|------|---------|
| `LiCSBAS_lib/LiCSBAS_tools_lib.py` | `resolve_url()` — 2-tier page resolution; `extract_url_licsar()` — 4-tier file resolution with `.future` HTML parsing; `_extract_ceda_url_from_xml()` — XML→CEDA URL parser; `_extract_url_from_future_html()` — `.future` catalogue parser; 30 s timeout on all HTTP requests |
| `bin/LiCSBAS01_get_geotiff.py` | `fetch_listing()` — aggregated discovery across all 3 catalogues; detailed unavailable-IFG summary; graceful exit on empty listing; 30 s timeouts |
| `bin/LiCSBAS13_sb_inv.py` | **Bug fix:** resolved `IndexError` from stale loop variable accessing `gap_patch` |
| `bin/LiCSBAS_get_eqoffsets.py` | Metadata access via `resolve_url()` fallback |
| `LiCSBAS_lib/LiCSBAS_meta.py` | Version `1.15.2` (2026-04-04), author credit |
| `bcankara.sh` | 3-phase progress log (ASC/DSC/DECOMPAS); Decompas two-stage directory lookup; safe file/directory operations on resume |
| `.gitattributes` | Enforce LF line endings across platforms |

---

## 📋 Changelog

### v1.15.2 — 2026-04-04
- **feat:** 4-tier URL resolution for post-migration LiCSAR data
- **feat:** Aggregated IFG discovery across `original`, `.public`, and `.future` catalogues
- **feat:** `.future` HTML catalogue parsing for CEDA archive links
- **fix:** `IndexError` crash in `LiCSBAS13_sb_inv.py` (stale `gap_patch` index)
- **fix:** `bcankara.sh` ASC/DSC mode reset on period boundaries
- **fix:** `bcankara.sh` Decompas directory-not-found on resume
- **feat:** 3-phase progress log system (ASC → DSC → DECOMPAS) with resume capability
- **feat:** 30 s timeout on all HTTP requests to prevent indefinite hangs
- **docs:** Complete README rewrite with installation guide and changelog

### v1.15.0 — Upstream (comet-licsar)
- Base version from COMET team with standard SBAS processing pipeline

---

## ✅ Verified

Tested against frame `094D_04913_101213` (Merzifon, Turkey) on 2026-04-04:

- ✓ `LiCSAR_products/` — timeout (expected for migrated data)
- ✓ `LiCSAR_products.public/` — direct `.tif` present for new pairs; burst-overlap only for old pairs
- ✓ CEDA archive via XML parsing — HTTP 200, correct file size
- ✓ `LiCSAR_products.future/` HTML catalogue — resolves `<a href>` to `data.ceda.ac.uk` URLs
- ✓ Aggregated discovery merges IFGs from all 3 sources successfully
- ✓ `LiCSBAS13_sb_inv.py` inversion completes without `gap_patch` crash
- ✓ Multi-period batch processing with ASC/DSC/DECOMPAS resume

---

## 📚 Sample Data & Tutorial

| Resource | Link |
|----------|------|
| **Jupyter Notebook** | [licsbas_tutorial.ipynb](licsbas_tutorial.ipynb) |
| **Run on Binder** | [![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/comet-licsar/LiCSBAS/HEAD?labpath=licsbas_tutorial.ipynb) |
| **Run on Colab** | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/comet-licsar/LiCSBAS/blob/main/licsbas_tutorial.ipynb) |
| **Sample PDF** | [LiCSBAS_sample_CF.pdf](https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/documents/LiCSBAS_sample_CF.pdf) |
| **Sample Results** | [LiCSBAS_sample_CF.tar.gz](https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/sample/LiCSBAS_sample_CF.tar.gz) (63 MB) |

**Sample frame:** 124D_04854_171313 (Campi Flegrei, Italy) — 67 epochs, ~217 interferograms, 2016–2018.

---

## 📖 Citations

> Morishita, Y.; Lazecky, M.; Wright, T.J.; Weiss, J.R.; Elliott, J.R.; Hooper, A. **LiCSBAS: An Open-Source InSAR Time Series Analysis Package Integrated with the LiCSAR Automated Sentinel-1 InSAR Processor.** *Remote Sens.* 2020, 12, 424. https://doi.org/10.3390/RS12030424

> Morishita, Y. **Nationwide urban ground deformation monitoring in Japan using Sentinel-1 LiCSAR products and LiCSBAS.** *Prog. Earth Planet. Sci.* 2021, 8, 6. https://doi.org/10.1186/s40645-020-00402-7

> Lazecký, M. et al. **LiCSAR: An Automatic InSAR Tool for Measuring and Monitoring Tectonic and Volcanic Activity.** *Remote Sens.* 2020, 12, 2430. https://doi.org/10.3390/rs12152430

> Lazecký, M. et al. **Strategies for improving and correcting unwrapped interferograms implemented in LiCSBAS.** *Procedia Computer Science* 2024, 239, 2408-2412. https://doi.org/10.1016/j.procs.2024.06.435

---

## 🙏 Acknowledgements

LiCSBAS was originally developed by **Dr. Yu Morishita** during his research visit at the University of Leeds, funded by JSPS Overseas Research Fellowship. It is currently maintained by the **COMET team**.

This fork is maintained by **Dr. Burak Can KARA** ([bcankara.com](https://bcankara.com)) with contributions addressing the 2025 JASMIN data migration and critical bug fixes.

<p align="center">
  <a href="https://comet.nerc.ac.uk/"><img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/COMET_logo.png" height="50"></a>&nbsp;&nbsp;&nbsp;
  <a href="https://environment.leeds.ac.uk/see/"><img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/logo-leeds.png" height="50"></a>&nbsp;&nbsp;&nbsp;
  <a href="https://comet.nerc.ac.uk/COMET-LiCS-portal/"><img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/LiCS_logo.jpg" height="50"></a>
</p>

---

<details>
<summary><b>📜 Original Upstream README</b> <i>(click to expand)</i></summary>

<br>

LiCSBAS is an open-source package in Python and bash to carry out InSAR time series analysis using LiCSAR products (i.e., unwrapped interferograms and coherence) which are freely available on the [COMET-LiCS web portal](https://comet.nerc.ac.uk/COMET-LiCS-portal/).

It was originally developed by Dr. Yu Morishita during his research visit stay at the University of Leeds and is currently maintained by the COMET team while his original version (that is also further developed) exists on his [original site](https://github.com/yumorishita). Here we try keep a unity and ingest new functionality to this version that contains various updates but a lot of experimental functions that were developed within a programming learning curve of several COMET members.

With LiCSBAS, users can easily derive the time series and velocity of the displacement if sufficient LiCSAR products are available in the area of interest. LiCSBAS also contains visualization tools to interactively display the time series of displacement to help investigation and interpretation of the results.

[<img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/comet-lics-web.png" height="180">](https://comet.nerc.ac.uk/COMET-LiCS-portal/) <img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/sample_vel.png" height="180"> <img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/sample_ts.png" height="180">

<img src="https://raw.githubusercontent.com/wiki/yumorishita/LiCSBAS/images/LiCSBAS_plot_ts.py_demo_small.gif" alt="Demonstration Video"/>

THIS IS RESEARCH CODE PROVIDED TO YOU "AS IS" WITH NO WARRANTIES OF CORRECTNESS. USE AT YOUR OWN RISK.

For full documentation, see the original [Wiki](https://github.com/yumorishita/LiCSBAS/wiki).

---

*Yu Morishita (PhD)*
*JSPS Overseas Research Fellow (June 2018 – March 2020)*
*Visiting Researcher, COMET, School of Earth and Environment, University of Leeds*
*Chief Researcher, Geography and Crustal Dynamics Research Center, GSI*

</details>

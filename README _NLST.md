# NLST_Dataset

Utilities and documentation for working with the **National Lung Screening Trial (NLST)** low‑dose CT data — from downloading via TCIA/NBIA to preprocessing and dataset manifests ready for ML.

> ⚠️ **No data is stored in this repository.** NLST/TCIA data must be requested and downloaded from official sources. Follow all data‑use agreements and institutional IRB/DUA requirements.

---

## Contents
- [Overview](#overview)
- [Data Access & Permissions](#data-access--permissions)
- [Repository Structure](#repository-structure)
- [Environment](#environment)
- [Quickstart](#quickstart)
- [Download from TCIA / NBIA](#download-from-tcia--nbia)
- [Preprocessing](#preprocessing)
- [Cohort & Labels](#cohort--labels)
- [Manifests & Splits](#manifests--splits)
- [Basic QA & Stats](#basic-qa--stats)
- [Reproducibility](#reproducibility)
- [Citations](#citations)
- [License](#license)

---

## Overview
This repo helps you:
1. **Download** NLST LDCT scans from TCIA using NBIA or scripts.  
2. **Convert & normalize** DICOM → NIfTI (and optionally export PNG slices).  
3. **Assemble labels** and create cohort manifests.  
4. **Produce train/val/test splits** for ML pipelines.

---

## Data Access & Permissions
- **NLST on TCIA:** Obtain a **.tcia** manifest from The Cancer Imaging Archive (TCIA). You may need to accept usage terms.  
- **No PHI:** Only de‑identified data; **do not** commit any data files to this repo.  
- **IRB/DUA:** Ensure your institution permits use of NLST for your project.

> Tip: The **NBIA Data Retriever** (desktop app) downloads DICOMs from a `.tcia` manifest reliably.

---

## Repository Structure
```text
NLST_Dataset/
├─ scripts/
│  ├─ download_tcia_manifest.py     # optional helper around NBIA/TCIA
│  ├─ dicom_to_nifti.py             # DICOM → NIfTI, orientation, resampling, windowing
│  ├─ build_manifest_csv.py         # assemble series/patient-level CSV
│  ├─ cohort_select.py              # inclusion/exclusion rules + labels
│  ├─ compute_stats.py              # QA metrics (spacing, slices, HU ranges)
│  └─ visualize_series.py           # quick axial previews
├─ configs/
│  └─ default.yaml                  # paths and preprocessing params
├─ notebooks/
│  └─ examples.ipynb
├─ manifests/                       # metadata only (no images)
│  ├─ tcia_manifest.tcia            # not tracked
│  ├─ series_manifest.csv
│  ├─ cohort_labels.csv
│  └─ splits/{train,val,test}.csv
├─ data/                            # (gitignored)
│  ├─ raw/                          # DICOM hierarchy from TCIA
│  ├─ interim/                      # converted NIfTI
│  └─ processed/                    # resampled/cleaned for ML
├─ .gitignore
└─ README.md
```

> Adjust paths or filenames if your scripts use different names.

---

## Environment
Use a virtual environment or conda.

```bash
python -m venv .venv
source .venv/bin/activate              # Windows: .venv\Scripts\activate
pip install -r requirements.txt        # if provided
```

**Suggested Python packages:**
`pydicom`, `nibabel`, `SimpleITK`, `opencv-python-headless`,  
`numpy`, `pandas`, `tqdm`, `pyyaml`, `matplotlib`.

---

## Quickstart
1) **Place your TCIA manifest** at `manifests/tcia_manifest.tcia`.  
2) **Download DICOMs** (see next section).  
3) **Convert to NIfTI** and normalize:

```bash
python scripts/dicom_to_nifti.py \
  --src data/raw \
  --dst data/interim \
  --jobs 8
```

4) **Resample** to isotropic spacing & apply lung window (optional):

```bash
python scripts/dicom_to_nifti.py \
  --src data/interim --dst data/processed \
  --resample 1.0 --window lung --clip -1000 400
```

5) **Build a manifest** and **create splits**:

```bash
python scripts/build_manifest_csv.py \
  --root data/processed \
  --out manifests/series_manifest.csv

python scripts/cohort_select.py \
  --in manifests/series_manifest.csv \
  --labels manifests/cohort_labels.csv \
  --out manifests/splits/
```

---

## Download from TCIA / NBIA

### A) NBIA Data Retriever (GUI)
1. Open `manifests/tcia_manifest.tcia` in **NBIA Data Retriever**.  
2. Choose `data/raw/` as the destination.  
3. Start download (preserve folder hierarchy).

### B) Scripted (if you use TCIA APIs/tokens)
If you maintain an authenticated helper:

```bash
python scripts/download_tcia_manifest.py \
  --manifest manifests/tcia_manifest.tcia \
  --out data/raw \
  --jobs 8
```

---

## Preprocessing
Common steps (parameterize via `configs/default.yaml`):

- **Convert** DICOM → NIfTI (`.nii.gz`)
- **Reorient** to RAS
- **Resample** to e.g., **1.0 mm isotropic**
- **Intensity**: HU clamping (e.g., `[-1000, 400]`), optional z‑score
- **Lung masking** (optional): crop to lung fields
- **Slice export** (optional): export PNGs for 2D models
- **QC**: remove corrupt or incomplete series

Example `configs/default.yaml` snippet:
```yaml
paths:
  raw: data/raw
  interim: data/interim
  processed: data/processed
preprocess:
  resample_mm: 1.0
  window: lung      # none|lung
  clip: [-1000, 400]
  reorient: RAS
export:
  slices_png: false
```

---

## Cohort & Labels
Define your prediction target and follow‑up window (e.g., incident cancer within N months). Example label CSV columns:

```
patient_id,study_uid,series_uid,cancer_label,split,notes
```

Keep label derivations scripted and reproducible.

---

## Manifests & Splits
- **Series manifest**: paths, UIDs, spacing, slice count, checksum.  
- **Splits**: patient‑level disjoint `{train,val,test}.csv` stored under `manifests/splits/`.  
- **Reproducible seeds**: record RNG seeds and the git commit for each split export.

---

## Basic QA & Stats
```bash
python scripts/compute_stats.py \
  --root data/processed \
  --out manifests/stats.csv
```
Recommended checks:
- Missing slices / inconsistent spacing
- HU distributions and window sanity checks
- Scanner/site counts (if available)

---

## Reproducibility
- Config‑driven pipelines (`configs/default.yaml`)
- Logged seeds, package versions, and git commit hashes
- Idempotent scripts that gracefully skip existing outputs

---

## Citations
Please cite data sources and tools you use:
- **NLST** (National Lung Screening Trial) — add the official citation for NLST.  
- **TCIA**: Clark K, et al. *The Cancer Imaging Archive (TCIA): Maintaining and Operating a Public Information Repository.* J Digit Imaging. 2013.

---

## License
Code in this repository is released under the **MIT License** (update if different).  
Data remains under NLST/TCIA terms and **is not included** in this repository.

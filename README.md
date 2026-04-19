# Concept Bottleneck Models — Fork

This is a fork of [yewsiang/ConceptBottleneck](https://github.com/yewsiang/ConceptBottleneck) (Koh et al., ICML 2020). For the original paper, datasets, CUB experiments, and Docker setup, refer to the upstream README and repo.

---

## Changes in this fork

### OAI data pipeline (`data/OAI/scripts/`)

The upstream repo pointed to [epierson9/pain-disparities](https://github.com/epierson9/pain-disparities) for OAI data processing without providing runnable scripts. This fork adds a complete, self-contained data preparation pipeline for the OAI dataset.

#### Scripts

| Script | Description |
|---|---|
| `data/OAI/scripts/1_download_data.ipynb` | Download raw OAI DICOM images and label files via NDA |
| `data/OAI/scripts/2_check_downloaded_data.ipynb` | Verify downloaded DICOMs, inspect DICOM tags, check label coverage |
| `data/OAI/scripts/3_prepare_data.ipynb` | Full preprocessing pipeline → produces `manifest.csv` + `.npy` images |

#### `3_prepare_data.ipynb` — what it does

Builds `(X, y)` pairs from raw OAI DICOMs and `oai_kxrsemiquant01.txt`, using preprocessing functions directly from the pain-disparities submodule (`submodule/pain-disparities/`).

**Step 1 — Find valid pairs**
- Walks `ROOT_DIR` for all DICOM `001` files, filters to `BodyPartExamined == KNEE`
- Loads `oai_kxrsemiquant01.txt` (tab-separated, skip description row 2)
- Applies `readprj` filter (`readprj==15` for timepoints 00–48m; recodes `readprj==42→37` per OAI notes)
- Maps side codes `{1: right, 2: left}` — same as `NonImageData.side_mappings`
- Inner joins on `(src_subject_id, interview_date)` → one row per knee side

**Step 2 — Image preprocessing** (via `XRayImageDataset` from pain-disparities)
- Patches `constants_and_util.py` path guards before import (avoids `assert os.path.exists` failures)
- Instantiates a minimal `XRayImageDataset` proxy without the full directory structure
- `get_resized_pixel_array_from_dicom_image()` — resize to 1024×1024 (bilinear, `cv2`)
- `compute_dataset_image_statistics_and_divide_by_max()` — divide by global max → [0, 1], compute dataset mean/std
- `cut_images_in_two()` → `cut_image_in_half()` — split bilateral image with 1.1× margin; radiological convention: right side of film = patient's left knee
- `make_images_RGB_and_zscore()` — replicate grayscale → (3, H, W) RGB, z-score with dataset-wide stats
- Saves as `{subject_id}_{side}.npy`, shape `(3, 1024, ~563)`; skips already-processed files (re-runnable)
- Saves `stats.json` with `global_max`, `global_mean`, `global_std`

**Step 3 — Label processing** (mirrors `NonImageData.load_semiquantitative_xray_data()`)
- Numeric coercion of all label columns
- JSN truncation: floor fractional grades (e.g. 1.4→1) — JSN annotations use fractions to indicate progression within a grade, not fractional severity
- Fill non-JSN/non-KLG concept columns with 0 for participants who never had `xrkl >= 2` at any recorded timepoint; drop rows where a KLG≥2 participant is missing concept scores (mirrors pain-disparities logic)
- Drop rows missing `xrkl`
- KLG merge: 0+1→0, shift others down → 4-level label (0–3) for CBM target

**Step 4 — Manifest**
- Saves `data/OAI/processed/manifest.csv` with columns:
  - Identifiers: `src_subject_id`, `interview_date`, `visit`, `side`, `readprj`, `barcode`
  - Paths: `dicom_path`, `processed_image_path`
  - Label: `xrkl` (raw), `xrkl_merged` (4-level)
  - 10 kept concepts: `xrosfm xrscfm xrjsm xrostm xrsctm xrosfl xrscfl xrjsl xrostl xrsctl`
  - 8 reference concepts (low sparsity, kept for reference): `xrcyfm xrchm xrcytm xrattm xrcyfl xrchl xrcytl xrattl`

#### Labels and concepts

The 10 kept concepts pass a ≥95% sparsity filter (used in CBM training). The 8 reference concepts are retained in the CSV but not used as CBM concept targets. KLG (Kellgren-Lawrence Grade) is the primary task label.

| Column | Description |
|---|---|
| `xrkl` / `xrkl_merged` | KL grade (raw 0–4 / merged 0–3) |
| `xrjsm` / `xrjsl` | Joint space narrowing, medial / lateral |
| `xrosfm` / `xrosfl` | Osteophytes, femoral medial / lateral |
| `xrscfm` / `xrscfl` | Subchondral sclerosis, femoral medial / lateral |
| `xrostm` / `xrostl` | Osteophytes, tibial medial / lateral |
| `xrsctm` / `xrsctl` | Subchondral sclerosis, tibial medial / lateral |

#### Directory structure

```
data/OAI/
├── 1243742/image03/          # raw DICOM downloads (one per timepoint folder)
│   └── 12m/1.E.1/{subject_id}/{yyyymmdd}/{series}/001
├── Package_1243743/
│   └── oai_kxrsemiquant01.txt   # semi-quantitative X-ray label file
└── processed/
    ├── manifest.csv             # all (X, y) pairs with metadata
    ├── stats.json               # normalisation constants
    └── images/
        └── {subject_id}_{side}.npy   # shape (3, 1024, ~563), float32
```

#### Submodules

```
submodule/
├── pain-disparities/   # Pierson et al. 2019 — image + label preprocessing code
└── nda-tools/          # NDA download tooling
```

To initialise submodules after cloning:
```bash
git submodule update --init --recursive
```

---

## Original repo

See [yewsiang/ConceptBottleneck](https://github.com/yewsiang/ConceptBottleneck) for:
- Paper abstract, CUB experiments, Docker setup
- `CUB/README.md` for CUB data processing and evaluation
- `OAI/README.md` for original OAI experiment commands

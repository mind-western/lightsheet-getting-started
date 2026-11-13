---
marp: true
theme: default
paginate: true
header: "Getting started with lightsheet data at MIND"
style: |
  section {
    font-size: 1.4rem;
  }
  section.title {
    text-align: center;
    justify-content: center;
  }
  code {
    font-size: 0.85rem;
  }
  table {
    font-size: 1.1rem;
  }
  h1 { color: #2c5f8a; }
  h2 { color: #3a7abf; }
  a { color: #2c7fb8; }
---

<!-- _class: title -->

# Getting started with lightsheet data

### From Imaris datasets to quantitative results

**MIND Platform, Western University**
https://github.com/mind-western/lightsheet-getting-started
https://www.mind-western.ca/lightsheet-getting-started

---

## The workflow

```
microscope ──► core: stitch to Imaris (.ims)
                    │
                    ▼
   /nfs/trident3/lightsheet/<pi>/<project>/raw/
                    │
                    ▼   SPIMpack (spimpack-ui)
   <project>/bids/
                    │
                    ▼   SPIMquant
   <project>/derivatives/spimquant_<version>/
```

| Step | Where |
|---|---|
| Viewing data (napari) | CBS **VDI** |
| Making the BIDS dataset (`spimpack-ui`) | CBS **VDI** |
| Running SPIMquant | CBS **SLURM cluster** |

---

## Prerequisites: CBS servers

Docs: https://hackmd.io/@CompCore/cbs_servers

- **VDI:** Linux virtual desktop
  - Omnissa Horizon Client → `vdi.robarts.ca`, log in as `<westernid>@uwo.ca`
  - For viewing data and interactive tools
- **SLURM cluster:** for running SPIMquant
  - From a VDI terminal: `ssh <westernid>@rri-cbs-slurm`
  - Needs a CBS **Power** account
  - First login: connect once, then wait an hour for setup

> Home directories on the VDI and on SLURM are **separate**. Install tools on each system where you use them.

---

## Prerequisites: tools and data

- **pixi** (SPIMpack, SPIMquant): on the VDI *and* SLURM
  ```bash
  curl -fsSL https://pixi.sh/install.sh | bash
  ```
- **uv** (napari): on the VDI
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```
- **Data** on `/nfs/trident3/lightsheet/`, mounted on the VDI and SLURM
- **Stitching** to Imaris is done by the **core**, which puts the datasets in `raw/`

---

## Project layout

| Folder | Contents |
|---|---|
| `raw/` | Imaris datasets (placed by the core) |
| `bids/` | BIDS dataset made by SPIMpack |
| `etc/` | SPIMpack manifest (`manifest.yml`, `scans.tsv`), notes |
| `derivatives/` | Pipeline outputs, e.g. `spimquant_v0.10.0/` |

```
lightsheet/<pi_id>/<project_id>/raw/ims_4x_stitched/a_AS134F1/*.ims
                                                    └ <bag_id>_<subject_id>
```

Full naming standard and validator:
https://github.com/khanlab/mind-lightsheet-validator

---

## View your Imaris data

- **Windows:** Imaris Viewer
- **Linux (CBS VDI):** napari with the `napari-zarrnii` plugin

```bash
uv tool install 'napari[pyside6]' --with napari-zarrnii
napari     # then drag in an .ims file
```

Check that the whole brain is there, all channels are present, and there are no
obvious stitching problems.

---

## Make a BIDS dataset with SPIMpack (VDI)

```bash
pixi global install spimpack -c https://prefix.dev/akhanf/khanlab -c conda-forge
spimpack-ui      # opens http://localhost:8501
```

1. **Dataset description:** name, authors, license
2. **Scans table:** add your `.ims` datasets, one row per file
   - **subject:** the `<subject_id>` from the folder name
   - **stains:** set **all** stain channels, usually listed in the Imaris filename
   - **orientation:** see next slide
3. **Validate & Download:** save `manifest.yml` and `scans.tsv` to `etc/`

---

## Orientation

| Acquisition | `orientation_string_xyz` |
|---|---|
| Standard mouse whole-brain mount (default) | `RPI` |
| Brain placed the opposite way (rotated 180°) | `LAI` |
| Anything else (e.g. marmoset hemisphere) | **Work it out and confirm it** |

A wrong orientation leads to failed or mirrored registrations.

---

## Write the BIDS dataset

```bash
P=/nfs/trident3/lightsheet/<pi>/<project>
spimpack package --manifest $P/etc/manifest.yml \
  --output-dir $P/bids --backend symlink
```

```
bids/
├── dataset_description.json
├── participants.tsv
├── samples.tsv
└── sub-AS134F1/micr/
    ├── sub-AS134F1_sample-brain_acq-4x_SPIM.ims  -> raw/...
    └── sub-AS134F1_sample-brain_acq-4x_SPIM.json
```

Don't edit `bids/` by hand. Change the manifest and re-run.

---

## Run SPIMquant (SLURM)

On the SLURM login node:

```bash
pixi global install \
  --channel https://prefix.dev/conda-forge \
  --channel https://prefix.dev/bioconda \
  --channel https://prefix.dev/akhanf/khanlab \
  spimquant

BIDS=/nfs/trident3/lightsheet/<pi>/<project>/bids
OUT=/nfs/trident3/lightsheet/<pi>/<project>/derivatives/spimquant_v0.10.0

spimquant $BIDS $OUT participant --executor slurm -n      # dry run
spimquant $BIDS $OUT participant --executor slurm --jobs 10 --retries 1
```

- Each step runs as its own SLURM job. Monitor with `squeue -u $USER`.
- Leave the VDI terminal open until the run finishes.

---

## SPIMquant: stains and options

SPIMquant picks channels **by stain name**:

- **Registration:** `PI`, `YOPRO`, `YoPro`, `AutoF`, ...
- **Segmentation:** `Abeta`, `AlphaSynuclein`, `Iba1`, `NeuN`, `ChAT`, ...
- **Vessels:** `CD31`, `Lectin`

| Option | What it does |
|---|---|
| `--participant-label ...` | Only process these subjects |
| `--template {ABAv3,DSURQE,gubra,MBMv3,turone}` | Registration template |
| `--template-crop {left,right}` | Register to one hemisphere |
| `--stains-for-reg/seg/vessels ...` | Override stain choices |

---

## Outputs and group stats

- `micr/*_space-<template>_SPIM.nii.gz`: data registered to the template
- `parc/*_dseg.nii.gz`: atlas parcellation in subject space
- `seg/*_mask.ozx`: full-resolution segmentations
- `tabular/*_segstats.tsv`: region-wise quantification

Group level (after adding group columns to `participants.tsv`):

```bash
spimquant $BIDS $OUT group \
  --group-stats-model "metric ~ C(treatment) + age" \
  --group-stats-pairwise treatment \
  --executor slurm --jobs 10
```

---

## Getting help

- **CBS (VDI, SLURM, storage):** https://hackmd.io/@CompCore/cbs_servers, support-cbs-server@uwo.ca
- **Folder naming:** https://github.com/khanlab/mind-lightsheet-validator
- **SPIMpack:** https://github.com/khanlab/SPIMpack
- **SPIMquant:** https://github.com/khanlab/SPIMquant and https://spimquant.readthedocs.io
- **Full guide:** https://github.com/mind-western/lightsheet-getting-started

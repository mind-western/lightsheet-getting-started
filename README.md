# Getting started with lightsheet data at MIND (Western)

A step-by-step guide to working with lightsheet microscopy data in our labs: from
stitched Imaris datasets on `trident3`, to a BIDS dataset, to quantitative results
with [SPIMquant](https://github.com/khanlab/SPIMquant).

```
microscope ──► core: stitch to Imaris (.ims) ──► /nfs/trident3/lightsheet/<pi>/<project>/raw/
                                                         │
                                                         ▼
                                       SPIMpack (spimpack-ui) ──► <project>/bids/
                                                                        │
                                                                        ▼
                                                SPIMquant ──► <project>/derivatives/spimquant_<version>/
```

## Contents

1. [Prerequisites](#1-prerequisites)
2. [View your Imaris data](#2-view-your-imaris-data)
3. [Make a BIDS dataset with SPIMpack](#3-make-a-bids-dataset-with-spimpack)
4. [Run SPIMquant](#4-run-spimquant)
5. [Getting help](#5-getting-help)

---

## 1. Prerequisites

This guide assumes the following are in place.

### 1.1 Access to the CBS server

You need an account on the CBS server; all commands below are run there unless
stated otherwise. Ask your PI if you don't have access yet.

### 1.2 pixi (Python tools: SPIMpack, SPIMquant, ...)

[pixi](https://pixi.prefix.dev) installs our Python tools and their non-Python
dependencies into isolated environments. Install it once in your home directory:

```bash
curl -fsSL https://pixi.sh/install.sh | bash
source ~/.bashrc    # or open a new terminal
pixi --version
```

Our tools are published on the `khanlab` channel on prefix.dev
(`https://prefix.dev/akhanf/khanlab`), and are installed with `pixi global install`
(see sections 3 and 4).

### 1.3 uv (napari)

[uv](https://docs.astral.sh/uv/) is used to install [napari](https://napari.org),
the image viewer we use on Linux:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv --version
```

### 1.4 Data lives on `/nfs/trident3`

Lightsheet data are stored under `/nfs/trident3/lightsheet/`. `trident3` is our
newest fileserver and can be NFS-mounted from the cluster, so it is the preferred
location for active projects.

### 1.5 Stitching is done by the core

Stitching the raw TIFF tiles into Imaris (`.ims`) datasets is done by the imaging
core. The stitched datasets are placed on `trident3` in your project's
`raw/ims_<objective>x[<zoom>]_stitched/` folder. This guide starts from there.

### 1.6 Folder structure

The `lightsheet/` folder follows the MIND naming standard, described in full (with a
browser-based validator) at
**[khanlab/mind-lightsheet-validator](https://github.com/khanlab/mind-lightsheet-validator)**.
In short:

```
/nfs/trident3/lightsheet/
└── <pi_id>/                          # lowercase letters only, e.g. prado
    └── <project_id>/                 # lowercase, letters/numbers/_, e.g. mouse_app_lecanemab_batch3
        ├── README.md                 # recommended: description, species, stains, contact
        ├── raw/
        │   ├── tif_4x/               # raw tiles from the microscope
        │   └── ims_4x_stitched/      # stitched Imaris datasets (from the core)
        │       ├── a_AS134F1/        # <bag_id>_<subject_id>[_<modifier>]
        │       │   └── *.ims
        │       └── b_AS134F3/
        │           └── *.ims
        ├── bids/                     # made by SPIMpack (section 3)
        ├── derivatives/              # pipeline outputs, e.g. spimquant_v0.10.0/
        └── etc/                      # spreadsheets, notes, anything else
```

The `<subject_id>` in each sample folder name becomes the BIDS participant label,
so check the names before building the BIDS dataset.

---

## 2. View your Imaris data

Before processing, open a few datasets to check they look right (whole brain present,
all channels there, no obvious stitching problems).

- **Windows:** use the free [Imaris Viewer](https://imaris.oxinst.com/imaris-viewer).
- **Linux (e.g. on the CBS server):** use napari with the
  [napari-zarrnii](https://github.com/khanlab/napari-zarrnii) plugin, which reads
  `.ims` files directly.

Install napari as a uv tool (one time):

```bash
uv tool install 'napari[pyside6]' --with napari-zarrnii
```

Then launch it and drag in an `.ims` file (or use *File → Open*):

```bash
napari
```

> napari needs a graphical session. Run it from a remote desktop session on the
> CBS server, not a plain SSH terminal.

---

## 3. Make a BIDS dataset with SPIMpack

[SPIMpack](https://github.com/khanlab/SPIMpack) packages your existing Imaris
datasets into a [BIDS](https://bids-specification.readthedocs.io/en/stable/modality-specific-files/microscopy.html)
dataset. It doesn't copy or convert the images. It creates symlinks to your `.ims`
files and writes the metadata (stains, orientation, voxel size, ...) that
downstream tools need.

### 3.1 Install SPIMpack

```bash
pixi global install spimpack -c https://prefix.dev/akhanf/khanlab -c conda-forge
```

To update to the latest release later, run `pixi global update spimpack`.

### 3.2 Launch the UI

```bash
spimpack-ui
```

A browser window opens at `http://localhost:8501`.

> **Running on the server over SSH?** Forward the port, then open
> `http://localhost:8501` in the browser on your own computer:
>
> ```bash
> ssh -L 8501:localhost:8501 <user>@<cbs-server>
> ```

### 3.3 Fill in the dataset

1. **Dataset description:** dataset name, authors and license.
2. **Scans table:** add your Imaris datasets, one row per `.ims` file
   (e.g. everything in `raw/ims_4x_stitched/*/`). For each row, check:
   - **subject:** the `<subject_id>` from the folder name (letters and numbers only).
   - **sample:** usually `brain`.
   - **acquisition (optional):** objective/zoom, e.g. `4x`. This matters if you have
     more than one acquisition per sample.
   - **stains (`sample_staining`):** set **all** stain channels, in channel order.
     These are usually listed in the Imaris filename (e.g. `..._Abeta_YoPro_CD31.ims`).
     SPIMquant uses these names to decide which channel to register and which to
     segment, so spell them consistently (see [section 4.4](#44-stains)).
   - **orientation (`orientation_string_xyz`):** see below.
3. **Validate & Download:** fix any errors shown, then save `manifest.yml` and
   `scans.tsv`. A good place for them is your project's `etc/` folder.

#### Orientation

The orientation string tells downstream tools how the image axes map onto the
anatomy, which template registration needs to get right.

| Acquisition | `orientation_string_xyz` |
|---|---|
| Standard mouse whole-brain mount (default) | `RPI` |
| Brain placed the opposite way (rotated 180°) | `LAI` |
| Anything else (e.g. marmoset hemisphere, custom mounts) | **Work it out and confirm it** (see note) |

> **Custom orientations:** if your acquisition differs from the standard mount, you
> need to work out the correct orientation string yourself and confirm it. For
> example, open the data in napari next to the template, or run SPIMquant on one
> subject and check the registration. Ask if you are unsure. A wrong orientation
> leads to failed or mirrored registrations.

### 3.4 Write the BIDS dataset

Use the manifest from the UI to write the dataset into your project's `bids/` folder:

```bash
spimpack package \
  --manifest /nfs/trident3/lightsheet/<pi>/<project>/etc/manifest.yml \
  --output-dir /nfs/trident3/lightsheet/<pi>/<project>/bids \
  --backend symlink
```

You should end up with something like:

```
bids/
├── dataset_description.json
├── participants.tsv
├── samples.tsv
├── samples.json
└── sub-AS134F1/
    └── micr/
        ├── sub-AS134F1_sample-brain_acq-4x_SPIM.ims  -> ../../../raw/ims_4x_stitched/a_AS134F1/....ims
        └── sub-AS134F1_sample-brain_acq-4x_SPIM.json
```

Don't edit files in `bids/` by hand. To change something, load the manifest in
`spimpack-ui` again (*Load existing manifest.yml to edit*), fix it, and re-run
`spimpack package`.

---

## 4. Run SPIMquant

[SPIMquant](https://github.com/khanlab/SPIMquant) registers each brain to a template
atlas, segments pathology/cell stains and vessels, and produces region-wise
statistics. Full documentation: <https://spimquant.readthedocs.io>.

### 4.1 Install

Install the latest release with pixi:

```bash
pixi global install \
  --channel https://prefix.dev/conda-forge \
  --channel https://prefix.dev/bioconda \
  --channel https://prefix.dev/akhanf/khanlab \
  spimquant
```

Check that it works and see all options:

```bash
spimquant --help
```

### 4.2 Dry run

SPIMquant is a [Snakebids](https://github.com/khanlab/snakebids) BIDS app:
`spimquant <bids_dir> <output_dir> <analysis_level> [options]`. Always start with a
dry run (`-n`), which lists the jobs that would run without running them:

```bash
BIDS=/nfs/trident3/lightsheet/<pi>/<project>/bids
OUT=/nfs/trident3/lightsheet/<pi>/<project>/derivatives/spimquant_v0.10.0

spimquant $BIDS $OUT participant -n
```

Name the output folder after the SPIMquant version you are running
(`pixi global list` shows it), as the folder standard requires.

To test on a single subject first, add `--participant-label AS134F1`.

### 4.3 Participant-level run

```bash
spimquant $BIDS $OUT participant --cores all
```

Useful options:

| Option | What it does |
|---|---|
| `--cores N` / `--cores all` | Number of CPU cores to use |
| `--participant-label ...` | Only process these subjects |
| `--template {ABAv3,DSURQE,gubra,MBMv3,turone}` | Registration template (default `ABAv3`, Allen mouse) |
| `--template-crop {left,right}` | Register to one hemisphere of the template |
| `--sample-staining ...` | Override stain names from the BIDS sidecars |
| `--stains-for-reg ...` | Candidate stains for registration (first one found is used) |
| `--stains-for-seg ...` | Stains to segment and quantify |
| `--stains-for-vessels ...` | Stains to use for vessel segmentation |
| `--report` | Write an HTML report of the workflow |

Processing whole brains takes significant memory and disk. See SPIMquant's
[hardware notes](https://spimquant.readthedocs.io/en/latest/getting_started/hardware.html).

### 4.4 Stains

SPIMquant picks channels by **name**, using the stains you entered in SPIMpack:

- **Registration** uses the first stain found from: `PI`, `YOPRO`, `YoPro`, `YoPro1`,
  `AutoF`, `autof`.
- **Segmentation** runs on stains such as `Abeta`, `BetaAmyloid`, `AlphaSynuclein`,
  `Iba1`, `NeuN`, `Olig2`, `ChAT`, `VAChT`, `Thy1GFP`, ...
- **Vessels** uses `CD31` or `Lectin`.

If your stain names differ, either fix them in SPIMpack or pass them explicitly with
the options above. Run `spimquant --help` for the full default lists.

### 4.5 Outputs

Key outputs in `$OUT`:

- `sub-*/micr/*_space-<template>_SPIM.nii.gz`: SPIM data registered to the template
- `sub-*/parc/*_dseg.nii.gz`: atlas parcellation in subject space
- `sub-*/seg/*_mask.ozx`: full-resolution segmentations
- `sub-*/tabular/*_segstats.tsv`: region-wise quantification

View the volumetric results in napari (section 2).

### 4.6 Group-level statistics

After the participant-level run, add your group variables to `bids/participants.tsv`
(e.g. `treatment`, `genotype`, `sex`, `age`; set them in SPIMpack so they survive
re-packaging), then:

```bash
spimquant $BIDS $OUT group \
  --group-stats-model "metric ~ C(treatment) + age" \
  --group-stats-pairwise treatment \
  --group-stats-label treatment_analysis \
  --cores all
```

Results (merged tables, per-region stats, heatmaps and statistical maps) are written
to `$OUT/group/<label>/`. See the
[SPIMquant README](https://github.com/khanlab/SPIMquant#group-level-statistical-analysis)
for more complex designs.

---

## 5. Getting help

- Folder naming / validator: [khanlab/mind-lightsheet-validator](https://github.com/khanlab/mind-lightsheet-validator/issues)
- BIDS packaging: [khanlab/SPIMpack](https://github.com/khanlab/SPIMpack/issues)
- Processing: [khanlab/SPIMquant](https://github.com/khanlab/SPIMquant/issues) and
  [docs](https://spimquant.readthedocs.io)
- Problems with this guide: open an issue in this repository.

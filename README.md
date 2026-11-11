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

### Where each step runs

| Step | Where | Why |
|---|---|---|
| Viewing data (napari) | CBS **VDI** | Graphical desktop |
| Making the BIDS dataset (`spimpack-ui`) | CBS **VDI** | Runs in a web browser |
| Running SPIMquant | CBS **SLURM cluster** | Needs many cores and lots of memory |

## Contents

1. [Prerequisites](#1-prerequisites)
2. [View your Imaris data](#2-view-your-imaris-data)
3. [Make a BIDS dataset with SPIMpack](#3-make-a-bids-dataset-with-spimpack)
4. [Run SPIMquant](#4-run-spimquant)
5. [Getting help](#5-getting-help)

---

## 1. Prerequisites

This guide assumes the following are in place.

### 1.1 Access to the CBS servers

The CBS servers at Robarts have two parts that we use (full docs:
**[CBS server guides](https://hackmd.io/@CompCore/cbs_servers)**):

- **VDI (virtual desktop):** a Linux desktop you connect to with the
  [Omnissa Horizon Client](https://hackmd.io/@CompCore/cbs_vdi), server
  `vdi.robarts.ca`, logging in with `<westernid>@uwo.ca`. Use it to **look at
  data** and run interactive tools (napari, `spimpack-ui`).
- **SLURM cluster:** compute nodes with many cores, lots of memory and GPUs. Use it
  to **run SPIMquant**. Log in from the VDI (or the campus network) with:

  ```bash
  ssh <westernid>@rri-cbs-slurm
  ```

  See the [CBS SLURM guide](https://hackmd.io/@CompCore/cbs_slurm) for details.
  The first time, connect and disconnect once, then wait an hour for your account
  to be set up before connecting again.

This tutorial assumes you already have a CBS **Power** account, which is needed for
SLURM access (a Basic account only gets the VDI). Ask your PI if you don't have one.

> **Your home directory on the VDI and on SLURM are separate.** Install pixi, uv and
> the tools below on each system where you'll use them: napari and SPIMpack on the
> VDI, SPIMquant on the SLURM login node.

### 1.2 pixi (Python tools: SPIMpack, SPIMquant, ...)

[pixi](https://pixi.prefix.dev) installs our Python tools and their non-Python
dependencies into isolated environments. Install it once in your home directory:

```bash
curl -fsSL https://pixi.sh/install.sh | bash
source ~/.bashrc    # or open a new terminal
pixi --version
```

Do this on both the VDI and the SLURM login node.

Our tools are published on the `khanlab` channel on prefix.dev
(`https://prefix.dev/akhanf/khanlab`), and are installed with `pixi global install`
(see sections 3 and 4).

### 1.3 uv (napari)

[uv](https://docs.astral.sh/uv/) is used to install [napari](https://napari.org),
the image viewer we use on Linux. You only need it on the VDI:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv --version
```

### 1.4 Data lives on `/nfs/trident3`

Lightsheet data are stored under `/nfs/trident3/lightsheet/`. `trident3` is our
newest fileserver and is mounted on both the VDI and the SLURM cluster, so it is
the preferred location for active projects.

### 1.5 Stitching is done by the core

Stitching the raw TIFF tiles into Imaris (`.ims`) datasets is done by the imaging
core. The stitched datasets are placed on `trident3` in your project's
`raw/ims_<objective>x[<zoom>]_stitched/` folder. This guide starts from there.

### 1.6 Folder structure

Each project has the following layout:

| Folder | Contents |
|---|---|
| `raw/` | Imaris datasets (placed here by the core) |
| `bids/` | BIDS dataset made by SPIMpack (section 3) |
| `etc/` | SPIMpack manifest (`manifest.yml`, `scans.tsv`), spreadsheets, notes, etc. |
| `derivatives/` | Pipeline outputs, e.g. SPIMquant (section 4) |

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
        ├── etc/                      # SPIMpack manifest, spreadsheets, notes
        └── derivatives/              # pipeline outputs, e.g. spimquant_v0.10.0/
```

The `<subject_id>` in each sample folder name becomes the BIDS participant label,
so check the names before building the BIDS dataset.

---

## 2. View your Imaris data

Before processing, open a few datasets to check they look right (whole brain present,
all channels there, no obvious stitching problems).

- **Windows:** use the free [Imaris Viewer](https://imaris.oxinst.com/imaris-viewer).
- **Linux (CBS VDI):** use napari with the
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

Run napari on the VDI desktop, not on the SLURM cluster. Large datasets open
faster on the CBS-Heavy VDI (Power accounts).

---

## 3. Make a BIDS dataset with SPIMpack

[SPIMpack](https://github.com/khanlab/SPIMpack) packages your existing Imaris
datasets into a [BIDS](https://bids-specification.readthedocs.io/en/stable/modality-specific-files/microscopy.html)
dataset. It doesn't copy or convert the images. It creates symlinks to your `.ims`
files and writes the metadata (stains, orientation, voxel size, ...) that
downstream tools need.

Do this section on the **VDI**.

### 3.1 Install SPIMpack

```bash
pixi global install spimpack -c https://prefix.dev/akhanf/khanlab -c conda-forge
```

To update to the latest release later, run `pixi global update spimpack`.

### 3.2 Launch the UI

```bash
spimpack-ui
```

A browser window opens on the VDI desktop at `http://localhost:8501`.

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

Do this section on the **SLURM cluster**. From a VDI terminal:

```bash
ssh <westernid>@rri-cbs-slurm
```

You run `spimquant` on the login node. It doesn't do the processing there:
it submits each step as a SLURM job to the compute nodes and keeps track of them.

### 4.1 Install

On the SLURM login node, install the latest release with pixi:

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

spimquant $BIDS $OUT participant --executor slurm -n
```

Name the output folder after the SPIMquant version you are running
(`pixi global list` shows it), as the folder standard requires.

To test on a single subject first, add `--participant-label AS134F1`.

### 4.3 Participant-level run

From a terminal on the VDI, logged in to the SLURM login node, submit the
workflow to SLURM:

```bash
spimquant $BIDS $OUT participant --executor slurm --jobs 10 --retries 1
```

`spimquant` keeps running on the login node until all jobs finish, which can take
hours to days. Your VDI session doesn't time out, so just leave the terminal open.
You can also use `tmux` or `screen` if you prefer.

- `--executor slurm`: submit each step as a SLURM job instead of running it locally.
- `--jobs 10`: at most 10 jobs queued or running at once.
- `--retries 1`: rerun a failed job once with more memory and time. This also
  covers transient shared-filesystem delays.

Check on your jobs with `squeue -u $USER`. Each job's log is under
`$OUT/.snakemake/slurm_logs/`.

> **Running SPIMquant inside a single SLURM job** (e.g. with `sbatch` or an
> interactive `salloc` session, using `--cores all` instead of `--executor slurm`)
> is possible but not recommended except for short tests. A job is limited to one
> node and a 48 h maximum, and you lose the per-step parallelism and retries.

Useful options:

| Option | What it does |
|---|---|
| `--executor slurm` / `--jobs N` | Submit to SLURM, up to N jobs at once |
| `--participant-label ...` | Only process these subjects |
| `--template {ABAv3,DSURQE,gubra,MBMv3,turone}` | Registration template (default `ABAv3`, Allen mouse) |
| `--template-crop {left,right}` | Register to one hemisphere of the template |
| `--sample-staining ...` | Override stain names from the BIDS sidecars |
| `--stains-for-reg ...` | Candidate stains for registration (first one found is used) |
| `--stains-for-seg ...` | Stains to segment and quantify |
| `--stains-for-vessels ...` | Stains to use for vessel segmentation |
| `--report` | Write an HTML report of the workflow |

Steps that use a GPU (e.g. vessel segmentation with vesselFM) need extra setup.
See the [SPIMquant cluster docs](https://spimquant.readthedocs.io/en/latest/usage/workflows.html#slurm).

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

View the volumetric results in napari on the VDI (section 2).

### 4.6 Group-level statistics

After the participant-level run, add your group variables to `bids/participants.tsv`
(e.g. `treatment`, `genotype`, `sex`, `age`; set them in SPIMpack so they survive
re-packaging), then:

```bash
spimquant $BIDS $OUT group \
  --group-stats-model "metric ~ C(treatment) + age" \
  --group-stats-pairwise treatment \
  --group-stats-label treatment_analysis \
  --executor slurm --jobs 10
```

Results (merged tables, per-region stats, heatmaps and statistical maps) are written
to `$OUT/group/<label>/`. See the
[SPIMquant README](https://github.com/khanlab/SPIMquant#group-level-statistical-analysis)
for more complex designs.

---

## 5. Getting help

- CBS accounts, VDI, SLURM and storage: [CBS server guides](https://hackmd.io/@CompCore/cbs_servers),
  or email support-cbs-server@uwo.ca

- Folder naming / validator: [khanlab/mind-lightsheet-validator](https://github.com/khanlab/mind-lightsheet-validator/issues)
- BIDS packaging: [khanlab/SPIMpack](https://github.com/khanlab/SPIMpack/issues)
- Processing: [khanlab/SPIMquant](https://github.com/khanlab/SPIMquant/issues) and
  [docs](https://spimquant.readthedocs.io)
- Problems with this guide: open an issue in this repository.

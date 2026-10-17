# cryo_datasets

Data repository of *Multi-Noise2Noise: Physics-Aware Self-Supervised Denoising for
Cryo-Electron Microscopy* (Grzesiuk, Karanowski, Kuczbański, Rzepiela, Frey, Zięba;
Bioinformatics). It holds everything needed to reproduce the paper without re-processing the
raw movies:

- motion-corrected, non-averaged particle movie stacks (HDF5) and their RELION STAR files,
- the training / validation / test partition and the evaluation subsets,
- reference maps and tight molecular masks,
- trained model checkpoints,
- reconstruction results.

The training and evaluation code is in
[Grigori200/multinoise2noise](https://github.com/Grigori200/multinoise2noise)
([10.5281/zenodo.23427776](https://doi.org/10.5281/zenodo.23427776)). Large files are versioned
with [DVC](https://dvc.org/) and stored on Google Drive; this git repository contains only the
`.dvc` pointer files and this README.

## Datasets

| Directory | Specimen | Source | Symmetry | Frames | Dose / frame (e⁻/Å²) | Box (px) | Pixel size (Å/px) |
|-----------|----------|--------|----------|--------|----------------------|----------|-------------------|
| `10025/` | T20S proteasome | EMPIAR-10025 | D7 | 38 | 1.3947 | 300 | 0.9856 |
| `10474/` | Human light-chain apoferritin | EMPIAR-10474 | O | 30 | 1.36 | 384 | 0.706667 |
| `10948/` | GroEL | EMPIAR-10948 | D7 | 40 | 1.0 | 360 | 0.813 |
| `10025_sim/` | T20S proteasome, simulated | PDB 6BDF | D7 | 38 | – | 300 | 0.9856 |
| `10474_sim/` | Apoferritin, simulated | PDB 6WX6 | O | 30 | – | 384 | 0.706667 |
| `10948_sim/` | GroEL, simulated | PDB 1SX3 | D7 | 40 | – | 360 | 0.813 |

Number of particles per partition (from `paper/<ds>/training/<ds>_split.json`):

| Dataset | All | Train | Validation | Test (half 1 / half 2) | Micrographs |
|---------|-----|-------|------------|------------------------|-------------|
| 10025 | 34,297 | 8,488 | 17,226 | 8,583 (4,291 / 4,292) | 196 |
| 10948 | 49,670 | 12,519 | 24,915 | 12,236 (6,118 / 6,118) | 466 |
| 10474 | 203,903 | 50,847 | 102,037 | 51,019 (25,509 / 25,510) | 326 |
| 10025_sim | 80,000 | 20,000 | 40,000 | 20,000 (10,000 / 10,000) | 200 |
| 10474_sim | 51,200 | 12,800 | 25,600 | 12,800 (6,400 / 6,400) | 200 |
| 10948_sim | 80,000 | 20,000 | 40,000 | 20,000 (10,000 / 10,000) | 200 |

### Provenance

**Experimental data.** Movies were downloaded from EMPIAR and processed in RELION: movie
import, motion correction (RELION's MotionCor2 implementation, dose weighting, 5 × 5 patches,
B-factor 150), CTFFIND4, 2D classification and class ranking, Topaz picking, extraction,
3D auto-refinement and postprocessing (parameters: Supplementary Table S1 of the paper). The
CTF parameters, coordinates, orientations and shifts of the refined particles are those in
`<ds>.star`. Because RELION writes only frame-averaged micrographs, the per-frame,
motion-corrected particle movies were then computed with the motion-correction and extraction
pipelines of the code repository (`python -m source.correct_motion` and
`python -m source.extract`, configs in `configs_motion_correction/` and `configs_extract/`),
with dose weighting applied per frame
and CTF phase flipping (sign correction) applied during extraction.

**Simulated data.** Movies were simulated with TEM Simulator from the PDB models listed above,
without radiation damage. `extraction/clean/` holds the noise-free particle images used only
for image-level evaluation (PSNR, FRC); they are never used for training.

## Directory layout

```
cryo_datasets/
├── <ds>/                          one directory per dataset (DVC: <ds>.dvc)
│   ├── <ds>.star                  all particles (RELION 3.1 STAR: optics + particles)
│   ├── <ds>_subset.star           particles of the original training runs (see below)
│   ├── <ds>_model.mrc             map of the atomic model
│   ├── subsets/
│   │   └── denoised_<N>_<r>.star  evaluation subsets of the test set (see below)
│   ├── extraction/                particle movie stacks (HDF5, dataset `data`)
│   │   ├── <ds>.h5                experimental: (particles, frames, H, W)
│   │   ├── noisy/<ds>.h5          simulated: noisy movies
│   │   └── clean/<ds>.h5          simulated: clean images
│   ├── mask.mrc                   mask used by the reconstruction step of the training pipeline
│   └── half_{1,2}_reconstruction.mrc  raw-data half-maps used by the training pipeline
├── checkpoints/<ds>/*.ckpt        trained models (DVC: checkpoints.dvc)
├── results/10474/<method>/...     all reconstructions of denoised 10474 test subsets (DVC: results.dvc)
└── paper/                         everything the paper's evaluation reads (DVC: paper.dvc)
    ├── <ds>/
    │   ├── training/
    │   │   ├── <ds>_split.star    all particles with a train / val / test `split` column
    │   │   └── <ds>_split.json    particle counts and checks of the partition
    │   ├── reference/
    │   │   └── <ds>_model.mrc     reference map for model-to-map FSC and local resolution
    │   └── masks/
    │       ├── <ds>_mask_tight.mrc   tight molecular mask derived from the reference map
    │       └── <ds>_mask_tight.json  mask parameters and coverage statistics
    ├── results/<ds>/<method>/Refine_4/<r>/denoised_mean/   half-maps of the paper's metrics
    ├── results/<ds>_sim/frame_sweep/                       FRC files of Fig. 2
    └── evaluation/                metrics, tables and figures of the paper (expected outputs)
```

`paper/` is self-contained: the evaluation of the code repository (`configs_eval/paper.yaml`)
reads only files below it, so `dvc pull paper.dvc` is enough to recompute every number,
table and figure.

### Training, validation and test partition

Each dataset is split at the particle level into training (25 %), validation (50 %) and test
(25 %) sets. The splits were drawn at random, stratified by micrograph, and no particle belongs
to two splits.

- `paper/<ds>/training/<ds>_split.star` is the authoritative partition: every particle of `<ds>.star`
  with a `split` column (`train`, `val`, `test`). `rlnRandomSubset` of the test particles
  defines the two half-sets used for half-map reconstructions. It is read by the code
  repository with `datamodule.split_mode=column`.
- `<ds>_subset.star` is the file of the original training runs. It contains the
  `rlnRandomSubset == 2` half of `<ds>.star`, re-split into two new halves: half 1 is the
  training set, half 2 is the test set. The validation set is the remaining half of
  `<ds>.star`.

`paper/<ds>/training/<ds>_split.star` was built from these files with
`python -m source.datasets.split_star --full <ds>.star --subset <ds>_subset.star
--test-halves subsets/denoised_1_1.star --out paper/<ds>/training/<ds>_split.star`. For 10025, one test
particle is listed in both halves of `subsets/denoised_1_1.star`; the partition keeps its
first occurrence (half 2), recorded under `test_halves_conflicts_kept_first` in
`10025_split.json`. For 10474, one test particle is listed twice in the same half
(`test_halves_duplicate_rows` in `10474_split.json`).

### Evaluation subsets and results

`subsets/denoised_<N>_<r>.star` contains a random `1/N` of the test set (repetition
`r = 1..5`, `N = 1, 2, 4, …, 2048`), divided into two disjoint halves by `rlnRandomSubset`.
Each half is an evaluation subset of its own and is reconstructed separately
(`half_1_reconstruction.mrc`, `half_2_reconstruction.mrc`). The paper's reconstruction metrics
use `N = 4`: 5 files × 2 halves = 10 test subsets of 12.5 % of the test set each. Particles
keep the orientations refined on the raw data. Files are drawn independently, so different
files share particles. In `10025/subsets/denoised_4_*.star` (only), some particles occur
twice within a half or in both halves of a file (6–14 repeated rows and 4–15 shared
particles per file).

`results/10474/<method>/Refine_<N>/<r>/` holds the reconstructions of
`10474/subsets/denoised_<N>_<r>.star` denoised by `<method>`: `denoised_mean/` (used for the paper's
metrics) and `denoised_full_dose/` (an alternative image mode), each with
`half_{1,2}_reconstruction.mrc`, `reconstruction.mrc`, sharpened maps and `fsc.csv`. Methods:
`raw`, `noise2noise` (Topaz-Denoise baseline), `noise2noise_ctf`, `multi_noise2noise`,
`multi_noise2noise_ctf`, `multi_noise2noise_dw`, `multi_noise2noise_ctf_dw` (DW variants for
experimental data only).

`paper/results/<ds>/<method>/Refine_4/<r>/denoised_mean/` holds the files of these
reconstructions that the paper's evaluation reads: `half_{1,2}_reconstruction.mrc` of
`r = 1..5`, and `half_1_sharp.mrc` of `r = 1` (rendered in Figs. 7 and 8).

`paper/results/<ds>_sim/frame_sweep/split_<N>/frc_metrics_test.csv` (simulated datasets only)
holds the FRC between denoised test images and clean projections of the model trained with `N`
frames averaged into the input, for single, averaged and whole-recording inference (Fig. 2).

`paper/evaluation/` holds the outputs of the evaluation package of the code repository
(`configs_eval/paper.yaml`) computed from the files above:

- `reconstruction_fsc.csv` – model-to-map FSC metrics of every (dataset, method, subset,
  half-map) reconstruction of `Refine_4/<r>/denoised_mean`,
- `tables/fsc_recon.{tex,csv}` – Table 4 (mean ± standard deviation over the 10 half-maps),
- `locres_summary.csv`, `locres_histograms.csv` – local model-to-map resolution inside the
  symmetrised mask core,
- `figures/` – Figs. 2 and 5–8 and Supp. Fig. S2 as PNG and PDF; `frame_sweep.csv` and
  `locres_grid_{real,sim}.json` hold the plotted values, colour ranges and isosurface levels.

### Reference maps and masks

`paper/<ds>/reference/<ds>_model.mrc` is the reference for model-to-map FSC and local
resolution. The tight mask is `auto_mask(reference, 0.05 × max(reference))`, extended by 3
voxels and with a 10-voxel raised-cosine edge, computed with
`python -m source.evaluation.make_mask` of the code repository.
`paper/<ds>/masks/<ds>_mask_tight.json` reports which fraction of the box it covers and that
all reference voxels above mean + 1σ lie inside it.

## Checkpoints

`checkpoints/<ds>/` contains one PyTorch Lightning checkpoint per variant; each stores the full
Hydra configuration of its run. Names map to the experiment configs
`configs_train/experiment/<ds>/<model>/<name>.yaml` of the code repository:

| Checkpoint | Method | Experiment config |
|------------|--------|-------------------|
| `base.ckpt` | Noise2Noise (Topaz-Denoise baseline) | `multinoise2noise/base` |
| `base_ctf.ckpt` | Noise2Noise + CTF | `multinoise2noise_ctf/base` |
| `default.ckpt` | Multi-Noise2Noise | `multinoise2noise/default` |
| `default_ctf.ckpt` | Multi-Noise2Noise + CTF | `multinoise2noise_ctf/default` |
| `dose_weighted.ckpt` | Multi-Noise2Noise + DW (experimental only) | `multinoise2noise/dose_weighted` |
| `dose_weighted_ctf.ckpt` | Multi-Noise2Noise + CTF + DW (experimental only) | `multinoise2noise_ctf/dose_weighted` |
| `dose_weighted{,_ctf}_legacy.ckpt` | the same runs with the loss configuration they were trained with | `*/dose_weighted_legacy` |
| `split_<k>.ckpt` | frame-split ablation (Fig. 2, simulated only) | `multinoise2noise/split_<k>` |

`dose_weighted{,_ctf}.ckpt` hold the weights of the `_legacy` runs with the loss configuration
of the manuscript (dose weighting of the target undone, weights applied to the per-pixel loss);
they load into the current `dose_weighted` configs. Other files (`scunet*`) are not part of
the paper.

## Getting the data

Requirements: Python, git, and disk space: `paper/` is about 70 GB, `10025/` about 470 GB
and `results/` about 1.5 TB (sizes are listed in each `.dvc` file).

```bash
git clone https://github.com/cryo-EM-team/cryo_datasets.git
cd cryo_datasets
pip install -r requirements.txt          # dvc, dvc-gdrive
```

### Google Drive access through a Google Cloud service account

DVC needs credentials to read the Google Drive remote. A service account works without a
browser, e.g. on a cluster:

1. In the [Google Cloud console](https://console.cloud.google.com/), create (or select) a
   project and enable the **Google Drive API** for it (*APIs & Services → Library*).
2. Create a service account (*IAM & Admin → Service Accounts → Create*); no roles are needed.
3. Open the service account, *Keys → Add key → Create new key → JSON*, and store the downloaded
   file outside the repository, readable only by you (`chmod 600 key.json`). Never commit it.
4. Give the service account's e-mail address (`…@<project>.iam.gserviceaccount.com`) read
   access to the Drive folder of the remote (ask the maintainers to share it).
5. Configure DVC **locally** (written to `.dvc/config.local`, which git ignores):

   ```bash
   dvc remote modify --local drive gdrive_use_service_account true
   dvc remote modify --local drive gdrive_service_account_json_file_path /path/to/key.json
   ```

Without a service account, `dvc pull` asks for a Google login in the browser; an own OAuth
client can be configured with `gdrive_client_id` and `gdrive_client_secret` (also `--local`).

### Pulling

```bash
dvc pull paper.dvc            # evaluation inputs and expected outputs
dvc pull checkpoints.dvc      # all checkpoints
dvc pull 10948.dvc            # one dataset
dvc pull                      # everything
```

On clusters without direct internet access, load the site proxy first.

## Licences

The experimental movies come from EMPIAR and the atomic models from the PDB; both are
distributed under Creative Commons terms (see the EMPIAR and PDB entries). Please cite the
original EMPIAR depositions together with the paper when using the processed data.

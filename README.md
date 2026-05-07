# ICPR-TVRID
Competition on Privacy-Preserving Person Re-Identification from Top-View RGB-Depth Camera (TVRID).

## Overview
TVRID was the ICPR 2026 competition on top-view person re-identification with aligned RGB and Depth. The competition has now concluded, and this repository is kept as an archive of the benchmark, baselines, and evaluation code. The benchmark captures 88 identities with four overhead Intel RealSense D455 cameras observing each passage twice (IN/OUT) across four geometric contexts: flat ground, ascent, descent, and oblique roof view. Submissions were ranked lists evaluated with CMC@1/5/10 and mAP, and the primary leaderboard metric was `overallMap` (mean of per-track overall mAP).

Tracks:
- RGB Re-ID (privacy-unconstrained)
- Depth Re-ID (privacy-preserving)
- Cross-modal RGB↔Depth retrieval

Competition page: https://www.codabench.org/competitions/12315/
Competition report: https://arxiv.org/abs/2605.04977

## Data download and layout
- Download from Zenodo: https://zenodo.org/records/17909410 (DOI: 10.5281/zenodo.17909410). Two zips are provided:
  - **Original**: full-resolution RGB/Depth streams.
  - **Extracted**: depth-guided crops (300×300) centered on each pedestrian—recommended for quick start and matches this repo’s defaults.
- The ground-truth labels are now available in the Zenodo release and can be downloaded locally.
- Unzip the **extracted** archive into `data/DB_extracted` so the structure matches:
  - `data/DB_extracted/train_labels.csv`
  - `data/DB_extracted/public_test_labels.csv`
  - `data/DB_extracted/train/` (cropped train data)
  - `data/DB_extracted/test_public/` (cropped public test data)

Important note:
- `public_test_labels.csv` now includes the ground-truth labels and can be used for local evaluation.
- It should still not be used as a training-time validation split unless you explicitly want to evaluate on that labeled CSV.

If you prefer to re-extract from the original zip, place it separately and adapt paths/transforms as needed. Update paths via CLI flags or Hydra configs if you store data elsewhere.

## Environment
Create and activate a virtual environment, then install requirements:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Baseline training
- Defaults live in `config/train.yaml` and `config/data/data.yaml`.
- `data.eval_csv` is a **validation split used during training**.

Because this is a competition, participants should create their own train/validation split **from `train_labels.csv`**, typically by splitting **by identity** (to avoid identity leakage across splits). The training set contains 62 unique identities; for example, a 50/50 split would allocate 31 identities to train and 31 identities to validation, keeping each identity entirely in one split.

Example (RGB track) with a user-defined split:
```bash
python train.py track=rgb data.root=data/DB_extracted   data.train_csv=data/DB_extracted/train_split.csv   data.eval_csv=data/DB_extracted/valid_split.csv   trainer.max_epochs=20
```

Recommended before generating submission rankings: train on the full training set (maximize data usage). You can set the same CSV for both arguments:
```bash
python train.py track=rgb data.root=data/DB_extracted   data.train_csv=data/DB_extracted/train_labels.csv   data.eval_csv=data/DB_extracted/train_labels.csv   trainer.max_epochs=20
```

Change `track=depth` or `track=cross` to train the other baselines; adjust batch size/devices as needed.

## Generate rankings (submission CSVs)
Legacy submission instructions are kept here for reference.

You can either:
1) Train the baseline yourself (see section above), or
2) Download the official pretrained baseline checkpoints from the GitHub **Releases** (they are not stored in the repository due to file size constraints).

After downloading, place them in `baselines_weights/` (or update the `--checkpoint` path accordingly).

Expected checkpoint paths:
- `baselines_weights/rgb_weights.ckpt`
- `baselines_weights/depth_weights.ckpt`
- `baselines_weights/cross_weights.ckpt`

Release page (checkpoints):
```text
https://github.com/RaphaelDel/ICPR-2026-TVRID/releases/tag/weights
```

Generate ranking CSVs for Codabench submission (public test split):
```bash
# RGB
python eval_generate.py --checkpoint baselines_weights/rgb_weights.ckpt --track rgb --data-root data/DB_extracted --labels-csv data/DB_extracted/public_test_labels.csv --output outputs/rankings_rgb.csv

# Depth
python eval_generate.py --checkpoint baselines_weights/depth_weights.ckpt --track depth --data-root data/DB_extracted --labels-csv data/DB_extracted/public_test_labels.csv --output outputs/rankings_depth.csv

# Cross-modal (RGB→Depth)
python eval_generate.py --checkpoint baselines_weights/cross_weights.ckpt --track cross --data-root data/DB_extracted --labels-csv data/DB_extracted/public_test_labels.csv --output outputs/rankings_cross.csv
```

Each CSV must contain columns:
`query_gallery_id, query_path, gallery_id, gallery_path, rank, distance`.

## Local evaluation
For local evaluation, use the downloaded labeled CSV directly as the ground-truth reference in the scoring step. The example below is for the RGB track, but you can adapt it for Depth and Cross by switching `--track` and output names.

1) Generate rankings using the labeled CSV as the reference file:
```bash
python eval_generate.py --checkpoint baselines_weights/rgb_weights.ckpt --track rgb \
  --data-root data/DB_extracted --labels-csv data/DB_extracted/public_test_labels.csv \
  --eval-subdir train --output outputs/rankings_rgb_valid.csv
```

2) Score those rankings against the same labeled CSV:
```bash
python eval_score.py --rankings outputs/rankings_rgb_valid.csv --secret-map data/DB_extracted/public_test_labels.csv
```

The official Codabench evaluation uses hidden test labels; your submissions are scored server-side with the same `bundle/scoring_program/scoring.py`, producing `scores.json`, `detailed_results.html`, and the primary `overallMap`.

## Submission packaging
- Create a zip with the three CSV files at the archive root (exact names required):
  - `rankings_rgb.csv`
  - `rankings_depth.csv`
  - `rankings_cross.csv`

Example:
```bash
cd outputs
zip submission.zip rankings_rgb.csv rankings_depth.csv rankings_cross.csv
```

Submit the zip on the Codabench page above. The leaderboard sorts by `overallMap`, computed as the mean of per-track `*mAP`, with CMC@1/5/10 and mAP reported per track and scenario.

## How to participate
This section is retained for archival reference; the competition is no longer accepting new submissions.
- Register on the Codabench competition page and clone this repository.
- Prepare data as described, generate ranked lists for one or more tracks, and package them into the zip format above.
- Upload the zip; Codabench runs the scoring program to produce `scores.json` and leaderboard entries.
- For test phases, only the public labels are available locally; final scoring uses hidden test labels on the server.

## Winners
Thanks to all participants for the final leaderboard results. Congratulations to the podium teams for each track:

| Track | Leader | Team | CMC@1 | mAP |
| --- | --- | --- | --- | --- |
| RGB Track | 🥇 1st | Hien Pham Duy et al. | 100.0% | 100.0% |
| RGB Track | 🥈 2nd | Carlo Metta | 100.0% | 99.98% |
| RGB Track | 🥉 3rd | Oron Nir et al. | 99.84% | 99.92% |
| Depth Track | 🥇 1st | Jin-Hui Jiang et al. | 98.86% | 99.38% |
| Depth Track | 🥈 2nd | Oron Nir et al. | 97.19% | 98.34% |
| Depth Track | 🥉 3rd | Md Rashidunnabi et al. | 90.44% | 94.33% |
| Cross-Modal Track | 🥇 1st | Md Rashidunnabi et al. | 99.84% | 99.92% |
| Cross-Modal Track | 🥈 2nd | Jin-Hui Jiang et al. | 97.89% | 98.76% |
| Cross-Modal Track | 🥉 3rd | Oron Nir et al. | 91.50% | 94.88% |

Winning teams and affiliations:

- Hien Pham Duy et al.: Hien Pham Duy, Bao Tran; University of Information Technology, VNU-HCM, Vietnam; Vietnam National University, Ho Chi Minh City, Vietnam.
- Carlo Metta: Carlo Metta; IST CNR, Pisa, Italy.
- Oron Nir et al.: Oron Nir, Eliyahu Strugo, Ariel Shamir; Reichman University; Microsoft Corporation.
- Jin-Hui Jiang et al.: Jin-Hui Jiang, Yu-Fan Lin, Fu-En Yang, Yu-Chiang Frank Wang, Chih-Chung Hsu; Institute of Computational Intelligence, National Yang Ming Chiao Tung University; Institute of Intelligent Systems, National Yang Ming Chiao Tung University; Institute of Data Science, National Cheng Kung University; NVIDIA, Taipei, Taiwan.
- Md Rashidunnabi et al.: Md Rashidunnabi, Kailash A. Hambarde, João C. Neves, Vasco Lopes, Hugo Proença; IT: Instituto de Telecomunicações, Covilhã, Portugal; University of Beira Interior, Covilhã, Portugal; NOVA LINCS, University of Beira Interior, Covilhã, Portugal; DeepNeuronic, Covilhã, Portugal.

Competition organisers:

- Raphaël Delecluse, Ph.D. Candidate at Université de Lille / IMT Nord Europe / EXPLAIN, whose research focuses on multimodal person re-identification and trajectory prediction.
- Prof. Hazem Wannous, Professor at Université de Lille and CRIStAL Lab, with expertise in computer vision, 3D human analysis, and multimodal fusion.
- Prof. Laurent Grisoni, Professor at IMT Nord Europe, with expertise in vision-based motion analysis and human-computer interaction.

## Dataloader quickstart for your own models
The dataset interface lives in `utils/data.py` and is meant to be reusable beyond the provided Lightning baselines.

- Core pieces: `DataConfig` (paths, modality, sequence length/sampling, normalization), `build_transforms` (per-modality torchvision transforms), and `UnifiedReIDDataset`.
- Modalities: set `modality` to `rgb`, `depth`, or `rgbd` (cross-modal) to control which inputs are loaded. Depth masking of RGB can be enabled with `mask_rgb_with_depth`.
- Sequences: `sequence.length` picks how many frames per passage to sample (evenly or randomly via `sequence.sampling`); transforms are applied after stacking to keep spatial alignment.

CSV requirements:
- For **training** (train/validation splits): CSVs must contain
  `gallery_id, person_id, cam_name, cam_id, passage_name, passage_id, path` (same schema as `train_labels.csv`).
- For **ranking generation on test splits**: the CSV only needs
  `gallery_id, path`.

Paths are relative to `train_subdir` or `eval_subdir`.

Sample usage with plain PyTorch:
```python
from utils.data import DataConfig, UnifiedReIDDataset, build_transforms
from torch.utils.data import DataLoader

cfg = DataConfig(
    root="data/DB_extracted",
    train_csv="data/DB_extracted/train_labels.csv",
    # For training-time validation, provide a labeled split derived from train_labels.csv:
    # eval_csv="data/DB_extracted/valid_split.csv",
    modality="rgb",          # or depth / rgbd
    sequence={"length": 5},  # set >1 for temporal sampling
)
rgb_t, depth_t = build_transforms(cfg.transforms)
train_set = UnifiedReIDDataset(
    csv_path=cfg.train_csv, root=cfg.root, modality=cfg.modality, mode="train",
    sequence=cfg.sequence, rgb_transform=rgb_t, depth_transform=depth_t,
    train_subdir=cfg.train_subdir, eval_subdir=cfg.eval_subdir,
)
loader = DataLoader(train_set, batch_size=8, shuffle=True, num_workers=4)
batch = next(iter(loader))
rgb = batch.get("rgb")      # [B,C,H,W] if modality includes RGB
depth = batch.get("depth")  # [B,1,H,W] if modality includes depth
```

For quick integration with Lightning, `UnifiedReIDDataModule` (also in `utils/data.py`) wraps the same dataset and applies the config defaults found in `config/data/data.yaml`.

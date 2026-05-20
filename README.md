# geneFlow

Conditional Flow Matching for predicting CRISPRi single-cell perturbation responses (Virtual Cell Challenge).

## Setup

```bash
git clone https://github.com/DhruvaSakhare/geneFlow-deepLearning.git
cd geneFlow-deepLearning

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Data

Download the Virtual Cell Challenge `training.h5ad` and `validation.h5ad` files and place them anywhere; record the paths.

Edit `perturbation_fm/configs/default.yaml` and update:

```yaml
train_path: /path/to/training.h5ad
val_path:   /path/to/validation.h5ad
emb_path:   /path/to/embeddings/esm2_embeddings.pt
save_dir:   /path/to/checkpoints/
```

## 1. Generate ESM-2 gene embeddings

```bash
mkdir -p embeddings
python -c "
from perturbation_fm.data.preprocess import preprocess
import numpy as np
d = preprocess('/path/to/training.h5ad', '/path/to/validation.h5ad')
genes = sorted(set(np.asarray(d['train_genes'])[~d['train_ctrl_mask']].tolist() +
                   np.asarray(d['val_genes'])[~d['val_ctrl_mask']].tolist()))
print(' '.join(genes))
" > /tmp/genes.txt

python -m perturbation_fm.embeddings.esm2_embed \
    --genes $(cat /tmp/genes.txt) \
    --out embeddings/esm2_embeddings.pt \
    --device cpu
```

## 2. Train

```bash
python -m perturbation_fm.training.train \
    --config perturbation_fm/configs/default.yaml
```

Trains for 100 epochs on a single A100 (~24h). Validation is evaluated every 5 epochs; the best checkpoint by VCC-aligned score `S = r_Pearson + (PDS_VCC - 0.533)` is saved to `save_dir/best_model.pt`.

## 3. Evaluate

### Cell-mean baseline (validation split)

```bash
python -m perturbation_fm.evaluation.eval_baseline_only \
    --config perturbation_fm/configs/default.yaml
```

Expected: `Pearson r = 0.238, PDS-VCC = 0.533, DES = 0.138, MAE = 0.034`.

### Best checkpoint on held-out test set

```bash
python -m perturbation_fm.evaluation.eval_test \
    --config perturbation_fm/configs/default.yaml \
    --ckpt /path/to/checkpoints/best_model.pt \
    --test_path /path/to/test.h5ad
```

Expected: `Pearson r = 0.167, PDS-VCC = 0.587, DES = 0.331, MAE = 0.041`.

## Reproducing reported numbers

Run steps 1 → 2 → 3 in order with `seed: 42` (default in `default.yaml`). Numbers in the report come from running `eval_baseline_only` on the validation split and `eval_test` on the held-out test split using the checkpoint selected at epoch 45.

| Metric    | Test baseline | Test model | Δ        |
|-----------|---------------|------------|----------|
| Pearson r | 0.237         | 0.167      | −30%     |
| PDS-VCC   | 0.533         | 0.587      | +11.5%   |
| DES       | 0.157         | 0.331      | +20.6%   |
| MAE       | 0.027         | 0.041      | clip 0   |

## Repository layout

```
perturbation_fm/
    configs/            YAML config files
    data/               Dataset, preprocessing, train/val split
    embeddings/         ESM-2 + GenePT embedding scripts
    model/              FlowNet (velocity network) + ResBlock/AdaLN
    training/           Training loop
    evaluation/         Metric definitions + eval entry points
```

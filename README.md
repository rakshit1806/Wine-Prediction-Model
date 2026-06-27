# Wine Prediction Model

A simple ML project that trains a Random Forest regressor to predict wine quality. It uses **DVC** for dataset versioning and **MLflow** for experiment tracking.

---

## Repository Structure

```
Wine-Prediction-Model/
├── .dvc/                    # DVC internal config and local cache
│   ├── config               # Remote storage configuration
│   └── cache/               # Local copy of tracked data (auto-generated)
├── .dvcignore               # Files DVC should skip
├── .gitignore               # Git ignores the actual CSV (DVC tracks it instead)
├── wine_data_set.csv        # Dataset (ignored by Git, tracked by DVC)
├── wine_data_set.csv.dvc    # DVC pointer file (committed to Git)
├── train.py                 # Training script with MLflow logging
├── utils.py                 # Data loading helpers
└── requirements.txt         # Python dependencies
```

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| Git | Source control |
| Python 3.10+ | Runtime |
| DVC | Dataset versioning |
| AWS CLI (optional) | S3 remote access for DVC |

Install DVC with S3 support:

```bash
pip install 'dvc[s3]'
```

Install project dependencies:

```bash
pip install -r requirements.txt
```

---

## Onboarding: Step-by-Step Setup

Follow these steps in order when setting up the repo from scratch.

### 1. Clone the Repository

Always work inside the project directory — **not** the parent `Personal/` folder.

```bash
git clone git@github.com:iam-veeramalla/Wine-Prediction-Model.git
cd Wine-Prediction-Model
```

> **Common mistake:** Running `dvc init` from `/Users/<you>/Personal` fails because that folder is not a Git repo. Always `cd` into `Wine-Prediction-Model` first.

---

### 2. Initialize DVC

Run this once per repo to create the `.dvc/` directory and `.dvcignore`.

```bash
dvc init
```

**What gets created:**

| File / Folder | Purpose |
|---------------|---------|
| `.dvc/` | DVC configuration and local cache |
| `.dvcignore` | Patterns for files DVC should ignore |

Commit the DVC scaffold to Git:

```bash
git add .dvc .dvcignore
git commit -m "Initialize DVC"
```

---

### 3. Track the Dataset with DVC

The main dataset is `wine_data_set.csv` at the repo root.

```bash
dvc add wine_data_set.csv
```

**What happens:**

1. DVC computes an MD5 hash of the file
2. A copy is stored in `.dvc/cache/`
3. `wine_data_set.csv.dvc` is created (a small pointer file)
4. `wine_data_set.csv` is added to `.gitignore` so Git never stores the raw CSV

Example pointer file:

```yaml
outs:
- md5: 0d18849e41a76563c932f3f4ed44c2fe
  size: 358
  hash: md5
  path: wine_data_set.csv
```

Commit the pointer — not the CSV:

```bash
git add wine_data_set.csv.dvc .gitignore
git commit -m "Track wine_data_set.csv with DVC"
```

---

### 4. Configure a Remote Storage (S3)

Point DVC to an S3 bucket so data can be shared across machines and teammates.

```bash
dvc remote add -d wine-remote s3://mlops-zero-to-hero
```

This writes to `.dvc/config`:

```ini
[core]
    remote = wine-remote
['remote "wine-remote"']
    url = s3://mlops-zero-to-hero
```

Commit the remote config:

```bash
git add .dvc/config
git commit -m "Configure DVC S3 remote"
```

Ensure AWS credentials are configured before pushing:

```bash
aws configure
# or export AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
```

---

### 5. Push Data to Remote

Upload the dataset to S3:

```bash
dvc push
```

Verify status:

```bash
dvc status
# Expected: Data and pipelines are up to date.
```

---

### 6. Pull Data on a New Machine

After cloning the repo, the CSV won't be present locally. Restore it from the remote:

```bash
git clone git@github.com:iam-veeramalla/Wine-Prediction-Model.git
cd Wine-Prediction-Model
dvc pull
```

This reads `wine_data_set.csv.dvc`, downloads the file from S3, and places `wine_data_set.csv` in the repo root.

---

## Training the Model

Start an MLflow tracking server (if not already running):

```bash
mlflow server --host 0.0.0.0 --port 7006
```

Train using the DVC-tracked dataset:

```bash
python train.py --csv wine_data_set.csv
```

### CLI Options

| Flag | Default | Description |
|------|---------|-------------|
| `--csv` | `data/wine_sample.csv` | Path to the CSV file |
| `--target` | `quality` | Target column name |
| `--experiment` | `wine-prediction` | MLflow experiment name |
| `--run` | `run-2` | MLflow run name |
| `--n-estimators` | `50` | Random Forest tree count |
| `--max-depth` | `5` | Max tree depth |
| `--test-size` | `0.2` | Hold-out fraction |
| `--random-state` | `42` | Random seed |

Override the MLflow server URL:

```bash
export MLFLOW_TRACKING_URI=http://localhost:7006
python train.py --csv wine_data_set.csv
```

---

## Git vs DVC: What Goes Where

| Artifact | Stored in Git | Stored in DVC/S3 |
|----------|---------------|------------------|
| Source code (`train.py`, `utils.py`) | Yes | No |
| DVC pointer (`wine_data_set.csv.dvc`) | Yes | No |
| DVC config (`.dvc/config`) | Yes | No |
| Raw dataset (`wine_data_set.csv`) | **No** | **Yes** |
| DVC cache (`.dvc/cache/`) | No | Local only |

---

## Updating the Dataset

When the CSV changes:

```bash
# 1. Edit or replace wine_data_set.csv
dvc add wine_data_set.csv          # Updates the .dvc pointer
dvc push                           # Upload new version to S3
git add wine_data_set.csv.dvc
git commit -m "Update wine dataset"
git push
```

---

## Useful DVC Commands

```bash
dvc status          # Check if local data matches .dvc pointers
dvc pull            # Download missing data from remote
dvc push            # Upload local data to remote
dvc add <file>      # Start tracking a new file
dvc remove <file>   # Stop tracking a file
dvc cache dir       # Show local cache location
```

---

## Troubleshooting

### `failed to initiate DVC - /Users/<you>/Personal is not tracked by any supported SCM tool`

You ran a DVC command from the wrong directory. Fix:

```bash
cd /path/to/Wine-Prediction-Model
dvc init   # or whichever command you were running
```

### `CSV not found` when running train.py

The dataset hasn't been pulled yet:

```bash
dvc pull
python train.py --csv wine_data_set.csv
```

### `dvc push` fails with S3 errors

Check AWS credentials and bucket access:

```bash
aws s3 ls s3://mlops-zero-to-hero
```

### `wine_data_set.csv` shows up as untracked in Git

That is expected. Only `wine_data_set.csv.dvc` should be committed. Confirm `.gitignore` contains:

```
/wine_data_set.csv
```

---

## Resetting DVC (Start Fresh for Learning)

To wipe DVC artifacts and rebuild from scratch:

```bash
rm -rf .dvc .dvcignore wine_data_set.csv.dvc
# Keep wine_data_set.csv — that is your actual data
```

Then repeat steps 2–5 above.

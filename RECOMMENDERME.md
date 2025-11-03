# Recommender-System — Quick Start (Local repo)

Ringkasan  
Repositori ini berisi contoh proyek Recommender-System yang siap dijalankan secara lokal dan disimpan ke Git. Termasuk:
- Implementasi sederhana Collaborative Filtering (UserKNN), Content-based (TF‑IDF) dan Hybrid aggregator.
- Skrip CLI untuk training dan inferensi.
- Contoh data sample.
- Unit tests dan GitHub Actions CI (pytest).
- Struktur proyek yang rapi (src, scripts, data, tests).
- Skrip PowerShell helper `run_project_files.ps1` untuk membuat semua file scaffold otomatis.

Quick setup (singkat)
1. Jika belum, simpan file README.md ini dan (opsional) skrip `run_project_files.ps1` di folder yang Anda inginkan.
2. Jalankan PowerShell script untuk membuat semua file project (opsional):
   - Menulis file ke folder saat ini:
     ```powershell
     pwsh -ExecutionPolicy Bypass -File .\run_project_files.ps1
     ```
   - Menulis file ke folder khusus dan inisialisasi git:
     ```powershell
     pwsh -ExecutionPolicy Bypass -File .\run_project_files.ps1 -ProjectDir "C:\path\to\project" -InitializeGit:$true
     ```
3. Buat virtual environment dan pasang dependensi:
   - Unix / macOS:
     ```bash
     python -m venv .venv
     source .venv/bin/activate
     pip install -r requirements.txt
     ```
   - Windows (PowerShell):
     ```powershell
     python -m venv .venv
     .\.venv\Scripts\Activate.ps1
     pip install -r requirements.txt
     ```
4. Contoh training (menggunakan sample data):
   ```bash
   python scripts/train.py --data data/sample/ratings.csv --items data/sample/items.csv --output models/
   ```
5. Contoh rekomendasi:
   ```bash
   python scripts/predict.py --model models/cf_knn.pkl --user-id 1 --k 5
   ```

Struktur proyek
```
.
├── README.md
├── LICENSE
├── .gitignore
├── requirements.txt
├── pyproject.toml
├── data/
│   └── sample/
│       ├── ratings.csv
│       └── items.csv
├── src/
│   └── recommender/
│       ├── __init__.py
│       ├── data.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── cf.py
│       │   ├── content.py
│       │   └── hybrid.py
│       ├── evaluate.py
│       └── utils.py
├── scripts/
│   ├── train.py
│   └── predict.py
├── tests/
│   └── test_models.py
└── .github/
    └── workflows/
        └── ci.yml
```

Deskripsi singkat komponen
- `src/recommender/data.py`: loader & split train/test.
- `src/recommender/models/cf.py`: UserKNN simple matrix-based KNN.
- `src/recommender/models/content.py`: TF‑IDF item similarity.
- `src/recommender/models/hybrid.py`: gabungkan skor model dengan bobot.
- `src/recommender/evaluate.py`: precision@k & recall@k.
- `src/recommender/utils.py`: save/load model (joblib).
- `scripts/train.py`: melatih model dan menyimpan artifact (.pkl).
- `scripts/predict.py`: memuat model dan mengeluarkan rekomendasi.

Contoh data sample
- `data/sample/ratings.csv`: format `user_id,item_id,rating`
- `data/sample/items.csv`: format `item_id,title,description` (untuk content-based)

Testing & CI
- `pytest` digunakan; contoh test ada di `tests/test_models.py`.
- `.github/workflows/ci.yml` menjalankan `pytest` pada push ke branch `main`.

Lisensi
- MIT (lihat file `LICENSE`).

Catatan
- Proyek ini dimaksudkan sebagai starting point yang mudah dimodifikasi. Jika ingin saya buatkan repository GitHub (push otomatis) atau tambahkan notebook demo `.ipynb` dan data yang lebih besar, beri tahu saya.

------------------------------------------------------------
run_project_files.ps1
(PS1 helper — full script included below; copy this block into a file named `run_project_files.ps1` if you want the scaffold written automatically.)
------------------------------------------------------------

```powershell
<#
run_project_files.ps1

Creates a Recommender-System project structure and writes the files provided by the user into the chosen project directory.

Usage:
  pwsh -ExecutionPolicy Bypass -File .\run_project_files.ps1
  pwsh -ExecutionPolicy Bypass -File .\run_project_files.ps1 -ProjectDir "C:\path\to\project" -InitializeGit:$true

Parameters:
  -ProjectDir (string) : target folder where files will be written (default: current directory)
  -InitializeGit (switch) : if supplied, run git init and make an initial commit (if git available)

This script writes:
- README.md
- LICENSE
- .gitignore
- requirements.txt
- pyproject.toml
- src/recommender/__init__.py
- src/recommender/data.py
- src/recommender/models/cf.py
- src/recommender/models/content.py
- src/recommender/models/hybrid.py
- src/recommender/evaluate.py
- src/recommender/utils.py
- scripts/train.py
- scripts/predict.py
- data/sample/ratings.csv
- data/sample/items.csv
- tests/test_models.py
- .github/workflows/ci.yml
#>

param(
  [string] $ProjectDir = (Get-Location).ProviderPath,
  [switch] $InitializeGit
)

Set-StrictMode -Version Latest
$ErrorActionPreference = "Stop"

function Ensure-Dir {
  param([string]$Path)
  if (-not (Test-Path -LiteralPath $Path)) {
    New-Item -ItemType Directory -Path $Path -Force | Out-Null
    Write-Host "Created directory: $Path"
  }
}

function Write-FileSafely {
  param(
    [Parameter(Mandatory=$true)][string] $Path,
    [Parameter(Mandatory=$true)][string] $Content,
    [string] $Encoding = "utf8"
  )
  $dir = Split-Path -Path $Path -Parent
  if ($dir -and -not (Test-Path -LiteralPath $dir)) {
    New-Item -ItemType Directory -Path $dir -Force | Out-Null
  }
  Set-Content -LiteralPath $Path -Value $Content -Encoding $Encoding -Force
  Write-Host "Wrote: $Path"
}

# Prepare directories
$proj = Resolve-Path -LiteralPath $ProjectDir
$root = $proj.Path
Ensure-Dir -Path $root
Ensure-Dir -Path (Join-Path $root "src\recommender\models")
Ensure-Dir -Path (Join-Path $root "scripts")
Ensure-Dir -Path (Join-Path $root "data\sample")
Ensure-Dir -Path (Join-Path $root "tests")
Ensure-Dir -Path (Join-Path $root ".github\workflows")

# Files content (taken from user's provided snippets)
$readme = @'
# Recommender-System — Quick Start (Local repo)

Ringkasan
Repositori ini berisi contoh proyek Recommender-System yang siap dijalankan secara lokal dan disimpan ke Git. Termasuk:
- Implementasi sederhana Collaborative Filtering (UserKNN),
  Content-based (TF-IDF) dan Hybrid aggregator.
'@

$license = @'
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
'@

$gitignore = @'
# Byte-compiled / optimized
__pycache__/
*.py[cod]
*$py.class

# Virtual env
.venv/
venv/
env/

# Jupyter
.ipynb_checkpoints
'@

$requirements = @'
numpy>=1.21
pandas>=1.3
scikit-learn>=1.0
scipy>=1.7
joblib
tqdm
pytest
python-dotenv
'@

$pyproject = @'
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "recommender-system"
'@

$init_py = @'
"""Recommender package"""
__version__ = "0.1.0"
'@

$data_py = @'
import pandas as pd
from typing import Tuple
import numpy as np

def load_ratings(path: str) -> pd.DataFrame:
    """Load ratings CSV with columns: user_id, item_id, rating"""
    df = pd.read_csv(path)
    required = {"user_id", "item_id", "rating"}
    if not required.issubset(set(df.columns)):
        raise ValueError(f"Ratings CSV must contain columns: {required}")
    df = df[['user_id','item_id','rating']].copy()
    return df
'@

$cf_py = @'
import numpy as np
import pandas as pd
from sklearn.neighbors import NearestNeighbors
from typing import List, Tuple, Dict

class UserKNN:
    """Simple user-based KNN on rating matrix using cosine similarity"""

    def __init__(self, n_neighbors: int = 20, metric: str = "cosine"):
        self.n_neighbors = n_neighbors
        self.metric = metric
        self.model = None
        self.user_index = None
        self.item_index = None
        self.ratings_matrix = None

    def fit(self, ratings: pd.DataFrame):
        users = ratings["user_id"].unique()
        items = ratings["item_id"].unique()
        self.user_index = {u:i for i,u in enumerate(users)}
        self.item_index = {j:i for i,j in enumerate(items)}
        matrix = np.zeros((len(users), len(items)), dtype=float)
        for r in ratings.itertuples(index=False):
            matrix[self.user_index[r.user_id], self.item_index[r.item_id]] = r.rating
        self.ratings_matrix = matrix
        k = min(self.n_neighbors, max(1, matrix.shape[0]-1))
        if k >= 1:
            self.model = NearestNeighbors(n_neighbors=k, metric=self.metric).fit(matrix)

    def recommend(self, user_id, k=10):
        if self.user_index is None or user_id not in self.user_index:
            return []
        uid = self.user_index[user_id]
        if self.model is None:
            return []
        distances, neighbors = self.model.kneighbors(self.ratings_matrix[uid].reshape(1,-1), n_neighbors=self.model.n_neighbors)
        scores = np.zeros(self.ratings_matrix.shape[1])
        for nb in neighbors.flatten():
            scores += self.ratings_matrix[nb]
        user_rated = self.ratings_matrix[uid] > 0
        scores[user_rated] = -np.inf
        top_idx = np.argsort(-scores)[:k]
        inv_item_index = {v:k for k,v in self.item_index.items()}
        return [(inv_item_index[i], float(scores[i])) for i in top_idx if scores[i] != -np.inf]
'@

$content_py = @'
from typing import List, Tuple
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import linear_kernel
import numpy as np

class ContentRecommender:
    """Content-based recommender using TF-IDF on item metadata (e.g., title + description)"""

    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=5000, stop_words="english")
        self.item_ids = None
        self.tfidf = None
        self.cosine_sim = None

    def fit(self, items: pd.DataFrame, text_col: str = "text"):
        self.item_ids = items["item_id"].astype(int).tolist()
        corpus = items[text_col].fillna("").astype(str).tolist()
        self.tfidf = self.vectorizer.fit_transform(corpus)
        self.cosine_sim = linear_kernel(self.tfidf, self.tfidf)

    def recommend(self, item_id, k=10) -> List[Tuple[int, float]]:
        if item_id not in self.item_ids:
            return []
        idx = self.item_ids.index(int(item_id))
        sim_scores = list(enumerate(self.cosine_sim[idx]))
        sim_scores = sorted(sim_scores, key=lambda x: x[1], reverse=True)
        return [(int(self.item_ids[i]), float(score)) for i, score in sim_scores[1:k+1]]
'@

$hybrid_py = @'
from typing import List, Tuple, Dict, Any

class HybridRecommender:
    """
    Combine multiple recommenders by weighted score aggregation.
    Recommenders must implement recommend(user_id or item_id, k).
    """

    def __init__(self, recommenders: Dict[str, Any], weights: Dict[str, float] = None):
        self.recommenders = recommenders
        self.weights = weights or {name: 1.0 for name in recommenders.keys()}

    def recommend(self, user_id=None, item_id=None, k: int = 10):
        scores = {}
        for name, rec in self.recommenders.items():
            if not hasattr(rec, "recommend"):
                continue
            w = float(self.weights.get(name, 1.0))
            try:
                if user_id is not None:
                    out = rec.recommend(user_id, k=50)
                elif item_id is not None:
                    out = rec.recommend(item_id, k=50)
                else:
                    continue
                for iid, s in out:
                    scores[iid] = scores.get(iid, 0.0) + w * float(s)
            except Exception:
                continue
        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:k]
        return ranked
'@

$evaluate_py = @'
from typing import List

def precision_at_k(recommended: List[int], actual: List[int], k: int) -> float:
    rec_k = recommended[:k]
    if not rec_k:
        return 0.0
    return len(set(rec_k) & set(actual)) / float(k)

def recall_at_k(recommended: List[int], actual: List[int], k: int) -> float:
    rec_k = recommended[:k]
    if not actual:
        return 0.0
    return len(set(rec_k) & set(actual)) / float(len(set(actual)))
'@

$utils_py = @'
import joblib
from typing import Any
from pathlib import Path

def save_model(obj: Any, path: str):
    p = Path(path)
    p.parent.mkdir(parents=True, exist_ok=True)
    joblib.dump(obj, str(p))

def load_model(path: str) -> Any:
    return joblib.load(path)
'@

$scripts_train = @'
#!/usr/bin/env python3
"""
Train example recommenders and save artifacts.

Usage:
  python scripts/train.py --data data/sample/ratings.csv --items data/sample/items.csv --output models/
"""
import argparse
import os
import pandas as pd
from src.recommender.data import load_ratings
from src.recommender.models.cf import UserKNN
from src.recommender.models.content import ContentRecommender
from src.recommender.utils import save_model

def main(args):
    os.makedirs(args.output, exist_ok=True)
    ratings = load_ratings(args.data)
    print("Loaded ratings:", ratings.shape)

    cf = UserKNN(n_neighbors=20)
    cf.fit(ratings)
    save_model(cf, os.path.join(args.output, "cf_knn.pkl"))
    print("Saved CF model.")

    if args.items:
        items = pd.read_csv(args.items)
        if "text" not in items.columns:
            items["text"] = items.get("title","").fillna("") + " " + items.get("description","").fillna("")
        content = ContentRecommender()
        content.fit(items, text_col="text")
        save_model(content, os.path.join(args.output, "content_tfidf.pkl"))
        print("Saved content model.")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--data", required=True)
    parser.add_argument("--items", required=False)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()
    main(args)
'@

$scripts_predict = @'
#!/usr/bin/env python3
"""
Load a model and produce top-K recommendations.

Usage:
  python scripts/predict.py --model models/cf_knn.pkl --user-id 1 --k 10
"""
import argparse
from src.recommender.utils import load_model

def main(args):
    model = load_model(args.model)
    if args.user_id is not None:
        recs = model.recommend(int(args.user_id), k=args.k)
        print("Recommendations for user", args.user_id)
        for item, score in recs:
            print(item, score)
    elif args.item_id is not None:
        recs = model.recommend(int(args.item_id), k=args.k)
        print("Similar items to", args.item_id)
        for item, score in recs:
            print(item, score)
    else:
        print("Provide --user-id or --item-id")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--model", required=True)
    parser.add_argument("--user-id", type=int)
    parser.add_argument("--item-id", type=int)
    parser.add_argument("--k", type=int, default=10)
    args = parser.parse_args()
    main(args)
'@

$ratings_csv = @'
user_id,item_id,rating
1,10,5
1,11,4
2,10,4
2,12,5
3,11,3
'@

$items_csv = @'
item_id,title,description
10,Apple iPhone,Smartphone by Brand A
11,Samsung Galaxy,Android smartphone family
12,MacBook Pro,Notebook laptop by Brand A
13,Dell XPS,High-end Windows laptop
'@

$tests_py = @'
import pandas as pd
from src.recommender.models.content import ContentRecommender
from src.recommender.models.cf import UserKNN

def test_content_recommender_basic():
    items = pd.DataFrame({
        "item_id": [1,2,3],
        "text": ["apple banana", "banana orange", "apple fruit"]
    })
    rec = ContentRecommender()
    rec.fit(items, text_col="text")
    out = rec.recommend(1, k=2)
    assert len(out) == 2
    assert out[0][0] in [2,3]

def test_userknn_fit_recommend():
    ratings = pd.DataFrame({
        "user_id":[1,1,2,2,3],
        "item_id":[10,11,10,12,11],
        "rating":[5,4,4,5,3]
    })
    cf = UserKNN(n_neighbors=2)
    cf.fit(ratings)
    recs = cf.recommend(1, k=2)
    assert isinstance(recs, list)
'@

$ci_yaml = @'
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run tests
        run: pytest -q
'@

# Write files
Write-FileSafely -Path (Join-Path $root "README.md") -Content $readme
Write-FileSafely -Path (Join-Path $root "LICENSE") -Content $license
Write-FileSafely -Path (Join-Path $root ".gitignore") -Content $gitignore
Write-FileSafely -Path (Join-Path $root "requirements.txt") -Content $requirements
Write-FileSafely -Path (Join-Path $root "pyproject.toml") -Content $pyproject

Write-FileSafely -Path (Join-Path $root "src\recommender\__init__.py") -Content $init_py
Write-FileSafely -Path (Join-Path $root "src\recommender\data.py") -Content $data_py
Write-FileSafely -Path (Join-Path $root "src\recommender\models\cf.py") -Content $cf_py
Write-FileSafely -Path (Join-Path $root "src\recommender\models\content.py") -Content $content_py
Write-FileSafely -Path (Join-Path $root "src\recommender\models\hybrid.py") -Content $hybrid_py
Write-FileSafely -Path (Join-Path $root "src\recommender\evaluate.py") -Content $evaluate_py
Write-FileSafely -Path (Join-Path $root "src\recommender\utils.py") -Content $utils_py

Write-FileSafely -Path (Join-Path $root "scripts\train.py") -Content $scripts_train
Write-FileSafely -Path (Join-Path $root "scripts\predict.py") -Content $scripts_predict

Write-FileSafely -Path (Join-Path $root "data\sample\ratings.csv") -Content $ratings_csv
Write-FileSafely -Path (Join-Path $root "data\sample\items.csv") -Content $items_csv

Write-FileSafely -Path (Join-Path $root "tests\test_models.py") -Content $tests_py
Write-FileSafely -Path (Join-Path $root ".github\workflows\ci.yml") -Content $ci_yaml

# Optionally initialize git and make initial commit
if ($InitializeGit) {
  if (Get-Command git -ErrorAction SilentlyContinue) {
    Push-Location $root
    if (-not (Test-Path -LiteralPath (Join-Path $root ".git"))) {
      git init
      git add --all
      git commit -m "Initial commit: add recommender-system starter files"
      Write-Host "Initialized git repository and performed initial commit."
    } else {
      Write-Host "Git repository already exists at $root"
    }
    Pop-Location
  } else {
    Write-Host "git not found in PATH; skipping git init/commit."
  }
}

Write-Host "`nAll files created under: $root"
Write-Host "Next steps:"
Write-Host "  1) cd into $root"
Write-Host "  2) create and activate a Python venv"
Write-Host "  3) pip install -r requirements.txt"
Write-Host "  4) run: python scripts/train.py --data data/sample/ratings.csv --items data/sample/items.csv --output models/"
Write-Host "  5) run: python scripts/predict.py --model models/cf_knn.pkl --user-id 1 --k 5"
```

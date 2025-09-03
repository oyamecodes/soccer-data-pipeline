# Soccer Data Pipeline — Architecture & Implementation Guide

> Concrete architectural diagram + full documentation for a reproducible, testable, cost-aware pipeline on Google Cloud Platform (free-tier-friendly). No Terraform included.

---

## Table of contents
1. Overview
2. Architecture diagram (Mermaid)
3. Component responsibilities
4. Data formats & schema
5. Ingestion (GitHub Actions -> GCS)
6. Processing (Cloud Run ETL)
7. Storage (BigQuery schemas & partitioning)
8. Modeling (BigQuery ML + Vertex AI path)
9. Evaluation & validation strategy
10. CI/CD, testing & reproducibility
11. Monitoring, logging & cost controls
12. Security & IAM
13. Sample code snippets (GitHub Actions, Cloud Run Dockerfile, Python ETL skeleton)
14. Acceptance criteria checklist
15. Next steps & rollout

---

## 1. Overview
This document defines a concrete architecture to ingest JSON match data stored in a GitHub repository into Google Cloud, normalize it, store it in BigQuery, and train a reproducible model that predicts the season winner for a given league.

Primary goals:
- Use GCP free-tier-friendly services where possible (BigQuery sandbox, Cloud Storage, Cloud Run)
- Keep pipeline batch-oriented (daily/weekly) and cost-aware
- Provide reproducible training and CI/CD to rerun end-to-end jobs on updates


## 2. Architecture diagram (Mermaid)

```mermaid
flowchart LR
  subgraph SOURCE
    GH[GitHub repo (JSON files)]
  end

  subgraph INGEST
    GHAction[GitHub Action]
    GCS[GCS Bucket (raw/ingest/)]
  end

  subgraph PROCESS
    CR[Cloud Run ETL]
    BQ[BigQuery (staging -> normalized)]
  end

  subgraph MODEL
    BQML[BigQuery ML]
    VA[Vertex AI (optional)]
    MODEL_REG[Model artifact / metadata]
  end

  subgraph CI_CD
    GB[GitHub Actions (CI) / Cloud Build]
  end

  subgraph MON
    LOGS[Cloud Logging]
    ALERTS[Budget Alerts]
  end

  GH -->|push| GHAction -->|upload JSON| GCS
  GCS -->|pull / trigger| CR -->|write| BQ
  BQ -->|train/query| BQML
  BQML --> MODEL_REG
  MODEL_REG -->|optional| VA
  CR --> LOGS
  BQ --> LOGS
  GB --> CR
  GB --> BQML
  LOGS --> ALERTS
```

> The mermaid diagram above maps the main flow: GitHub pushes JSON -> GitHub Action uploads to GCS -> Cloud Run ETL normalizes and writes to BigQuery -> BigQuery ML trains model. CI/CD (GitHub Actions) orchestrates tests and retraining, and Cloud Logging / Budget Alerts provide monitoring.


## 3. Component responsibilities
- **GitHub repo**: Holds JSON files per league-season, schema docs, ETL and model code, CI YAMLs. Use a directory layout like `/data/<league>/<season>.json`, `/etl/`, `/models/`, `/docs/`.

- **GitHub Action (ingest)**: On `push` to `main` (or a `data/` path), upload changed/added JSON files to a GCS bucket (`gs://soccer-ingest-raw`). Optionally include metadata (uploader, commit sha).

- **GCS (raw)**: Immutable raw JSON objects stored by path `raw/<league>/<season>/<file>.json`. Use a lifecycle policy to purge > 1 year if cost is a concern.

- **Cloud Run ETL**: Containerized Python service triggered manually, by schedule (Cloud Scheduler -> Pub/Sub -> Cloud Run), or by GitHub Actions. It:
  1. Reads JSON files from GCS.
  2. Validates & normalizes records (flatten scores, fill missing fields, parse dates).
  3. Writes to a staging BigQuery table (append) and then to the normalized production table (partitioned).

- **BigQuery**:
  - `dataset: soccer_data`
  - `matches_raw` (optional) to store JSON as-is (for auditing)
  - `matches_normalized` (partitioned by `season` or `date`) for feature engineering and analytics
  - `season_results` (yearly aggregated table) with final standings and champion labels

- **BigQuery ML**:
  - Train baseline and production models using SQL or export features to Vertex AI for custom training.

- **CI/CD** (GitHub Actions or Cloud Build): Run unit tests (ETL), integration tests (dry-run ETL against a small in-memory or test GCS bucket), and retrain model on `push` to `main` (or on schedule).

- **Monitoring & Alerts**: Cloud Logging, Cloud Monitoring (for Cloud Run/BigQuery), and Budget Alerts for cost control.


## 4. Data formats & schema
### 4.1 Input JSON (sample)
You provided an example JSON with `name` and `matches` array. Key variations to handle:
- Matches can have only `ft` (full-time) or both `ht` and `ft`
- Some fields (time) may be missing or inconsistent
- Team names can vary between seasons (rename mapping needed)

### 4.2 Normalized `matches_normalized` schema (recommended)

| column | type | notes |
|---|---:|---|
| league | STRING | e.g., "Österr. Bundesliga" |
| season | STRING | e.g., "2010/11" |
| round | STRING | e.g., "Matchday 1" |
| match_date | DATE | parsed from date + timezone handling |
| match_time | STRING | keep raw if timezone unknown |
| team_home | STRING | standardized team name |
| team_away | STRING | standardized team name |
| ht_home | INT64 | nullable |
| ht_away | INT64 | nullable |
| ft_home | INT64 | nullable |
| ft_away | INT64 | nullable |
| winner | STRING | derived: "home"/"away"/"draw" |
| points_home | INT64 | derived (3/1/0) |
| points_away | INT64 | derived |
| schema_version | STRING | e.g., "v1" |
| source_commit | STRING | Git SHA that produced record |
| ingestion_ts | TIMESTAMP | ETL run timestamp |

Partitioning: `PARTITION BY RANGE_BUCKET` on season or `PARTITION BY DATE(match_date)` depending on query patterns. For season-level analytics, partitioning by season (string) is okay; BigQuery recommends date partitions for time-based pruning — consider `season_start_date` as partition column.

Clustering: `CLUSTER BY league, team_home, team_away` to reduce scan costs when filtering by league/team.


## 5. Ingestion — GitHub Actions -> GCS (concrete)
**Trigger**: `on: push` (paths: `data/**`)

**Behavior**:
- For every changed `.json` file under `data/`, upload to `gs://soccer-ingest-raw/<league>/<season>/<filename>.json` using `google-github-actions/upload-cloud-storage@v0` or `gsutil` in the action.
- Optionally set metadata `x-goog-meta-commit-sha`.

**Idempotence**: Overwrite using content-hash file name or commit-sha in path.


## 6. Processing — Cloud Run ETL
**Why Cloud Run?** Serverless (free-tier requests), containerized, easy to test locally.

**ETL flow per run**:
1. List files in GCS path (optionally filter by prefix/commit).
2. For each file:
   - Load JSON into Python dicts.
   - Validate structure against `schema.json` (use `jsonschema`).
   - Normalize into row-level match records.
   - Standardize team names (map aliases to canonical names via a lookup table in repo or BigQuery table `team_aliases`).
   - Write to `matches_raw` (optional) and append to `matches_normalized`.
3. Commit a summary report (counts, errors) to Cloud Logging and return non-zero exit on fatal errors.

**Idempotency**: Use `source_commit` and `file_path` columns — skip if record for that file+sha already exists.

**Scaling**: Cloud Run concurrency can be tuned; for many files the GitHub Action could call Cloud Run per-file in parallel (watch cost).


## 7. Storage — BigQuery schemas & partitioning
### `matches_normalized` (DDL example)

```sql
CREATE TABLE IF NOT EXISTS `project.soccer_data.matches_normalized` (
  league STRING,
  season STRING,
  round STRING,
  match_date DATE,
  match_time STRING,
  team_home STRING,
  team_away STRING,
  ht_home INT64,
  ht_away INT64,
  ft_home INT64,
  ft_away INT64,
  winner STRING,
  points_home INT64,
  points_away INT64,
  schema_version STRING,
  source_commit STRING,
  ingestion_ts TIMESTAMP
)
PARTITION BY DATE(match_date)
CLUSTER BY league, team_home, team_away;
```

### `season_results` (aggregated)
Contains per-season final standings and label `champion_team`.
Columns: `league, season, team, points, wins, draws, losses, goals_for, goals_against, position, champion BOOL`.

Populate via `SQL` aggregation from `matches_normalized`.


## 8. Modeling
Start with **BigQuery ML** for reproducibility and minimal infra. Example approaches:

### 8.1 Problem framing
- **Target**: For each season, predict `champion_team` among teams in that season.
- Two modeling styles:
  1. **Per-team classification**: For each (season, team) row, target = 1 if team was champion else 0. Then train a binary classifier producing a probability per team; pick argmax per season.
  2. **Ranking / multiclass**: Train a multiclass model where each class is a canonical team label (works if teams are stable across seasons, otherwise re-index per season).

Recommendation: Start with **per-team binary classification** using `CREATE MODEL` with `BOOSTED_TREE_CLASSIFIER` in BigQuery ML.

### 8.2 Features (examples)
- Historical: previous-season final position, previous-season points
- Rolling team-level features within season up to a cutoff (e.g., after 10 matchdays): goals_for_avg, goals_against_avg, win_pct, clean_sheet_pct
- Squad/transfer features (optional): big signings, manager change flags
- External: ELO rating per team

Feature engineering can be done in SQL (window functions) in BigQuery.

### 8.3 Training SQL (sketch)
```sql
CREATE OR REPLACE MODEL `project.soccer_data.models.champion_predictor_v1`
OPTIONS(
  model_type='BOOSTED_TREE_CLASSIFIER',
  input_label_cols=['is_champion']
) AS
SELECT
  team,
  season,
  avg_goals_for_rolling_10,
  avg_goals_against_rolling_10,
  win_rate_rolling_10,
  previous_season_points,
  is_champion
FROM
  `project.soccer_data.team_features_training_table`
WHERE
  season < '2019/20';
```

### 8.4 Evaluation
- Use `ML.EVALUATE` and `ML.ROC` on holdout seasons.
- Top-1 accuracy: for each season, pick team with highest predicted probability and compare to actual champion.
- Top-3 accuracy: check if actual champion is within top 3 predicted teams.


## 9. Evaluation & validation strategy
- **Temporal holdout**: Train on seasons up to `T-2`, validate on `T-1`, test on `T`.
- **Cross-season CV**: Rolling origin evaluation (walk-forward validation) across seasons.
- **Baselines**:
  - Last season's champion (repeat-champion baseline).
  - Simple heuristic: team with highest previous-season points.

- **Metrics**: accuracy, top-3 accuracy, log-loss, calibration (reliability curve).


## 10. CI/CD, testing & reproducibility
- **Repository structure** (suggested)
```
/data/               # JSON files
/etl/
  Dockerfile
  main.py
  requirements.txt
/models/
  bqml_sql/
/docs/
  soccer-data-pipeline-architecture.md
/tests/
  test_etl.py
.gitignore
.github/workflows/ingest.yml
.github/workflows/ci.yml
```

- **CI workflows**:
  - `ci.yml`: run unit tests (python -m pytest), lint, security checks
  - `ingest.yml`: on `data/**` push: upload to GCS and optionally call Cloud Run ETL
  - `retrain.yml` (scheduled): run training job via `bq` or `gcloud` commands

- **Reproducibility**:
  - Pin Python deps with `requirements.txt`
  - Use `bq` and `gcloud` CLIs in the same versions in CI images
  - Keep model SQL in repo and use `CREATE OR REPLACE MODEL` steps in CI to reproduce


## 11. Monitoring, logging & cost controls
- **Logging**: Cloud Run and BigQuery logs to Cloud Logging. Log ETL summary (rows processed, errors).
- **Monitoring**: Create alerts for ETL failures (error rate > 0) and long-running jobs.
- **Cost control**:
  - BigQuery: limit query sizes in CI (use `--maximum_bytes_billed` flag)
  - Set GCP Budget & Alerts: threshold at e.g., $5, $10 during prototyping
  - Use BigQuery sandbox and keep queries under 1 TB/month
  - Use partitioning/clustering to limit scanned data


## 12. Security & IAM
- Principle of least privilege: give GitHub Action service account permissions to only write to the specific GCS bucket and run Cloud Run invocation if needed.
- BigQuery: grant dataset-level roles only to the CI service account or owner.
- Secrets: store service account keys in GitHub Secrets or better use Workload Identity Federation (recommended long-term) to avoid long-lived keys.


## 13. Sample code snippets
### 13.1 GitHub Actions (ingest.yml) — snippet
```yaml
name: Ingest JSON to GCS
on:
  push:
    paths:
      - 'data/**'

jobs:
  upload:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Upload changed JSON to GCS
        uses: google-github-actions/upload-cloud-storage@v1
        with:
          path: 'data/**.json'
          destination: 'gs://soccer-ingest-raw/'
        env:
          GCP_PROJECT: ${{ secrets.GCP_PROJECT }}
          GOOGLE_APPLICATION_CREDENTIALS: ${{ secrets.GCP_SA_KEY }}
```

### 13.2 Cloud Run Dockerfile (skeleton)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "main:app", "-b", "0.0.0.0:8080"]
```

### 13.3 Python ETL skeleton (`etl/main.py`)
```python
from flask import Flask, request
from google.cloud import storage, bigquery
import json
import os
import datetime

app = Flask(__name__)

bq = bigquery.Client()
storage_client = storage.Client()

DATASET = os.environ.get('BQ_DATASET', 'soccer_data')
TABLE = os.environ.get('BQ_TABLE', 'matches_normalized')

@app.route('/run_etl', methods=['POST'])
def run_etl():
    bucket_name = os.environ.get('INGEST_BUCKET')
    prefix = request.json.get('prefix')
    bucket = storage_client.bucket(bucket_name)
    blobs = bucket.list_blobs(prefix=prefix)
    rows_to_insert = []
    for blob in blobs:
        content = blob.download_as_text()
        obj = json.loads(content)
        # parse and normalize
        # for example, iterate obj['matches'] and produce row dicts
        # rows_to_insert.append(row)

    # write to BigQuery
    table_ref = bq.dataset(DATASET).table(TABLE)
    errors = bq.insert_rows_json(table_ref, rows_to_insert)
    if errors:
        return {"status": "error", "errors": errors}, 500
    return {"status": "ok", "rows": len(rows_to_insert)}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### 13.4 BigQuery: example aggregation to `season_results`
```sql
CREATE OR REPLACE TABLE `project.soccer_data.season_results` AS
SELECT
  league,
  season,
  team,
  SUM(points) as points,
  SUM(CASE WHEN winner = 'home' AND team = team_home THEN 1 WHEN winner = 'away' AND team = team_away THEN 1 ELSE 0 END) as wins,
  SUM(CASE WHEN winner = 'draw' THEN 1 ELSE 0 END) as draws,
  SUM(CASE WHEN (winner = 'home' AND team <> team_home) OR (winner = 'away' AND team <> team_away) THEN 1 ELSE 0 END) as losses,
  SUM(ft_home) as goals_for, -- careful: this needs conditional aggregation per team
  -- ... more aggregations ...
FROM `project.soccer_data.matches_normalized`
GROUP BY league, season, team;
```

(Notes: the above SQL is a sketch — implement per-team conditional aggregation correctly.)


## 14. Acceptance criteria checklist
- [ ] Automated ingestion from GitHub to GCS (GH Action created)
- [ ] ETL job (Cloud Run) that normalizes JSON and writes to BigQuery
- [ ] BigQuery `matches_normalized` table partitioned and clustered
- [ ] `season_results` table with champion label
- [ ] Reproducible model (BigQuery ML) training step + evaluation
- [ ] CI workflow to run ETL unit tests and retrain model
- [ ] Monitoring (Cloud Logging) and Budget Alerts


## 15. Next steps & rollout
1. Create GCP project and enable APIs: Cloud Storage, BigQuery, Cloud Run, Cloud Build, Cloud Logging.
2. Provision a service account with minimal permissions and add its key to GitHub Secrets (or use Workload Identity Federation).
3. Add GitHub Action `ingest.yml` and push a sample `data/` JSON file.
4. Deploy Cloud Run ETL and trigger a run from the GitHub Action or Cloud Scheduler.
5. Run BigQuery aggregation SQL to populate `season_results` and verify champion labels.
6. Create a BQ ML model (train) and run `ML.EVALUATE` on holdout seasons.

---

## Appendix — tips & gotchas
- Team names: keep a canonical mapping table — many datasets use different naming conventions.
- Date/timezones: input dates may lack timezone — store as DATE when time is unclear.
- Missing HT or FT: prefer FT for final outcome; HT only for mid-match features.
- If dataset grows: consider Dataflow/Apache Beam for large-scale transforms.


---

*Document created by ChatGPT — ask to iterate on any section, add more code snippets, or convert any SQL sketch into runnable SQL.*


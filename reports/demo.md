# Demo

## Contexte

Modele de scoring credit entraine et versionne dans MLflow, expose via FastAPI.
Chaque prediction peut etre journalisee dans PostgreSQL.
La stack locale couvre aussi Streamlit, Prometheus, Grafana, Evidently et quelques benchmarks de performance.

## Architecture

```text
MLflow Model Registry
        ->
FastAPI API
        ->
PostgreSQL prediction logs
        ->
Monitoring:
  - Streamlit
  - Prometheus / Grafana
  - Evidently
        ->
Performance:
  - cProfile
  - batching
  - ONNX Runtime PoC
```

## Stack locale

```powershell
docker compose down -v
docker compose up --build -d
Start-Sleep -Seconds 30
docker compose ps
```

Services attendus :

```text
credexp_api
credexp_db
credexp_streamlit
credexp_prometheus
credexp_grafana
```

Reperes :

- Docker Compose orchestre toute la stack locale.
- L'API charge le modele au demarrage.
- PostgreSQL stocke les logs de prediction.
- Prometheus scrape `/metrics`.
- Grafana affiche les metriques techniques.

## Verifs rapides

```powershell
uv run ruff check .
uv run ruff format --check .
uv run pytest -q
```

## MLflow local

```powershell
uv run mlflow ui --backend-store-uri sqlite:///mlflow/mlflow.db --default-artifact-root ./mlflow/artifacts
```

URL :

```text
http://127.0.0.1:5000
```

Reperes :

- modele cible dans le registry : `credit_scoring_model`
- version de demo mentionnee dans la doc : `Version 2 / Production`

## API FastAPI

URL :

```text
http://127.0.0.1:8000/docs
```

Endpoints :

```text
GET  /health
GET  /model-info
POST /predict
POST /predict_batch
GET  /metrics
```

Commande :

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8000/health
```

Payload type :

```json
{
  "sk_id_curr": 123456,
  "features": {
    "EXT_SOURCE_1": 0.52,
    "EXT_SOURCE_2": 0.71,
    "EXT_SOURCE_3": 0.41
  }
}
```

Prediction manuelle :

```powershell
$body = @{
  sk_id_curr = 123456
  features = @{
    EXT_SOURCE_1 = 0.52
    EXT_SOURCE_2 = 0.71
    EXT_SOURCE_3 = 0.41
  }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:8000/predict" `
  -ContentType "application/json" `
  -Body $body
```

Lecture brute :

```text
proba_default
decision
threshold
model_name
model_version
latency_ms
```

## PostgreSQL

Initialisation :

```powershell
$env:DATABASE_URL="postgresql+psycopg://postgres:postgres@localhost:5432/credexp"
uv run python scripts/init_db.py
```

Verif table :

```powershell
docker exec -it credexp_db psql -U postgres -d credexp -c "\dt"
```

Verif donnees :

```powershell
docker exec -it credexp_db psql -U postgres -d credexp -c "SELECT created_at, sk_id_curr, model_version, proba_default, decision, latency_ms FROM predictions ORDER BY created_at DESC LIMIT 10;"
```

Reperes :

- table cle : `predictions`
- journalisation des inputs, outputs, version du modele et latence

## Jeu de donnees de demo

```powershell
1..50 | ForEach-Object { .\api_examples\test_api.ps1 }
docker exec -it credexp_db psql -U postgres -d credexp -c "SELECT COUNT(*) FROM predictions;"
```

## Streamlit

URL :

```text
http://127.0.0.1:8501
```

Pages :

- `Scoring Client`
- `Monitoring Dev`

Elements utiles :

- payload JSON editable
- score et decision
- latence
- dernieres predictions
- distributions scores / decisions

## Prometheus

URL :

```text
http://127.0.0.1:9090/targets
```

Repere :

```text
credexp_api    UP
```

PromQL utiles :

```promql
http_requests_total
```

```promql
sum(rate(http_requests_total[1m]))
```

## Grafana

URL :

```text
http://127.0.0.1:3000
```

Identifiants :

```text
admin / admin
```

Datasource :

```text
http://prometheus:9090
```

Dashboard :

```text
Credit Scoring API Monitoring
```

Points de lecture :

- debit requetes
- statuts HTTP
- latence moyenne
- total des requetes

## Drift Evidently

Commande :

```powershell
$env:DATABASE_URL="postgresql+psycopg://postgres:postgres@localhost:5432/credexp"
uv run python scripts/monitoring_drift.py --limit 500
```

Sorties :

```text
reports/monitoring/evidently_drift.html
reports/monitoring/evidently_drift_meta.json
notebooks/06_drift_monitoring.ipynb
```

Lecture brute :

- comparaison reference `data/processed/reference.parquet` vs donnees de production simulee
- interpretation indicative si le volume est faible

## Benchmarks / perf

```powershell
uv run python scripts/profile_inference.py
uv run python scripts/benchmark_api.py
uv run python scripts/benchmark_batching.py
uv run python scripts/benchmark_onnx.py
```

Artifacts :

```text
reports/performance/cprofile_inference_top20.txt
reports/performance/inference_benchmark.json
reports/performance/api_benchmark.json
reports/performance/batching_benchmark.json
reports/performance/onnx_benchmark.json
```

Reperes :

- batching = principal gain pratique
- ONNX Runtime = PoC d'optimisation

## Git / historique

```powershell
git status
git branch --show-current
git log --oneline --decorate --graph -15
```

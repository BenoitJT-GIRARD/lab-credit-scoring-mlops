# Demo notes - credit scoring service

## Fil rouge

- objectif : montrer la chaine complete de serving et de suivi autour du modele
- ordre naturel : repo -> stack -> API -> stockage -> monitoring -> drift -> perf -> CI/CD
- deux environnements utiles : local multi-services, remote public plus simple a partager

## Ce qui compte dans le repo

- artefacts de serving figes dans `artifacts/models/`
- code d'API dans `src/credexp/serving/`
- stockage / logging dans `src/credexp/db/`
- monitoring drift dans `src/credexp/monitoring/`
- UI dans `streamlit_app/`
- orchestration locale dans `docker-compose.yml`
- deploy remote dans `deploy/huggingface/`
- automatisation dans `.github/workflows/`

Commandes :

```powershell
git branch --show-current
git log --oneline --decorate --graph -15
```

Captures utiles :

```text
reports/screenshots/01_github_history.png
reports/screenshots/02_github_actions_success.png
reports/screenshots/17_github_action_deploy_hf_success.png
```

## Architecture retenue

Local :

```text
Docker Compose
  |- FastAPI
  |- PostgreSQL
  |- Streamlit
  |- Prometheus
  `- Grafana
```

Remote :

```text
Hugging Face Docker Space
  |- Nginx : 7860
  |- Streamlit : /
  |- FastAPI : /api
  `- Supabase PostgreSQL
```

Choix retenus :

- stack locale complete pour valider les integrations
- stack remote volontairement compacte pour un deploy simple
- MLflow garde la partie train / registry
- le deploy embarque un export joblib autonome

## Artefacts de serving

```text
artifacts/models/pipeline.joblib
artifacts/models/threshold.json
artifacts/models/feature_columns.json
```

Pourquoi ce format :

- pas de dependance runtime a MLflow
- image Docker reproductible
- chargement simple au startup
- ordre des features fige explicitement

Limite assumee :

- duplication entre registry et export de serving
- acceptable ici car l'objectif principal est la robustesse de deploiement

## Lancement local

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

Point de controle :

- la stack donne tout le circuit en local, y compris le stockage et le monitoring

## API

Docs :

```text
http://127.0.0.1:8000/docs
https://bijeytis-prjperso-credexp.hf.space/api/docs
```

Routes a verifier :

```text
GET  /health
GET  /model-info
POST /predict
POST /predict_batch
GET  /metrics
```

Commandes :

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8000/health
```

```powershell
$body = @{
  sk_id_curr = 123456
  features = @{
    EXT_SOURCE_1 = 0.52
    EXT_SOURCE_2 = 0.71
    EXT_SOURCE_3 = 0.41
  }
} | ConvertTo-Json -Depth 5
```

```powershell
Invoke-RestMethod -Method Post -Uri "http://127.0.0.1:8000/predict" -ContentType "application/json" -Body $body
Invoke-RestMethod -Method Post -Uri "https://bijeytis-prjperso-credexp.hf.space/api/predict" -ContentType "application/json" -Body $body
```

Notes :

- modele charge une seule fois au startup
- validation Pydantic stricte
- endpoint batch garde du sens pour la perf
- logging base en best effort pour ne pas bloquer le scoring

Captures :

```text
reports/screenshots/03_fastapi_docs.png
reports/screenshots/04_fastapi_predict_response.png
reports/screenshots/16_huggingface_space_api_docs.png
```

## Stockage des predictions

Initialisation locale :

```powershell
$env:DATABASE_URL="postgresql+psycopg://postgres:postgres@localhost:5432/credexp"
uv run python scripts/init_db.py
```

Verification :

```powershell
docker exec -it credexp_db psql -U postgres -d credexp -c "\dt"
docker exec -it credexp_db psql -U postgres -d credexp -c "SELECT created_at, sk_id_curr, model_version, proba_default, decision, latency_ms FROM predictions ORDER BY created_at DESC LIMIT 10;"
```

Trafic de test :

```powershell
1..50 | ForEach-Object { .\api_examples\test_api.ps1 }
```

Pourquoi stocker autant :

- audit d'une prediction
- base pour le drift
- lecture rapide des latences et decisions
- meme schema local / remote

Limite assumee :

- payload complet en base = pratique pour un PoC, a durcir ensuite selon volumetrie et RGPD

Captures :

```text
reports/screenshots/07_postgres_predictions.png
reports/screenshots/14_supabase_predictions.png
reports/screenshots/19_supabase_prediction_from_hf.png
```

## Interface Streamlit

URLs :

```text
http://127.0.0.1:8501
https://bijeytis-prjperso-credexp.hf.space
```

Pages :

- `Scoring Client`
- `Monitoring Dev`

Utilite reelle :

- point d'entree simple pour tester le scoring
- vue legere sur les predictions recemment stockees
- lecture plus metier que Grafana

Captures :

```text
reports/screenshots/05_streamlit_scoring.png
reports/screenshots/06_streamlit_monitoring.png
reports/screenshots/15_huggingface_space_streamlit.png
```

## Monitoring technique

Prometheus :

```text
http://127.0.0.1:9090/targets
```

PromQL :

```promql
http_requests_total
sum(rate(http_requests_total[1m]))
```

Grafana :

```text
http://127.0.0.1:3000
admin / admin
```

Lecture retenue :

- trafic
- statuts HTTP
- latence moyenne

Pourquoi garder Grafana en plus de Streamlit :

- Streamlit = lecture fonctionnelle du service
- Grafana = lecture exploitation / systeme

Captures :

```text
reports/screenshots/08_prometheus_target_up.png
reports/screenshots/09_grafana_dashboard.png.png
```

## Drift

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

Lecture :

- reference = donnees de dev stabilisees
- current = fenetre extraite des predictions stockees
- interpretation surtout qualitative si faible volume

Limite assumee :

- sans labels recents, on suit surtout la derive de donnees et les signaux techniques

Capture :

```text
reports/screenshots/10_evidently_drift_report.png
```

## Performance

Commandes :

```powershell
uv run python scripts/profile_inference.py
uv run python scripts/benchmark_api.py
uv run python scripts/benchmark_batching.py
uv run python scripts/benchmark_onnx.py
```

Sorties a ouvrir :

```text
reports/performance/cprofile_inference_top20.txt
reports/performance/inference_benchmark.json
reports/performance/api_benchmark.json
reports/performance/batching_benchmark.json
reports/performance/onnx_benchmark.json
notebooks/07_performance_optimization.ipynb
```

Constats :

- le batching donne le gain le plus concret
- ONNX valide une piste, mais ne remplace pas a lui seul le preprocessing Python
- le choix final reste le pipeline sklearn complet pour garder le comportement stable

Captures :

```text
reports/screenshots/12_performance_notebook.png
reports/screenshots/13_onnx_benchmark_json.png
```

## MLflow

```powershell
uv run mlflow ui --backend-store-uri sqlite:///mlflow/mlflow.db --default-artifact-root ./mlflow/artifacts
```

```text
http://127.0.0.1:5000
```

Ce que je veux retrouver rapidement :

- runs d'entrainement
- comparaison des modeles
- modele `credit_scoring_model`
- version retenue avant export de serving

Capture :

```text
reports/screenshots/11_mlflow_registry_model_v2.png
```

## Qualite et automatisation

```powershell
uv run ruff check .
uv run ruff format --check .
uv run pytest -q
```

```text
reports/coverage/coverage.xml
reports/coverage/html/
.github/workflows/ci.yml
.github/workflows/deploy_huggingface.yml
```

Notes :

- couverture volontairement moderee mais explicite
- cible prioritaire : code de prod et logique critique
- le workflow CI reste simple et lisible
- le deploy remote repasse par les controles avant upload

## Derniers checks utiles

- verifier que le Space repond bien sur `/api/health`
- garder un payload JSON deja pret
- garder les captures ouvertes en secours
- si le remote ralentit, basculer sur la demo locale
- eviter d'exposer les secrets ou valeurs sensibles a l'ecran

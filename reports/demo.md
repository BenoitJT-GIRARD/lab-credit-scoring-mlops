# Demo - Credit Scoring MLOps Project

Ce document sert de guide de démonstration pour la soutenance.

Objectif : montrer rapidement que le projet couvre l'API de scoring, Docker, CI/CD, stockage des prédictions, monitoring technique, drift, optimisation de performance, déploiement distant Hugging Face et stockage distant Supabase.

## 1. Message d'ouverture

Phrase de contexte :

```text
Le projet simule la mise en production d'un modèle de scoring crédit pour l'entreprise fictive Prêt à Dépenser. Le modèle LightGBM a été entraîné, évalué et versionné en Partie 1. En Partie 2, je l'ai intégré dans une stack de déploiement complète avec API, base de données, monitoring, CI/CD et déploiement distant.
```

Phrase d'architecture :

```text
J'ai deux déploiements complémentaires. En local, Docker Compose lance la stack complète avec FastAPI, PostgreSQL, Streamlit, Prometheus et Grafana. En remote, Hugging Face Spaces lance un conteneur unique avec Nginx, Streamlit et FastAPI. Le modèle est embarqué sous forme d'artefact joblib versionné, et les prédictions sont stockées dans Supabase via une variable secrète DATABASE_URL.
```

## 2. Architecture à présenter

### Local

```text
Docker Compose
  ├── FastAPI
  ├── PostgreSQL
  ├── Streamlit
  ├── Prometheus
  └── Grafana
```

### Remote

```text
Hugging Face Docker Space
  ├── Nginx : 7860
  ├── Streamlit : /
  ├── FastAPI : /api
  ├── modèle joblib embarqué
  └── Supabase PostgreSQL
```

### Artefacts modèle

```text
artifacts/models/
  pipeline.joblib
  threshold.json
  feature_columns.json
```

Phrase :

```text
MLflow reste utilisé pour le tracking et le registry pendant l'entraînement. Pour le déploiement, j'embarque les artefacts minimaux nécessaires afin d'obtenir un conteneur autonome et reproductible.
```

## 3. Git et GitHub

Commandes locales :

```powershell
git status
git branch --show-current
git log --oneline --decorate --graph -15
```

À montrer :

- branche `develop` ;
- commits explicites ;
- dépôt public GitHub ;
- historique de versions.

Screenshot associé :

```text
reports/screenshots/01_github_history.png
```

## 4. GitHub Actions

Page GitHub :

```text
Repository -> Actions
```

À montrer :

- workflow `ci` ;
- workflow `deploy-huggingface` ;
- jobs verts ;
- tests ;
- coverage ;
- build Docker ;
- déploiement Hugging Face.

Screenshots associés :

```text
reports/screenshots/02_github_actions_success.png
reports/screenshots/17_github_action_deploy_hf_success.png
```

Phrase :

```text
Le workflow CI vérifie le lint, le format, les tests avec coverage et le build Docker. Le workflow de déploiement Hugging Face relance les contrôles qualité, construit l'image du Space et déploie uniquement si tout passe.
```

## 5. Stack locale Docker Compose

Commande :

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

Phrase :

```text
Docker Compose orchestre la stack locale complète. C'est le mode de démonstration le plus proche d'une architecture production-like multi-services.
```

## 6. API FastAPI locale

Swagger local :

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

Healthcheck :

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

Prédiction PowerShell :

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

Screenshots associés :

```text
reports/screenshots/03_fastapi_docs.png
reports/screenshots/04_fastapi_predict_response.png
```

Phrase :

```text
Le modèle est chargé une seule fois au démarrage de l'API et réutilisé pour toutes les requêtes. Cela évite de recharger le modèle à chaque appel et réduit la latence.
```

## 7. API Hugging Face remote

URL du Space :

```text
https://bijeytis-prjperso-credexp.hf.space
```

Swagger distant :

```text
https://bijeytis-prjperso-credexp.hf.space/api/docs
```

Healthcheck distant :

```text
https://bijeytis-prjperso-credexp.hf.space/api/health
```

Routes :

```text
/              -> Streamlit
/api/health    -> FastAPI healthcheck
/api/docs      -> Swagger API
/api/predict   -> prédiction
/api/metrics   -> métriques Prometheus exposées par FastAPI
```

À montrer :

1. ouvrir le Space ;
2. ouvrir `/api/docs` ;
3. tester `/api/health` ;
4. tester `/api/predict` ;
5. vérifier que la prédiction est stockée dans Supabase.

Screenshots associés :

```text
reports/screenshots/15_huggingface_space_streamlit.png
reports/screenshots/16_huggingface_space_api_docs.png
```

Phrase :

```text
Le Space Hugging Face utilise un seul conteneur. Nginx route la racine vers Streamlit et le préfixe /api vers FastAPI.
```

## 8. PostgreSQL local

Initialisation :

```powershell
$env:DATABASE_URL="postgresql+psycopg://postgres:postgres@localhost:5432/credexp"
uv run python scripts/init_db.py
```

Vérification table :

```powershell
docker exec -it credexp_db psql -U postgres -d credexp -c "\dt"
```

Vérification données :

```powershell
docker exec -it credexp_db psql -U postgres -d credexp -c "SELECT created_at, sk_id_curr, model_version, proba_default, decision, latency_ms FROM predictions ORDER BY created_at DESC LIMIT 10;"
```

Générer plusieurs prédictions :

```powershell
1..50 | ForEach-Object { .\api_examples\test_api.ps1 }
```

Screenshot associé :

```text
reports/screenshots/07_postgres_predictions.png
```

Phrase :

```text
Chaque prédiction est stockée avec l'entrée, la sortie, la version du modèle, le score, la décision et la latence. Cela permet l'audit, le monitoring et le drift.
```

## 9. Supabase remote

Supabase est utilisé comme PostgreSQL distant.

Secret utilisé :

```text
DATABASE_URL
```

Format :

```text
postgresql+psycopg://USER:PASSWORD@HOST:PORT/postgres?sslmode=require
```

À montrer :

- Supabase ;
- Table Editor ;
- table `predictions` ;
- ligne générée par le Space Hugging Face.

Screenshots associés :

```text
reports/screenshots/14_supabase_predictions.png
reports/screenshots/19_supabase_prediction_from_hf.png
```

Phrase :

```text
Le code applicatif reste le même entre PostgreSQL local et Supabase. Seule la variable DATABASE_URL change.
```

## 10. Streamlit

URL locale :

```text
http://127.0.0.1:8501
```

URL remote :

```text
https://bijeytis-prjperso-credexp.hf.space
```

Pages :

- `Scoring Client`
- `Monitoring Dev`

À montrer :

- payload JSON éditable ;
- score ;
- décision ;
- latence ;
- dernières prédictions ;
- distribution des scores ;
- distribution des décisions.

Screenshots associés :

```text
reports/screenshots/05_streamlit_scoring.png
reports/screenshots/06_streamlit_monitoring.png
reports/screenshots/15_huggingface_space_streamlit.png
```

## 11. Prometheus

URL :

```text
http://127.0.0.1:9090/targets
```

Repère :

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

Screenshot associé :

```text
reports/screenshots/08_prometheus_target_up.png
```

Phrase :

```text
Prometheus scrape les métriques exposées par FastAPI sur /metrics. Cela permet de suivre l'activité technique de l'API.
```

## 12. Grafana

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

- débit de requêtes ;
- statuts HTTP ;
- latence moyenne ;
- total des requêtes.

Screenshot associé :

```text
reports/screenshots/09_grafana_dashboard.png
```

Phrase :

```text
Grafana permet une lecture Dev/Ops des métriques techniques. Streamlit et Grafana ne visent pas le même public : Streamlit est orienté métier, Grafana est orienté exploitation.
```

## 13. Drift Evidently

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

Screenshot associé :

```text
reports/screenshots/10_evidently_drift_report.png
```

Phrase :

```text
Le drift compare une référence issue des données de développement à une fenêtre de données de production. Ici, le flux de production est simulé, donc l'interprétation est qualitative, mais l'architecture est celle d'un monitoring réel.
```

## 14. MLflow

Commande locale :

```powershell
uv run mlflow ui --backend-store-uri sqlite:///mlflow/mlflow.db --default-artifact-root ./mlflow/artifacts
```

URL :

```text
http://127.0.0.1:5000
```

À montrer :

- expérience ;
- runs ;
- modèle `credit_scoring_model` ;
- version retenue.

Screenshot associé :

```text
reports/screenshots/11_mlflow_registry_model_v2.png
```

Phrase :

```text
MLflow a été utilisé pour le tracking, la comparaison des modèles et le registry. Le déploiement utilise ensuite un export joblib minimal afin d'avoir une image Docker autonome.
```

## 15. Performance

Commandes :

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

Notebook :

```text
notebooks/07_performance_optimization.ipynb
```

Screenshots associés :

```text
reports/screenshots/12_performance_notebook.png
reports/screenshots/13_onnx_benchmark_json.png
```

Phrase :

```text
Le principal gain pratique vient du batching, qui réduit le coût répété du preprocessing et de la validation. ONNX Runtime a été testé comme PoC sur l'estimateur LightGBM, avec un gain mesurable mais moins structurant que le batching.
```

## 16. Pytest coverage

Commande :

```powershell
uv run pytest -q
```

Rapports :

```text
reports/coverage/coverage.xml
reports/coverage/html/
```

Screenshot recommandé :

```text
reports/screenshots/18_pytest_coverage_report.png
```

Phrase :

```text
Les tests génèrent un rapport de couverture. Le seuil minimal est volontairement modéré car le dépôt contient beaucoup de code expérimental, de notebooks et de scripts de training. Les tests ciblent prioritairement les composants critiques de production : API, schémas, IO et logique métier.
```

## 17. Dockerfiles et workflows

Structure finale :

```text
docker-compose.yml
docker/
  api.Dockerfile
  prometheus.Dockerfile
deploy/huggingface/
  Dockerfile
.github/workflows/
  ci.yml
  deploy_huggingface.yml
```

Phrase :

```text
L'image docker/api.Dockerfile sert à la stack locale et au build CI. Le Dockerfile Hugging Face est séparé car le Space doit exposer un seul port public et utilise Nginx pour router / vers Streamlit et /api vers FastAPI. Le Dockerfile Prometheus embarque la configuration Prometheus pour éviter les problèmes de volume local sous Windows.
```

## 18. Script oral 15 minutes

### 0:00 - 2:00 : contexte

```text
Je présente la mise en production simulée d'un modèle de scoring crédit. Le modèle LightGBM a été entraîné et versionné en Partie 1, puis déployé et monitoré en Partie 2.
```

### 2:00 - 4:00 : architecture

Montrer README ou schéma.

```text
Localement, Docker Compose lance une stack complète. En remote, Hugging Face lance un conteneur unique avec Nginx, Streamlit, FastAPI et connexion Supabase.
```

### 4:00 - 6:30 : API

Montrer Swagger local ou remote.

```text
L'API expose /predict et /predict_batch, valide les entrées avec Pydantic et charge le modèle une seule fois au démarrage.
```

### 6:30 - 8:30 : stockage

Montrer PostgreSQL ou Supabase.

```text
Chaque prédiction est loggée avec les inputs, outputs, version du modèle, score, décision et latence.
```

### 8:30 - 10:30 : monitoring

Montrer Streamlit, Prometheus, Grafana.

```text
Streamlit sert à la démonstration utilisateur et au monitoring simplifié. Prometheus/Grafana servent au monitoring technique.
```

### 10:30 - 12:00 : drift

Montrer notebook 06 et Evidently.

```text
Le drift compare les données de production aux données de référence issues du développement.
```

### 12:00 - 13:30 : performance

Montrer notebook 07.

```text
Le batching est l'optimisation principale, ONNX est un PoC complémentaire.
```

### 13:30 - 15:00 : CI/CD et conclusion

Montrer GitHub Actions.

```text
La CI lance les tests, le coverage, le lint, le build Docker. Le workflow Hugging Face redéploie automatiquement le Space après validation.
```

## 19. Questions probables

### Pourquoi FastAPI ?

```text
FastAPI fournit une API performante, typée, documentée automatiquement avec OpenAPI/Swagger, et s'intègre bien à Docker, pytest et au monitoring.
```

### Pourquoi Streamlit et Grafana ?

```text
Streamlit vise l'utilisateur métier et la démonstration interactive. Grafana vise le monitoring technique Dev/Ops.
```

### Pourquoi PostgreSQL / Supabase ?

```text
Les prédictions sont structurées, requêtables et auditables. Supabase permet de démontrer la même logique avec une base distante managée.
```

### Pourquoi ne pas charger MLflow directement sur Hugging Face ?

```text
MLflow est utilisé pour le tracking et le registry pendant l'entraînement. Pour le déploiement, j'embarque les artefacts minimaux afin d'obtenir une image autonome, plus robuste et plus simple à reproduire.
```

### Pourquoi le coverage n'est pas plus élevé ?

```text
Le dépôt contient beaucoup de code expérimental, de notebooks et de scripts de training. Les tests ciblent les composants critiques pour la production : API, validation d'entrée, IO et logique métier. Le seuil est volontairement modéré mais explicite.
```

### Pourquoi ONNX n'est pas l'optimisation principale ?

```text
ONNX accélère l'inférence du modèle, mais le preprocessing reste dans Python/sklearn. Le batching réduit davantage les surcoûts répétés et constitue l'optimisation la plus impactante dans cette architecture.
```

### Comment gérer le drift en production réelle ?

```text
Il faudrait monitorer une fenêtre glissante de prédictions, comparer aux données de référence, déclencher des alertes si les seuils sont dépassés, puis investiguer et éventuellement réentraîner le modèle.
```

### Que se passe-t-il si Supabase est indisponible ?

```text
Le logging base est best-effort : l'API continue de renvoyer une prédiction et logge l'erreur. La disponibilité du scoring est donc découplée du stockage.
```

## 20. Conclusion

Phrase finale :

```text
Le projet couvre l'ensemble du cycle de vie MLOps attendu : modèle versionné, API, Docker, CI/CD, monitoring, drift, performance, stockage production et déploiement distant. La stack locale est complète et la version distante permet une démonstration publique réaliste avec Hugging Face et Supabase.
```

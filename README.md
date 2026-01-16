# MLOps Lab 01 — Churn Model

Projet pédagogique complet couvrant Git, DVC, CI/CD, Docker, Kubernetes et MLflow autour d'une API FastAPI de prédiction de churn.

## 📚 Aperçu des labs

- **Lab 3 - DVC** : Versionnement des données/pipelines ([README_LAB3_DVC.md](README_LAB3_DVC.md))
- **Lab 4 - CI/CD** : Pipeline GitHub Actions (tests, build, déploiement simulé) ([README_LAB4_CICD.md](README_LAB4_CICD.md))
- **Lab 5 - Docker** : Conteneurisation + Docker Compose ([README_LAB5_DOCKER.md](README_LAB5_DOCKER.md))
- **Lab 6 - Kubernetes** : Déploiement sur K8s avec health checks, volumes persistants, monitoring ([README_LAB6_K8S.md](README_LAB6_K8S.md))
- **Lab 7 - MLflow** : Gestion du cycle de vie des modèles, versioning, promotion, rollback ([README_LAB7_MLFLOW.md](README_LAB7_MLFLOW.md))

## Arborescence (extrait)
```
mlops-lab-01/
 ├─ data/             # données (raw/processed) gérées par DVC
 ├─ logs/             # logs API (predictions.log)
 ├─ models/           # modèles entraînés .joblib
 ├─ registry/         # current_model.txt, metadata.json, train_stats.json
 ├─ mlflow/           # MLflow tracking (artifacts, mlflow.db)
 ├─ k8s/              # Manifests Kubernetes (deployment, service, pvc, etc.)
 ├─ src/              # api.py, train.py, promote.py, rollback.py, etc.
 ├─ .github/workflows/ # CI/CD
 ├─ Dockerfile
 ├─ docker-compose.yml
 ├─ dvc.yaml / dvc.lock
 └─ requirements.txt
```

## Prérequis
- Python 3.12 recommandé (aligné avec l'image Docker)
- Docker + Docker Compose
- Git, DVC
- Kubernetes (Minikube ou Docker Desktop)
- MLflow

## Installation locale
```bash
python -m venv venv_mlops
venv_mlops\Scripts\activate  # Windows
pip install --upgrade pip
pip install -r requirements.txt
```

## Données & DVC
```bash
# récupérer les données suivies par DVC
dvc pull
# exécuter le pipeline
python src/prepare_data.py
python src/train.py
```
Assurez-vous que `registry/current_model.txt` pointe vers un modèle présent dans `models/`.

## API FastAPI (local)
```bash
uvicorn src.api:app --reload --port 8000
# santé
curl http://localhost:8000/health
# prédiction
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d "{\"tenure_months\":18,\"num_complaints\":1,\"avg_session_minutes\":60.0,\"plan_type\":\"standard\",\"region\":\"AF\"}"
```

## Conteneur Docker
```bash
# build (Python 3.11 dans Dockerfile)
docker build --no-cache -t churn-api:latest .
# run
docker run -d --name churn-api -p 8000:8000 churn-api:latest
# logs (prédictions)
docker exec churn-api cat /app/logs/predictions.log
```
Monter les logs en volume pour les consulter localement : `-v ${PWD}/logs:/app/logs`.

## Docker Compose
```bash
docker compose up -d
curl http://localhost:8000/health
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d "{...}"
docker compose down
```

## CI/CD (GitHub Actions)
- Workflow principal : `.github/workflows/ci-cd.yml` (tests, build, artefacts, déploiement simulé).
- Secrets/variables à définir : `PY_VERSION`, `F1_GATE_THRESHOLD`, `DEMO_SECRET`, `APP_ENV` (cf. README_LAB4_CICD.md).

## Notes sur les modèles
- Entraînement : `src/train.py` produit un modèle horodaté dans `models/` et met à jour `registry/current_model.txt`.
- Évaluation : `src/evaluate.py` pour métriques supplémentaires.
- API charge le modèle courant à partir du registry.

## Dépannage rapide
- Mismatch sklearn/Python : l’image Docker est en Python 3.11, sklearn 1.8.0. Garder la même version localement pour sérialiser/désérialiser les modèles.
- Modèle manquant : regénérer `models/` via `python src/train.py` puis vérifier `registry/current_model.txt`.
- Logs absents localement : monter `logs/` en volume avec Docker/Compose.

## Références
- [README_LAB3_DVC.md](README_LAB3_DVC.md)
- [README_LAB4_CICD.md](README_LAB4_CICD.md)
- [README_LAB5_DOCKER.md](README_LAB5_DOCKER.md)
- [README_LAB6_K8S.md](README_LAB6_K8S.md) - **Kubernetes deployment**
- [README_LAB7_MLFLOW.md](README_LAB7_MLFLOW.md) - **MLflow lifecycle management**

## 🚀 Quick Start (MLflow + Kubernetes)

### MLflow Tracking Server
```bash
# Terminal 1: MLflow server
mlflow server --backend-store-uri sqlite:///mlflow/mlflow.db \
              --default-artifact-root file:///mlflow/artifacts \
              --host 127.0.0.1 --port 5000
```

### Train with MLflow
```bash
# Terminal 2: Train model
python src/train.py
# Automatically registers model in MLflow Registry as version v1, v2, etc.
```

### Promote to Production
```bash
python src/promote.py
# Sets alias "production" to latest version
```

### Kubernetes Deployment
```bash
# Start Minikube
minikube start

# Apply manifests
kubectl apply -f k8s/pvc.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Access API
curl http://127.0.0.1:30080/health
curl http://127.0.0.1:30080/docs
```

### Rollback
```bash
# Automatic rollback to previous version
python src/rollback.py

# Or explicit version
python src/rollback.py 2
```

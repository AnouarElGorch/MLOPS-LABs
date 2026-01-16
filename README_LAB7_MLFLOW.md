# Lab 7 : Gestion du Cycle de Vie des Modèles avec MLflow
## Versioning, Registry, Promotion & Rollback

---

## 🎓 Introduction : MLflow Model Registry

### Le Problème en Production

```
Situation typique SANS MLflow:
─────────────────────────────────

Data Scientist:
  ✓ Entraîne modèle_v1
  ✓ Sauvegarde dans models/churn_model_v1.joblib
  ✓ Modifie registry/current_model.txt
  
DevOps/Engineering:
  ❓ Quelle est la version actuelle?
  ❓ Quels paramètres ont produit ce modèle?
  ❓ Puis-je revenir à la version précédente rapidement?
  ❓ Qui a changé le modèle?

API Production:
  ❌ Charge un fichier local (fragile)
  ❌ Pas de versioning
  ❌ Rollback = modification manuelle + redéploiement
```

### La Solution : MLflow Model Registry

```
Avec MLflow:
──────────────

MLflow Server (127.0.0.1:5000)
        ↓
┌───────────────────────────────┐
│   MODEL REGISTRY              │
│                               │
│ churn_model                   │
│  ├── v1 (registered)         │
│  │   └── staging              │
│  ├── v2 (registered)         │
│  │   └── production           │
│  └── v3 (registered)         │
│      └── (aucun alias)        │
│                               │
│ État centralisé + tracé       │
│ Versionning automatique       │
│ Promotion en 1 commande       │
│ Rollback en 1 commande        │
└───────────────────────────────┘
        ↓
    API charge
  models:/churn_model@production
```

---

## 🏗️ Architecture MLflow Model Registry

### Concepts Clés

```
┌──────────────────────────────────────────────────────────────────┐
│                  MLflow Model Registry                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  REGISTERED MODEL: "churn_model"                                │
│  │                                                              │
│  ├─ Model Version 1                                            │
│  │  ├── Source Run:       mlflow-run-001                       │
│  │  ├── Création:         2025-12-15 10:30:00                  │
│  │  ├── Stage:            None                                 │
│  │  ├── Alias:            None                                 │
│  │  └── Artifacts:        MLflow artifacts store              │
│  │                                                              │
│  ├─ Model Version 2  ⭐ COURANT EN PRODUCTION                  │
│  │  ├── Source Run:       mlflow-run-002                       │
│  │  ├── Création:         2025-12-15 11:45:00                  │
│  │  ├── Stage:            Production                           │
│  │  ├── Alias:            production                           │
│  │  └── Artifacts:        MLflow artifacts store              │
│  │                                                              │
│  └─ Model Version 3                                            │
│     ├── Source Run:       mlflow-run-003                       │
│     ├── Création:         2025-12-15 12:15:00                  │
│     ├── Stage:            Staging                              │
│     ├── Alias:            staging                              │
│     └── Artifacts:        MLflow artifacts store              │
│                                                                  │
│  ALIAS = nom logique pointant vers une version                 │
│  Examples: production, staging, champion, challenger           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Cycle de Vie Complet

```
DÉVELOPPEMENT          ENTRAÎNEMENT           REGISTRY           PRODUCTION
──────────            ────────────           ────────           ──────────

Data Scientist               │
      │                      │
      └──→ train.py ─────────┤
            (script)         │
                             ↓
                    ┌─────────────────────┐
                    │ mlflow.log_param()  │
                    │ mlflow.log_metrics()│ ← Parameters, Metrics
                    │ mlflow.log_artifact()
                    │ log_model() ────────┐
                    └─────────────────────┘
                             ↓
                    ┌─────────────────────┐
                    │  MLflow Backend     │  ← SQLite / Database
                    │  (Tracking URI)     │  ← Artifacts Store
                    │  127.0.0.1:5000     │
                    └─────────────────────┘
                             ↓
                    ┌─────────────────────┐
                    │ MODEL REGISTRY      │  Version créée
                    │ churn_model v2      │  (automatique)
                    └─────────────────────┘
                             │
                             ↓
                    promote.py
                    (script)
                             │
                             ↓
                    ┌─────────────────────┐
                    │ UPDATE ALIAS        │
                    │ production ─→ v2    │
                    └─────────────────────┘
                             │
                             ↓
                                           API charges
                                      models:/churn_model@production
                                           ↓
                                      v2 déployée en prod
                                      ✓ Utilisateurs heureux
                                      ✓ Métriques en hausse
                                           │
                                           │ Problème détecté?
                                           ↓
                                      rollback.py
                                      (script)
                                           │
                                           ↓
                                      ┌─────────────┐
                                      │ production  │
                                      │  ─→ v1      │
                                      └─────────────┘
                                           │
                                           ↓
                                      API recharge v1
                                      ✓ Problème résolu
```

---

## 🚀 Étape 1 : Entraînement & Logging (src/train.py)

### Code MLflow Ajouté

```python
# Configuration
mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("mlops-lab-01")

# Création du run
with mlflow.start_run(run_name=f"train-{version}") as run:
    run_id = run.info.run_id
    
    # Enregistrement des paramètres
    mlflow.log_param("version", version)
    mlflow.log_param("seed", seed)
    mlflow.log_param("gate_f1", gate_f1)
    
    # Enregistrement des métriques
    mlflow.log_metrics(metrics)  # F1, accuracy, precision, recall
    
    # Tags (métadonnées humaines)
    mlflow.set_tag("data_file", DATA_PATH.name)
    mlflow.set_tag("model_file", model_filename)
    
    # Artifacts (fichiers)
    mlflow.log_artifact(str(model_path), artifact_path="exported_models")
    
    # REGISTRATION dans le Model Registry
    mlflow.sklearn.log_model(
        sk_model=pipe,
        artifact_path="model",
        registered_model_name="churn_model",  # Crée une nouvelle version
    )
```

### Processus Détaillé

```
1. mlflow.set_experiment("mlops-lab-01")
   └─ Groupe toutes les exécutions (runs) sous ce nom
      Permet: Comparaison A/B, historique

2. mlflow.start_run(run_name=f"train-{version}")
   └─ Crée une exécution traçable unique
      Stocke: Tous les logs, paramètres, métriques

3. mlflow.log_param() / mlflow.log_metrics()
   └─ Enregistre pour reproducibilité
      Permet: Rejouer exact avec les mêmes paramètres

4. mlflow.set_tag()
   └─ Ajoute contexte humain
      Aide: Debugging, correlation avec données/code

5. mlflow.log_artifact()
   └─ Attache le fichier modèle au run
      Bénéfice: Historique complet, traçabilité

6. mlflow.sklearn.log_model(..., registered_model_name="churn_model")
   └─ CRÉATION DE VERSION dans le Registry
      Crée: churn_model v1 (première fois)
            churn_model v2 (deuxième fois)
            churn_model v3 (troisième fois)
      Automatique: Versioning numérique incrémental
```

### Exécution

```powershell
python src/train.py

# Output:
# [Epoch 1/...] Loss: 0.45
# [OK] Modèle sauvegardé : models\churn_model_v1_20260115_101530.joblib
# [DEPLOY] Modèle activé : churn_model_v1_20260115_101530.joblib
# [DEPLOY] Alias stable : models\model.joblib
# [MLflow] Run enregistré : 12ab34cd5e6f | Model Registry : churn_model v2
```




---

## 📌 Étape 2 : Promotion en Production (src/promote.py)

### Objectif

```
AVANT (sans promote.py):
  ❌ Promotion manuelle via fichier texte
  ❌ Risque de confusion (quelle version est active?)
  ❌ Pas de trace
  ❌ Perte de temps

APRÈS (avec promote.py):
  ✅ Promotion programmatique
  ✅ Un alias = une version active
  ✅ Historique complet dans MLflow
  ✅ Automatisable en pipeline CI/CD
```

### Code

```python
# src/promote.py

import mlflow
from mlflow.tracking import MlflowClient

MODEL_NAME = "churn_model"
ALIAS = "production"

mlflow.set_tracking_uri("http://127.0.0.1:5000")
client = MlflowClient()

# Cherche toutes les versions du modèle
mvs = client.search_model_versions(f"name='{MODEL_NAME}'")
if not mvs:
    raise SystemExit(f"Aucune version trouvée pour {MODEL_NAME}.")

# Prend la plus récente
latest_version = max(int(mv.version) for mv in mvs)

# Assigne l'alias "production" à cette version
client.set_registered_model_alias(MODEL_NAME, ALIAS, str(latest_version))
print(f"Modèle activé : {MODEL_NAME}@{ALIAS} -> v{latest_version}")
```

### Exécution

```powershell
# Après train.py v2
python src/promote.py
# [Output] Modèle activé : churn_model@production -> v2

# Vérifier
curl http://127.0.0.1:5000/api/2.0/mlflow/registered-models/get?name=churn_model
# → "alias_list": ["production"]
```

### Flux

```
1. Script détecte la PLUS RÉCENTE version
   (ex: v2)
   
2. Assigne alias "production" à v2
   ├─ v1: (aucun alias)
   └─ v2: production ⭐
   
3. API rechargera v2
   (cache invalidé)
```



---

## 🔄 Étape 3 : Rollback Automatique (src/rollback.py)

### Motivation

```
SCÉNARIO RÉEL:
──────────────
  ✅ v1 en production (F1=0.82)
  ✅ Entraîne v2 (F1=0.85) → Promotion
  ❌ v2 CAUSE DES FAUX NÉGATIFS (clients quittent!)
  
SANS MLflow:
  ⏱️ 30 minutes pour:
    1. Trouver ancienne version
    2. Recompiler le code
    3. Redémarrer containers
    4. Tester à nouveau
  💔 Dégâts = perte de clients
  
AVEC MLflow + rollback.py:
  ⏱️ 5 SECONDES pour:
    1. python src/rollback.py
  ✅ v1 réactivée immédiatement
  ✅ Zéro downtime
```

### Code

```python
# src/rollback.py - Fonctions clés

def _list_versions(client: MlflowClient) -> list[int]:
    """Toutes les versions existantes."""
    versions = client.search_model_versions(f"name='{MODEL_NAME}'")
    return sorted({int(v.version) for v in versions})

def _get_current_version(client: MlflowClient) -> Optional[int]:
    """Version pointée par l'alias 'production'."""
    mv = client.get_model_version_by_alias(MODEL_NAME, ALIAS)
    return int(mv.version)

def _set_alias(client: MlflowClient, version: int) -> None:
    """Change l'alias pour pointer vers une version."""
    client.set_registered_model_alias(MODEL_NAME, ALIAS, str(version))

def main(target: Optional[str] = None) -> None:
    """
    Deux modes:
    - Rollback auto: revenir à la version PRÉCÉDENTE
    - Promotion explicite: aller à la version N
    """
    if target is None:
        # Rollback: version précédente
        previous = versions[idx - 1]
        _set_alias(client, previous)
        print(f"[OK] rollback => v{current} -> v{previous}")
    else:
        # Explicit: version N
        v = int(target)
        _set_alias(client, v)
        print(f"[OK] activation => v{v}")
```

### Cas d'Usage

```powershell
# Cas 1: Rollback automatique
python src/rollback.py
# v2 → v1 (version PRÉCÉDENTE)

# Cas 2: Activation explicite
python src/rollback.py 3
# → v3 (spécifier le numéro)
```

### Sécurité

```
Rollback impossible:
  python src/rollback.py
  # Quand: Déjà sur la version la plus ANCIENNE
  # Résultat: ValueError + message clair
  
Activation explicite:
  python src/rollback.py 99
  # Quand: v99 n'existe pas
  # Résultat: ValueError + liste des versions valides
```



---

## 🔗 Étape 4 : API Intégrée à MLflow (src/api.py)

### Évolution

```
AVANT (chargement local):
──────────────────────────

api.py:
  def get_current_model_name():
    return CURRENT_MODEL_PATH.read_text()  ← Fichier texte!
  
  def load_model_if_needed():
    path = MODELS_DIR / name
    model = joblib.load(path)             ← Disque local
    
Problèmes:
  ❌ Couplage fort avec filesystem
  ❌ Pas tracé (qui a changé?)
  ❌ Manque context (version, paramètres d'entraînement)
  ❌ Pas de versioning transparent

APRÈS (chargement MLflow):
──────────────────────────

api.py:
  def get_current_model_name():
    client = MlflowClient()
    mv = client.get_model_version_by_alias("churn_model", "production")
    return f"churn_model@production (v{mv.version})"
    
  def load_model_if_needed():
    model = mlflow.sklearn.load_model("models:/churn_model@production")
    
Bénéfices:
  ✅ Transparent: alias = version
  ✅ Tracé: chaque alias pointe vers run
  ✅ Contexte: run contient paramètres/métriques
  ✅ Dynamique: rollback = rechargement auto
```

### Code

```python
# src/api.py

MLFLOW_TRACKING_URI = "http://127.0.0.1:5000"
MODEL_NAME = "churn_model"
ALIAS = "production"
MODEL_URI = f"models:/{MODEL_NAME}@{ALIAS}"

def get_current_model_name() -> str:
    """Récupère le nom+version depuis MLflow Registry."""
    mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)
    client = MlflowClient()
    mv = client.get_model_version_by_alias(MODEL_NAME, ALIAS)
    return f"{MODEL_NAME}@{ALIAS} (v{mv.version})"

def load_model_if_needed() -> tuple[str, Any]:
    """Charge depuis MLflow avec cache."""
    mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)
    
    cache_key = MODEL_URI
    
    # Réutiliser si en cache
    if _model_cache["name"] == cache_key and _model_cache["model"]:
        return cache_key, _model_cache["model"]
    
    # Sinon, charger depuis MLflow
    model = mlflow.sklearn.load_model(MODEL_URI)
    
    _model_cache["name"] = cache_key
    _model_cache["model"] = model
    return cache_key, model
```

### Comportement

```
WORKFLOW COMPLET:
─────────────────

1. Client appelle: POST /predict
   Payload: {tenure_months: 48, ...}

2. load_model_if_needed()
   ├─ Vérifier cache
   └─ Si vide: mlflow.sklearn.load_model("models:/churn_model@production")
   
3. Model chargé contient:
   ├─ Preprocessing (StandardScaler, OneHotEncoder)
   ├─ Modèle (LogisticRegression)
   └─ Métadonnées (run_id, version, paramètres)

4. Prédiction effectuée
   Résultat: {"prediction": 0, "probability": 0.23, "model": "v2"}

5. Dans le contexte:
   Si rollback.py exécuté ENTRE 2 et 4:
   └─ Cache invalidé
   └─ PROCHAIN appel: reload v1 automatiquement
      ✅ ZÉRO DOWNTIME!
```



---

## 🔀 Étape 5 : Stages vs Alias (Concepts Avancés)

### Stages (Ancien système)

```
Registry stages (ancien MLflow):
  ├─ None        (nouveau modèle)
  ├─ Staging     (test)
  ├─ Production  (en prod)
  └─ Archived    (vieux)

Limitations:
  ❌ Une version = UN stage
  ❌ Pas d'alias multiples
  ❌ Transitioned_at = timestamp

Exemple:
  v1: Production (2024-01-01)
  v2: Staging (2024-01-05)
  → Pour passer v2 en prod, stage(v1) = Archived
  → Complexe
```

### Alias (Nouveau système) ⭐

```
Registry aliases (MLflow 2.0+):
  ├─ production    → v2
  ├─ staging       → v3
  ├─ champion      → v2
  └─ challenger    → v3

Avantages:
  ✅ Une version = PLUSIEURS alias
  ✅ Alias multiples
  ✅ Shadow deployment: champion vs challenger

Exemple:
  churn_model@production ─→ v2  (utilisée en prod)
  churn_model@staging ───→ v3  (testée avant)
  churn_model@champion ──→ v2  (meilleure F1)
  churn_model@challenger → v3  (nouvelle candidate)
```

### Cas: A/B Testing avec Alias

```
SETUP:
──────

mlflow.set_registered_model_alias("churn_model", "champion", "2")
mlflow.set_registered_model_alias("churn_model", "challenger", "3")

API avec A/B:
──────────────

def load_model_for_user(user_id):
    if user_id % 2 == 0:
        # 50% des users: champion (v2)
        return mlflow.sklearn.load_model("models:/churn_model@champion")
    else:
        # 50% des users: challenger (v3)
        return mlflow.sklearn.load_model("models:/churn_model@challenger")

Résultats:
──────────

Log JSON:
  {
    "user_id": "user_123",
    "model_alias": "challenger",
    "prediction": 1,
    "probability": 0.67
  }

Analyse:
  Champion (v2) → F1 = 0.82, churn_rate = 15%
  Challenger (v3) → F1 = 0.88, churn_rate = 12%
  → Champion remplacé par Challenger
```

---

## 🎯 Étape 6 : Workflow Complet du Projet

### Schéma Résumé

```
DATA SCIENTIST                    MLFLOW SERVER              PRODUCTION API
──────────────────────            ──────────────              ───────────────

python train.py
  ├─ Charge data
  ├─ Entraîne
  ├─ Évalue
  ├─ Enregistre params/metrics ──→ Tracking
  ├─ Log artifacts ─────────────→ Artifacts Store
  └─ log_model() ───────────────→ MODEL REGISTRY (v2)
                                   │
python promote.py                  │
  ├─ search_model_versions() ←────┤
  ├─ Détecte v2
  └─ set_alias(v2, "production") ──→ UPDATE alias
                                   │
                                   └─→ API recharge
                                       churn_model@production
                                       ↓
                                   Production serve v2
                                   ✅ Utilisateurs heureux


PROBLÈME DÉTECTÉ (v2 bugué):

python rollback.py
  ├─ _get_current_version() ←──┐
  ├─ Détecte v2 actuel         │
  ├─ Calcule previous = v1      │
  └─ set_alias(v1, "production") → UPDATE alias
                                  │
                                  └─→ API recharge
                                      churn_model@production (v1)
                                      ↓
                                  Production serve v1
                                  ✅ Problème résolu
                                  ✅ Zéro downtime
```

### Checklist: Reproduction

```powershell
# 1. Démarrer MLflow Server
$Env:MLFLOW_BACKEND_STORE_URI = "sqlite:///mlflow/mlflow.db"
$Env:MLFLOW_DEFAULT_ARTIFACT_ROOT = "file:///mlflow/artifacts"
mlflow server --backend-store-uri $Env:MLFLOW_BACKEND_STORE_URI `
              --default-artifact-root $Env:MLFLOW_DEFAULT_ARTIFACT_ROOT `
              --host 127.0.0.1 --port 5000

# 2. Terminal 2: Venv + Train
& 'C:/Users/anoua/projects/MLOPS/mlops-lab-01/venv_mlops/Scripts/Activate.ps1'
python src/train.py
# → v1 créée, registered

# 3. Prometheus v1 en prod
python src/promote.py
# → production = v1

# 4. Train à nouveau
python src/train.py
# → v2 créée, registered

# 5. Promouvoir v2
python src/promote.py
# → production = v2

# 6. Vérifier API charge v2
curl http://127.0.0.1:30080/health
# {"model": "churn_model@production (v2)"}

# 7. Rollback
python src/rollback.py
# → production = v1

# 8. Vérifier API charge v1
curl http://127.0.0.1:30080/health
# {"model": "churn_model@production (v1)"}
```

---

## 🔒 Bonnes Pratiques

### 1️⃣ Toujours Logger les Paramètres

```python
# ❌ Mauvais
mlflow.log_metrics({"accuracy": 0.95})

# ✅ Bon
mlflow.log_param("learning_rate", 0.01)
mlflow.log_param("epochs", 100)
mlflow.log_metrics({"accuracy": 0.95})
# → Reproductibilité garantie
```

### 2️⃣ Tags pour le Contexte Humain

```python
# ✅ Ajouter toujours
mlflow.set_tag("data_file", "processed.csv")
mlflow.set_tag("model_file", "churn_model_v1_20260115.joblib")
mlflow.set_tag("model_type", "LogisticRegression")
mlflow.set_tag("owner", "data_science_team")
# → Facile de trouver qui/quoi/quand
```

### 3️⃣ Versioning Explicite

```python
# ❌ Mauvais
version = "v1"  # Hardcodé

# ✅ Bon
from datetime import datetime
version = f"v{datetime.now().strftime('%Y%m%d_%H%M%S')}"
mlflow.log_param("version", version)
```

### 4️⃣ Cache intelligemment en Production

```python
# ✅ Cache basé sur alias, pas filename
cache_key = f"models:/{MODEL_NAME}@{ALIAS}"

if _cache.get(cache_key):
    return _cache[cache_key]
else:
    model = mlflow.sklearn.load_model(cache_key)
    _cache[cache_key] = model
    return model
```

### 5️⃣ Rollback Sûr

```python
# ✅ Vérifications avant rollback
versions = _list_versions()
if len(versions) < 2:
    raise ValueError("Au moins 2 versions requises")
    
current = _get_current_version()
if current == versions[0]:
    raise ValueError("Déjà sur version minimale")
```

---

## 📊 Architecture Finale: MLOps Complete

```
┌───────────────────────────────────────────────────────────────────┐
│                     MLOPS STACK COMPLET                           │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  DATA SCIENTIST              MLFLOW                  OPERATIONS   │
│  ──────────────────          ──────────              ─────────    │
│                                                                   │
│  data/                       MLflow Server          src/         │
│  ├─ raw.csv                  (127.0.0.1:5000)       ├─ train.py  │
│  └─ processed.csv            ├─ Tracking             ├─ api.py    │
│                              │  (params, metrics)    ├─ promote.py│
│  src/train.py                ├─ Model Registry       └─ rollback.py
│  ├─ Load data                │  (v1, v2, v3)                     │
│  ├─ Entraîne                 ├─ Artifacts Store                   │
│  ├─ Log to MLflow            │  (joblib files)                    │
│  └─ Register model ─────────→├─ Aliases                           │
│                              │  (production, staging)            │
│  models/                     └─ UI Dashboard                      │
│  ├─ churn_model_v1_.joblib              ↑                        │
│  ├─ churn_model_v2_.joblib              │                        │
│  └─ model.joblib (alias stable)         │                        │
│                                      Query Registry              │
│  registry/                             ↓                        │
│  ├─ current_model.txt      src/promote.py  src/rollback.py      │
│  ├─ metadata.json          Update alias ── Query version         │
│  └─ train_stats.json       "production@v2"                       │
│                                      ↓                           │
│  logs/                          Load Model                       │
│  ├─ predictions.log        src/api.py                            │
│  └─ drift_monitoring.log   mlflow.sklearn.load_model()          │
│                           (dynamique, cache)                     │
│                                      ↓                           │
│  Kubernetes Pod                 PRODUCTION API                   │
│  ├─ Container API           (127.0.0.1:8000)                     │
│  ├─ PVC (registry, models)  ├─ /health → v2                      │
│  └─ Service (NodePort)      ├─ /predict → v2                     │
│                             └─ /ready → v2                       │
│                                      ↓                           │
│                            USERS (Prédictions)                   │
│                                                                   │
│  Monitoring                                                      │
│  src/monitor_drift.py                                            │
│  ├─ Charge train_stats.json                                      │
│  ├─ Détecte changements                                          │
│  └─ Alert si drift                                               │
│     → Trigger retraining                                         │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---



## 🎓 Résumé: Cycle MLflow Complet

| Étape | Script | Action | Résultat |
|-------|--------|--------|----------|
| 1 | `train.py` | Entraîne + Register | Model Registry: v2 créée |
| 2 | `promote.py` | Assigne alias | production → v2 |
| 3 | `api.py` | Charge via alias | Serve v2 |
| 4 | `rollback.py` | Change alias | production → v1 |
| 5 | `api.py` | Recharge (cache off) | Serve v1 |

**Temps total rollback: 5 secondes ⚡**

---

## 🚀 Commandes Rapides

```powershell
# Démarrer MLflow
mlflow server --backend-store-uri sqlite:///mlflow/mlflow.db `
              --default-artifact-root file:///mlflow/artifacts `
              --host 127.0.0.1 --port 5000

# Train + Registry
python src/train.py

# Promouvoir
python src/promote.py

# Rollback
python src/rollback.py

# ou spécifier version
python src/rollback.py 1

# Test API
curl http://127.0.0.1:30080/health
curl http://127.0.0.1:30080/docs
```

---

## 📚 Ressources

- [MLflow Docs](https://mlflow.org/docs/)
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)
- [MLflow API Reference](https://mlflow.org/docs/latest/python_api/mlflow.html)
- [Registered Models & Versions](https://mlflow.org/docs/latest/model-registry.html#concepts)

---

**Lab 7:** Janvier 2026 | Gestion du Cycle de Vie | ✅ Complète


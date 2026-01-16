# Lab 6 : Déploiement Kubernetes d'un Système MLOps Churn

## 🎯 Vue d'ensemble du Lab

Ce lab vous guide à travers le **déploiement en production** d'un système complet de Machine Learning Operations (MLOps) pour la prédiction de churn client. Vous allez construire une architecture **cloud-native** utilisant Kubernetes, en mettant en place :

- **Containerisation** : Empaquetage de votre application dans une image Docker
- **Orchestration** : Gestion et scaling automatique avec Kubernetes
- **Persistance** : Stockage des modèles et logs sur des volumes persistants
- **Monitoring** : Surveillance de la dérive des données en production
- **Automation** : Exécution programmée des jobs d'entraînement et de monitoring

### Qu'est-ce que vous allez apprendre ?

1. **Docker** : Créer une image containerisée de votre API FastAPI
2. **Kubernetes** : Déployer et gérer votre application dans un cluster K8s
3. **MLOps** : Implémenter un pipeline complet (train → serve → monitor)
4. **Health Checks** : Configurer des probes pour la haute disponibilité
5. **Drift Detection** : Détecter automatiquement les changements de données
6. **Automation** : Planifier des tâches avec CronJobs

---

## 📋 Objectifs du Lab

- ✅ Containeriser l'application MLOps avec Docker
- ✅ Configurer Kubernetes (Minikube ou Docker Desktop)
- ✅ Déployer les composants (Deployment, Service, ConfigMap, Secret)
- ✅ Monter des volumes persistants (PVC) pour stocker modèles et logs
- ✅ Implémenter les health checks (livenessProbe, readinessProbe, startupProbe)
- ✅ Monitorer la dérive des données (drift monitoring)
- ✅ Automatiser les tâches (Jobs et CronJobs)

---

## � Étape 1 : Dockeriser l'Application

### Pourquoi Docker ?

Docker permet de **packager votre application** avec toutes ses dépendances dans une image immuable. Cela garantit que votre application s'exécute de la même manière **partout** (développement, test, production).

### Fichier: `Dockerfile`

### Pourquoi Docker ?

Docker permet de **packager votre application** avec toutes ses dépendances dans une image immuable. Cela garantit que votre application s'exécute de la même manière **partout** (développement, test, production).

### Fichier: `Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "src.api:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Explication ligne par ligne :**

| Ligne | Explication |
|-------|-------------|
| `FROM python:3.12-slim` | Utilise l'image de base Python 3.12 (slim = image légère) |
| `WORKDIR /app` | Définit `/app` comme répertoire de travail dans le conteneur |
| `COPY requirements.txt .` | Copie le fichier de dépendances dans le conteneur |
| `RUN pip install ...` | Installe toutes les dépendances Python |
| `COPY . .` | Copie tout le code source dans le conteneur |
| `EXPOSE 8000` | Déclare que le port 8000 sera utilisé |
| `CMD ["uvicorn", ...]` | Lance FastAPI au démarrage du conteneur |

### Processus de Build

### Processus de Build

```
1. Minikube configure Docker
   ↓
2. Docker construit l'image couche par couche
   ↓
3. Image churn-api:v1 est disponible dans Minikube
   ↓
4. Kubernetes peut la utiliser pour créer des pods
```

**Commandes:**

```powershell
# Configurer Docker pour utiliser Minikube
minikube docker-env | Invoke-Expression

# Construire l'image
docker build -t churn-api:v1 .

# Vérifier que l'image existe
minikube image ls | Select-String churn-api

# (Optionnel) Vérifier localement
docker images | Select-String churn-api
```


## 🚀 Étape 2 : Configurer Kubernetes

### Qu'est-ce que Kubernetes ?

Kubernetes (K8s) est un **orchestrateur de conteneurs** qui :
- **Déploie** et gère vos conteneurs Docker
- **Scale** automatiquement (crée plus de pods si besoin)
- **Redémarre** les pods qui crashent
- **Distribute** le trafic entre les pods
- **Persiste** les données avec des volumes

### Architecture Kubernetes pour ce Lab

```
┌─────────────────────────────────────────┐
│         Kubernetes Cluster              │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────────────────────────┐  │
│  │    Deployment: churn-api         │  │
│  │  (2 replicas = 2 pods)           │  │
│  │                                  │  │
│  │  ┌─────────┐  ┌─────────┐       │  │
│  │  │ Pod 1   │  │ Pod 2   │       │  │
│  │  │ :8000   │  │ :8000   │       │  │
│  │  └────┬────┘  └────┬────┘       │  │
│  └───────┼────────────┼────────────┘  │
│          │            │                │
│  ┌───────▼────────────▼────────────┐  │
│  │    Service: NodePort :30080     │  │
│  │    (Load Balancer)              │  │
│  └────────────────────────────────┘  │
│          ↓                            │
│  ┌───────────────────────────────┐   │
│  │  PVC: churn-storage           │   │
│  │  (Volume Persistant)          │   │
│  │  - /app/models                │   │
│  │  - /app/registry              │   │
│  │  - /app/logs                  │   │
│  └───────────────────────────────┘   │
│                                       │
└───────────────────────────────────────┘
         ↓ Accès externe
   http://127.0.0.1:30080
```

### Démarrer Kubernetes

**Avec Minikube:**
```powershell
# Démarrer Minikube
minikube start

# Vérifier le cluster
kubectl cluster-info
kubectl get nodes
```

**Avec Docker Desktop:**
1. Ouvrir Docker Desktop
2. Settings → Kubernetes → Enable Kubernetes
3. Attendre le démarrage (quelques minutes)

**Vérifier que tout fonctionne:**
```powershell
kubectl get nodes
# Output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.28.3
```



---

## 📦 Étape 3 : Déploiement Kubernetes (Deployment)

### Qu'est-ce qu'un Deployment ?

Un **Deployment** est un objet Kubernetes qui :
- Spécifie **combien de copies** (replicas) de votre application doivent tourner
- Gère le **rolling update** (mise à jour sans downtime)
- Redémarre les pods qui crashent
- Permet l'**auto-scaling**

**Fichier:** `k8s/deployment.yaml`
**Fichier:** `k8s/deployment.yaml`

### Structure du Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: churn-api
spec:
  replicas: 2  # Créer 2 copies (pods) de l'application
  selector:
    matchLabels:
      app: churn-api
  template:
    metadata:
      labels:
        app: churn-api
    spec:
      containers:
        - name: api
          image: churn-api:v1  # Image Docker créée précédemment
          ports:
            - containerPort: 8000
          env:  # Variables d'environnement
            - name: MODEL_NAME
              valueFrom:
                configMapKeyRef:
                  name: churn-config
                  key: MODEL_NAME
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: churn-config
                  key: LOG_LEVEL
            - name: MONITORING_TOKEN
              valueFrom:
                secretKeyRef:
                  name: churn-secret
                  key: MONITORING_TOKEN
          volumeMounts:  # Monter les volumes
            - name: churn-volume
              mountPath: /app/registry
              subPath: registry
            - name: churn-volume
              mountPath: /app/models
              subPath: models
            - name: churn-volume
              mountPath: /app/logs
              subPath: logs
          # Health checks
          startupProbe:
            httpGet:
              path: /startup
              port: 8000
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 30
      volumes:
        - name: churn-volume
          persistentVolumeClaim:
            claimName: churn-storage
```

### Explication des Sections Clés

#### 1. **replicas: 2**
Crée 2 pods identiques. Si un pod crash, Kubernetes en redémarre automatiquement un pour maintenir 2 replicas actifs.

#### 2. **Environment Variables**
```yaml
env:
  - name: MODEL_NAME
    valueFrom:
      configMapKeyRef:  # Lire depuis ConfigMap
        name: churn-config
        key: MODEL_NAME
  - name: MONITORING_TOKEN
    valueFrom:
      secretKeyRef:  # Lire depuis Secret
        name: churn-secret
        key: MONITORING_TOKEN
```

**Avantage:** Pas de secrets en dur dans le YAML. Les valores sensibles sont stockées séparément.

#### 3. **volumeMounts**
```yaml
volumeMounts:
  - name: churn-volume
    mountPath: /app/registry  # Où montrer dans le conteneur
    subPath: registry  # Sous-dossier du PVC
```

**Utilité:** Partager un même volume PVC entre plusieurs pods, en les isolant dans des sous-dossiers.

#### 4. **Health Checks (Probes)**

```yaml
startupProbe:  # Vérifie le démarrage
  httpGet:
    path: /startup
    port: 8000
  failureThreshold: 30      # 30 tentatives max
  periodSeconds: 5          # Toutes les 5 secondes
  # = 150 secondes d'attente max pour démarrer

readinessProbe:  # Vérifie la disponibilité
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 5    # Attendre 5s avant de vérifier
  periodSeconds: 10         # Vérifier toutes les 10s

livenessProbe:  # Vérifie que c'est vivant
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 30         # Vérifier toutes les 30s
```

**Comment ça marche:**
1. Pod démarre → startupProbe teste `/startup`
2. Si `/startup` répond 200 OK, le pod est "started"
3. readinessProbe teste `/ready` en continu
4. Si OK → pod reçoit du trafic ; sinon → trafic déviéé
5. livenessProbe teste `/health`
6. Si échoue plusieurs fois → pod redémarré

**Endpoints correspondants dans `src/api.py`:**

```python
@app.get("/startup")
def startup():
    # Vérifie que registry existe et current_model.txt n'est pas vide
    # Retour 503 si problème

@app.get("/ready")
def ready():
    # Vérifie que le modèle peut être chargé
    # Retour 503 si problème

@app.get("/health")
def health():
    # Vérification basique
    # Retour 200 si OK
```

### Déployer l'Application

**Commandes:**
### Déployer l'Application

**Commandes:**

```powershell
# Appliquer le deployment
kubectl apply -f k8s/deployment.yaml

# Voir les deployments
kubectl get deployments
# Output:
# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# churn-api   2/2     2            2           10s

# Voir les pods créés
kubectl get pods -l app=churn-api
# Output:
# NAME                         READY   STATUS    RESTARTS   AGE
# churn-api-7655fd649b-tg6fz   1/1     Running   0          10s
# churn-api-7655fd649b-vzd4l   1/1     Running   0          9s

# Voir les détails
kubectl describe deployment churn-api
```



---

## 🔌 Étape 4 : Exposer l'Application avec un Service

**Fichier:** `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: churn-api-service
spec:
  type: NodePort
  selector:
    app: churn-api  # Sélectionne les pods avec ce label
  ports:
    - port: 80           # Port du service
      targetPort: 8000   # Port du pod
      nodePort: 30080    # Port externe (30000-32767)
```

### Comment le trafic arrive jusqu'au pod

```
Navigateur                  Minikube Node
127.0.0.1:30080   ────→    Node:30080
                           ↓
                    NodePort Service
                           ↓
                    Load Balancer
                      (round-robin)
                           ↓
              ┌────────────┴────────────┐
              ↓                         ↓
            Pod 1:8000              Pod 2:8000
           (Instance 1)            (Instance 2)
```

**Commandes:**

```powershell
# Appliquer le service
kubectl apply -f k8s/service.yaml

# Voir les services
kubectl get svc
# Output:
# NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
# churn-api-service     NodePort   10.96.123.45   <none>        80:30080/TCP

# Voir les détails
kubectl describe svc churn-api-service
```

**Accéder à l'API:**

```powershell
# Via NodePort (recommandé)
http://127.0.0.1:30080/docs         # Swagger UI
http://127.0.0.1:30080/health       # Health check
http://127.0.0.1:30080/predict      # POST pour prédiction

# Alternative: port-forward
kubectl port-forward svc/churn-api-service 8000:80
# Puis: http://localhost:8000/docs
```


---

### 4. **ConfigMap**

Stocke les configurations non-sensibles :
- `MODEL_NAME` : Nom du modèle courant
- `LOG_LEVEL` : Niveau de logging

**Exemple de création:**
```bash
kubectl create configmap churn-config \
  --from-literal=MODEL_NAME="churn_model_v1" \
  --from-literal=LOG_LEVEL="INFO"
```



---

### 5. **Secret Kubernetes**

Stocke les données sensibles :
- `MONITORING_TOKEN` : Token pour les alertes externes

**Exemple de création:**
```bash
kubectl create secret generic churn-secret \
  --from-literal=MONITORING_TOKEN="your-token-here"
```


---

### 6. **Volumes Persistants (PVC)**

**Fichier:** `k8s/pvc.yaml` (à créer si absent)

Permet le partage de données entre pods :
- Modèles (`/app/models`)
- Registry (`/app/registry`)
- Logs (`/app/logs`)

**Commandes:**
```bash
kubectl apply -f k8s/pvc.yaml
kubectl get pvc
kubectl describe pvc churn-storage
```



---

## 🔄 Étape 5 : L'API FastAPI et ses Endpoints

### Architecture de l'API

Votre API FastAPI expose plusieurs endpoints pour :
- **Santé** : Vérifier que l'API fonctionne
- **Prédictions** : Faire des prédictions sur le churn
- **Monitoring** : Pour les Kubernetes probes

---

## 🔄 Étape 5 : L'API FastAPI et ses Endpoints

### Architecture de l'API

Votre API FastAPI expose plusieurs endpoints pour :
- **Santé** : Vérifier que l'API fonctionne
- **Prédictions** : Faire des prédictions sur le churn
- **Monitoring** : Pour les Kubernetes probes

### Health Checks

**Endpoint:** `GET /health`  
**Utilisé par:** livenessProbe (toutes les 30 secondes)  
**Retour:** 
```json
{
  "status": "ok",
  "current_model": "churn_model_v1_20260115_020000.joblib"
}
```
**Signification:** "L'API est vivante et le modèle est chargé"

---

**Endpoint:** `GET /startup`  
**Utilisé par:** startupProbe (pendant le démarrage)  
**Retour:**
```json
{
  "status": "ok",
  "current_model": "churn_model_v1_20260115_020000.joblib"
}
```
**Signification:** "L'application a démarré correctement"  
**Retour 503 si:** Registry manquant, current_model.txt vide, etc.

---

**Endpoint:** `GET /ready`  
**Utilisé par:** readinessProbe (toutes les 10 secondes)  
**Retour:**
```json
{
  "status": "ready",
  "current_model": "churn_model_v1_20260115_020000.joblib"
}
```
**Signification:** "L'API est prête à recevoir du trafic"  
**Retour 503 si:** Modèle ne peut pas être chargé

### Prédictions

- `POST /predict` - Prédiction de churn avec features client

**Exemple:**
```bash
curl -X POST http://127.0.0.1:30080/predict \
  -H "Content-Type: application/json" \
  -d '{
    "tenure_months": 48,
    "num_complaints": 1,
    "avg_session_minutes": 45,
    "plan_type": "premium",
    "region": "EU",
    "request_id": "req-001"
  }'
```


---

## 📊 Monitoring et Drift Detection

### Préparation des données

**Fichier:** `src/prepare_data.py`

Génère `registry/train_stats.json` contenant :
- Moyennes des features numériques
- Écart-types pour chaque feature

**Commande:**
```bash
$pod = "churn-api-7655fd649b-tg6fz"
kubectl exec -it $pod -c api -- python src/prepare_data.py
```



---

### Monitoring de dérive

**Fichier:** `src/monitor_drift.py`

Détecte les changements dans les distributions des features en production :
- Compare les moyennes observées aux moyennes d'entraînement
- Utilise un score Z pour mesurer la dérive
- Déclenche une alerte si Z ≥ seuil (défaut: 2.0)

**Commande:**
```bash
kubectl exec -it $pod -c api -- python src/monitor_drift.py
```

**Résultat attendu:**
```
=== Drift check sur N requêtes récentes ===
- tenure_months: mean_prod=... | mean_train=... | z=...
- num_complaints: mean_prod=... | mean_train=... | z=...
- avg_session_minutes: mean_prod=... | mean_train=... | z=...
Résultat : aucun drift détecté.
```



---

## 🤖 Automation : Jobs et CronJobs

### Job d'Entraînement

**Fichier:** `k8s/job-train.yaml`

Lance un job unique pour entraîner le modèle avec accès au PVC.

**Commandes:**
```bash
kubectl apply -f k8s/job-train.yaml
kubectl get jobs
kubectl logs job/churn-train
```


---

### CronJob de Monitoring

**Fichier:** `k8s/cron-drift.yaml` (à créer)

Lance automatiquement le drift monitoring selon un calendrier.

**Template:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: churn-drift-monitor
spec:
  schedule: "*/5 * * * *"  # Toutes les 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          volumes:
            - name: churn-volume
              persistentVolumeClaim:
                claimName: churn-storage
          containers:
            - name: drift
              image: churn-api:v1
              command: ["python", "src/monitor_drift.py"]
              volumeMounts:
                - name: churn-volume
                  mountPath: /app/registry
                  subPath: registry
                - name: churn-volume
                  mountPath: /app/logs
                  subPath: logs
```



---

## 🔐 NetworkPolicy (Optionnel)

**Fichier:** `k8s/networkpolicy.yaml`

Restreint le trafic réseau entre pods.



---

## 📋 Checklist de Déploiement

- [ ] Minikube ou Docker Desktop Kubernetes démarré
- [ ] Image Docker construite : `churn-api:v1`
- [ ] PVC créée et montée
- [ ] ConfigMap configurée
- [ ] Secret configuré
- [ ] Deployment déployée
- [ ] Service exposée (NodePort sur 30080)
- [ ] Health checks fonctionnels
- [ ] Job d'entraînement exécuté
- [ ] Prédictions testées via `/predict`
- [ ] Drift monitoring exécuté

---

## 🧪 Tests et Validation

### Vérifier les pods
```bash
kubectl get pods -l app=churn-api -o wide
kubectl describe pod <pod-name>
```



---

### Accéder à l'API

**Via port-forward:**
```bash
kubectl port-forward svc/churn-api-service 8000:80
# Accès à http://localhost:8000/docs (Swagger UI)
```

**Via NodePort:**
```
http://127.0.0.1:30080/docs
```



---

### Vérifier les logs de pod

```bash
$pod = "churn-api-7655fd649b-tg6fz"
kubectl logs $pod -c api
kubectl logs $pod -c api --tail=50 -f
```



---

## 📁 Structure des Fichiers K8s

```
k8s/
├── deployment.yaml          # Déploiement principal
├── service.yaml             # Service d'exposition
├── configmap.yaml           # Variables de configuration
├── secret.yaml              # Variables sensibles
├── pvc.yaml                 # Volume persistant
├── job-train.yaml           # Job d'entraînement
├── cron-drift.yaml          # CronJob de monitoring
└── networkpolicy.yaml       # Politique réseau
```

---

## 🚀 Commandes Utiles

```bash
# Déployer tous les fichiers K8s
kubectl apply -f k8s/

# Voir tous les ressources
kubectl get all

# Supprimer le déploiement
kubectl delete deployment churn-api
kubectl delete svc churn-api-service

# Port forward
kubectl port-forward svc/churn-api-service 8000:80

# Exec dans un pod
$pod = kubectl get pods -l app=churn-api -o jsonpath="{.items[0].metadata.name}"
kubectl exec -it $pod -c api -- bash
```

---

## 📝 Notes et Problèmes Rencontrés

### Problème : `LogisticRegression multi_class attribute error`
**Solution:** Retrainer le modèle avec la version actuelle de scikit-learn.
```bash
python src/train.py
```

### Problème : `Kubernetes refuses connection`
**Solution:** Démarrer Minikube ou activer Kubernetes dans Docker Desktop.
```bash
minikube start
```

### Problème : `train_stats.json introuvable`
**Solution:** Exécuter `prepare_data.py` dans le pod pour générer les fichiers.
```bash
kubectl exec -it $pod -c api -- python src/prepare_data.py
```

---

## 📚 Ressources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Minikube Guide](https://minikube.sigs.k8s.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Scikit-learn Documentation](https://scikit-learn.org/)

---

**Auteur:** Lab MLOps  
**Date:** Janvier 2026  
**Statut:** ✅ En cours / ⚠️ À completer


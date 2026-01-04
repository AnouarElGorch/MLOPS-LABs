# Lab 4 : Mise en place d'un pipeline CI/CD complet pour un projet Machine Learning

Ce Lab couvre les étapes essentielles pour mettre en place un pipeline CI/CD (Continuous Integration / Continuous Deployment) complet pour votre projet Machine Learning, en utilisant GitHub Actions.

---

## Étape 1 : Créer le dépôt GitHub et connecter le remote

**Description :**
Créer un dépôt GitHub pour votre projet et le connecter à votre dépôt local. Cela permet de synchroniser votre code et de déclencher automatiquement les workflows CI/CD.

**Instructions :**

1. Aller sur **GitHub → New Repository**
2. Nommer le dépôt : `mlops-lab-01`
3. Copier l'URL HTTPS du dépôt

**Connecter le remote local :**

```bash
git remote add origin https://github.com/<USER>/mlops-lab-01.git
git branch -M main
git push -u origin main
```

**Screenshot :**

<img width="1881" height="768" alt="image" src="https://github.com/user-attachments/assets/80e86cc1-a942-4f54-ab4f-903152b1e43f" />


---

## Étape 2 : Définir les secrets et variables GitHub

**Description :**
Les secrets et variables GitHub permettent de stocker des informations sensibles et des configurations sans les exposer dans le code. Ils sont utilisés par les workflows pour l'authentification, la configuration, et les paramètres d'environnement.

**Instructions :**

Aller dans : **GitHub → Repository → Settings → Secrets and variables → Actions → New repository secret**

**Créer les variables et secrets suivants :**

| Nom | Type | Valeur |
|-----|------|--------|
| `PY_VERSION` | Variable | `3.10` |
| `F1_GATE_THRESHOLD` | Variable | `0.70` |
| `DEMO_SECRET` | Secret | `CI/CD demo secret for students` |
| `APP_ENV` | Variable | `staging` |

**Utilisation dans les workflows :**
```yaml
env:
  PY_VERSION: ${{ vars.PY_VERSION }}
  F1_GATE_THRESHOLD: ${{ vars.F1_GATE_THRESHOLD }}
  DEMO_SECRET: ${{ secrets.DEMO_SECRET }}
  APP_ENV: ${{ vars.APP_ENV }}
```

**Screenshot (Configuration) :**

<img width="956" height="612" alt="image" src="https://github.com/user-attachments/assets/33c5e4f8-4be8-494a-938a-502a7fb5040c" />

**Screenshot (Secrets créés) :**



<img width="997" height="475" alt="image" src="https://github.com/user-attachments/assets/35606e98-8dea-4197-979c-0c141e28c0de" />

---

## Étape 3 : Créer le workflow CI/CD

**Description :**
Le workflow CI/CD automatise les étapes de test et de déploiement. Il contient deux jobs principaux :
- **Job `ci`** : Installe Python, exécute les scripts, et upload les artefacts (testé à chaque push)
- **Job `cd`** : Simule un déploiement SSH (uniquement sur la branche `main`)

**Fichier à créer : `.github/workflows/ci-cd.yml`**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  PY_VERSION: ${{ vars.PY_VERSION }}
  F1_GATE_THRESHOLD: ${{ vars.F1_GATE_THRESHOLD }}
  DEMO_SECRET: ${{ secrets.DEMO_SECRET }}
  APP_ENV: ${{ vars.APP_ENV }}

jobs:
  ci:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ env.PY_VERSION }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PY_VERSION }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    
    - name: Prepare data
      run: python src/prepare_data.py
    
    - name: Train model
      run: python src/train.py
    
    - name: Evaluate model
      run: python src/evaluate.py
    
    - name: Upload model artifact
      uses: actions/upload-artifact@v3
      with:
        name: trained-model
        path: models/
        retention-days: 30
    
    - name: Upload metrics artifact
      uses: actions/upload-artifact@v3
      with:
        name: metrics-report
        path: reports/
        retention-days: 30

  cd:
    needs: ci
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Download artifacts
      uses: actions/download-artifact@v3
      with:
        path: artifacts/
    
    - name: Simulate SSH deployment
      run: |
        echo "Deploying to staging environment: ${{ env.APP_ENV }}"
        echo "F1 Score must be above: ${{ env.F1_GATE_THRESHOLD }}"
        echo "Secret demo: ${{ env.DEMO_SECRET }}"
        ls -la artifacts/
    
    - name: Verify deployment
      run: echo "Deployment completed successfully!"
```

**Screenshot :**

<img width="375" height="66" alt="image" src="https://github.com/user-attachments/assets/2fdf813d-c643-41ba-ab1d-8d652bbf4f99" />


---

## Étape 4 : Commit, push et vérification des workflows

**Description :**
Commit le workflow sur GitHub et vérifiez son exécution dans l'interface GitHub Actions. Consultez les logs pour voir les détails de chaque étape.

**Instructions :**

```bash
git add .github/workflows/ci-cd.yml
git commit -m "Add CI/CD pipeline workflow"
git push origin main
```

**Vérifier l'exécution :**

1. Allez dans : **GitHub → Actions**
2. Cliquez sur le workflow en cours d'exécution
3. Consultez les logs de chaque job

<img width="1372" height="551" alt="image" src="https://github.com/user-attachments/assets/a62c99fb-ba65-4b51-bcbb-a907ffb165ad" />


**Éléments à vérifier :**

### ✅ Job `ci` (Continuous Integration)
- Installation de Python ${{ env.PY_VERSION }}
- Installation des dépendances
- Exécution des scripts (prepare_data, train, evaluate)
- Upload des artefacts (modèles et métriques)

**Screenshot (Job CI) :**

<img width="1849" height="851" alt="image" src="https://github.com/user-attachments/assets/51818db3-d9c3-4f25-b482-8dcc0e1fb3c7" />


### ✅ Job `cd` (Continuous Deployment)
- Téléchargement des artefacts
- Simulation du déploiement SSH
- Vérification des variables d'environnement
- Message de déploiement réussi

**Screenshot (Job CD) :**

<img width="1915" height="657" alt="image" src="https://github.com/user-attachments/assets/edc5bc3e-8e2d-4476-a402-c9e4fe66d54b" />


### 📦 Artefacts produits
Les artefacts suivants sont générés pendant l'exécution :
- **trained-model** : Modèles entraînés dans `models/`
- **metrics-report** : Rapports de métriques dans `reports/`

**Screenshot (Artefacts) :**

<img width="1430" height="245" alt="image" src="https://github.com/user-attachments/assets/94058ed0-2bec-4e8b-8935-967ace7d595c" />


---

## Résumé et Bonnes Pratiques

**Checklist de validation :**
- ✅ Dépôt GitHub créé et connecté
- ✅ Variables et secrets GitHub configurés
- ✅ Workflow CI/CD créé et actif
- ✅ Job CI exécuté avec succès
- ✅ Job CD déployé sur la branche `main`
- ✅ Artefacts générés et téléchargeables


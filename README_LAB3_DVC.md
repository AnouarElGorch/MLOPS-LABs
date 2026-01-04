# Lab3 : Versionnement des données et pipelines ML avec DVC

Ce Lab couvre les étapes essentielles pour mettre en place le versionnement des données et des pipelines de Machine Learning en utilisant DVC (Data Version Control).

## Étape 1 : Initialisation de DVC dans le projet

**Description :**
La première étape consiste à installer et initialiser DVC dans votre projet. Cela crée les fichiers de configuration nécessaires (.dvc) pour commencer à suivre vos données et modèles.

Commandes typiques :
```bash
pip install dvc
dvc init
```

![Capture d'écran Étape 1 : Initialisation](placeholder_screen_1.png)

---

## Étape 2 : Versionner les données brutes avec DVC

**Description :**
Une fois DVC initialisé, nous ajoutons nos données brutes (par exemple, `data/raw.csv`) au suivi de DVC. Cela crée un fichier `.dvc` (ex: `raw.csv.dvc`) qui sert de pointeur vers les données réelles, tandis que le fichier de données lui-même est ajouté au `.gitignore`.

Commandes typiques :
```bash
dvc add data/raw.csv
git add data/raw.csv.dvc data/.gitignore
git commit -m "Add raw data"
```

![Capture d'écran Étape 2 : Versionner les données](placeholder_screen_2.png)

---

## Étape 3 : Configuration d'un remote DVC

**Description :**
Pour partager les données avec d'autres collaborateurs ou pour les sauvegarder, nous configurons un stockage distant ("remote"). Ici, nous simulons un remote local (un dossier sur le disque), mais cela pourrait être un bucket S3, Google Drive, etc.

Commandes typiques :
```bash
mkdir dvc_storage
dvc remote add -d localremote dvc_storage
git add .dvc/config
git commit -m "Configure local remote"
```

![Capture d'écran Étape 3 : Configuration remote](placeholder_screen_3.png)

---

## Étape 4 : Push des données dans le remote DVC

**Description :**
Maintenant que le remote est configuré, nous pouvons pousser (push) nos données versionnées vers ce stockage distant. Cela assure que les données sont sécurisées et accessibles.

Commandes typiques :
```bash
dvc push
```

![Capture d'écran Étape 4 : Push des données](placeholder_screen_4.png)

---

## Étape 5 : Simulation d’une collaboration

**Description :**
Pour vérifier que le versionnement fonctionne, nous simulons une collaboration en supprimant les données locales (`data/raw.csv` et le cache). Ensuite, nous utilisons `dvc pull` pour récupérer les données depuis le remote, prouvant ainsi la reproductibilité de l'environnement.

Commandes typiques :
```bash
rm data/raw.csv
rm -rf .dvc/cache
dvc pull
```

![Capture d'écran Étape 5 : Simulation collaboration](placeholder_screen_5.png)

---

## Étape 6 : Création d’un pipeline reproductible dvc.yaml

**Description :**
DVC permet de définir des pipelines ML complets dans un fichier `dvc.yaml`. Ce fichier spécifie les étapes (stages) comme la préparation (`prepare`), l'entraînement (`train`) et l'évaluation (`evaluate`), ainsi que leurs dépendances et sorties. Cela garantit que chaque étape n'est réexécutée que si ses dépendances ont changé.

Exemple de structure `dvc.yaml` :
```yaml
stages:
  prepare:
    cmd: python src/prepare_data.py
    deps: ...
    outs: ...
  train:
    cmd: python src/train.py
    deps: ...
    outs: ...
  evaluate:
    cmd: python src/evaluate.py
    deps: ...
```

![Capture d'écran Étape 6 : Pipeline dvc.yaml](placeholder_screen_6.png)

---

## Étape 7 : Reproduire automatiquement tout le pipeline

**Description :**
Enfin, avec le fichier `dvc.yaml` en place, nous pouvons utiliser la commande `dvc repro` pour exécuter tout le pipeline. DVC analysera le graphe de dépendances et n'exécutera que les étapes nécessaires.

Commandes typiques :
```bash
dvc repro
```

![Capture d'écran Étape 7 : Reproduire le pipeline](placeholder_screen_7.png)

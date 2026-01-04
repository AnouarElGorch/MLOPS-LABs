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
<img width="1016" height="100" alt="image" src="https://github.com/user-attachments/assets/fd9d0ba2-8cc1-46d1-a3c7-e6a25383f41b" />
<img width="782" height="392" alt="image" src="https://github.com/user-attachments/assets/6c3ffd36-227e-4560-8e5f-77cabfbef4d3" />


---

## Étape 2 : Versionner les données brutes avec DVC

**Description :**
Une fois DVC initialisé, nous ajoutons nos données brutes (par exemple, `data/raw.csv`) au suivi de DVC. Cela crée un fichier `.dvc` (ex: `raw.csv.dvc`) qui sert de pointeur vers les données réelles, tandis que le fichier de données lui-même est ajouté au `.gitignore`.
<img width="266" height="208" alt="image" src="https://github.com/user-attachments/assets/f32dab01-7c0d-4436-a1c9-0adf11fead53" />

Commandes typiques :
```bash
dvc add data/raw.csv
git add data/raw.csv.dvc data/.gitignore
git commit -m "Add raw data"
```
<img width="1018" height="244" alt="image" src="https://github.com/user-attachments/assets/2245a68a-8ce0-4288-87af-2e3d851bdf22" />
<img width="386" height="136" alt="image" src="https://github.com/user-attachments/assets/8a923b82-d670-4b53-b418-fef45924d67b" />

<img width="1001" height="197" alt="image" src="https://github.com/user-attachments/assets/927d6e49-05bf-4c2e-8723-353f64bd2904" />


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

<img width="946" height="311" alt="image" src="https://github.com/user-attachments/assets/0598e203-8420-4f14-aa19-300b77e4fdc8" />
<img width="1003" height="132" alt="image" src="https://github.com/user-attachments/assets/3ae638bf-62aa-4a43-9157-0e0e7f25b1c7" />


---

## Étape 4 : Push des données dans le remote DVC

**Description :**
Maintenant que le remote est configuré, nous pouvons pousser (push) nos données versionnées vers ce stockage distant. Cela assure que les données sont sécurisées et accessibles.

Commandes typiques :
```bash
dvc push
```
<img width="978" height="94" alt="image" src="https://github.com/user-attachments/assets/ff5ce107-f085-445a-9ba6-29655a49e19e" />
<img width="1114" height="546" alt="image" src="https://github.com/user-attachments/assets/8606b6a8-c88a-4322-8b0a-472f410e6c6c" />


---

## Étape 5 : Simulation d’une collaboration

**Description :**
Pour vérifier que le versionnement fonctionne, nous simulons une collaboration en supprimant les données locales (`data/raw.csv` et le cache). Ensuite, nous utilisons `dvc pull` pour récupérer les données depuis le remote, prouvant ainsi la reproductibilité de l'environnement.


<img width="723" height="275" alt="image" src="https://github.com/user-attachments/assets/cd3183a7-0783-4d3f-b48b-64b79424074f" />


Commandes typiques :
```bash
rm data/raw.csv
rm -rf .dvc/cache
dvc pull
```
<img width="996" height="182" alt="image" src="https://github.com/user-attachments/assets/28fddf69-9315-4753-a681-5e1c4535480a" />

<img width="581" height="165" alt="image" src="https://github.com/user-attachments/assets/513fbb64-b318-4b76-ac4a-b4d40026d97c" />



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

<img width="499" height="615" alt="image" src="https://github.com/user-attachments/assets/5972d453-0378-41a2-884d-11a7e99c024d" />
<img width="993" height="224" alt="image" src="https://github.com/user-attachments/assets/b4576b7d-fa23-43af-a7fa-ffafbe3b51bd" />

---

## Étape 7 : Reproduire automatiquement tout le pipeline

**Description :**
Enfin, avec le fichier `dvc.yaml` en place, nous pouvons utiliser la commande `dvc repro` pour exécuter tout le pipeline. DVC analysera le graphe de dépendances et n'exécutera que les étapes nécessaires.

Commandes typiques :
```bash
dvc repro
```
<img width="1002" height="376" alt="image" src="https://github.com/user-attachments/assets/6087e514-e034-4533-91c7-3567b2e90052" />
<img width="959" height="163" alt="image" src="https://github.com/user-attachments/assets/c243db31-271b-4dbc-84f0-f009730c9971" />



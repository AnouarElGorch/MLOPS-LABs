# Lab5 : Du Notebook au déploiement conteneurisé d’un modèle de Machine Learning

## Étape 1 : Vérifier l’installation de Docker

Instructions :
- Ouvrez un terminal (PowerShell ou bash).
- Vérifiez la version de Docker :

```bash
docker --version
```

- Vérifiez que le démon Docker fonctionne :

```bash
docker ps
```

<img width="875" height="159" alt="image" src="https://github.com/user-attachments/assets/8e423772-887d-4f47-b657-721622e41104" />


---

## Étape 2 : Lancer un serveur Nginx dans un conteneur

Instructions :
- Lancez un conteneur Nginx en arrière-plan :

```bash
docker run -d -p 8080:80 --name demo-nginx nginx
```

<img width="987" height="326" alt="image" src="https://github.com/user-attachments/assets/b1ad7ddd-0ac8-4e6b-88f8-745993b8fe40" />


- Ouvrez un navigateur et accédez à :

```
http://localhost:8080
```

<img width="1138" height="432" alt="image" src="https://github.com/user-attachments/assets/d1d22e9e-44b4-4c70-8a51-9c54fa77f603" />


- Vérifiez que la page par défaut de Nginx s’affiche.
- Listez les conteneurs en cours d’exécution :

```bash
docker ps
```

<img width="1458" height="136" alt="image" src="https://github.com/user-attachments/assets/55272d5d-05ed-452f-8718-3dbb0afabe6d" />


- Arrêtez puis supprimez le conteneur :

```bash
docker stop demo-nginx
docker rm demo-nginx
```

<img width="514" height="113" alt="image" src="https://github.com/user-attachments/assets/63350b8e-f44a-47ca-a6fd-a15e133a2e4a" />


---

## Étape 3 : Ouvrir un shell Linux isolé dans un conteneur

Instructions :
- Lancez un conteneur Linux interactif (exemple avec ubuntu) :

```bash
docker run -it --name demo-ubuntu ubuntu bash
```

<img width="976" height="165" alt="image" src="https://github.com/user-attachments/assets/912c68a2-0aea-42a8-96d6-0cc89a311d9d" />


- Dans le shell à l’intérieur du conteneur, exécutez quelques commandes :

```bash
ls
cat /etc/os-release
pwd
```

<img width="1234" height="449" alt="image" src="https://github.com/user-attachments/assets/4b415da2-91a8-43f3-bf7a-075ca8adf949" />


- Installez un paquet (exemple) :

```bash
apt-get update
apt-get install -y curl
```

<img width="1285" height="644" alt="image" src="https://github.com/user-attachments/assets/113afc25-55f6-4c57-b505-bf0673d7a016" />


- Quittez le shell :

```bash
exit
```

<img width="361" height="82" alt="image" src="https://github.com/user-attachments/assets/86731208-5e51-40be-8c63-b8d27cc54f7b" />


- Vérifiez que le conteneur existe toujours mais est arrêté :

```bash
docker ps -a
```

<img width="1213" height="97" alt="image" src="https://github.com/user-attachments/assets/b15fa43f-b23e-4323-8388-b86ea763ae9c" />


- Supprimez ce conteneur :

```bash
docker rm demo-ubuntu
```

<img width="509" height="78" alt="image" src="https://github.com/user-attachments/assets/128d2da4-eb83-407c-809f-ddf266c7536d" />


---

## Étape 4 : Comprendre la structure d’une commande docker run

Instructions :
- Structure générale :

```bash
docker run [options] image [commande] [arguments]
```

- Lancez de nouveau Nginx avec des options clairement visibles :

<img width="847" height="79" alt="image" src="https://github.com/user-attachments/assets/6ef0ef12-9fdc-4c43-b367-7387ac6521b5" />


- Identifiez le rôle de chaque élément :
  - `-d` : détaché (arrière-plan)
  - `--name demo-nginx` : nom du conteneur
  - `-p 8080:80` : port hôte 8080 → port conteneur 80
  - `nginx` : image utilisée
- Arrêtez et supprimez encore une fois le conteneur :

<img width="506" height="123" alt="image" src="https://github.com/user-attachments/assets/1c176394-c0bd-4a26-82d2-bed7f1db1058" />


---

## Étape 5 : Conteneuriser l’API churn du projet mlops-lab-01

- Vérifiez que l’arborescence contient au minimum :

```bash
mlops-lab-01/
 ├── data/
 ├── logs/
 ├── models/
 ├── registry/
 └── src/
```

<img width="506" height="123" alt="image" src="https://github.com/user-attachments/assets/29a3eff6-5aab-40be-b3f6-06987415e066" />


- Vérifiez que l’API fonctionne encore en local (optionnel mais recommandé) :

<img width="675" height="180" alt="image" src="https://github.com/user-attachments/assets/fe8f9f0a-b62f-49b4-82dc-47bafa447b8e" />

<img width="736" height="48" alt="image" src="https://github.com/user-attachments/assets/0c1abac3-f4fb-419a-b27b-020ef64ea20c" />

---

## Étape 6 : Créer un fichier requirements.txt pour l’image Docker

Instructions :
1. Dans le dossier `mlops-lab-01`, créez un fichier `requirements.txt`.
2. Ajoutez le contenu suivant (version minimale) :

```
fastapi
uvicorn[standard]
pydantic
scikit-learn
pandas
numpy
joblib
```

3. Enregistrez le fichier. Remarque : ajoutez vos libs supplémentaires si besoin (`python-dotenv`, `loguru`, etc.).

<img width="386" height="309" alt="image" src="https://github.com/user-attachments/assets/f51a454b-0020-44e1-9fa8-92bdeb0c81da" />


---

## Étape 7 : Créer un Dockerfile pour l’API churn

Instructions :
1. Dans le dossier `mlops-lab-01`, créez un fichier nommé `Dockerfile`.
2. Collez le contenu attendu (voir capture) puis sauvegardez.

<img width="794" height="510" alt="image" src="https://github.com/user-attachments/assets/725185a2-288b-46fd-acc2-32b6473fe383" />


---

## Étape 8 : Préparer un modèle actif avant de construire l’image

Instructions :
1. Assurez-vous qu’un modèle entraîné existe déjà dans `models/` :

<img width="809" height="334" alt="image" src="https://github.com/user-attachments/assets/56394120-bcfd-41ce-a2a7-8318fea4ac26" />


2. Assurez-vous que `registry/current_model.txt` contient le nom du modèle actif :

<img width="596" height="114" alt="image" src="https://github.com/user-attachments/assets/1bb2e4fd-e505-4711-9dbd-da5be6317acb" />


3. Vérifiez à nouveau le contenu du répertoire `models/` :

<img width="907" height="101" alt="image" src="https://github.com/user-attachments/assets/02939b9a-15d1-4e9c-a175-f49aa9dd4cda" />


---

## Étape 9 : Construire l’image Docker du projet churn

Instructions :
1. Dans `mlops-lab-01`, construisez l’image :

<img width="1374" height="697" alt="image" src="https://github.com/user-attachments/assets/f2d44862-46aa-47aa-a857-4f1249fbc030" />


2. Vérifiez la présence de l’image (`churn-api` dans REPOSITORY) :

<img width="688" height="169" alt="image" src="https://github.com/user-attachments/assets/7588d015-45ec-4f9f-b56c-5b55f0e86eca" />


---

## Étape 10 : Lancer l’API churn dans un conteneur

Instructions :
1. Lancez un conteneur basé sur l’image et vérifiez qu’il tourne :

<img width="1377" height="160" alt="image" src="https://github.com/user-attachments/assets/edceccf6-833e-4de8-b792-b50a9ab47985" />


2. Testez le endpoint `/health` (curl, Postman, navigateur) :

<img width="1081" height="480" alt="image" src="https://github.com/user-attachments/assets/8c6debd1-cb50-486e-8a74-ea255ea39845" />


3. Testez une requête POST `/predict` avec un JSON du lab (tenure, complaints, etc.) :

<img width="1443" height="832" alt="image" src="https://github.com/user-attachments/assets/f584da1b-bac3-4b82-9d7a-6b9747edee94" />

> Si vous rencontrez un problème de version scikit-learn 1.8 non compatible avec Python 3.10, modifiez l’image vers Python 3.11 dans le Dockerfile.

---

## Étape 11 : Vérifier les logs générés à l’intérieur du conteneur

Instructions :
1. Listez les fichiers dans le conteneur :

<img width="972" height="220" alt="image" src="https://github.com/user-attachments/assets/656102b8-db72-42cc-bc33-57e6cf02dc84" />


2. Vérifiez que l’application écrit des logs à l’exécution :

<img width="933" height="49" alt="image" src="https://github.com/user-attachments/assets/4e849cdb-8937-428a-8e20-6a4028d6e233" />


3. Affichez quelques lignes du fichier de logs des prédictions :

<img width="1006" height="363" alt="image" src="https://github.com/user-attachments/assets/200e1ccb-0e72-4e92-bfeb-95fee84a57ef" />


4. Arrêtez et supprimez le conteneur après test :

<img width="820" height="128" alt="image" src="https://github.com/user-attachments/assets/06e88f55-3f16-48eb-a0f9-0abd66a51cdd" />


---

## Étape 12 : Orchestration locale avec Docker Compose

<img width="720" height="472" alt="image" src="https://github.com/user-attachments/assets/16f9e352-68e1-4935-9433-db9b7ec4e4ac" />


---

## Étape 13 : Démarrer l’API via Docker Compose

<img width="998" height="272" alt="image" src="https://github.com/user-attachments/assets/f109c1be-b043-4728-a4c8-46a70586933d" />


- Test GET :

<img width="956" height="429" alt="image" src="https://github.com/user-attachments/assets/3d8ad4cc-0353-48d1-906e-1c73846215fe" />


- Test POST :

<img width="931" height="827" alt="image" src="https://github.com/user-attachments/assets/87e1c961-088a-4c45-808f-057f3bd2f78e" />


- Arrêt :

<img width="992" height="189" alt="image" src="https://github.com/user-attachments/assets/35e5f976-a832-4be7-bfed-490456f9a0b7" />


---

## Étape 14 : Lancer les services en arrière-plan et observer les logs

Instructions :
- Lancez les services en mode détaché.

<img width="982" height="137" alt="image" src="https://github.com/user-attachments/assets/37483010-b4e4-4d07-a2a6-fce357aa1e42" />


- Vérifiez les conteneurs en cours d’exécution :

<img width="979" height="149" alt="image" src="https://github.com/user-attachments/assets/567725d4-be3e-468b-91fd-1fde933b0277" />


- Affichez les logs du service :

<img width="986" height="493" alt="image" src="https://github.com/user-attachments/assets/72d61c15-1848-4bac-972d-deee72b9eac5" />


- Testez `/health` et `/predict` pendant que les logs défilent. Arrêtez les services :

<img width="779" height="788" alt="image" src="https://github.com/user-attachments/assets/b2e6109e-97ca-4b64-8cd0-4f18efc4da92" />

<img width="659" height="314" alt="image" src="https://github.com/user-attachments/assets/aba7d4be-3e32-4e31-b018-a16befc2b9d6" />

<img width="993" height="182" alt="image" src="https://github.com/user-attachments/assets/9ef06048-a352-4dd1-84e6-855843f35706" />


---

## Étape 15 : Lier Docker Compose au reste du cours (Git + DVC)

Assurez-vous que :
- le projet `mlops-lab-01` est versionné avec Git (lab Git),
- les données et modèles lourds sont suivis par DVC (lab DVC),
- l’API est conteneurisée via Docker (lab Docker).

<img width="990" height="195" alt="image" src="https://github.com/user-attachments/assets/54ab1050-fbb4-4d35-bbca-a2a0de4c5d20" />


Notez dans votre cours :
- MLOps local : pipeline + API + monitoring
- Git : versionnement du code et de la structure
- DVC : versionnement des données / modèles
- Docker / Compose : déploiement reproductible de l’API

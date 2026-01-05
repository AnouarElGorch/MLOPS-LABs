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

![image.png](attachment:1211d5a0-4ccb-4c39-b824-82502ba4370e:image.png)

---

## Étape 2 : Lancer un serveur Nginx dans un conteneur

Instructions :
- Lancez un conteneur Nginx en arrière-plan :

```bash
docker run -d -p 8080:80 --name demo-nginx nginx
```

![image.png](attachment:efd20580-1fef-4163-a0cd-b9a802f624f4:image.png)

- Ouvrez un navigateur et accédez à :

```
http://localhost:8080
```

![image.png](attachment:d6e2d566-f19d-47fd-8180-63595e518798:image.png)

- Vérifiez que la page par défaut de Nginx s’affiche.
- Listez les conteneurs en cours d’exécution :

```bash
docker ps
```

![image.png](attachment:241ae74b-3cc6-4a25-b857-259a65870fc1:image.png)

- Arrêtez puis supprimez le conteneur :

```bash
docker stop demo-nginx
docker rm demo-nginx
```

![image.png](attachment:e34df6e6-8c13-412a-991b-4c9801620485:image.png)

---

## Étape 3 : Ouvrir un shell Linux isolé dans un conteneur

Instructions :
- Lancez un conteneur Linux interactif (exemple avec ubuntu) :

```bash
docker run -it --name demo-ubuntu ubuntu bash
```

![image.png](attachment:2c8c47c8-dfed-4c74-a934-0a3640df50dd:image.png)

- Dans le shell à l’intérieur du conteneur, exécutez quelques commandes :

```bash
ls
cat /etc/os-release
pwd
```

![image.png](attachment:6416ea8f-bccd-49f0-950c-8f8afb2e6191:image.png)

- Installez un paquet (exemple) :

```bash
apt-get update
apt-get install -y curl
```

![image.png](attachment:79851f81-cc64-44a9-871c-729b0b7cd989:image.png)

- Quittez le shell :

```bash
exit
```

![image.png](attachment:a5c75f80-9459-46b7-9d72-705eaaa9b9ca:image.png)

- Vérifiez que le conteneur existe toujours mais est arrêté :

```bash
docker ps -a
```

![image.png](attachment:7dcf006c-37d2-4170-bcba-ccbf2470d317:image.png)

- Supprimez ce conteneur :

```bash
docker rm demo-ubuntu
```

![image.png](attachment:f754a5d2-933f-4f13-b66f-a0f313b8096e:image.png)

---

## Étape 4 : Comprendre la structure d’une commande docker run

Instructions :
- Structure générale :

```bash
docker run [options] image [commande] [arguments]
```

- Lancez de nouveau Nginx avec des options clairement visibles :

![image.png](attachment:563b2a9f-1948-4ece-b3e9-75fa77e44d66:image.png)

- Identifiez le rôle de chaque élément :
  - `-d` : détaché (arrière-plan)
  - `--name demo-nginx` : nom du conteneur
  - `-p 8080:80` : port hôte 8080 → port conteneur 80
  - `nginx` : image utilisée
- Arrêtez et supprimez encore une fois le conteneur :

![image.png](attachment:98e482e4-35c2-457c-9fb5-ac03c8d9b400:image.png)

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

![image.png](attachment:abe785b6-6c0d-4b8c-8afd-28f788ada93b:image.png)

- Vérifiez que l’API fonctionne encore en local (optionnel mais recommandé) :

![image.png](attachment:a5ec6830-87c5-4817-816a-d1a9125edb55:image.png)
![image.png](attachment:0a20f2fb-c444-482a-a552-b068c5b95980:image.png)

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

![image.png](attachment:d60975c9-6db1-4846-a19f-2501627fd678:image.png)

---

## Étape 7 : Créer un Dockerfile pour l’API churn

Instructions :
1. Dans le dossier `mlops-lab-01`, créez un fichier nommé `Dockerfile`.
2. Collez le contenu attendu (voir capture) puis sauvegardez.

![image.png](attachment:f8d04c0d-0b79-4172-aac9-b6dc19a9ea5d:image.png)

---

## Étape 8 : Préparer un modèle actif avant de construire l’image

Instructions :
1. Assurez-vous qu’un modèle entraîné existe déjà dans `models/` :

![image.png](attachment:4d73ddee-794e-42ce-90f4-4b8e655d0cb8:image.png)

2. Assurez-vous que `registry/current_model.txt` contient le nom du modèle actif :

![image.png](attachment:3f722c49-925b-4db9-afe6-eda71e00cafc:image.png)

3. Vérifiez à nouveau le contenu du répertoire `models/` :

![image.png](attachment:77f15d52-3e55-415c-a8ec-8e43ce8fe7fd:image.png)

---

## Étape 9 : Construire l’image Docker du projet churn

Instructions :
1. Dans `mlops-lab-01`, construisez l’image :

![image.png](attachment:bd847fce-1307-4427-b865-a19e7ad199a8:image.png)

2. Vérifiez la présence de l’image (`churn-api` dans REPOSITORY) :

![image.png](attachment:0f4dc189-8802-45a5-a457-9d2786fea8d1:image.png)

---

## Étape 10 : Lancer l’API churn dans un conteneur

Instructions :
1. Lancez un conteneur basé sur l’image et vérifiez qu’il tourne :

![image.png](attachment:b7ca5e24-b519-48fb-9b9f-fa8597ca6a4d:image.png)

2. Testez le endpoint `/health` (curl, Postman, navigateur) :

![image.png](attachment:73476f5c-5e2d-4f4a-acec-e715be04276f:image.png)
![image.png](attachment:5b554cb4-e803-4478-a09e-a88393873083:image.png)

3. Testez une requête POST `/predict` avec un JSON du lab (tenure, complaints, etc.) :

![image.png](attachment:9ef3f0b7-e0dd-467d-b76e-3189a2bb95ee:image.png)

> Si vous rencontrez un problème de version scikit-learn 1.8 non compatible avec Python 3.10, modifiez l’image vers Python 3.11 dans le Dockerfile.

---

## Étape 11 : Vérifier les logs générés à l’intérieur du conteneur

Instructions :
1. Listez les fichiers dans le conteneur :

![image.png](attachment:a869edd5-11fe-42e6-b54f-fcb49d47e00b:image.png)

2. Vérifiez que l’application écrit des logs à l’exécution :

![image.png](attachment:d40af29f-9aae-4379-9d1b-f810f6ea06a5:image.png)

3. Affichez quelques lignes du fichier de logs des prédictions :

![image.png](attachment:8fe98262-4516-4707-92e1-fc2ac68f0b60:image.png)

4. Arrêtez et supprimez le conteneur après test :

![image.png](attachment:73716706-a31d-4a49-954c-d05d2e77ea30:image.png)

---

## Étape 12 : Orchestration locale avec Docker Compose

![image.png](attachment:e2327b91-df2a-4e8e-beff-9a52368de922:image.png)

---

## Étape 13 : Démarrer l’API via Docker Compose

![image.png](attachment:be3c320d-9c28-42e6-a955-2278ab66ec1c:image.png)

- Test GET :

![image.png](attachment:5c83ea28-09c0-49af-9d90-4015205820af:image.png)

- Test POST :

![image.png](attachment:7c8baa2f-c742-4eb2-96cb-e0a2d819eb52:image.png)

- Arrêt :

![image.png](attachment:8964a703-8f8b-41de-935e-6c93afe53f62:image.png)

---

## Étape 14 : Lancer les services en arrière-plan et observer les logs

Instructions :
- Lancez les services en mode détaché.

![image.png](attachment:c3849690-000c-4cea-9afe-6426aaf7c371:image.png)

- Vérifiez les conteneurs en cours d’exécution :

![image.png](attachment:2e11b151-8312-4652-b8eb-f2e53c6a12df:image.png)

- Affichez les logs du service :

![image.png](attachment:e8333428-e05a-435e-b6e8-3d61eaedcb56:image.png)

- Testez `/health` et `/predict` pendant que les logs défilent. Arrêtez les services :

![image.png](attachment:3d578212-8581-4cc2-98f6-ea83b1d68195:image.png)
![image.png](attachment:20e03b7d-fceb-4839-8af8-ebf3a6b6d805:image.png)
![image.png](attachment:57ba35ad-fc2e-4c0d-b922-82837b7019b4:image.png)

---

## Étape 15 : Lier Docker Compose au reste du cours (Git + DVC)

Assurez-vous que :
- le projet `mlops-lab-01` est versionné avec Git (lab Git),
- les données et modèles lourds sont suivis par DVC (lab DVC),
- l’API est conteneurisée via Docker (lab Docker).

![image.png](attachment:2cd92dfb-48e9-42b2-aefb-cdeb414970b2:image.png)

Notez dans votre cours :
- MLOps local : pipeline + API + monitoring
- Git : versionnement du code et de la structure
- DVC : versionnement des données / modèles
- Docker / Compose : déploiement reproductible de l’API

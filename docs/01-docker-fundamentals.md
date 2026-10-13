# Docker Fundamentals — De Dockerfile à un conteneur Django

> **Objectif :** comprendre les fondamentaux de Docker en utilisant une vraie application Django comme laboratoire.
>
> Cette documentation fait partie de mon parcours d'apprentissage Docker → Docker Compose → Production → AWS ECS/Fargate.

---

## 📚 Table des matières

1. [Pourquoi Docker ?](#1-pourquoi-docker)
2. [Le modèle mental de Docker](#2-le-modèle-mental-de-docker)
3. [Image vs conteneur](#3-image-vs-conteneur)
4. [Le Dockerfile](#4-le-dockerfile)
5. [Comprendre chaque instruction](#5-comprendre-chaque-instruction)
6. [Le build context](#6-le-build-context)
7. [`.dockerignore`](#7-dockerignore)
8. [Construire une image](#8-construire-une-image)
9. [Layers et cache Docker](#9-layers-et-cache-docker)
10. [Créer et lancer un conteneur](#10-créer-et-lancer-un-conteneur)
11. [Variables d'environnement](#11-variables-denvironnement)
12. [Les ports : `EXPOSE` vs `-p`](#12-les-ports-expose-vs--p)
13. [`ENTRYPOINT` et `CMD`](#13-entrypoint-et-cmd)
14. [Observer et diagnostiquer un conteneur](#14-observer-et-diagnostiquer-un-conteneur)
15. [`run`, `start` et `restart`](#15-run-start-et-restart)
16. [Supprimer conteneurs et images](#16-supprimer-conteneurs-et-images)
17. [Les erreurs rencontrées pendant l'apprentissage](#17-les-erreurs-rencontrées-pendant-lapprentissage)
18. [Schéma récapitulatif](#18-schéma-récapitulatif)
19. [Commandes essentielles](#19-commandes-essentielles)
20. [Exercices](#20-exercices)
21. [Ce que je retiens](#21-ce-que-je-retiens)
22. [Prochaine étape](#22-prochaine-étape)

---

# 1. Pourquoi Docker ?

Une application ne dépend pas uniquement de son code.

Elle dépend également de :

* la version de Python ;
* les packages Python ;
* les bibliothèques système ;
* les variables d'environnement ;
* la configuration ;
* les services externes comme PostgreSQL ou Redis.

Une application peut donc fonctionner sur une machine et ne pas fonctionner sur une autre.

Docker permet de construire un environnement reproductible pour l'application.

L'idée fondamentale est :

> **Je construis une image contenant ce dont mon application a besoin, puis je peux créer des conteneurs à partir de cette image.**

Dans mon projet Django :

```text
                    Dockerfile
                         │
                         │ docker build
                         ▼
                       IMAGE
                         │
                         │ docker run
                         ▼
                     CONTAINER
                         │
                         ▼
                      Gunicorn
                         │
                         ▼
                       Django
```

Docker ne se limite donc pas à la commande `docker run`.

Il faut comprendre toute la chaîne :

```text
Dockerfile
    ↓
Build context
    ↓
docker build
    ↓
Image
    ↓
docker run
    ↓
Container
    ↓
Process
    ↓
Network / Ports / Environment
```

---

# 2. Le modèle mental de Docker

Le premier concept à comprendre est la différence entre **image** et **conteneur**.

```text
                    IMAGE
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
       CONTAINER 1        CONTAINER 2
```

Une image est un modèle.

Un conteneur est une instance créée à partir de ce modèle.

Une même image peut donc créer plusieurs conteneurs.

Par exemple :

```bash
docker run --name app-1 apprendre-docker-web
docker run --name app-2 apprendre-docker-web
```

Les deux conteneurs utilisent la même image :

```text
apprendre-docker-web
        │
        ├── app-1
        │
        └── app-2
```

Supprimer `app-1` ne supprime pas automatiquement l'image.

---

# 3. Image vs conteneur

## Image

Une image contient notamment :

* un système de fichiers ;
* Python ;
* les dépendances installées ;
* le code de l'application ;
* certaines variables/configurations définies dans l'image ;
* les métadonnées nécessaires au démarrage.

On peut considérer l'image comme un **modèle immuable**.

Pour voir les images :

```bash
docker images
```

---

## Conteneur

Un conteneur est une instance d'une image.

Il possède notamment :

* un nom ;
* un ID ;
* un état ;
* un processus principal ;
* un environnement ;
* une configuration réseau ;
* éventuellement des volumes ;
* éventuellement des ports publiés.

Pour voir les conteneurs en cours :

```bash
docker ps
```

Pour voir tous les conteneurs, même ceux qui sont arrêtés :

```bash
docker ps -a
```

---

## À retenir

```text
IMAGE
  ↓ docker run
CONTAINER
```

L'image existe indépendamment du conteneur.

Ainsi :

```bash
docker rm docker-lab
```

supprime le conteneur, mais l'image :

```text
apprendre-docker-web
```

reste disponible.

Pour supprimer l'image :

```bash
docker rmi apprendre-docker-web
```

---

# 4. Le Dockerfile

Voici le Dockerfile utilisé pour mon application Django :

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .

RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]

CMD ["web"]
```

Ce fichier décrit comment construire l'image et définit également le comportement du conteneur lorsqu'il démarre.

---

# 5. Comprendre chaque instruction

## 5.1 `FROM`

```dockerfile
FROM python:3.11-slim
```

`FROM` définit l'image de base.

Ici, je pars d'une image contenant Python 3.11 sur une base Debian slim.

Je n'ai donc pas besoin d'installer Python moi-même.

```text
python:3.11-slim
        │
        ▼
   mon image
        │
        ├── Python
        ├── système de base
        └── dépendances de mon application
```

---

## 5.2 `WORKDIR`

```dockerfile
WORKDIR /app
```

Définit le répertoire de travail dans l'image.

Les instructions suivantes utilisent donc `/app` comme répertoire courant lorsque cela s'applique.

Par exemple :

```dockerfile
COPY requirements.txt .
```

copie le fichier vers :

```text
/app/requirements.txt
```

---

## 5.3 `RUN`

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

`RUN` exécute une commande **pendant le build de l'image**.

Il ne s'agit pas d'une commande exécutée à chaque démarrage du conteneur.

Ici :

1. les index APT sont mis à jour ;
2. `curl` est installé ;
3. les fichiers d'index APT sont supprimés afin de réduire la taille de la couche.

Autre exemple :

```dockerfile
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt
```

Les dépendances Python sont installées dans l'image pendant le build.

### Différence importante

```text
docker build
     │
     ├── RUN apt-get ...
     ├── RUN pip install ...
     │
     ▼
   IMAGE
```

Puis :

```text
docker run
     │
     ▼
 CONTAINER
```

Les instructions `RUN` appartiennent donc principalement à la phase de **construction**.

---

## 5.4 `ENV`

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
```

Ces variables sont définies dans l'environnement du conteneur issu de cette image.

### `PYTHONDONTWRITEBYTECODE=1`

Demande à Python de ne pas générer les fichiers `.pyc`.

### `PYTHONUNBUFFERED=1`

Permet aux sorties Python d'être disponibles plus directement dans stdout/stderr.

C'est utile dans Docker car les logs peuvent ensuite être récupérés avec :

```bash
docker logs <container>
```

En production, cette logique est particulièrement importante pour envoyer les logs vers un système comme CloudWatch.

---

## 5.5 `COPY`

```dockerfile
COPY requirements.txt .
```

Copie `requirements.txt` depuis le **build context** vers l'image.

Puis :

```dockerfile
COPY . .
```

copie le projet dans `/app`.

Grâce à :

```dockerfile
WORKDIR /app
```

on obtient notamment :

```text
/app
├── manage.py
├── requirements.txt
├── crud/
├── ...
└── entrypoint.sh
```

---

## 5.6 Pourquoi copier `requirements.txt` séparément ?

On utilise :

```dockerfile
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

plutôt que :

```dockerfile
COPY . .

RUN pip install ...
```

L'objectif est de profiter du cache Docker.

Si je modifie uniquement :

```text
views.py
```

Docker peut réutiliser la couche :

```text
RUN pip install ...
```

Il n'a donc pas besoin de réinstaller toutes les dépendances Python.

En revanche, si :

```text
requirements.txt
```

change, la couche d'installation doit être reconstruite.

---

## 5.7 `EXPOSE`

```dockerfile
EXPOSE 8000
```

Cette instruction indique que l'application utilise le port `8000` dans le conteneur.

Mais :

> **`EXPOSE` ne publie pas le port sur ma machine.**

Pour rendre le service accessible depuis l'hôte, je dois utiliser `-p` lors du lancement du conteneur.

---

## 5.8 `ENTRYPOINT`

```dockerfile
ENTRYPOINT ["/app/entrypoint.sh"]
```

Définit le programme principal lancé lorsque le conteneur démarre.

Le script a été copié dans l'image avec :

```dockerfile
COPY entrypoint.sh /app/entrypoint.sh
```

Puis rendu exécutable :

```dockerfile
RUN chmod +x /app/entrypoint.sh
```

Le script permet de lancer différents rôles :

```sh
#!/bin/sh
set -e

case "$1" in
  web)
    echo "Starting Django web server..."
    python manage.py collectstatic --noinput
    exec gunicorn --bind 0.0.0.0:8000 crud.wsgi:application
    ;;

  celery)
    echo "Starting Celery worker..."
    exec celery -A crud worker --loglevel=info
    ;;

  beat)
    echo "Starting Celery Beat..."
    exec celery -A crud beat --loglevel=info
    ;;

  *)
    exec "$@"
    ;;
esac
```

---

## 5.9 `CMD`

```dockerfile
CMD ["web"]
```

Dans mon cas, `CMD` fournit l'argument par défaut à `ENTRYPOINT`.

Docker lance donc conceptuellement :

```bash
/app/entrypoint.sh web
```

Dans le script :

```sh
case "$1" in
```

`$1` vaut :

```text
web
```

Le script entre donc dans la branche :

```sh
web)
```

et démarre Gunicorn.

### Même image, différents rôles

Je peux utiliser la même image avec un autre argument :

```bash
docker run apprendre-docker-web celery
```

Docker lancera alors conceptuellement :

```bash
/app/entrypoint.sh celery
```

Le script choisira la branche `celery`.

Même principe pour Beat :

```bash
docker run apprendre-docker-web beat
```

### À retenir

Dans notre Dockerfile :

```dockerfile
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["web"]
```

signifie :

```text
ENTRYPOINT
     +
CMD
     ↓
/app/entrypoint.sh web
```

`ENTRYPOINT` définit le programme principal.

`CMD` fournit ici son argument par défaut.

---

# 6. Le build context

Lorsque je lance :

```bash
docker build -t apprendre-docker-web .
```

le dernier `.` représente le **build context**.

Le build context correspond aux fichiers que Docker peut utiliser pendant la construction.

C'est notamment ce qui permet :

```dockerfile
COPY requirements.txt .
```

et :

```dockerfile
COPY . .
```

Docker ne peut pas simplement aller lire n'importe quel fichier de mon ordinateur.

Les fichiers doivent faire partie du contexte de build et ne pas être exclus par `.dockerignore`.

---

# 7. `.dockerignore`

Mon `.dockerignore` contient notamment :

```text
.git
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env
pip-log.txt
pip-delete-this-directory.txt
.tox
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.gitignore
```

Il est important d'y exclure également les secrets locaux, par exemple :

```text
.env
.env.*
```

à adapter selon la convention utilisée par le projet.

Pourquoi ?

Parce que le Dockerfile contient :

```dockerfile
COPY . .
```

Sans `.dockerignore`, un `.env` présent dans le build context pourrait être copié dans l'image.

### Deux mécanismes différents

```text
.dockerignore
     │
     └── contrôle ce qui entre dans le build context

--env-file .env
     │
     └── injecte des variables au démarrage du conteneur
```

Le fait d'exclure `.env` du build context n'empêche donc pas :

```bash
docker run --env-file .env ...
```

de fonctionner.

---

# 8. Construire une image

Commande :

```bash
docker build -t apprendre-docker-web .
```

Décomposition :

```text
docker build
     │
     ├── -t apprendre-docker-web
     │        └── nom/tag de l'image
     │
     └── .
          └── build context
```

Une fois le build terminé :

```bash
docker images
```

peut afficher :

```text
REPOSITORY              TAG       IMAGE ID       SIZE
apprendre-docker-web    latest    5d42...        297MB
```

L'image est maintenant disponible localement.

---

# 9. Layers et cache Docker

Docker construit l'image par couches.

Les différentes instructions du Dockerfile participent à cette construction.

Le cache permet d'éviter de refaire inutilement certaines étapes.

Par exemple :

```dockerfile
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

Si je modifie uniquement :

```text
views.py
```

Docker peut réutiliser les étapes précédentes.

Si je modifie :

```text
requirements.txt
```

Docker doit reconstruire l'installation des dépendances.

Pendant mon build, j'ai notamment vu :

```text
#7 WORKDIR /app CACHED
#8 RUN apt-get ... CACHED
#9 COPY requirements.txt . DONE
#10 RUN pip install... DONE
#11 COPY . . DONE
```

Le mot :

```text
CACHED
```

signifie que Docker a pu réutiliser une étape déjà construite.

### Pourquoi c'est important ?

Une bonne organisation du Dockerfile permet donc :

* des builds plus rapides ;
* moins de travail inutile ;
* un meilleur usage du cache Docker.

---

# 10. Créer et lancer un conteneur

Première tentative :

```bash
docker run --name docker-lab apprendre-docker-web
```

Cette commande :

1. utilise l'image `apprendre-docker-web` ;
2. crée un nouveau conteneur ;
3. démarre le processus principal ;
4. lance `/app/entrypoint.sh` ;
5. lui transmet `web` comme argument.

Schématiquement :

```text
docker run
    │
    ▼
apprendre-docker-web
    │
    ▼
nouveau conteneur
    │
    ▼
/app/entrypoint.sh web
    │
    ▼
Gunicorn
    │
    ▼
Django
```

---

# 11. Variables d'environnement

Lors de ma première tentative :

```bash
docker run --name docker-lab apprendre-docker-web
```

Django a retourné :

```text
django.core.exceptions.ImproperlyConfigured:
The SECRET_KEY setting must not be empty.
```

Pourquoi ?

Parce que le conteneur n'avait pas reçu la variable :

```text
SECRET_KEY
```

Docker ne lit pas automatiquement mon `.env` lorsque je fais simplement :

```bash
docker run ...
```

Je peux utiliser :

```bash
docker run \
  --name docker-lab2 \
  --env-file .env \
  apprendre-docker-web
```

`--env-file .env` signifie :

> charger les variables du fichier `.env` et les injecter dans l'environnement du conteneur.

Cette fois, Django a démarré :

```text
Starting Django web server...

164 static files copied to '/app/staticfiles', 472 post-processed.

Starting gunicorn 23.0.0
Listening at: http://0.0.0.0:8000
Booting worker
```

---

# 12. Les ports : `EXPOSE` vs `-p`

C'est une distinction fondamentale.

Dans le Dockerfile :

```dockerfile
EXPOSE 8000
```

indique :

```text
Le conteneur utilise le port 8000.
```

Mais cela ne rend pas automatiquement le service disponible sur :

```text
localhost:8000
```

Pour publier le port :

```bash
docker run -p 9000:8000 ...
```

La syntaxe est :

```text
-p PORT_HÔTE:PORT_CONTENEUR
```

Donc :

```text
-p 9000:8000
    │     │
    │     └── port dans le conteneur
    │
    └──────── port sur ma machine
```

Avec :

```bash
docker run \
  --name docker-test \
  -p 9000:8000 \
  --env-file .env \
  apprendre-docker-web
```

je peux accéder à :

```text
http://localhost:9000
```

Le chemin est :

```text
Navigateur
    │
    ▼
localhost:9000
    │
    │ port mapping
    ▼
container:8000
    │
    ▼
Gunicorn
    │
    ▼
Django
```

### Le port interne ne change pas

Django continue d'écouter sur :

```text
0.0.0.0:8000
```

Je change uniquement le port utilisé depuis l'extérieur.

Je pourrais également utiliser :

```bash
-p 8080:8000
```

et accéder à :

```text
http://localhost:8080
```

Django écouterait toujours sur :

```text
8000
```

dans le conteneur.

---

# 13. `ENTRYPOINT` et `CMD`

Mon Dockerfile contient :

```dockerfile
ENTRYPOINT ["/app/entrypoint.sh"]

CMD ["web"]
```

Ces deux instructions fonctionnent ensemble.

## `ENTRYPOINT`

Définit le programme principal :

```text
/app/entrypoint.sh
```

## `CMD`

Fournit l'argument par défaut :

```text
web
```

Docker lance donc :

```bash
/app/entrypoint.sh web
```

Le script reçoit :

```text
$1 = web
```

et exécute :

```sh
web)
```

qui démarre :

```bash
exec gunicorn --bind 0.0.0.0:8000 crud.wsgi:application
```

### Pourquoi avoir un `ENTRYPOINT` ?

Dans ce projet, il permet de centraliser le démarrage des différents rôles :

```text
                 même image
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
         web       celery      beat
          │          │          │
       Gunicorn    Worker      Beat
```

Chaque conteneur peut avoir un rôle différent tout en utilisant la même image.

---

# 14. Observer et diagnostiquer un conteneur

## `docker ps`

Voir les conteneurs actifs :

```bash
docker ps
```

---

## `docker ps -a`

Voir tous les conteneurs :

```bash
docker ps -a
```

Cela permet notamment de voir un conteneur arrêté.

Exemple :

```text
STATUS
Exited (1)
```

ou :

```text
Up 4 seconds
```

### `Exited (1)`

Le processus principal s'est terminé avec une erreur.

### `Exited (0)`

Le processus s'est terminé sans erreur.

---

## `docker logs`

Afficher les logs :

```bash
docker logs docker-test
```

Suivre les logs en temps réel :

```bash
docker logs -f docker-test
```

C'est l'une des premières commandes à utiliser lorsqu'un conteneur ne fonctionne pas comme prévu.

---

## `docker inspect`

Commande :

```bash
docker inspect docker-test
```

Cette commande affiche la configuration détaillée du conteneur.

Dans mon expérience, j'ai notamment pu observer :

```json



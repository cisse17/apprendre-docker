# 02 — Docker Networking, Volumes & Compose

## Objectifs

À la fin de ce chapitre, je dois être capable de :

* Expliquer comment les conteneurs communiquent entre eux.
* Comprendre la résolution DNS sur un réseau Docker.
* Distinguer la communication interne et la publication des ports.
* Comprendre les volumes nommés, les bind mounts et les volumes anonymes.
* Utiliser Docker Compose pour démarrer plusieurs services.
* Comprendre les limites de `depends_on`.
* Inspecter un conteneur et identifier les informations importantes.

---

## 1. Réseaux Docker

### 1.1 Lister les réseaux

```bash
docker network ls
```

Cette commande affiche les réseaux Docker existants.

Pour examiner un réseau en détail :

```bash
docker network inspect docker-lab-network
```

On peut notamment observer :

* Le driver réseau.
* Le sous-réseau et la passerelle.
* Les conteneurs connectés.
* Les adresses IP et les alias réseau.

### 1.2 Créer un réseau personnalisé

```bash
docker network create docker-lab-network
```

Cette commande crée un réseau Docker.

Un réseau `bridge` personnalisé permet notamment aux conteneurs connectés de se joindre par leur nom ou leur alias grâce au DNS intégré de Docker.

### 1.3 Connecter deux conteneurs

```bash
docker run -d \
  --name lab-nginx-1 \
  --network docker-lab-network \
  nginx:alpine

docker run -d \
  --name lab-nginx-2 \
  --network docker-lab-network \
  nginx:alpine
```

| Option         | Signification                             |
| -------------- | ----------------------------------------- |
| `-d`           | Exécute le conteneur en arrière-plan.     |
| `--name`       | Attribue un nom au conteneur.             |
| `--network`    | Connecte le conteneur au réseau spécifié. |
| `nginx:alpine` | Image utilisée pour créer le conteneur.   |

Pour vérifier les connexions :

```bash
docker network inspect docker-lab-network
```

### 1.4 Tester la communication entre les conteneurs

```bash
docker exec lab-nginx-1 wget -qO- http://lab-nginx-2
```

La commande renvoie le HTML de la page d'accueil Nginx si la communication fonctionne.

**À retenir :** sur un réseau Docker personnalisé, un conteneur peut joindre un autre conteneur par son nom, sans devoir connaître son adresse IP.

### 1.5 `localhost` désigne le conteneur lui-même

Dans un conteneur, `localhost` ou `127.0.0.1` désigne ce même conteneur, pas les autres conteneurs du projet.

Dans notre application Django, si le service PostgreSQL s'appelle `db`, Django peut utiliser :

```text
DB_HOST=db
DB_PORT=5432
```

De même, si Redis est défini sous le nom de service `redis`, les autres services peuvent le joindre via `redis:6379`.

Les adresses IP des conteneurs peuvent changer lors de leur recréation. Il est donc préférable d'utiliser les noms de services plutôt que des adresses IP codées en dur.

### 1.6 Communication interne et publication des ports

Exemple :

```bash
docker run -d --name mon-nginx -p 8080:80 nginx:alpine
```

La syntaxe est :

```text
-p PORT_HOTE:PORT_CONTENEUR
```

Ici, le port `8080` de la machine hôte redirige vers le port `80` du conteneur.

Deux conteneurs connectés au même réseau Docker peuvent communiquer sans publier leurs ports sur l'hôte.

`EXPOSE` dans un Dockerfile est une métadonnée qui documente le port prévu. À lui seul, il ne publie pas le port sur la machine hôte.

---

## 2. Volumes Docker

Un conteneur est remplaçable. Les données importantes doivent être stockées de manière à survivre à sa suppression et à sa recréation.

Les volumes sont particulièrement importants pour PostgreSQL : ils permettent de conserver les données de la base indépendamment du cycle de vie du conteneur.

### 2.1 Lister et inspecter les volumes

```bash
docker volume ls
```

Cette commande liste les volumes Docker.

Pour examiner un volume :

```bash
docker volume inspect apprendre-docker_postgres_data
```

On peut notamment observer son nom, son driver, ses labels et son emplacement géré par Docker.

### 2.2 Volume nommé

Exemple dans `compose.yaml` :

```yaml
services:
  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Il faut distinguer deux éléments :

* `postgres_data` est le nom logique du volume dans le fichier Compose.
* `/var/lib/postgresql/data` est le chemin dans le conteneur où PostgreSQL stocke ses données.

Compose crée généralement un nom physique préfixé par le nom du projet, par exemple `apprendre-docker_postgres_data`.

**À retenir :** un volume nommé est adapté aux données qui doivent persister, notamment celles de PostgreSQL.

Attention : supprimer un volume peut entraîner la perte définitive des données qu'il contient. Il ne faut pas supprimer le volume PostgreSQL uniquement pour reconstruire une image ou redémarrer un service.

### 2.3 Bind mount

Exemple :

```yaml
services:
  web:
    volumes:
      - .:/app
```

Le dossier du projet sur la machine hôte est monté dans `/app` dans le conteneur.

En développement, cela permet de modifier le code sur la machine hôte sans reconstruire systématiquement l'image.

Un bind mount masque le contenu de l'image présent au chemin cible tant que le montage est actif.

### 2.4 Volume anonyme

Exemple :

```yaml
services:
  web:
    volumes:
      - .:/app
      - /app/staticfiles
```

`/app/staticfiles` est un point de montage sans nom de volume explicite. Docker lui associe un volume anonyme.

Dans cet exemple, le montage anonyme sur `/app/staticfiles` est plus spécifique que le bind mount parent `.:/app`. Il masque le contenu de ce chemin provenant du montage parent.

### 2.5 Comparaison des montages

| Type           | Exemple                                  | Usage courant                                                      |
| -------------- | ---------------------------------------- | ------------------------------------------------------------------ |
| Volume nommé   | `postgres_data:/var/lib/postgresql/data` | Persister les données d'un service.                                |
| Bind mount     | `.:/app`                                 | Partager un dossier de l'hôte, notamment le code en développement. |
| Volume anonyme | `/app/staticfiles`                       | Monter un volume sans nom explicite dans le fichier Compose.       |

---

## 3. Docker Compose

Docker Compose permet de décrire plusieurs services dans un fichier YAML, puis de les construire et de les démarrer ensemble.

Dans notre application Django, les services sont :

| Service       | Rôle                                                                     |
| ------------- | ------------------------------------------------------------------------ |
| `web`         | Application Django et serveur web.                                       |
| `db`          | Base de données PostgreSQL.                                              |
| `redis`       | Redis, utilisé selon la configuration comme broker ou backend de tâches. |
| `celery`      | Worker qui exécute les tâches asynchrones.                               |
| `celery_beat` | Planification des tâches périodiques.                                    |

### 3.1 Construire les images

```bash
docker compose build
```

Cette commande construit les images des services qui possèdent une configuration `build`.

Plusieurs services peuvent utiliser le même contexte de construction tout en ayant des configurations ou des images associées distinctes.

### 3.2 Démarrer les services

```bash
docker compose up -d
```

Compose crée, si nécessaire :

* Le réseau du projet.
* Les volumes déclarés.
* Les conteneurs des services.

L'option `-d` démarre les services en arrière-plan.

### 3.3 Vérifier l'état des services

```bash
docker compose ps
```

Pour afficher également les conteneurs arrêtés :

```bash
docker compose ps -a
```

Pour suivre les journaux du service web :

```bash
docker compose logs -f web
```

Un conteneur affiché comme `Up` est en cours d'exécution, mais cela ne garantit pas à lui seul que l'application fonctionne correctement ou répond aux requêtes.

### 3.4 Le réseau créé par Compose

Compose crée généralement un réseau par défaut pour le projet, par exemple :

```text
apprendre-docker_default
```

Les services peuvent communiquer sur ce réseau en utilisant leurs noms de service.

Exemple :

```text
Django ─────► db:5432
Django ─────► redis:6379
Celery ─────► redis:6379
```

Ces communications internes ne nécessitent pas que les ports soient publiés sur la machine hôte.

Pour inspecter le réseau :

```bash
docker network ls
docker network inspect apprendre-docker_default
```

Le nom exact dépend du nom du projet Compose et des éventuelles personnalisations.

### 3.5 Comprendre `depends_on`

Exemple :

```yaml
services:
  web:
    depends_on:
      - db
      - redis
```

`depends_on` indique à Compose de démarrer les dépendances avant le service qui les utilise.

En configuration simple, cela ne garantit pas que PostgreSQL ou Redis soit prêt à accepter les connexions.

Il faut distinguer :

* **Démarré :** le conteneur a été lancé.
* **Prêt :** le service à l'intérieur du conteneur est réellement disponible.

Pour améliorer la fiabilité, on peut configurer un `healthcheck` et utiliser une condition de dépendance basée sur l'état de santé lorsque cela est approprié. L'application doit également gérer les erreurs de connexion et les nouvelles tentatives si nécessaire.

---

## 4. Inspecter un conteneur

Pour inspecter le conteneur du service `web` :

```bash
docker inspect "$(docker compose ps -q web)"
```

`docker compose ps -q web` récupère l'identifiant du conteneur associé au service `web`. Cet identifiant est ensuite transmis à `docker inspect`.

Champs importants :

| Champ                      | Signification                                |
| -------------------------- | -------------------------------------------- |
| `Config.Image`             | Image utilisée par le conteneur.             |
| `Config.Cmd`               | Commande configurée pour le démarrage.       |
| `Config.Entrypoint`        | Point d'entrée du conteneur.                 |
| `Config.WorkingDir`        | Répertoire de travail.                       |
| `Config.Env`               | Variables d'environnement configurées.       |
| `HostConfig.Binds`         | Informations sur les bind mounts configurés. |
| `Mounts`                   | Montages effectifs du conteneur.             |
| `NetworkMode`              | Mode ou réseau configuré.                    |
| `NetworkSettings.Networks` | Réseaux, adresses IP et alias.               |
| `Ports`                    | Correspondances de ports publiées.           |

### Précaution de sécurité

La sortie complète de `docker inspect` peut contenir des variables d'environnement sensibles.

Bonnes pratiques :

* Exclure `.env` et les fichiers de secrets du contexte de construction avec `.dockerignore`.
* Éviter d'incorporer des secrets dans une image avec `COPY . .`.
* Ne pas committer les fichiers contenant des identifiants.
* Si un secret a été exposé, le révoquer ou le renouveler.

**Attention à PostgreSQL :** modifier uniquement `POSTGRES_PASSWORD` dans l'environnement ne change pas automatiquement le mot de passe du rôle d'une base déjà initialisée. Il faut également modifier le mot de passe dans PostgreSQL. Ne supprime pas le volume pour résoudre ce problème.

---

## 5. Commandes de référence

| Commande                         | Fonction                                                      |
| -------------------------------- | ------------------------------------------------------------- |
| `docker network ls`              | Lister les réseaux.                                           |
| `docker network create NOM`      | Créer un réseau.                                              |
| `docker network inspect NOM`     | Inspecter un réseau.                                          |
| `docker volume ls`               | Lister les volumes.                                           |
| `docker volume inspect NOM`      | Inspecter un volume.                                          |
| `docker compose build`           | Construire les images concernées.                             |
| `docker compose up -d`           | Créer et démarrer les services en arrière-plan.               |
| `docker compose ps`              | Afficher l'état des services.                                 |
| `docker compose ps -a`           | Inclure les conteneurs arrêtés.                               |
| `docker compose logs -f web`     | Suivre les journaux du service web.                           |
| `docker inspect CONTENEUR`       | Inspecter la configuration d'un conteneur.                    |
| `docker exec CONTENEUR COMMANDE` | Exécuter une commande dans un conteneur en cours d'exécution. |

---

## 6. Points essentiels à retenir

1. Les conteneurs connectés au même réseau personnalisé peuvent communiquer par leur nom grâce au DNS Docker.
2. Dans un conteneur, `localhost` désigne ce conteneur lui-même.
3. `-p` publie un port vers l'hôte ; il n'est généralement pas nécessaire pour la communication interne.
4. Les volumes nommés permettent de persister les données au-delà du cycle de vie d'un conteneur.
5. Un bind mount partage un chemin de l'hôte avec le conteneur.
6. Compose organise les services, le réseau et les volumes d'une application.
7. `depends_on` simple gère l'ordre de démarrage, mais ne garantit pas que les dépendances soient prêtes.
8. `docker inspect` permet d'examiner la configuration réelle, mais peut révéler des secrets.

---

## 7. Exercices de révision

### Exercice 1 — Réseau

1. Créer un réseau `revision-network`.
2. Démarrer deux conteneurs Nginx sur ce réseau.
3. Depuis le premier conteneur, récupérer la page du second en utilisant son nom.
4. Expliquer pourquoi aucun `-p` n'est nécessaire pour ce test.

### Exercice 2 — Volumes

1. Repérer le volume PostgreSQL dans le fichier Compose.
2. Identifier son nom logique et le chemin cible dans le conteneur.
3. Expliquer ce qui risque d'arriver aux données si le volume est supprimé.

### Exercice 3 — Compose

1. Exécuter `docker compose ps`.
2. Inspecter le réseau par défaut du projet.
3. Identifier les noms de services utilisables comme noms d'hôte.
4. Expliquer pourquoi `depends_on` ne garantit pas toujours que Django puisse immédiatement se connecter à PostgreSQL.

---

## Fin du Jour 2

Le Jour 2 a couvert les réseaux, les volumes et l'orchestration locale avec Docker Compose.

Le Jour 3 portera sur l'amélioration du Dockerfile : cache de construction, taille des images, exécution avec un utilisateur non privilégié, gestion des processus et préparation à la production sur ECS Fargate.

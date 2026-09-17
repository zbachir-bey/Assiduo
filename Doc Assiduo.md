# Installation du projet Assiduo — Symfony 7.4 + Docker + MySQL

Ce document explique comment installer et lancer le projet en local, sans rien avoir à deviner.

## Prérequis

- Linux (testé sur Debian)
- Docker + Docker Compose Plugin
- Git

### Installer Docker (si pas déjà fait)

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin git
sudo usermod -aG docker $USER
```

Déconnecte-toi/reconnecte-toi (ou `newgrp docker`) pour que l'ajout au groupe `docker` prenne effet.

Vérifie que Docker fonctionne :
```bash
docker --version
docker compose version
sudo systemctl status docker
```

## 1. Cloner le projet

```bash
git clone <URL_DU_REPO> Assiduo
cd Assiduo
```

## 2. Lancer l'environnement Docker

```bash
docker compose build --no-cache
docker compose up -d
docker compose ps
```

Le conteneur `php` (FrankenPHP) doit passer à l'état `healthy` après quelques secondes.

## 3. Accéder à l'application

Ouvre dans le navigateur :
```
https://localhost
```

Le certificat TLS est auto-signé (généré par Caddy/FrankenPHP) : accepte l'avertissement de sécurité ("Avancé" → "Continuer quand même").

Tu dois voir la page d'accueil **Symfony 7.4**.

## Détails techniques du setup

Le projet est basé sur le template [dunglas/symfony-docker](https://github.com/dunglas/symfony-docker), qui utilise FrankenPHP (PHP + serveur web Caddy en un seul binaire), avec deux adaptations :

### Version Symfony fixée en 7.4 (LTS)

Le template installe par défaut la dernière version de Symfony. La version est forcée via une variable dans `.env` :
```
SYMFONY_VERSION=7.4.*
```

**Pourquoi 7.4 et pas la dernière version (8.x) ?**
La 7.4 est une version **LTS (Long Term Support)** : support des bugs jusqu'à fin 2027, sécurité jusqu'à fin 2028 environ. Les versions non-LTS (8.0, 8.1...) ne sont supportées que ~8 mois. Pour un projet destiné à durer, la LTS est le choix par défaut.

### Base de données MySQL (au lieu de PostgreSQL par défaut)

Le template installe PostgreSQL par défaut. La configuration a été adaptée pour MySQL 8.0.32, suivant la documentation officielle du template (`docs/mysql.md`).

Trois fichiers ont été modifiés :

**`compose.yaml`** — le service `database` utilise l'image `mysql` au lieu de `postgres`, avec les variables d'environnement, le healthcheck et le point de montage adaptés à MySQL.

**`.env`** — la ligne `DATABASE_URL` active pointe vers MySQL et utilise le nom du service Docker (`database`) plutôt qu'une IP comme `127.0.0.1`, pour que le conteneur PHP puisse résoudre correctement l'hôte de la base :
```
DATABASE_URL="mysql://app:!ChangeMe!@database:3306/app?serverVersion=8.0.32&charset=utf8mb4"
```

**`Dockerfile`** — l'extension PHP installée est `pdo_mysql` au lieu de `pdo_pgsql`.

Le pack ORM Doctrine a été installé avec :
```bash
docker compose exec php composer req symfony/orm-pack
```

## Commandes utiles

| Action | Commande |
|---|---|
| Voir les logs de l'app | `docker compose logs php --tail=50` |
| Voir les logs de la base | `docker compose logs database --tail=50` |
| Arrêter les conteneurs | `docker compose down` |
| Tout arrêter + supprimer les volumes (⚠️ efface la base) | `docker compose down -v` |
| Reconstruire l'image après un changement de Dockerfile | `docker compose up -d --build` |
| Exécuter une commande Composer dans le conteneur | `docker compose exec php composer <commande>` |
| Exécuter une commande Symfony (bin/console) | `docker compose exec php php bin/console <commande>` |
| Vérifier l'état des conteneurs | `docker compose ps` |

## Dépannage rapide

**Le conteneur `php` redémarre en boucle avec "Composer could not find a composer.json file"**
Le projet Symfony n'a pas encore été généré dans le dossier. Voir la section suivante.

**Régénérer le projet Symfony depuis zéro (si besoin)**
```bash
docker compose down -v
# Supprimer uniquement les fichiers générés par Symfony (PAS les fichiers Docker : Dockerfile, compose*.yaml, frankenphp/, .env, docs/, .git/)
sudo rm -rf bin config public src var vendor
sudo rm -f composer.json composer.lock symfony.lock .env.dev .gitignore

docker compose run --rm --no-deps php composer create-project symfony/skeleton:"7.4.*" temp --prefer-dist --no-interaction
docker compose run --rm --no-deps php sh -c "mv temp/* temp/.[!.]* . 2>/dev/null; true"
sudo rm -rf temp

docker compose up -d
```

**Erreur "permission denied while trying to connect to the docker API"**
Ton utilisateur n'est pas dans le groupe `docker` :
```bash
sudo usermod -aG docker $USER
newgrp docker
```

**Erreur "propriétaire douteux détecté" avec Git**
Certains fichiers ont été créés par `root` via Docker/`sudo`. Autorise le dépôt :
```bash
git config --global --add safe.directory <chemin_absolu_du_dossier>
```

## Partage avec l'équipe

Une fois le repo poussé sur GitHub/GitLab, chaque collègue n'a qu'à faire :
```bash
git clone <URL_DU_REPO>
cd Assiduo
docker compose build
docker compose up -d
```
et ouvrir `https://localhost`.

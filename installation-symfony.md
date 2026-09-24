# Installation de Symfony 7.4 sans Docker sur une VM Linux

Ce document explique, étape par étape, comment installer Symfony et faire tourner le projet **sans Docker**, directement sur une VM Linux (Debian/Ubuntu). Il couvre aussi comment récupérer le code déjà existant du projet avec Git.

Toutes les commandes sont à taper dans un terminal.

---

## 1. Installer les prérequis système

```bash
sudo apt update
sudo apt install -y php php-cli php-fpm php-xml php-mbstring php-curl php-zip php-intl php-mysql php-sqlite3 unzip git curl
```

Vérifie que PHP est bien installé (il faut au moins PHP 8.2, idéalement 8.3+) :
```bash
php -v
```

---

## 2. Installer Composer

Composer est l'outil qui gère les dépendances (bibliothèques) de PHP/Symfony.

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

Vérifie l'installation :
```bash
composer --version
```

---

## 3. Installer Symfony CLI

C'est l'outil officiel pour créer et lancer un projet Symfony.

```bash
curl -sS https://get.symfony.com/cli/installer | bash
sudo mv ~/.symfony5/bin/symfony /usr/local/bin/symfony
```

Vérifie que tout est prêt :
```bash
symfony check:requirements
```

Si un message indique que la configuration a changé d'emplacement (avertissement à partir de la version 5.17.0 du CLI), applique ceci une fois :
```bash
symfony server:stop --all
symfony proxy:stop
mv ~/.symfony5 ~/.config/symfony-cli
```

---

## 4. Configurer Git (obligatoire avant toute action Git)

Symfony CLI fait un `git init` automatique à la création d'un projet, donc Git a besoin de connaître ton identité :

```bash
git config --global user.email "ton_email@exemple.com"
git config --global user.name "Ton Nom"
```

---

## 5A. Créer un nouveau projet Symfony 7.4 (si le projet n'existe pas encore)

```bash
cd ~
symfony new mon_projet --version=7.4 --webapp
cd mon_projet
```

- `--version=7.4` force la version LTS (Long Term Support), supportée plus longtemps qu'une version standard.
- `--webapp` installe un squelette complet (Twig, formulaires, sécurité...). Sans ce flag, tu obtiens une base minimale, plutôt pour une API.

**Pourquoi la 7.4 et pas une version plus récente ?**
La 7.4 est LTS : elle reçoit des correctifs de bugs jusqu'à fin 2027 et des correctifs de sécurité jusqu'à fin 2028 environ. Les versions non-LTS ne sont supportées que ~8 mois.

➡️ Passe directement à l'étape 7 (lancer le serveur).

---

## 5B. Récupérer un projet Symfony déjà existant avec Git (cas le plus courant en équipe)

Si le projet existe déjà sur un dépôt Git (GitHub/GitLab), voici comment le récupérer.

### Cloner le projet pour la première fois

```bash
cd ~
git clone <URL_DU_REPO> mon_projet
cd mon_projet
```

Remplace `<URL_DU_REPO>` par l'URL donnée par ton collègue ou affichée sur la page GitHub du projet (bouton vert "Code").

### Installer les dépendances du projet

Un projet cloné n'a pas encore ses dépendances (dossier `vendor/`), il faut les installer :

```bash
composer install
```

### Récupérer les mises à jour du projet plus tard (une fois le projet déjà cloné)

Quand quelqu'un d'autre a poussé du nouveau code, récupère-le avec :

```bash
cd ~/mon_projet
git pull
composer install
```

Le `composer install` après un `git pull` est important : si de nouvelles dépendances ont été ajoutées (nouveau `composer.json`), il faut les télécharger.

### Envoyer ses propres modifications sur le dépôt

```bash
git add .
git commit -m "Description de ce que tu as changé"
git push
```

Si c'est la toute première fois que tu pousses depuis cette VM, Git te demandera de t'identifier (nom d'utilisateur GitHub + un **token**, pas ton mot de passe classique — voir la section Dépannage en bas de ce document).

---

## 6. Configurer la base de données

### Installer MySQL sur la VM

```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
```

Suis les instructions à l'écran (définir un mot de passe root, etc.).

### Configurer la connexion dans le projet

Ouvre le fichier `.env` à la racine du projet :

```bash
nano .env
```

Cherche la ligne `DATABASE_URL` et adapte-la à ta base MySQL locale, par exemple :

```
DATABASE_URL="mysql://user:password@127.0.0.1:3306/mon_projet?serverVersion=8.0"
```

Remplace `user`, `password` et `mon_projet` par tes propres informations.

Sauvegarde avec `Ctrl+O`, `Entrée`, puis quitte avec `Ctrl+X`.

### Créer la base de données

```bash
symfony console doctrine:database:create
```

### Appliquer les migrations (si le projet en contient déjà)

```bash
symfony console doctrine:migrations:migrate
```

---

## 7. Lancer le serveur de développement

```bash
symfony server:start
```

Le serveur tourne en continu dans ce terminal (il ne "finit" jamais tant que tu ne fais pas `Ctrl+C`) — c'est normal, laisse-le ouvert.

Par défaut, le serveur écoute uniquement sur `127.0.0.1` (localhost), donc **accessible seulement depuis la VM elle-même**.

### Accéder au site depuis ton navigateur si tu es en SSH sur la VM

Deux options :

**Option A — tunnel SSH (recommandé, ne modifie rien sur la VM)**

Depuis ta machine locale (pas depuis la VM), ouvre une connexion SSH avec redirection de port :
```bash
ssh -L 8000:127.0.0.1:8000 utilisateur@IP_DE_LA_VM
```
Puis ouvre `http://127.0.0.1:8000` dans le navigateur de ta machine locale.

**Option B — écouter sur toutes les IP (accès direct via l'IP de la VM)**

```bash
symfony server:start --allow-all-ip --port=8000
```
Puis accède directement via `http://IP_DE_LA_VM:8000` dans le navigateur (aucun tunnel nécessaire).

⚠️ Si vous êtes plusieurs sur la même VM, chacun doit utiliser un port différent (`--port=8000`, `--port=8001`, etc.) pour éviter les conflits.

### Vérifier qu'aucun pare-feu ne bloque le port (si besoin, Option B uniquement)

```bash
sudo iptables -L -n
```

Si la chaîne `INPUT` est en `ACCEPT` sans règle de blocage, rien à faire. Sinon, avec `ufw` (s'il est installé) :
```bash
sudo ufw allow 8000/tcp
```

---

## Commandes utiles au quotidien

| Action | Commande |
|---|---|
| Lancer le serveur | `symfony server:start` |
| Lancer le serveur accessible depuis le réseau | `symfony server:start --allow-all-ip --port=8000` |
| Arrêter le serveur | `symfony server:stop` ou `Ctrl+C` dans le terminal qui le fait tourner |
| Arrêter tous les serveurs | `symfony server:stop --all` |
| Installer les dépendances PHP | `composer install` |
| Ajouter une dépendance | `composer require nom-du-paquet` |
| Créer une entité Doctrine | `symfony console make:entity` |
| Créer une migration | `symfony console make:migration` |
| Appliquer les migrations | `symfony console doctrine:migrations:migrate` |
| Vider le cache Symfony | `symfony console cache:clear` |
| Récupérer le dernier code | `git pull` |
| Envoyer son code | `git add . && git commit -m "message" && git push` |
| Voir l'état des fichiers modifiés | `git status` |

---

## Dépannage

**`git commit` échoue avec "Identité d'auteur inconnue"**
Il faut configurer Git une fois (voir étape 4) :
```bash
git config --global user.email "ton_email@exemple.com"
git config --global user.name "Ton Nom"
```

**`git push` refuse le mot de passe ("Password authentication is not supported")**
GitHub n'accepte plus les mots de passe classiques pour `git push`. Il faut un **token d'accès personnel** :
1. Va sur `https://github.com/settings/tokens`
2. **Generate new token (classic)** → coche la case `repo` → génère
3. Copie le token (il ne sera plus jamais réaffiché)
4. Au moment du push, dans le champ "Password", colle le **token** (pas ton mot de passe GitHub)

Pour ne pas avoir à le retaper à chaque fois :
```bash
git config --global credential.helper store
```

**"Permission non accordée" en éditant un fichier**
Le fichier appartient à un autre utilisateur (souvent `root`). Utilise `sudo` pour l'éditer :
```bash
sudo nano nom_du_fichier
```

**"propriétaire douteux détecté" avec Git**
```bash
git config --global --add safe.directory /chemin/absolu/du/dossier
```

**Le site ne s'affiche pas dans le navigateur**
- Vérifie que `symfony server:start` tourne toujours (dans un terminal resté ouvert)
- Vérifie l'URL et le port exacts affichés par la commande (ex : `http://127.0.0.1:8001` si le port 8000 était déjà pris par quelqu'un d'autre)
- Si tu es en SSH, vérifie que tu utilises bien un tunnel SSH (Option A) ou `--allow-all-ip` (Option B)

**Deux collègues sur la même VM, conflit de port**
Chacun doit lancer son propre serveur sur un port différent :
```bash
symfony server:start --allow-all-ip --port=8001
```

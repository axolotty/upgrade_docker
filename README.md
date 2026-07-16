# upgrade_docker

Script Bash de mise à jour des images Docker d'un hôte : il parcourt les stacks Compose et
les conteneurs standalone, ne télécharge que ce qui a réellement changé, et relance
uniquement ce qui doit l'être.

Écrit pour tourner sans surveillance (cron, `--notify`), mais **à lire avant de l'automatiser** :
la section [Pièges et limites connues](#pièges-et-limites-connues) décrit des comportements
qui peuvent surprendre, dont certains ont déjà causé une interruption de service.

## Le contrat

C'est toute la logique du script, en une table :

| Situation | Ce que fait le script |
|---|---|
| Image tirée d'un registry, **pas de mise à jour** | Rien. Le conteneur n'est pas touché. |
| Image tirée d'un registry, **mise à jour dispo** | Pull, puis recréation du conteneur à l'identique. |
| Service Compose `build:`, **image de base à jour** | Rien. Aucune reconstruction. |
| Service Compose `build:`, **base périmée** | Pull de la base, `docker compose build`, puis `up -d`. |
| Image construite localement, absente de tout registry | Ignorée (`[LOCAL]`), jamais comptée en échec. |

Autrement dit : **rien ne bouge tant que rien n'a changé en amont.**

> ⚠️ Le script met à jour des **images**. Il ne redéploie pas tes modifications de code local :
> pour ça, c'est `docker compose up -d --build`.

## Installation

```bash
sudo cp upgrade_docker /usr/local/bin/upgrade_docker
sudo chmod +x /usr/local/bin/upgrade_docker
```

## Utilisation

```bash
# Voir ce qui serait fait, sans rien modifier
sudo upgrade_docker --dry-run

# Mettre à jour
sudo upgrade_docker

# Cas courants
sudo upgrade_docker --image nginx:latest          # une seule image
sudo upgrade_docker --exclude redis:7             # tout sauf celle-ci
sudo upgrade_docker --exclude-dir BACKUP          # ignorer les dossiers BACKUP/
sudo upgrade_docker --no-prune                    # garder les anciennes images
sudo upgrade_docker --compose-only                # seulement les stacks
sudo upgrade_docker --standalone-only             # seulement les conteneurs standalone
sudo upgrade_docker --no-build                    # ne rien reconstruire
sudo upgrade_docker --force-reboot                # recréer même sans changement de digest
sudo upgrade_docker --notify admin@example.com    # mail si échec
```

## Options

| Option | Description | Défaut |
|---|---|---|
| `--dry-run` | Vérifie les mises à jour sans rien modifier | - |
| `--parallel N` | Jobs simultanés | 20 |
| `--timeout N` | Timeout en secondes par pull | 300 |
| `--health-wait N` | Secondes d'observation d'un conteneur standalone recréé avant de supprimer l'ancien. `0` désactive la vérification | 5 |
| `--search-dir DIR` | Répertoire à scanner (répétable) | `/opt /home /docker_data` |
| `--exclude-dir NOM` | **Nom** de dossier à ignorer au scan (répétable) — s'ajoute à `vendor`, `node_modules`, `html/apps`, `.git`, `OLD`. Voir le piège plus bas | - |
| `--no-prune` | Ne pas supprimer les images obsolètes (`docker image prune -f`) | - |
| `--no-build` | Ne pas reconstruire les services Compose avec `build:` | - |
| `--compose-only` | Traite uniquement les stacks Compose | - |
| `--standalone-only` | Traite uniquement les images standalone | - |
| `--force` | Traite aussi les images déjà à jour — effet visible en `--dry-run` (signalées `[FORCE]`) ; en mode normal toutes les images sont déjà pullées | - |
| `--force-reboot` | Recrée les conteneurs même si le digest n'a pas changé | - |
| `--exclude IMAGE` | Exclut une image (répétable) | - |
| `--image IMAGE` | Traite uniquement cette image | - |
| `--notify EMAIL` | Envoie un mail **en cas d'échec uniquement** (nécessite `mail`) | - |
| `--log FILE` | Fichier de log | `/var/log/docker-update.log` |

## Configuration (optionnelle)

Pour figer ses propres défauts sans retaper d'options, créer `/etc/upgrade_docker.conf`
(système) ou `~/.config/upgrade_docker.conf` (utilisateur) :

```bash
# Stacks à ne jamais scanner (archives, doublons, clones de repo d'ops…)
EXCLUDE_DIRS=(vieille-stack mon-repo-ops)

# Autres défauts possibles
SEARCH_DIRS=(/opt /srv)
PARALLEL=10
NOTIFY_EMAIL=admin@exemple.fr
LOG_FILE=/var/log/docker-update.log
HEALTH_WAIT=5
```

Les options passées en **ligne de commande priment** sur ce fichier.

## Fonctionnement

### Stacks Docker Compose

- Scanne les répertoires configurés à la recherche de `docker-compose.yml` / `docker-compose.yaml`
- **Services avec `build:`** : le script lit les `FROM` du Dockerfile, vérifie si l'image de base
  a une mise à jour, et **ne reconstruit que dans ce cas** (base pullée, puis `docker compose build`).
  Sans ça, une image construite localement ne serait jamais mise à jour et sa base vieillirait
  indéfiniment. Désactivable avec `--no-build`.
  > La base est pullée **explicitement**, car `build --pull` ne rafraîchit pas le tag local :
  > la base serait alors vue comme périmée à chaque exécution, et reconstruite pour rien.
- Puis `docker compose pull` + `docker compose up -d --remove-orphans`. Le drapeau
  `--ignore-buildable` est ajouté **s'il est supporté** par la version de Compose installée :
  sans lui, `pull` sort en erreur sur les services `build:` et la stack entière serait
  comptée en échec
- Avec `--force-reboot` : `docker compose up -d --force-recreate`

### Images standalone

- Liste les images hors stacks Compose, pull parallèle avec `xargs -P`
- Les conteneurs concernés sont repérés **avant** le pull : après, le tag pointe déjà sur la
  nouvelle image et `--filter ancestor=` ne les retrouverait plus
- Si le digest change (ou avec `--force-reboot`), le conteneur est **recréé à l'identique**
  depuis son `docker inspect` : labels (Traefik…), ports, volumes (binds **et** volumes nommés),
  variables d'env, réseau (y compris `network_mode: container:xxx`), limites RAM/CPU/pids,
  devices, capabilities, DNS, log driver, user, workdir, commande et entrypoint
- **Bascule sécurisée** : l'ancien conteneur est *renommé*, puis supprimé seulement si le
  nouveau **démarre et survit** à `--health-wait` secondes d'observation (encore `running`,
  aucun redémarrage). Dans tous les autres cas — `docker run` en échec, ou conteneur qui
  démarre puis meurt — **rollback automatique** : l'ancien est restauré sous son nom et
  redémarré, et l'image fautive n'est pas adoptée
- Env et labels ne sont réinjectés que s'ils **diffèrent de l'image** : réinjecter ses défauts
  figerait l'ancienne version et annulerait la mise à jour
- Une image absente de tout registry est **ignorée**, pas comptée en échec — même verdict que
  le `--dry-run`. Un `RepoDigest` ne suffit pas à trancher (Compose en pose un sur les images
  qu'il construit) : seul le registry fait foi

### Dry-run

- Interroge l'API registry (aucun pull) pour comparer les digests
- Registries gérés : Docker Hub, GHCR, LSCR (alias de GHCR) et **tout registry conforme** — le
  jeton est obtenu en lisant le défi `www-authenticate`. Les registries privés **avec un port**
  (`registry.local:5000/app`) sont correctement analysés, de même que les dépôts publics servis
  en anonyme (sans jeton)
- Calcule la taille réelle des layers à télécharger et estime l'espace libéré après prune
- Pour les services `build:` : lit les `FROM` et vérifie l'**image de base** (les stages
  multi-étapes, `FROM $ARG` et `scratch` sont ignorés)
- Étiquettes : `[OK]` à jour · `[UPDATE]` mise à jour disponible · `[FORCE]` forcé mais déjà à
  jour · `[LOCAL]` image locale · `[BUILD]` service construit localement (base vérifiée)

Le dry-run et le run réel donnent **le même verdict sur les mêmes images** : ce qui est annoncé
`[LOCAL]` est bien ignoré, ce qui est annoncé `[UPDATE]` est bien mis à jour.

### Code de sortie

| Code | Signification |
|---|---|
| `0` | Aucun échec (y compris s'il n'y avait rien à faire) |
| `1` | Au moins une image en échec — c'est aussi ce qui déclenche `--notify` |

## Pièges et limites connues

### ⚠️ Le script démarre les stacks éteintes

Le chemin Compose se termine par `docker compose up -d`. Une stack **volontairement arrêtée**
qui se trouve dans un répertoire scanné sera donc **démarrée**. Le script ne fait aucune
différence entre « éteinte parce que cassée » et « éteinte parce que je l'ai voulu ».

Pour qu'une stack reste à l'arrêt : la déplacer dans un `OLD/` (exclu par défaut) ou l'ajouter
à `EXCLUDE_DIRS`.

### ⚠️ Deux dossiers de même nom = un seul projet Compose

Compose nomme un projet d'après le **nom du dossier**. Deux copies d'une stack dans
`/docker_data/monsite` et `/home/user/monsite` sont donc **le même projet `monsite`** : le
script les scanne toutes les deux, et la seconde `up -d` écrase le déploiement de la première.

C'est arrivé en vrai : un vieux clone oublié a repris la main sur un site en production et l'a
mis en boucle de crash pendant deux heures, avec une configuration périmée de deux mois.

Et **`--exclude-dir` ne sauve pas** de ce cas : il filtre sur le **nom** du dossier, pas sur le
chemin. `--exclude-dir monsite` exclurait les deux. La seule vraie réponse est de supprimer ou
de déplacer le doublon.

### Conteneurs qui sortent vite volontairement

Après avoir recréé un conteneur standalone, le script l'**observe pendant `--health-wait`
secondes** (5 par défaut) avant de supprimer l'ancien. Il n'est validé que s'il est encore
`running` et n'a **redémarré aucune fois** ; sinon, rollback.

C'est nécessaire parce qu'un `docker run` réussi ne prouve rien : il rend 0 dès que le
conteneur *démarre*. Une image qui casse l'application passe ce test, puis meurt et boucle sur
sa `restart policy` — l'ancien conteneur aurait alors été supprimé pour rien. Le critère fiable
est `RestartCount`, car un conteneur en boucle de crash repasse brièvement par `running` mais
ne peut pas cacher ses redémarrages.

La contrepartie : un conteneur dont le **travail se termine légitimement** pendant la fenêtre
d'observation (job one-shot, batch qui finit) est vu comme mort et déclenche un rollback à
tort. Le script ne recrée que des conteneurs qui *tournaient*, ce qui rend le cas rare — mais
si ça te concerne, `--health-wait 0` désactive la vérification et rétablit l'ancien
comportement.

Coût : `--health-wait` secondes par conteneur **effectivement recréé**, et rien du tout quand
il n'y a pas de mise à jour.

### Recréation standalone : ce qui n'est pas reproduit

La recréation reconstruit un `docker run` à partir de l'inspect. Quelques réglages ne sont pas
repris — le script **prévient explicitement** quand il en détecte :

- `ulimits`, `sysctls`, réservations **GPU**
- **alias réseau** et **IP statiques**
- entrypoint personnalisé **multi-arguments**

Pour ces cas, et de manière générale, une **stack Compose** reste préférable : le chemin Compose
reproduit tout fidèlement.

### Bon à savoir

- Une image sans conteneur (orpheline) est quand même pullée si elle a une mise à jour. Elle
  apparaît en `[UPDATE] … (aucun conteneur actif)` : utile pour repérer ce qui traîne.
- Un conteneur **arrêté** n'est jamais recréé : seuls les conteneurs en cours sont concernés
  (`--filter ancestor=` ne liste que ceux qui tournent). Son image, elle, sera pullée.
- Le dry-run fait une requête de manifest par image ; Docker Hub compte ces requêtes dans ses
  quotas anonymes (100 / 6 h par IP).

## Prérequis

- `docker` + plugin `compose`
- `jq`, `xargs`, `curl`
- Bash ≥ 4.3 (namerefs)
- Root, ou accès au socket Docker

## Logs

Fichier : `/var/log/docker-update.log` (modifiable via `--log` ou `LOG_FILE`).
Rotation automatique au-delà de **5 Mo** : le fichier est horodaté en `.bak`, et les `.bak` de
plus de **30 jours** sont supprimés.

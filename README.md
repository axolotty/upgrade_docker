# upgrade_docker

Script Bash de mise à jour automatique des images Docker (stacks Compose + images standalone).

## Installation

```bash
sudo cp upgrade_docker /usr/local/bin/upgrade_docker
sudo chmod +x /usr/local/bin/upgrade_docker
```

## Utilisation

```bash
# Vérifier sans modifier
sudo upgrade_docker --dry-run

# Mettre à jour
sudo upgrade_docker

# Options courantes
sudo upgrade_docker --parallel 5
sudo upgrade_docker --image nginx:latest
sudo upgrade_docker --exclude redis:7
sudo upgrade_docker --force          # Traite aussi les images déjà à jour (visible en --dry-run)
sudo upgrade_docker --force-reboot   # Recrée les conteneurs même si le digest n'a pas changé
sudo upgrade_docker --compose-only
sudo upgrade_docker --standalone-only
sudo upgrade_docker --no-prune
sudo upgrade_docker --no-build       # ne pas reconstruire les images construites localement
sudo upgrade_docker --notify admin@example.com
sudo upgrade_docker --search-dir /srv
```

## Options

| Option | Description | Défaut |
|---|---|---|
| `--dry-run` | Vérifie les mises à jour sans rien modifier | - |
| `--parallel N` | Jobs simultanés | 20 |
| `--timeout N` | Timeout en secondes par pull | 300 |
| `--search-dir DIR` | Répertoire à scanner (répétable) | `/opt /home /docker_data` |
| `--no-prune` | Ne pas supprimer les images obsolètes | - |
| `--no-build` | Ne pas reconstruire les services Compose avec `build:` (par défaut : reconstruit **uniquement si l'image de base a une MAJ**) | - |
| `--compose-only` | Traite uniquement les stacks Compose | - |
| `--standalone-only` | Traite uniquement les images standalone | - |
| `--force` | Traite aussi les images déjà à jour — effet visible en `--dry-run` (signalées `[FORCE]`) ; en mode normal toutes les images sont déjà pullées | - |
| `--force-reboot` | Recrée les conteneurs même si le digest n'a pas changé | - |
| `--exclude IMAGE` | Exclut une image (répétable) | - |
| `--image IMAGE` | Traite uniquement cette image | - |
| `--notify EMAIL` | Envoie un mail en cas d'échec (nécessite `mail`) | - |
| `--log FILE` | Fichier de log | `/var/log/docker-update.log` |

## Fonctionnement

### Stacks Docker Compose
- Scanne les répertoires configurés à la recherche de `docker-compose.yml` / `docker-compose.yaml`
- Exclut `vendor/`, `node_modules/`, `html/apps/`, `.git/`
- **Services avec `build:`** (image construite depuis un `Dockerfile`) : le script lit les `FROM`, **vérifie si l'image de base a une mise à jour**, et **ne reconstruit que dans ce cas** (base pullée, puis `docker compose build`). **Si les bases sont à jour, rien n'est touché.** Sans ça, une image construite localement ne serait jamais mise à jour et sa base vieillirait indéfiniment. Désactivable avec `--no-build`.
  > ⚠️ Ce script met à jour des **images**, il ne **redéploie pas** tes modifications de code locales — pour ça : `docker compose up -d --build`.
  > *(La base est pullée explicitement : `build --pull` ne rafraîchit pas le tag local, la base serait alors vue comme périmée à chaque exécution.)*
- Puis `docker compose pull --ignore-buildable` + `docker compose up -d --remove-orphans`
- Avec `--force-reboot` : `docker compose up -d --force-recreate`

### Images standalone
- Liste toutes les images locales hors stacks Compose
- Pull parallèle avec `xargs -P`
- Les conteneurs concernés sont repérés **avant** le pull (après, le tag pointe déjà sur la nouvelle image et ils deviendraient introuvables)
- Si le digest change (ou avec `--force-reboot`) : le conteneur est **recréé à l'identique** depuis son `docker inspect` — labels (Traefik…), ports, volumes (binds **et** volumes nommés), variables d'env, réseau (y compris `network_mode: container:xxx`), limites RAM/CPU/pids, devices, capabilities, DNS, log driver, user, workdir, commande et entrypoint
- **Bascule sécurisée** : l'ancien conteneur est *renommé*, puis supprimé **seulement si** le nouveau démarre — sinon **rollback automatique** sur l'ancien
- Env et labels ne sont réinjectés que s'ils **diffèrent de l'image** : réinjecter ses défauts figerait l'ancienne version et annulerait la mise à jour
- Images locales (sans registry) : ignorées automatiquement

### Dry-run
- Vérifie les digests distants via l'API registry (sans pull)
- Calcule la taille réelle des layers à télécharger
- Estime l'espace libéré après prune
- Pour les services `build:` : lit les `FROM` du Dockerfile et vérifie l'**image de base** (les stages multi-étapes, `FROM $ARG` et `scratch` sont ignorés)
- Labels : `[OK]` à jour · `[UPDATE]` mise à jour disponible · `[FORCE]` forcé mais déjà à jour · `[LOCAL]` image locale · `[BUILD]` service construit localement (base vérifiée)

## Limites connues (images standalone)

La recréation reconstruit un `docker run` à partir de l'inspect : quelques réglages ne sont **pas** reproduits — le script **prévient explicitement** quand il en détecte :

- `ulimits`, `sysctls`, réservations **GPU**
- **alias réseau** et **IP statiques**
- entrypoint personnalisé **multi-arguments**

Pour ces cas (et de manière générale), une **stack Compose** reste préférable : le chemin Compose (`docker compose up -d`) reproduit tout fidèlement.

## Prérequis

- `docker` + plugin `compose`
- `jq`
- `xargs`
- Root (ou accès socket Docker)

## Logs

Fichier : `/var/log/docker-update.log`  
Rotation automatique à 5 Mo (backup horodaté, nettoyage des backups > 30 jours).

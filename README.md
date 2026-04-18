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
sudo upgrade_docker --force          # Re-pull toutes les images (sans recréer les conteneurs si digest inchangé)
sudo upgrade_docker --force-reboot   # Recrée les conteneurs même si le digest n'a pas changé
sudo upgrade_docker --compose-only
sudo upgrade_docker --standalone-only
sudo upgrade_docker --no-prune
sudo upgrade_docker --notify admin@example.com
sudo upgrade_docker --search-dir /srv
```

## Options

| Option | Description | Défaut |
|---|---|---|
| `--dry-run` | Vérifie les mises à jour sans rien modifier | — |
| `--parallel N` | Jobs simultanés | 20 |
| `--timeout N` | Timeout en secondes par pull | 300 |
| `--search-dir DIR` | Répertoire à scanner (répétable) | `/opt /home /docker_data` |
| `--no-prune` | Ne pas supprimer les images obsolètes | — |
| `--compose-only` | Traite uniquement les stacks Compose | — |
| `--standalone-only` | Traite uniquement les images standalone | — |
| `--force` | Force le re-pull de toutes les images (sans recréer les conteneurs si digest inchangé) | — |
| `--force-reboot` | Recrée les conteneurs même si le digest n'a pas changé | — |
| `--exclude IMAGE` | Exclut une image (répétable) | — |
| `--image IMAGE` | Traite uniquement cette image | — |
| `--notify EMAIL` | Envoie un mail en cas d'échec (nécessite `mail`) | — |
| `--log FILE` | Fichier de log | `/var/log/docker-update.log` |

## Fonctionnement

### Stacks Docker Compose
- Scanne les répertoires configurés à la recherche de `docker-compose.yml` / `docker-compose.yaml`
- Exclut `vendor/`, `node_modules/`, `html/apps/`, `.git/`
- En mode normal : `docker compose pull` + `docker compose up -d --remove-orphans`
- Avec `--force-reboot` : `docker compose up -d --force-recreate`

### Images standalone
- Liste toutes les images locales hors stacks Compose
- Pull parallèle avec `xargs -P`
- Si l'image change de digest : recrée le conteneur via `docker run` (reconstruit depuis `docker inspect`)
- Images locales (sans registry) : ignorées automatiquement

### Dry-run
- Vérifie les digests distants via l'API registry (sans pull)
- Calcule la taille réelle des layers à télécharger
- Estime l'espace libéré après prune
- Labels : `[OK]` à jour · `[UPDATE]` mise à jour disponible · `[FORCE]` forcé mais déjà à jour · `[LOCAL]` image locale

## Prérequis

- `docker` + plugin `compose`
- `jq`
- `xargs`
- Root (ou accès socket Docker)

## Logs

Fichier : `/var/log/docker-update.log`  
Rotation automatique à 5 Mo (backup horodaté, nettoyage des backups > 30 jours).

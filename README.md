# upgrade_docker

Script Bash de mise à jour des conteneurs Docker **qui tournent** : stacks Compose
(y compris celles déployées par Portainer) et conteneurs standalone. Il ne télécharge
que ce qui a réellement changé, et ne relance que ce qui doit l'être.

## Le contrat

| Situation | Ce que fait le script |
|---|---|
| Image à jour | Rien. Le conteneur n'est pas touché. |
| Image d'un registry, mise à jour dispo | Pull, puis `up -d` de la stack. |
| Service Compose `build:`, base à jour | Rien. Aucune reconstruction. |
| Service Compose `build:`, base (`FROM`) périmée | Pull de la base, `docker compose build`, `up -d`. |
| Image construite localement, absente du registry | Ignorée, jamais comptée en échec. |
| Conteneur standalone, mise à jour dispo | Pull + signalement `[À RECRÉER]`. **Pas de recréation.** |
| Stack ou conteneur à l'arrêt | Invisible pour le script : il ne voit que ce qui tourne. |

Autrement dit : **rien ne bouge tant que rien n'a changé en amont**, et ce qui est
éteint reste éteint.

> ⚠️ Le script met à jour des **images**. Il ne redéploie pas tes modifications de code
> local : pour ça, c'est `docker compose up -d --build`.

## Installation

```bash
sudo cp upgrade_docker /usr/local/bin/upgrade_docker
sudo chmod +x /usr/local/bin/upgrade_docker
```

## Utilisation

```bash
sudo upgrade_docker --dry-run                     # voir ce qui serait fait
sudo upgrade_docker                               # mettre à jour

sudo upgrade_docker --image nginx:latest          # une seule image
sudo upgrade_docker --exclude plex                # tout sauf cette stack / ce conteneur
sudo upgrade_docker --no-build                    # ne rien reconstruire
sudo upgrade_docker --force-reboot                # recréer les stacks même sans changement
sudo upgrade_docker --notify admin@example.com    # mail si échec
```

## Options

| Option | Description | Défaut |
|---|---|---|
| `--dry-run` | Liste les mises à jour disponibles, ne touche à rien | - |
| `--image IMAGE` | Ne traite que les stacks/conteneurs qui utilisent cette image | - |
| `--exclude NOM` | Ignore une stack, un conteneur ou une image (répétable) | - |
| `--compose-only` | Stacks Compose uniquement | - |
| `--standalone-only` | Conteneurs standalone uniquement | - |
| `--no-build` | Ne reconstruit pas les services `build:` (même si leur base a bougé) | - |
| `--force-reboot` | Recrée les stacks même sans nouvelle image (`up -d --force-recreate`) | - |
| `--no-prune` | Garde les images obsolètes | - |
| `--timeout N` | Timeout d'un pull, en secondes, par stack ou image | 600 |
| `--log FILE` | Fichier de log | `/var/log/docker-update.log` |
| `--notify EMAIL` | Mail **en cas d'échec uniquement** (nécessite `mail`) | - |

## Fonctionnement

### Découverte par les conteneurs qui tournent

Pas de scan de répertoires : les stacks sont découvertes via les labels
`com.docker.compose.*` des conteneurs en cours d'exécution, qui donnent le projet,
ses fichiers compose et ses fichiers d'env. Conséquences :

- une stack **éteinte n'est jamais démarrée** ni même vue ;
- pas de piège du vieux clone oublié : seul le compose réellement déployé compte ;
- les stacks **Portainer** sont gérées : leurs labels pointent vers `/data/compose/N`
  (chemin vu depuis le conteneur Portainer), le script les traduit via le mountpoint
  du volume `portainer_data`. C'est pour ça qu'il faut être **root**.

### Services `build:`

Le script lit les `FROM` du Dockerfile (stages multi-étapes, `FROM $ARG` et `scratch`
ignorés) et vérifie si l'image de **base** a une mise à jour. Seulement dans ce cas :
pull explicite de la base, `docker compose build`, puis `up -d`. Sans ça, une image
construite localement vieillirait indéfiniment.

### `up -d` seulement si nécessaire

`docker compose up -d` n'est lancé que si un conteneur tourne sur une image plus
vieille que celle que porte son tag (ou avec `--force-reboot`). Un `up -d`
inconditionnel n'est pas neutre : Compose recrée systématiquement les services en
`network_mode: service:X`, soit un redémarrage pour rien à chaque passage.

### Standalone : pull, pas de recréation

Les conteneurs hors Compose sont pullés, puis **signalés** `[À RECRÉER]` : la
recréation reste manuelle. Un `docker run` reconstruit depuis l'inspect perd toujours
un réglage (alias réseau, ulimits, GPU…) ; la bonne réponse est de passer le conteneur
en stack Compose, qui elle est entièrement automatisée.

### Vérification des mises à jour

`docker buildx imagetools inspect` compare le digest du registry aux `RepoDigests`
locaux, avec les identifiants docker configurés. Résultat mis en cache par image :
Docker Hub répond 429 quand on l'interroge trop (bases partagées entre services).
Une image absente du registry est considérée comme buildée localement et ignorée en
silence.

### Code de sortie

| Code | Signification |
|---|---|
| `0` | Aucun échec (y compris s'il n'y avait rien à faire) |
| `1` | Au moins un échec, c'est aussi ce qui déclenche `--notify` |

## Prérequis

- `docker` + plugins `compose` et `buildx`
- `jq`, `awk`
- Bash ≥ 4
- Root (volume Portainer illisible sinon), ou accès au socket Docker si pas de Portainer

## Logs

Fichier : `/var/log/docker-update.log` (modifiable via `--log`), sortie dupliquée
sur le terminal.

## Historique

Cette version est une réécriture complète (168 lignes). La précédente (~1100 lignes :
scan de répertoires, recréation automatique des standalone avec rollback, dry-run par
API registry, fichier de configuration) reste disponible dans l'historique git si l'un
de ces comportements te manque.

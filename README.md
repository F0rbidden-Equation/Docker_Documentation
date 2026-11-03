# Docker_Documentation

# Docker — Guide pratique et commandes essentielles 🐳

> Fiche de révision Docker en français pour **Ubuntu**, **Kali Linux**, le développement web et les laboratoires de cybersécurité.
>
> Mise à jour : novembre 2026. Exemples destinés à un environnement de test local.

## Sommaire

1. [Comprendre Docker](#1-comprendre-docker)
2. [Installation](#2-installation)
3. [Service Docker et vérifications](#3-service-docker-et-vérifications)
4. [Images Docker](#4-images-docker)
5. [Conteneurs](#5-conteneurs)
6. [Ports et volumes](#6-ports-et-volumes)
7. [Réseaux](#7-réseaux)
8. [Dockerfile](#8-dockerfile)
9. [Docker Compose](#9-docker-compose)
10. [Mini-lab web local](#10-mini-lab-web-local)
11. [Diagnostic et dépannage](#11-diagnostic-et-dépannage)
12. [Nettoyage](#12-nettoyage)
13. [Bonnes pratiques de sécurité](#13-bonnes-pratiques-de-sécurité)
14. [Aide-mémoire rapide](#14-aide-mémoire-rapide)
15. [Références](#15-références)

---

## 1. Comprendre Docker

Docker permet d'exécuter des applications dans des **conteneurs** : des environnements isolés qui partagent le noyau du système hôte (sous Linux).

| Terme | Définition |
|---|---|
| **Image** | Modèle en lecture seule servant à créer des conteneurs. |
| **Conteneur** | Instance exécutable d'une image. |
| **Dockerfile** | Instructions permettant de construire une image. |
| **Registry** | Dépôt d'images, par exemple Docker Hub. |
| **Volume** | Stockage persistant indépendant du cycle de vie du conteneur. |
| **Network** | Réseau permettant la communication des conteneurs. |
| **Docker Compose** | Outil pour définir et gérer plusieurs services dans un fichier YAML. |

**Principe :** `Dockerfile → image → conteneur`.

Un conteneur **n'est pas une machine virtuelle complète** : les conteneurs Linux partagent le noyau de l'hôte. Ne pas considérer Docker comme une frontière de sécurité équivalente à une VM pour exécuter du code hostile.

## 2. Installation

### 2.1 Kali Linux (méthode simple officielle Kali)

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```

**Attention :** sur Kali, le paquet prévu pour Docker Engine est `docker.io`, pas `docker`. Pour Compose, vérifier d'abord les paquets disponibles :

```bash
apt-cache policy docker-compose-v2 docker-compose-plugin
```

Si `docker-compose-v2` est disponible dans les dépôts Kali :

```bash
sudo apt install docker-compose-v2
```

Sinon, suivre la documentation d'installation de Docker Compose pour la distribution et vérifier avec :

```bash
docker compose version
```

Ne pas mélanger sans vérification les paquets Docker des dépôts Kali avec ceux du dépôt officiel Docker CE.

### 2.2 Ubuntu (dépôt officiel Docker)

Ces commandes ciblent une **nouvelle installation**. Si Docker est déjà installé, vérifier les paquets existants et les éventuels conflits avant de changer de dépôt.

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF2
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF2

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Tester :

```bash
sudo docker run --rm hello-world
docker --version
docker compose version
```

### 2.3 Utiliser Docker sans `sudo` (facultatif)

```bash
sudo usermod -aG docker "$USER"
```

Se **déconnecter puis se reconnecter**, puis tester `docker ps`. Le groupe `docker` donne des privilèges équivalents à root : sur un poste sensible, conserver l'utilisation de `sudo` ou envisager le mode rootless.

## 3. Service Docker et vérifications

```bash
sudo systemctl status docker       # État du service
sudo systemctl start docker        # Démarrer
sudo systemctl stop docker         # Arrêter
sudo systemctl restart docker      # Redémarrer
sudo systemctl enable docker       # Activer au démarrage
sudo systemctl enable --now docker # Activer et démarrer

docker version                     # Client et serveur
docker info                        # Configuration et statistiques
docker compose version             # Version de Compose
```

## 4. Images Docker

```bash
docker image ls                    # Afficher les images
docker pull nginx:alpine           # Télécharger une image
docker image inspect nginx:alpine  # Informations détaillées
docker image history nginx:alpine  # Historique des couches
docker rmi nginx:alpine            # Supprimer une image inutilisée
docker image prune                 # Images pendantes inutilisées
```

Une image comporte généralement un **nom** et un **tag**, par exemple `nginx:alpine`. Pour une meilleure reproductibilité, fixer une version précise ou un digest plutôt que `latest`.

## 5. Conteneurs

### Créer et lancer

```bash
docker run --rm hello-world

docker run -d --name serveur-web -p 127.0.0.1:8080:80 nginx:alpine
# -d : mode détaché
# --name : nom du conteneur
# -p : port hôte:port conteneur
```

Visiter **http://localhost:8080**.

### Consulter et piloter

```bash
docker ps                                # Conteneurs en cours
docker ps -a                             # Tous les conteneurs
docker stop serveur-web                  # Arrêter
docker start serveur-web                 # Redémarrer
docker restart serveur-web              # Relancer
docker logs serveur-web                 # Lire les journaux
docker logs -f --tail 100 serveur-web   # Suivre les journaux
docker inspect serveur-web              # Inspecter la configuration
docker stats                            # CPU, RAM, réseau en direct
docker top serveur-web                  # Processus du conteneur
docker exec -it serveur-web /bin/sh     # Shell dans le conteneur
docker cp ./index.html serveur-web:/usr/share/nginx/html/index.html
docker rm -f serveur-web                # Supprimer (force l'arrêt)
```

> `docker exec` nécessite un conteneur **en cours d'exécution**. Les images Alpine utilisent généralement `/bin/sh` plutôt que `/bin/bash`.

### Interactif

```bash
docker run --rm -it ubuntu:24.04 bash
# exit : quitte le shell et supprime ce conteneur temporaire
```

### Options utiles de `docker run`

| Option | Fonction |
|---|---|
| `-d` | Exécuter en arrière-plan |
| `-it` | Terminal interactif |
| `--rm` | Supprimer automatiquement à l'arrêt |
| `--name NOM` | Donner un nom |
| `-p 8080:80` | Publier le port 80 sur le port 8080 de l'hôte |
| `-p 127.0.0.1:8080:80` | Publier uniquement sur localhost |
| `-e CLE=VALEUR` | Passer une variable d'environnement |
| `-v source:cible` | Monter un volume ou un dossier |
| `--network NOM` | Rattacher à un réseau |
| `--memory 512m` | Limiter la RAM |
| `--cpus 1` | Limiter l'utilisation CPU |
| `--read-only` | Système de fichiers racine en lecture seule |

## 6. Ports et volumes

### Ports

```bash
# Accès uniquement depuis la machine locale (préférable pour un lab)
docker run -d --name web-local -p 127.0.0.1:8081:80 nginx:alpine

# Port publié sur toutes les interfaces : risque d'exposition réseau
docker run -d --name web-public -p 8082:80 nginx:alpine

docker port web-local
```

`EXPOSE` dans un Dockerfile documente un port, mais **ne le publie pas** : il faut `-p` ou une configuration Compose adaptée.

### Volume nommé : données persistantes

```bash
docker volume create donnees-lab
docker volume ls
docker volume inspect donnees-lab

docker run --rm -it -v donnees-lab:/data alpine sh
# Dans le conteneur : echo test > /data/fichier.txt
# Quitter avec exit : le volume reste disponible

docker volume rm donnees-lab  # Seulement lorsqu'il n'est plus utilisé
```

### Bind mount : partager un dossier du PC

```bash
mkdir -p "$HOME/docker-demo/site"
printf '<h1>Mon laboratoire Docker</h1>\n' > "$HOME/docker-demo/site/index.html"

docker run -d --name site-local \
  -p 127.0.0.1:8083:80 \
  -v "$HOME/docker-demo/site:/usr/share/nginx/html:ro" \
  nginx:alpine
```

`ro` signifie *read-only* : le conteneur ne peut pas modifier ce dossier monté. Le fichier HTML est accessible depuis **http://localhost:8083**.

## 7. Réseaux

```bash
docker network ls
docker network create lab-net
docker network inspect lab-net
docker network connect lab-net site-local
docker network disconnect lab-net site-local
docker network rm lab-net
```

Les conteneurs connectés à un **réseau bridge utilisateur** peuvent se joindre par leur nom. Exemple : un service web peut joindre une base de données nommée `db` sur le réseau partagé, sans rendre le port SQL accessible depuis le PC ou le LAN.

## 8. Dockerfile

Un `Dockerfile` décrit la construction d'une image personnalisée.

### Exemple : site web statique

Arborescence :

```text
mon-site/
├── Dockerfile
└── index.html
```

`Dockerfile` :

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

`index.html` :

```html
<!doctype html>
<html lang="fr">
  <head><meta charset="utf-8"><title>Docker Lab</title></head>
  <body><h1>Bienvenue dans mon laboratoire Docker</h1></body>
</html>
```

Depuis le dossier `mon-site/` :

```bash
docker build -t mon-site:1.0 .
docker run -d --name mon-site -p 127.0.0.1:8090:80 mon-site:1.0
```

Ouvrir **http://localhost:8090**.

Instructions fréquentes :

| Instruction | Rôle |
|---|---|
| `FROM` | Image de base |
| `RUN` | Exécuter une commande à la construction |
| `COPY` | Copier des fichiers dans l'image |
| `WORKDIR` | Définir le répertoire de travail |
| `ENV` | Définir une variable d'environnement |
| `EXPOSE` | Documenter un port |
| `USER` | Choisir l'utilisateur du processus |
| `CMD` | Commande par défaut au démarrage |
| `ENTRYPOINT` | Exécutable principal du conteneur |

Éviter de copier des secrets dans les images. Utiliser un `.dockerignore` pour exclure `.git`, fichiers temporaires et secrets.

## 9. Docker Compose

Docker Compose permet de lancer une application composée de plusieurs services avec un fichier **`compose.yaml`**.

### Exemple minimal

Créer `compose.yaml` :

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8080:80"
    restart: unless-stopped
```

Commandes courantes à exécuter dans le dossier du fichier :

```bash
docker compose config           # Vérifier/afficher la configuration
docker compose up               # Démarrer au premier plan
docker compose up -d            # Démarrer en arrière-plan
docker compose up -d --build    # Construire si nécessaire puis démarrer
docker compose ps               # État des services
docker compose logs -f          # Journaux temps réel
docker compose logs -f web      # Journaux du service web
docker compose exec web sh      # Shell dans le service web
docker compose restart          # Redémarrer les services
docker compose stop             # Arrêter sans supprimer
docker compose start            # Redémarrer les services arrêtés
docker compose down             # Supprimer conteneurs et réseaux du projet
docker compose pull             # Télécharger les images plus récentes
docker compose down -v          # ATTENTION : supprimer aussi les volumes du projet
```

**Différence importante :** `stop` conserve les conteneurs ; `down` les supprime. `down` conserve normalement les volumes nommés, tandis que `down -v` les supprime.

> `docker compose` est la syntaxe moderne. `docker-compose` (avec tiret) correspond à l'ancien outil ou à une commande de compatibilité selon l'installation.

## 10. Mini-lab web local

Exemple sans vulnérabilité volontaire pour apprendre les commandes, les volumes et le réseau.

Arborescence :

```text
cyberlab/
├── compose.yaml
└── webroot/
    └── index.html
```

`webroot/index.html` :

```html
<!doctype html>
<html lang="fr">
  <head><meta charset="utf-8"><title>CyberLab</title></head>
  <body>
    <h1>CyberLab — environnement Docker local</h1>
    <p>Le serveur fonctionne.</p>
  </body>
</html>
```

`compose.yaml` :

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./webroot:/usr/share/nginx/html:ro
    networks:
      - lab-net

networks:
  lab-net:
    driver: bridge
```

Démarrer le lab :

```bash
cd cyberlab
docker compose config
docker compose up -d
docker compose ps
curl http://127.0.0.1:8080
```

Adresse : **http://localhost:8080**.

Modifier le fichier HTML dans `webroot/` et actualiser le navigateur : le changement apparaît sans reconstruire l'image grâce au bind mount.

```bash
docker compose logs -f web     # Vérifier les accès et erreurs
docker compose exec web sh     # Explorer le conteneur
docker compose down            # Arrêter et supprimer le lab
```

Pour un lab de sécurité contenant des applications volontairement vulnérables, préférer une **VM dédiée**, un réseau isolé et une publication limitée à `127.0.0.1`. Ne jamais l'exposer sur Internet.

## 11. Diagnostic et dépannage

```bash
# Docker est-il actif ?
sudo systemctl status docker

# Tous les conteneurs, même ceux arrêtés
docker ps -a

# Voir pourquoi un service s'arrête
docker logs --tail 100 NOM_CONTENEUR

# Voir les processus et ressources
docker stats

# Vérifier les ports publiés
docker port NOM_CONTENEUR

# Vérifier la configuration Compose
docker compose config

# Vérifier un service Compose
docker compose logs --tail 100 web

# Vérifier l'usage disque de Docker
docker system df

# Journaux du démon Docker sur Linux
sudo journalctl -u docker --no-pager -n 100
```

### Erreurs fréquentes

| Erreur | Piste de résolution |
|---|---|
| `permission denied` sur `/var/run/docker.sock` | Utiliser `sudo`, ou vérifier l'appartenance au groupe `docker`. |
| `Cannot connect to the Docker daemon` | Vérifier `sudo systemctl status docker`. |
| `port is already allocated` | Le port hôte est déjà occupé : changer le port dans `-p` ou Compose. |
| `no such container` | Vérifier le nom avec `docker ps -a`. |
| `docker: 'compose' is not a docker command` | Installer ou vérifier le plugin Compose compatible avec sa distribution. |
| Modifications HTML invisibles | Contrôler les chemins et bind mounts, puis actualiser le navigateur. |

## 12. Nettoyage

```bash
docker container prune   # Supprimer les conteneurs arrêtés
docker image prune       # Supprimer les images pendantes
docker network prune     # Supprimer les réseaux inutilisés
docker volume prune      # Supprimer les volumes inutilisés (attention aux données)
docker system df         # Afficher l'espace utilisé
```

**Prudence :**

```bash
docker system prune -a      # Supprime notamment les images non utilisées
docker system prune -a --volumes  # Peut également supprimer des données persistantes
```

Ne pas utiliser ces commandes globales sur un hôte hébergeant d'autres projets sans vérifier les ressources concernées.

## 13. Bonnes pratiques de sécurité

- Ne pas publier inutilement les ports sur `0.0.0.0` ; utiliser `127.0.0.1` pour les labs locaux.
- Vérifier l'origine des images et fixer des versions explicites lorsque la reproductibilité est importante.
- Éviter `--privileged`, les montages du socket `/var/run/docker.sock` et les bind mounts sensibles.
- Limiter les privilèges, utiliser un utilisateur non-root si possible, ajouter des limites de ressources.
- Garder Docker, les images et l'hôte à jour.
- Ne pas committer de mots de passe ni de secrets dans `compose.yaml`, `.env` ou `Dockerfile` ; stocker les secrets de manière adaptée et exclure `.env` du dépôt.
- Sauvegarder les volumes importants avant de nettoyer.
- **Docker n'est pas une sandbox de sécurité garantie** : isoler les labs vulnérables dans une VM dédiée.

## 14. Aide-mémoire rapide

| Besoin | Commande |
|---|---|
| Version | `docker --version` |
| Test | `docker run --rm hello-world` |
| Images | `docker images` |
| Conteneurs actifs | `docker ps` |
| Tous les conteneurs | `docker ps -a` |
| Lancer Nginx local | `docker run -d --name nginx-test -p 127.0.0.1:8080:80 nginx:alpine` |
| Logs | `docker logs -f nginx-test` |
| Terminal | `docker exec -it nginx-test sh` |
| Arrêter | `docker stop nginx-test` |
| Supprimer | `docker rm nginx-test` |
| Construire une image | `docker build -t monimage:1.0 .` |
| Démarrer Compose | `docker compose up -d` |
| État Compose | `docker compose ps` |
| Logs Compose | `docker compose logs -f` |
| Arrêter Compose | `docker compose down` |
| Stockage | `docker system df` |

## 15. Références

- [Documentation Docker](https://docs.docker.com/)
- [Installation Docker Engine sur Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Installation Docker sur Kali Linux](https://www.kali.org/docs/containers/installing-docker-on-kali/)
- [Docker Compose : démarrage](https://docs.docker.com/compose/gettingstarted/)
- [Référence des commandes Compose](https://docs.docker.com/reference/cli/docker/compose/)
- [Référence Dockerfile](https://docs.docker.com/reference/dockerfile/)

---

*Documentation pédagogique — environnement de test local et usages autorisés uniquement.*


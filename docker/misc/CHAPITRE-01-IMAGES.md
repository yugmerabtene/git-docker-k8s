# CHAPITRE-01 — Maîtriser les **images Docker** (théorie + pratique approfondie)

## 0) Prérequis & environnement

* Docker CLI ≥ 20.10 (idéalement Docker Desktop + WSL2 sous Windows, ou Docker Engine sous Linux).
* Savoir ouvrir un terminal et exécuter des commandes avec droits suffisants (`sudo` sous Linux).
* Avoir un compte Docker Hub (ou un registre privé) pour tester `push`.

---

## 1) Concepts fondamentaux (ce qu’est une image)

1. **Image = empilement de couches (layers)**
   Chaque instruction du Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`, etc.) crée une **couche de système de fichiers immuable**.
   Docker réutilise ces couches via un **cache de build**.

2. **Content-addressable storage**
   Chaque couche et l’image finale sont **adressées par leur digest SHA256**. C’est la base de la déduplication et du cache.

3. **Nom complet d’une image**

   ```
   [REGISTRY[:PORT]/][NAMESPACE/]REPOSITORY[:TAG|@DIGEST]
   ```

   Exemples :

   * `nginx:1.25` (implicite `docker.io/library/nginx:1.25`)
   * `ghcr.io/org/projet:prod`
   * `registry.example.com:5000/monapp:1.0`
   * `nginx@sha256:<digest>` (par digest, immuable)

4. **Tag vs Digest**

   * `:tag` est **mutable** (peut pointer vers une autre image demain).
   * `@sha256:…` est **immuable** (référentiel exact).
     Production : privilégier le **digest** pour garantir l’exécution d’exactement la même image.

5. **Manifeste multi-architecture**
   Une “image” publique (ex. `alpine:latest`) est souvent un **manifest list** pointant vers des builds pour `linux/amd64`, `linux/arm64`, etc. Docker choisit automatiquement selon ta plateforme (ou via `--platform`).

---

## 2) Lister les images locales — `docker images` / `docker image ls`

**Syntaxe**

```bash
docker images [OPTIONS] [REPOSITORY[:TAG]]
docker image ls [OPTIONS] [REPOSITORY[:TAG]]
```

**Options utiles**

* `-a`, `--all` : inclut **toutes** les images, y compris les couches intermédiaires (dangling).
* `-q`, `--quiet` : affiche uniquement les IDs d’images.
* `--digests` : montre aussi les digests SHA256.
* `--no-trunc` : n’abrège pas les IDs/digests.
* `--format` : sortie personnalisée (Go templates).
* `--filter`, `-f` : filtrage avancé. Filtres courants :

  * `dangling=true|false`
  * `reference=nginx:*` (glob)
  * `before=<image>` / `since=<image>`
  * `label=key` ou `label=key=value`

**Exemples**

```bash
# Basique
docker images

# Toutes les images (y compris intermédiaires)
docker images -a

# IDs seulement (pratique pour scripts)
docker images -q

# Références + taille au format custom
docker images --format '{{.Repository}}:{{.Tag}} -> {{.Size}}'

# Filtrer les images orphelines
docker images -f dangling=true

# Filtrer par motif de référence
docker images -f reference='ubuntu:*'
```

---

## 3) Télécharger depuis un registre — `docker pull`

**Syntaxe**

```bash
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

**Options importantes**

* `-a`, `--all-tags` : tire **tous les tags** d’un repo (attention à l’espace disque).
* `--platform linux/amd64|linux/arm64|…` : force l’archi ciblée (utile sur Mac M1/M2, ARM, CI multi-arch).
* `--quiet`, `-q` : sortie silencieuse.
* `--disable-content-trust` : ignore la vérification Notary (si trust activé).

**Exemples**

```bash
# Dernier tag (latest)
docker pull nginx

# Version précise
docker pull ubuntu:22.04

# Par digest (immutabilité)
docker pull nginx@sha256:2d4f27d8...

# Tirer toutes les variantes d'ubuntu (peut être massif)
docker pull ubuntu --all-tags

# Forcer l’architecture x86_64 depuis un hôte ARM
docker pull --platform linux/amd64 node:20
```

**Remarques**

* Docker ne retélécharge pas une couche déjà présente (cache local par digest).
* Sans registre explicitement indiqué, Docker utilise `docker.io`.

---

## 4) Supprimer des images — `docker rmi` / `docker image rm`

**Syntaxe**

```bash
docker rmi [OPTIONS] IMAGE [IMAGE...]
docker image rm [OPTIONS] IMAGE [IMAGE...]
```

**Options**

* `-f`, `--force` : supprime même si référencée par un conteneur **arrêté**.
* `--no-prune` : ne supprime pas les couches parentes non utilisées.

**Exemples**

```bash
# Par nom:tag
docker rmi nginx:latest

# Par ID
docker rmi 605c77e624dd

# Plusieurs
docker rmi ubuntu:22.04 nginx:1.25

# Forcé (dangereux si conteneurs s’appuient dessus)
docker rmi -f alpine
```

**Astuce de nettoyage**

```bash
# Supprimer absolument tout ce qui est image locale (radical)
docker rmi -f $(docker images -q)
```

---

## 5) Inspecter une image — `docker inspect`

**Syntaxe**

```bash
docker inspect [OPTIONS] IMAGE
```

**Options**

* `--type image` : force le type (en cas de nom ambigu).
* `--format '{{…}}'` : templating Go pour extraire un champ précis.

**Champs utiles (extraits)**

* `.Id` : SHA256 de l’image
* `.RepoTags` / `.RepoDigests`
* `.Architecture`, `.Os`
* `.Config.Env` : variables ENV baked in
* `.Config.Entrypoint` / `.Config.Cmd`
* `.RootFS.Layers` : liste des couches

**Exemples**

```bash
# Tout le JSON
docker inspect nginx

# Entrypoint et Cmd
docker inspect nginx --format '{{.Config.Entrypoint}} {{.Config.Cmd}}'

# Variables d’environnement définies dans l’image
docker inspect nginx --format '{{.Config.Env}}'

# Liste des couches
docker inspect nginx --format '{{json .RootFS.Layers}}'
```

---

## 6) Historique des couches — `docker history`

**Syntaxe**

```bash
docker history [OPTIONS] IMAGE
```

**Options**

* `--no-trunc` : évite l’ellipsage.
* `-q`, `--quiet` : IDs seulement.

**Exemple**

```bash
docker history --no-trunc ubuntu:22.04
```

Tu visualises, pour chaque couche, la commande à l’origine (`CREATED BY`), la taille et la date. Très utile pour **auditer** ce qui gonfle une image.

---

## 7) Construire des images — `docker build` (BuildKit)

**Syntaxe**

```bash
docker build [OPTIONS] PATH|URL|-
```

**Options clés**

* `-t`, `--tag name:tag` : nom+tag de l’image.
* `-f`, `--file` : Dockerfile alternatif.
* `--build-arg KEY=VALUE` : variables pour `ARG` dans Dockerfile.
* `--no-cache` : ignore le cache (reconstruit tout).
* `--pull` : tente de mettre à jour l’image de base `FROM`.
* `--platform` : cible (`linux/amd64`, `linux/arm64`, …).
* `--target` : étape d’un **multi-stage build**.
* `--label key=value` : métadonnées (owner, source, vcs, licence).
* `--progress=plain|tty|auto` : log du build.
* `--ssh default` : **forward SSH agent** (BuildKit) pour `RUN --mount=type=ssh`.
* `--secret id=…` : **secrets** BuildKit pour `RUN --mount=type=secret`.

**Exemples**

```bash
# Build simple
docker build -t monapp:1.0 .

# Build avec fichier spécifique + variables de build + pull frais
docker build -t monapp:prod -f Dockerfile.prod --build-arg APP_ENV=prod --pull .

# Build multi-arch (si buildx configuré)
docker buildx build --platform linux/amd64,linux/arm64 -t you/monapp:1.0 --push .
```

**Bonnes pratiques Dockerfile**

* Utiliser des images de base **officielles** et **minimales** (`alpine`, `ubi-micro`, `distroless`, selon besoin).
* **Multi-stage builds** pour ne garder que le runtime.
* **Ordonner** les `RUN` pour maximiser le cache (les étapes rarement modifiées d’abord).
* Nettoyer les caches (`apt-get clean`, `rm -rf /var/lib/apt/lists/*`).
* Fichier **`.dockerignore`** pour exclure `node_modules`, `.git`, artefacts, etc.
* **Épingler** des versions (`FROM python:3.11.8-slim`) pour la reproductibilité.
* Définir un `USER` non-root si possible.

**Exemple de multi-stage (Node → NGINX)**

```Dockerfile
# Stage 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: runtime minimal
FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

---

## 8) Taguer — `docker tag`

**Syntaxe**

```bash
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

**Exemples**

```bash
# Tag local → nommage pour un push
docker tag monapp:1.0 you/monapp:1.0

# Créer des canaux
docker tag monapp:1.0 you/monapp:stable
docker tag monapp:1.1-rc1 you/monapp:rc
```

**Note** : Tag = **alias** pointant vers le même ID. Zéro copie disque.

---

## 9) Publier — `docker push`

**Syntaxe**

```bash
docker push [OPTIONS] NAME[:TAG]
```

**Pré-requis**

```bash
docker login
```

**Exemples**

```bash
# Push d’un tag
docker push you/monapp:1.0

# Push d’une image multi-arch construite avec buildx (déjà --push)
docker buildx build --platform linux/amd64,linux/arm64 -t you/monapp:1.0 --push .
```

**Conseils**

* Pousser **:tag** lisibles (semver : `1.0.3`, `1.0`, `1`, `latest`) **et** publier le **digest** dans tes manifests de déploiement (immutabilité).

---

## 10) Sauvegarder / Restaurer — `docker save` & `docker load`

**Sauvegarde**

```bash
docker save -o nginx.tar nginx:latest
# ou compressé
docker save nginx:latest | gzip > nginx.tar.gz
```

**Restauration**

```bash
docker load -i nginx.tar
# ou
gunzip -c nginx.tar.gz | docker load
```

**Cas d’usage** : exporter une image vers une machine **sans accès** au registre.

---

## 11) Exporter un conteneur vs sauver une image — `docker export` / `docker import`

| Action             | Commande                           | Source         | Garde métadonnées (layers, history) |
| ------------------ | ---------------------------------- | -------------- | ----------------------------------- |
| Sauvegarder image  | `docker save`                      | Image          | Oui                                 |
| Charger image      | `docker load`                      | Archive `save` | Oui                                 |
| Exporter conteneur | `docker export <CONTAINER>`        | Conteneur      | Non (rootfs plat)                   |
| Importer rootfs    | `docker import` (→ nouvelle image) | Tar rootfs     | Non (recrée une image “bare”)       |

**Exemple**

```bash
docker export monctr > rootfs.tar
cat rootfs.tar | docker import - monimage:bare
```

---

## 12) Nettoyage d’images — `docker image prune` & co.

**Commandes utiles**

```bash
# Utilisation disque (images, layers, volumes, build cache)
docker system df
docker system df -v   # détail

# Supprimer images non référencées (orphelines)
docker image prune            # interactif
docker image prune -af        # agressif, sans confirmation
docker image prune -af --filter "until=72h"  # plus vieilles que 72h

# Tout nettoyer (dangereux : conteneurs/volumes/network non utilisés)
docker system prune -af --volumes
```

**Bonnes pratiques**

* Planifier un **prune** régulier en CI / dev machines.
* Sur serveurs prod, éviter `system prune` sans discernement : préférer des règles de rétention.

---

## 13) Manifeste multi-arch (avancé) — `docker manifest`

**Inspecter un manifest list**

```bash
docker manifest inspect nginx:latest
```

**Créer et pousser un manifest list (avec images pré-pushées)**

```bash
docker manifest create you/monapp:1.0 \
  you/monapp:1.0-amd64 \
  you/monapp:1.0-arm64

docker manifest annotate you/monapp:1.0 you/monapp:1.0-amd64 --os linux --arch amd64
docker manifest annotate you/monapp:1.0 you/monapp:1.0-arm64 --os linux --arch arm64

docker manifest push you/monapp:1.0
```

**Remarque** : avec `buildx`, on fait généralement un **build + push multi-arch** en une commande et cette étape est implicite.

---

## 14) Sécurité & intégrité (aperçu rapide)

* **Exécuter en tant qu’utilisateur non-root** (`USER`).
* **Réduire la surface** : images slim/alpine/distroless.
* **Reproductibilité** : pinner les versions, utiliser `@digest`.
* **SBOM / scan** (si dispo) : ex. `docker sbom`, `docker scout` (selon installation/versions).
* **Secrets en build** : préférer `--secret` (BuildKit) au lieu d’`ARG`.

---

## 15) Exercices dirigés (TP rapide)

1. **Pull + inspect + history**

```bash
docker pull nginx:1.25
docker inspect nginx:1.25 --format '{{.Id}} {{.Os}}/{{.Architecture}}'
docker history nginx:1.25 --no-trunc
```

2. **Build multi-stage + tag + push**

```bash
# Crée un Dockerfile multi-stage (ex. Node → Nginx ci-dessus)
docker build -t you/web:1.0 .
docker tag you/web:1.0 you/web:latest
docker login
docker push you/web:1.0
docker push you/web:latest
```

3. **Save / load**

```bash
docker save -o web.tar you/web:1.0
docker rmi you/web:1.0
docker load -i web.tar
```

4. **Prune sécurisé**

```bash
docker system df -v
docker image prune -af --filter "until=168h"   # images inutilisées > 7 jours
```

---

## 16) Cheatsheet (rappel)

* Lister / filtrer : `docker images -a`, `-f reference='repo:*'`, `--digests`, `--format`.
* Télécharger : `docker pull [--platform …] name:tag|@digest`.
* Supprimer : `docker rmi [-f]`.
* Inspecter : `docker inspect --format '{{…}}'`.
* Historique : `docker history [--no-trunc]`.
* Construire : `docker build -t name:tag [--build-arg …] [--no-cache] [--pull] [--target …]`.
* Taguer : `docker tag src:tag dst:tag`.
* Publier : `docker push`.
* Sauver/Charger : `docker save` / `docker load`.
* Export/Import rootfs : `docker export` / `docker import`.
* Nettoyer : `docker image prune` / `docker system prune`.
* Multi-arch : `docker buildx build --platform … --push` ou `docker manifest`.


# Chapitre-05 — Dockerfile & Build (BuildKit avancé)

## Objectifs d’apprentissage

* Écrire des **Dockerfiles** clairs, sûrs et performants : maîtriser les instructions, les scopes (`ARG`, `ENV`, `FROM`, `USER`, `WORKDIR`, `ENTRYPOINT`/`CMD`, `HEALTHCHECK`, etc.).
* Concevoir des **multi-stage builds** (builder → runtime) pour réduire la taille et la surface d’attaque.
* Exploiter **BuildKit** : `RUN --mount=type=cache|secret|ssh`, `COPY --from`, labels OCI, `.dockerignore`, optimisation des couches.
* Produire des images **multi-architecture** et traçables avec **buildx** (`--platform`, `--provenance`, `--sbom`, caches distants).
* Garantir **reproductibilité** et **gouvernance** (pinning versions, metadata, SBOM/provenance, politiques de tags).

## Pré-requis

* Docker Engine/CLI, BuildKit activé (Docker Desktop : par défaut ; Linux : `DOCKER_BUILDKIT=1`).
* Bases Linux (shell, permissions), notions réseau pour accès à des registries.

---

## 1) Fondamentaux du Dockerfile

### 1.1 En-tête BuildKit (recommandé)

```dockerfile
# syntax=docker/dockerfile:1.7
```

* Active les features récentes (ex. `RUN --mount=...`, `COPY --chmod/--chown`, améliorations de cache).

### 1.2 `FROM`, `ARG`, `ENV`

```dockerfile
ARG BASE_TAG=3.20
FROM alpine:${BASE_TAG} AS base

# ARG défini AVANT le FROM suivant si vous devez le réutiliser
ARG APP_VER=1.4.2
ENV TZ=UTC LANG=C.UTF-8
```

* `ARG` : disponible **à la construction** (non présent à l’exécution). Scope **local au stage** si déclaré après `FROM`.
* `ENV` : persiste **dans l’image** et sera visible au runtime.

### 1.3 `COPY` vs `ADD`

* **Préférer `COPY`** (comportement explicite).
* `ADD` **seulement** pour :
  a) extraire automatiquement un **tar** vers le FS, ou
  b) **télécharger** une URL (peu recommandé pour supply-chain).

```dockerfile
COPY --chown=10001:10001 --chmod=0755 ./bin/app /usr/local/bin/app
```

### 1.4 `WORKDIR`, `USER`, `SHELL`

```dockerfile
WORKDIR /app
# Créer un user non-root (ex. Alpine)
RUN addgroup -S app && adduser -S -G app -u 10001 app
USER 10001:10001
# SHELL utile côté Windows ou bash spécifique
```

### 1.5 `ENTRYPOINT` & `CMD` (formes exec vs shell)

* **Forme exec** (recommandée) : pas de shell implicite, signaux mieux relayés.

```dockerfile
ENTRYPOINT ["/usr/local/bin/app"]
CMD ["--help"]
```

* **Forme shell** : `ENTRYPOINT myapp.sh` (shell `/bin/sh -c`) → moins prévisible pour signaux/env.

### 1.6 `HEALTHCHECK`, `EXPOSE`, `STOPSIGNAL`, `ONBUILD`

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/health || exit 1

EXPOSE 8080              # métadonnée documentaire (aucune ouverture de port)
STOPSIGNAL SIGTERM       # signal d’arrêt conseillé
# ONBUILD : à éviter sauf images-modèles (déclenche une instruction dans l’enfant)
```

### 1.7 `.dockerignore` (critique)

* Réduire le **contexte** envoyé au démon (perf & sécurité).
* Empêcher l’embarquement involontaire de secrets/artefacts lourds.

Exemple générique :

```
.git
.gitignore
**/.env
**/node_modules
**/target
**/venv
*.pem
*.key
*.crt
*.log
dist/
build/
```

> Les patterns `!` ré-incluent des fichiers si nécessaire.

---

## 2) Multi-stage builds (patrons utiles)

### 2.1 Go (binaire statique, image runtime minimale)

```dockerfile
# syntax=docker/dockerfile:1.7

FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

### 2.2 Node.js (builder → runtime)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN --mount=type=cache,target=/root/.npm \
    npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev
USER node
EXPOSE 8080
ENTRYPOINT ["node","dist/server.js"]
```

### 2.3 Java (Maven → JRE distroless/jlink)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -DskipTests package

FROM gcr.io/distroless/java21-debian12:nonroot
WORKDIR /app
COPY --from=build /src/target/app.jar /app/app.jar
EXPOSE 8080
USER nonroot
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### 2.4 Python (wheelhouse + venv immuable)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS build
WORKDIR /w
COPY pyproject.toml poetry.lock ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --upgrade pip wheel build \
 && pip wheel --wheel-dir /wheels .

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /wheels /wheels
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-index --find-links=/wheels /wheels/*.whl \
 && useradd -u 10001 -r -s /sbin/nologin app \
 && rm -rf /wheels
COPY . .
USER 10001
EXPOSE 8000
ENTRYPOINT ["python","-m","app"]
```

---

## 3) BuildKit : `RUN --mount` (cache, secrets, ssh)

> Nécessite `# syntax=docker/dockerfile:1.x` + BuildKit activé.

### 3.1 Cache persistant entre builds

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

* Conserve un **cache** côté builder sans gonfler l’image.

### 3.2 Secrets (ne **pas** figer dans les couches)

```dockerfile
# build: docker build --secret id=npm_token,src=./.npm_token
RUN --mount=type=secret,id=npm_token \
    export NPM_TOKEN="$(cat /run/secrets/npm_token)" \
 && npm ci
```

### 3.3 Accès SSH (clés éphémères)

```dockerfile
# build: docker build --ssh default
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git
```

### 3.4 Montage depuis un autre stage (bind éphémère)

```dockerfile
# Copier des artefacts lourds sans créer de couche inutile
RUN --mount=type=bind,from=build,source=/out,target=/mnt/out \
    cp /mnt/out/app /usr/local/bin/app
```

---

## 4) `COPY` avancé & ordre des couches

* Grouper ce qui change **le moins** tôt (cache maximal).
* Exploiter `COPY --from=<stage>` pour extraire **uniquement** les artefacts nécessaires.
* Utiliser `--chown` et `--chmod` pendant `COPY` pour éviter un `chown` séparé (une couche en moins).

Exemple :

```dockerfile
COPY --from=build --chown=10001:10001 /out/app /usr/local/bin/app
```

---

## 5) Optimisations de taille & de surface

* **Base slim/distroless** lorsque possible.
* Nettoyer **dans la même couche** :

  ```dockerfile
  RUN apt-get update && apt-get install -y curl \
   && rm -rf /var/lib/apt/lists/*
  ```
* Supprimer docs/locales inutiles si acceptable (packages).
* `-ldflags="-s -w"` (Go), `strip` binaire, compilation statique si adaptée.
* **USER non-root** (et répertoires détenus par l’utilisateur).
* Éviter `ADD` avec URL (préférez `curl | tar` en build stage puis copier l’artefact).

---

## 6) Métadonnées & labels OCI (traçabilité)

Labels recommandés :

```dockerfile
LABEL org.opencontainers.image.title="acme-web" \
      org.opencontainers.image.description="Service web" \
      org.opencontainers.image.url="https://acme.example" \
      org.opencontainers.image.source="https://github.com/acme/web" \
      org.opencontainers.image.version="1.4.2" \
      org.opencontainers.image.revision="abc1234" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.licenses="Apache-2.0" \
      org.opencontainers.image.authors="Equipe Platform <platform@acme.example>"
```

* Fixez `BUILD_DATE` via `--build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)`.

---

## 7) Reproductibilité & gouvernance

* **Pinning** : images de base par tag **et** idéalement par **digest**.
* Gel des dépendances (lockfiles, `requirements.txt`, `package-lock.json`, `poetry.lock`).
* Variables d’ambiance déterministes : `TZ=UTC`, `LANG=C.UTF-8`.
* **SBOM** & **provenance** (voir section buildx) ; sauvegarder comme artefacts OCI.
* Politique : pas de `latest` en prod ; **déploiement par digest**.

---

## 8) `docker buildx` : multi-arch, SBOM, provenance, caches

### 8.1 Préparer un builder

```bash
docker buildx create --name builder-ci --use
docker buildx inspect --bootstrap
```

### 8.2 Multi-architecture + push

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/acme/web:1.4.2 \
  --push .
```

* Publie un **manifest list** (index multi-arch).

### 8.3 SBOM & provenance (attestations)

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/acme/web:1.4.2 \
  --provenance=true \
  --sbom=true \
  --push .
```

* Génère et pousse **attestations** (provenance SLSA-like) + **SBOM** (SPDX/CycloneDX).

### 8.4 Caches de build (CI/CD)

```bash
# Remplissage du cache vers un registry
docker buildx build \
  --cache-to=type=registry,ref=ghcr.io/acme/cache:web,mode=max \
  -t ghcr.io/acme/web:1.4.2 .

# Réutilisation du cache
docker buildx build \
  --cache-from=type=registry,ref=ghcr.io/acme/cache:web \
  -t ghcr.io/acme/web:1.4.3 .
```

* Alternatives : `type=local` (`--cache-to=type=local,dest=./.buildx-cache`).

---

## 9) Contextes & sources de build

* **Contexte local** (dossier courant) = **par défaut**.
* Contexte **Git** :

  ```bash
  docker build https://github.com/acme/web.git#main
  ```
* **Archives**/URL : possible mais moins contrôlé (supply-chain).

> Limiter le **contexte** via `.dockerignore`.

---

## 10) Sécurité de build (rappels)

* **Pas de secrets** dans le Dockerfile. Utiliser `RUN --mount=type=secret`.
* Ne **jamais** committer de clés dans le repo/contexte.
* Vérifier les **licences** (scans) et CVE (Trivy/Docker Scout).
* Préférer des bases **officielles**/maintenues (et mises à jour).

---

## 11) Exemples complets “prod-like”

### 11.1 Microservice web (Node) compact & traçable

```dockerfile
# syntax=docker/dockerfile:1.7
ARG NODE_VER=20-alpine
ARG APP_VER=1.4.2
ARG BUILD_DATE

FROM node:${NODE_VER} AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM node:${NODE_VER} AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN --mount=type=cache,target=/root/.npm npm run build

FROM node:${NODE_VER} AS runtime
WORKDIR /app
ENV NODE_ENV=production TZ=UTC LANG=C.UTF-8
LABEL org.opencontainers.image.title="acme-web" \
      org.opencontainers.image.version="${APP_VER}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://github.com/acme/web"
COPY --from=build /app/dist ./dist
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev \
 && addgroup -S app && adduser -S -G app -u 10001 app
USER 10001
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node dist/healthcheck.js || exit 1
ENTRYPOINT ["node","dist/server.js"]
```

Build & push multi-arch avec attestations :

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --build-arg APP_VER=1.4.2 \
  --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -t ghcr.io/acme/web:1.4.2 \
  --provenance=true --sbom=true \
  --push .
```

### 11.2 Go (binaire unique distroless)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
    go build -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER 65532
ENTRYPOINT ["/app"]
```

---

## 12) Commandes utiles & aide-mémoire

```bash
# Build local classique
docker build -t acme/app:1.0 .

# Build sans cache et en tirant la base à jour
docker build --no-cache --pull -t acme/app:clean .

# Build multi-arch + push + attestations
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/acme/app:1.0 --provenance --sbom --push .

# Caches registry
docker buildx build \
  --cache-to=type=registry,ref=ghcr.io/acme/cache:app,mode=max \
  --cache-from=type=registry,ref=ghcr.io/acme/cache:app \
  -t ghcr.io/acme/app:1.1 .

# Inspecter couches & metadata
docker history ghcr.io/acme/app:1.0
docker inspect ghcr.io/acme/app:1.0 | jq
```

---

## 13) Checklist de clôture (qualité du Dockerfile)

* Image de base **pinnée** (tag clair, idéalement **digest**).
* **Multi-stage** : pas d’outils de build dans le runtime ; binaire/artefacts **seuls**.
* `.dockerignore` propre ; **pas de secrets** embarqués.
* **USER non-root** ; permissions correctes au `COPY --chown/--chmod`.
* **HEALTHCHECK** pertinent ; `EXPOSE` documentaire ; `ENTRYPOINT` en **forme exec**.
* **Labels OCI** renseignés (source, version, révision, created, licences).
* **Taille** raisonnable (layers limitées, caches nettoyés).
* Build reproductible : locks/versions ; **SBOM & provenance** générés ; politique de **tags/digests** conforme.


# Chapitre-01 — Images Docker

## Objectifs d’apprentissage

* Comprendre le **modèle OCI** : couches (layers), manifeste, digest, index multi-architecture.
* Savoir **nommer**, **tirer** (pull), **construire** (build), **taguer**, **publier** (push) et **nettoyer** des images.
* Savoir **inspecter** (metadata, couches), **analyser l’historique**, **sauvegarder/restaurer** des images.
* Mettre en place des **bonnes pratiques** de taille, de reproductibilité et de gouvernance (tags vs digests).

## Pré-requis

* Docker Engine/CLI opérationnel.
* Bases Linux (shell), notions réseau (utile pour les registres).

---

## 1) Concepts fondamentaux (OCI)

* **Image** : empilement de **couches** en lecture seule (layers), immuable.
* **Manifeste** : description d’une image (config + liste des couches).
* **Digest** : empreinte SHA-256 du manifeste (`sha256:…`) ⇒ identifiant **immuable**.
* **Tag** : alias **mutable** (ex. `:1.4`, `:latest`) pointant vers un digest.
* **Index (manifest list)** : pointeur multi-arch (ex. `linux/amd64`, `linux/arm64`).

---

## 2) Nommage correct d’une image

**Grammaire :**
`[REGISTRY_HOST[:PORT]]/[NAMESPACE]/REPOSITORY[:TAG]` *ou* `@[DIGEST]`

**Exemples :**

* `ubuntu:22.04` (registre par défaut : Docker Hub, namespace implicite `library/`).
* `ghcr.io/monorg/monapp:web` (GitHub Container Registry).
* `registry.example.com/team/api@sha256:…` (par **digest** ⇒ immuable).

**Règles utiles :**

* Le tag par défaut est `:latest` **uniquement** si tag omis.
* Utiliser le **digest** pour déployer en prod (immutabilité).
* Garder des **tags sémantiques** (SemVer : `1.4.2`) et des **canaux** (`1.4`, `stable`, `rc`) pour l’humain.

---

## 3) Suite “docker image …” (panorama + options clés)

### 3.1 Lister

```bash
docker image ls                  # alias: docker images
docker image ls --digests
docker image ls --filter dangling=true
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.Size}}'
```

**Options utiles :**

* `--all` (inclure les images intermédiaires)
* `--filter reference=pattern` (ex. `myrepo:*`)
* `--digests` (afficher les digests)

### 3.2 Tirer (pull)

```bash
docker image pull nginx
docker image pull --platform linux/arm64 nginx:1.27
docker image pull --all-tags alpine
```

**Options clés :**

* `--platform` : tirer une variante spécifique (si multi-arch).
* `--all-tags` : tirer **tous** les tags du repo demandé.

### 3.3 Construire (build)

> CH-05 couvre la construction avancée ; ici, **bases nécessaires** au chapitre Images.

```bash
docker image build -t myapp:1.0 .
docker image build -t myapp:1.0 -f Dockerfile.prod .
docker image build --no-cache --pull -t myapp:clean .
```

**Options clés :**

* `-t` : nommage (repo:tag)
* `-f` : chemin du Dockerfile
* `--build-arg KEY=VAL` : paramètre de build
* `--no-cache` : ignorer le cache de build
* `--pull` : forcer la mise à jour de l’image de base
* `--platform` : construire pour une arch (via Buildx pour multi-arch réel)

### 3.4 Taguer

```bash
docker image tag myapp:1.0 registry.example.com/prod/myapp:1.0
docker image tag myapp:1.0 myapp:stable
```

* Un **tag** est un **pointeur mutable** vers un digest.

### 3.5 Pousser (push)

```bash
docker login registry.example.com
docker image push registry.example.com/prod/myapp:1.0
```

* Exige des **droits** sur le registre.
* **Conseil** : poussez des tags **versionnés**, pas seulement `latest`.

### 3.6 Inspecter (inspect)

```bash
docker image inspect myapp:1.0
docker image inspect --format '{{.Id}} {{.Os}}/{{.Architecture}}' myapp:1.0
docker image inspect --format '{{json .RootFS.Layers}}' myapp:1.0 | jq
```

* Accès aux **métadonnées**, couches, labels, config.

### 3.7 Historique (history)

```bash
docker image history myapp:1.0
docker image history --no-trunc myapp:1.0
```

* Visualise les **instructions** à l’origine des couches et leurs tailles.

### 3.8 Sauvegarder / Restaurer (save/load)

```bash
docker image save myapp:1.0 > myapp.tar
docker image load < myapp.tar
```

* **Transport** d’images entre hôtes **sans** registre.

### 3.9 Importer depuis un tar rootfs (import) — différent de save/load

```bash
cat rootfs.tar | docker image import - mybase:raw
```

* Crée une image à partir d’un **système de fichiers** (perd la meta d’historique).

### 3.10 Supprimer & nettoyer

```bash
docker image rm myapp:old
docker image prune           # supprime les images "dangling" (sans tag)
docker system df             # espace disque (images/containers/volumes)
docker system prune -a       # agressif : tout ce qui n’est pas référencé
```

* `rm` échoue si l’image est **utilisée** par un conteneur (même arrêté) ; supprimer d’abord le conteneur.

---

## 4) Commandes “classiques” alias de `docker image …`

* `docker images`  ≡ `docker image ls`
* `docker rmi`     ≡ `docker image rm`
* `docker pull`    ≡ `docker image pull`
* `docker push`    ≡ `docker image push`
* `docker build`   ≡ `docker image build`
* `docker history` ≡ `docker image history`
* `docker inspect` ≡ `docker image inspect`
* `docker save`/`docker load` idem.

---

## 5) Différence **save/load** vs **export/import**

* **save/load** (images) : conserve **manifeste + couches + config**. Parfait pour **transporter** une image OCI complète.
* **export/import** (conteneurs → image) : `docker export` capture un **rootfs** conteneur, puis `docker import` en fait une image **sans l’historique ni metadata**.

**Exemple :**

```bash
docker create --name t ubuntu:22.04 sleep infinity
docker export t > rootfs.tar
cat rootfs.tar | docker import - ubuntu:min
```

---

## 6) Multi-architecture (aperçu indispensable)

* Les images “officielles” modernes publient un **index** multi-arch.
* `docker pull --platform linux/arm64 node:20` tire la variante **arm64**.
* `docker inspect --format '{{.Architecture}}' node:20` vérifie la cible.

> La **construction** multi-arch (Buildx, QEMU/binfmt) est détaillée en CH-05/CH-11.

---

## 7) Bonnes pratiques de **construction** (niveau “images”)

*(Le détail avancé est en CH-05 ; voici les indispensables pour CH-01.)*

* **.dockerignore** : exclure `node_modules`, `target`, `.git`, etc.
* **Multi-stage** : builder → runtime (réduire la taille, surface d’attaque).
* **Pinning** : fixer versions (apt/yum/apk, langages, OS base).
* **Nettoyage** dans la même couche :

  ```Dockerfile
  RUN apt-get update && apt-get install -y curl \
      && rm -rf /var/lib/apt/lists/*
  ```
* **USER** non-root si possible : `USER 10001:10001`.
* **Labels OCI** pour la traçabilité :

  ```Dockerfile
  LABEL org.opencontainers.image.source="https://github.com/org/app" \
        org.opencontainers.image.version="1.4.2" \
        org.opencontainers.image.revision="abc1234"
  ```

---

## 8) Gouvernance des **tags** & digests

* **Tags sémantiques** : `1.4.2` (patch), `1.4` (minor), `1` (major).
* **Canaux** : `dev`, `rc`, `stable`.
* Éviter d’**écraser** un tag prod existant (politique d’**immutabilité** souhaitable).
* Déployer par **digest** (ex. `myapp@sha256:…`) pour une reproductibilité totale.
* Mettre en place des **politiques de rétention** et de nettoyage (cf. Annexe C du syllabus).

---

## 9) Exemples courants (pas-à-pas)

### 9.1 Tirer une image précise par digest

```bash
# 1) On récupère d’abord l’image et on repère son digest
docker pull nginx:1.27
docker image inspect --format '{{index .RepoDigests 0}}' nginx:1.27
# 2) On consomme par digest (immutable)
docker run -d --name web nginx@sha256:<digest>
```

### 9.2 Construire et pousser une image versionnée

```bash
# Build local
docker build -t ghcr.io/acme/api:1.4.2 .

# (si besoin) Login
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push
docker push ghcr.io/acme/api:1.4.2
```

### 9.3 Lister les images “dangling” et nettoyer

```bash
docker image ls --filter dangling=true
docker image prune -f
```

### 9.4 Sauvegarder et restaurer une image sur un autre hôte

```bash
docker save ghcr.io/acme/api:1.4.2 > api_1.4.2.tar
# scp api_1.4.2.tar user@remote:/tmp/
ssh user@remote 'docker load < /tmp/api_1.4.2.tar'
```

### 9.5 Extraire des infos ciblées (Go templates)

```bash
# ID, OS/ARCH, taille
docker image inspect myapp:1.0 \
  --format 'ID={{.Id}} OS/ARCH={{.Os}}/{{.Architecture}} Size={{.Size}}'

# Liste des couches en JSON
docker image inspect myapp:1.0 --format '{{json .RootFS.Layers}}' | jq
```

---

## 10) “Do & Don’t” rapides

**Do**

* Pinner les versions et les bases d’images.
* Nettoyer les caches (apt/apk/npm) **dans la même couche**.
* Utiliser `.dockerignore`.
* Publier des **tags versionnés** + un **digest** pour la prod.
* Documenter vos **labels OCI** (source, version, révision).

**Don’t**

* Ne mettez **jamais** de secrets dans l’image (ils finissent dans les couches).
* Évitez de dépendre de `latest` en prod.
* N’écrasez pas des tags stables déjà publiés (sauf politique assumée).
* N’oubliez pas de **pruner** régulièrement (images/volumes non utilisés).

---

## 11) Aide-mémoire (cheat-sheet minimal)

```bash
# Lister
docker images --digests
docker images --filter reference='acme/*'

# Pull
docker pull --platform linux/arm64 alpine:3.20

# Build (basique)
docker build -t acme/app:1.0 .

# Tag / Push
docker tag acme/app:1.0 ghcr.io/acme/app:1.0
docker login ghcr.io
docker push ghcr.io/acme/app:1.0

# Inspect / History
docker inspect acme/app:1.0
docker history acme/app:1.0

# Save / Load
docker save acme/app:1.0 > app.tar
docker load < app.tar

# Prune / Espace
docker image prune -f
docker system df
```

---

## 12) Checklist de clôture (qualité d’une image)

* Base **minimale** et **pin** de version (digest si possible).
* `.dockerignore` propre (évite d’embarquer des masses de fichiers).
* **Multi-stage** si build : runtime épuré.
* **USER** non-root ; pas de secrets ; labels OCI renseignés.
* Taille et nombre de couches **raisonnés** ; `history` justifiable.
* Tagging **cohérent** (SemVer/canaux) ; digest référencé pour la prod.
* Image **poussée** dans un registre de confiance.


# Chapitre-01 — Images Docker (version enrichie, **pas-à-pas**)

Objectif : repartir des bases **avec méthode** et t’amener jusqu’à des images **propres, traçables, reproductibles et prêtes prod** (tags + digest), tout en comprenant *exactement* ce que Docker manipule.

---

## Objectifs d’apprentissage (affinés)

* Comprendre le **modèle OCI** (layers, manifeste, digest, index multi-arch) et savoir **lire** ces infos.
* Savoir **nommer** correctement une image, **tirer** (`pull`), **construire** (`build`), **taguer**, **publier** (`push`) et **nettoyer**.
* Savoir **inspecter** (métadonnées / couches), **lire l’historique**, **sauvegarder/restaurer** (save/load vs export/import).
* Appliquer les bonnes pratiques : **.dockerignore**, **multi-stage**, **labels OCI**, **USER non-root**, **pinning**, **digest** en déploiement.

---

## Pré-requis & vérifications rapides

```bash
docker version            # client/serveur
docker info               # storage driver, cgroup driver, etc.
docker system df          # occupation disque (images/containers/volumes)
```

> Si tu es sur Windows, assure-toi que **WSL2** est activé et que Docker Desktop utilise WSL2.

---

## Plan d’apprentissage (étapes)

1. **Concepts OCI** → 2) **Nommage** → 3) **Lister / chercher** → 4) **Tirer** →
2. **Construire** → 6) **Tagger & pousser** → 7) **Inspecter / history** →
3. **Save/Load vs Export/Import** → 9) **Multi-arch** → 10) **Nettoyage** →
4. **Bonnes pratiques** → 12) **Parcours guidé** → 13) **FAQ & erreurs** → 14) **Checklist**

---

## 1) Concepts fondamentaux (OCI)

* **Image** : empilement de **couches** en lecture seule (layers). Chaque instruction Dockerfile produit (souvent) une couche.
* **Manifeste** : JSON décrivant **config** + **liste des couches**.
* **Digest** : empreinte **SHA-256** du manifeste → identifiant **immuable** (`@sha256:…`).
* **Tag** : alias **mutable** (ex. `:1.4.2`, `:stable`, `:latest`) → pratique mais **non garanti**.
* **Index (manifest list)** : “pointeur” vers plusieurs manifestes (amd64, arm64, …) pour une même **référence**.

Une *référence d’image* peut être un **tag** (`repo:1.4.2`) *ou* un **digest** (`repo@sha256:…`). En production, **préférer le digest**.

---

## 2) Nommage correct d’une image

**Grammaire**

```
[REGISTRY[:PORT]]/[NAMESPACE]/REPOSITORY[:TAG]   ou   [ ... ]@[DIGEST]
```

**Exemples**

* `ubuntu:22.04` → registre **Docker Hub** implicite + namespace `library/`.
* `ghcr.io/monorg/monapp:web` → GitHub Container Registry.
* `registry.example.com/team/api@sha256:deadbeef…` → par **digest** (immutabilité).

**Règles utiles**

* Si tu omets `:tag`, Docker suppose `:latest` (⚠️ **éviter** en prod).
* **Minuscules** et noms concis.
* **SemVer** : `1.4.2` + canaux (`1.4`, `rc`, `stable`) pour le confort humain ; **digest** pour déployer.

---

## 3) Lister / filtrer / chercher

```bash
# Lister images locales
docker image ls                       # alias: docker images
docker image ls --digests
docker image ls --filter dangling=true
docker image ls --filter reference='ghcr.io/monorg/*'
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.Size}}' | sort
```

> `dangling=true` = couches “orphelines” (tags supprimés).
> `--format` accepte les **Go templates** pour extraire des champs précis.

Recherche (Docker Hub uniquement) :

```bash
docker search nginx --limit 15
```

---

## 4) Tirer (pull) des images

```bash
docker pull nginx:1.27
docker pull --platform linux/arm64 nginx:1.27   # variante ARM64 si index multi-arch
docker pull --all-tags alpine                    # ⚠️ tire *tous* les tags du repo 'alpine'
```

**Points d’attention**

* `--platform` **tire** une variante spécifique. Pour **exécuter** une arch différente de la tienne, il faut l’émulation (QEMU/binfmt).
* `--all-tags` peut télécharger **beaucoup** de données : utilise-le avec précaution.

---

## 5) Construire (build) — bases indispensables

> Le chapitre “Dockerfile & Build” couvre le détail. Ici, on pose le **minimum vital**.

```bash
# Build avec tag
docker build -t myapp:1.0 .

# Dockerfile alternatif
docker build -t myapp:1.0 -f Dockerfile.prod .

# Build propre (ignore cache, rafraîchit l’image de base)
docker build --no-cache --pull -t myapp:clean .
```

**Options clés**

* `-t repo:tag` : nommage.
* `-f` : chemin Dockerfile.
* `--build-arg KEY=VAL` : variables **ARG** du Dockerfile.
* `--pull` : rafraîchit l’image de base.
* `--platform` : cible d’arch (multi-arch réel via **Buildx**).

**.dockerignore** (exemple)

```
.git
node_modules
target
__pycache__/
*.log
```

---

## 6) Tagger & pousser (push)

```bash
# Ajouter des tags
docker tag myapp:1.0 registry.example.com/prod/myapp:1.0
docker tag myapp:1.0 myapp:stable

# Authentification et push
docker login registry.example.com
docker push registry.example.com/prod/myapp:1.0
```

> **Astuce** : publie **toujours** un tag **versionné** (SemVer). Tu pourras **déployer par digest**.

---

## 7) Inspecter & lire l’historique

**Inspect**

```bash
docker image inspect myapp:1.0 | jq '.[0].Id, .[0].Os, .[0].Architecture'
docker image inspect --format '{{json .RootFS.Layers}}' myapp:1.0 | jq
docker image inspect --format '{{json .Config.Labels}}' myapp:1.0 | jq
```

**Historique**

```bash
docker image history myapp:1.0
docker image history --no-trunc myapp:1.0
```

> `history` révèle les **instructions** (RUN/COPY/…) et la **taille** par couche.
> Utile pour **optimiser** (fusionner RUN, nettoyer caches) et **auditer** ce qui compose l’image.

---

## 8) Sauvegarder/Restaurer vs Exporter/Importer

**Images** (avec métadonnées OCI) :

```bash
docker save ghcr.io/acme/api:1.4.2 > api_1.4.2.tar
docker load < api_1.4.2.tar
```

**Rootfs d’un conteneur** (sans l’historique Dockerfile) :

```bash
docker create --name t ubuntu:22.04 sleep infinity
docker export t > rootfs.tar
cat rootfs.tar | docker import - ubuntu:min
```

> `save/load` ≠ `export/import` : le premier conserve le **manifeste + layers**, le second **aplati** un rootfs en **une** image (sans historique).

---

## 9) Multi-architecture (aperçu utile)

* Les images modernes publient un **index** multi-arch.
* Tu peux voir quelle arch est tirée :

```bash
docker pull node:20
docker image inspect --format '{{.Os}}/{{.Architecture}}' node:20
```

* Inspection d’un **index** (via Buildx) :

```bash
docker buildx imagetools inspect nginx:1.27
```

---

## 10) Nettoyage & gestion d’espace

```bash
docker image ls --filter dangling=true
docker image prune -f                # supprime dangling
docker system df                     # récapitulatif espace
docker system prune -a               # agressif : supprime tout ce qui n’est pas référencé
```

> Si `rm` échoue : l’image est **utilisée** par au moins un conteneur (même arrêté). Supprime d’abord le conteneur.

---

## 11) Bonnes pratiques (niveau image)

* **Pinning** : versions OS/paquets/langages **fixées** (idéalement par **digest** de base).
* **Multi-stage** : une phase “build” lourde → une **runtime** **minimale**.
* **Nettoyage dans la même couche** :

  ```Dockerfile
  RUN apk add --no-cache curl \
   && rm -rf /var/cache/apk/*
  ```
* **USER non-root** :

  ```Dockerfile
  RUN adduser -D -u 10001 app
  USER 10001:10001
  ```
* **Labels OCI** (traçabilité) :

  ```Dockerfile
  LABEL org.opencontainers.image.title="api" \
        org.opencontainers.image.version="$VERSION" \
        org.opencontainers.image.revision="$GIT_SHA" \
        org.opencontainers.image.source="https://github.com/acme/api"
  ```
* **Base minimale** : `alpine`, `distroless`, `scratch` (selon besoin).

  > Attention aux libs (`glibc` vs `musl`) : certaines apps requièrent `glibc`.

---

## 12) Parcours guidé (pas-à-pas concret)

> On va : **tirer** → **lire digest** → **démarrer par digest** → **construire** → **tagger** → **pousser** → **inspecter** → **sauver/restaurer**.

**Étape 1 — Tirer & identifier le digest**

```bash
docker pull nginx:1.27
docker image inspect --format '{{index .RepoDigests 0}}' nginx:1.27
# → nginx@sha256:ABCD...
```

**Étape 2 — Exécuter par digest (immutabilité)**

```bash
docker run -d --name web -p 8080:80 nginx@sha256:ABCD...
curl -I http://localhost:8080
```

**Étape 3 — Construire une mini image**
`Dockerfile` :

```Dockerfile
FROM alpine:3.20
RUN adduser -D -u 10001 app
USER 10001:10001
WORKDIR /app
COPY hello.sh .
RUN chmod +x hello.sh
ENTRYPOINT ["./hello.sh"]
```

`hello.sh` :

```sh
#!/bin/sh
echo "Hello from $(uname -m) as user $(id -u)"
```

Build & run :

```bash
echo '#!/bin/sh\necho "Hello from $(uname -m) as user $(id -u)"' > hello.sh
docker build -t demo/hello:1.0 .
docker run --rm demo/hello:1.0
```

**Étape 4 — Tagger & pousser (ex. GHCR)**

```bash
docker tag demo/hello:1.0 ghcr.io/<org>/hello:1.0
echo "$GITHUB_TOKEN" | docker login ghcr.io -u <user> --password-stdin
docker push ghcr.io/<org>/hello:1.0
```

**Étape 5 — Inspecter l’historique & les labels**

```bash
docker image history demo/hello:1.0
docker image inspect --format '{{json .Config.Labels}}' demo/hello:1.0 | jq
```

**Étape 6 — Save/Load (transfert offline)**

```bash
docker save demo/hello:1.0 > hello.tar
# ... copier sur une autre machine ...
docker load < hello.tar
```

**Étape 7 — Nettoyer proprement**

```bash
docker stop web && docker rm web
docker image prune -f
docker system df
```

---

## 13) FAQ & erreurs fréquentes (solutions rapides)

* **`denied: requested access to the resource is denied` (push)**
  → Mauvais `docker login` ou droits insuffisants sur le registre.

* **`manifest unknown` / `pull access denied`**
  → Tag inexistant, repo privé, faute de frappe sur le nom.

* **`no space left on device`**
  → `docker system df` puis `docker system prune -a` (⚠️), ou agrandir le disque.

* **Je tire `arm64` sur une machine `amd64`, et ça ne démarre pas**
  → Il faut l’émulation (QEMU/binfmt) ou une cible `amd64`.

* **J’ai oublié `.dockerignore`, mon image est énorme**
  → Ajoute-le et reconstruis ; vérifie `docker history` pour localiser les couches lourdes.

---

## 14) Checklist finale (qualité image)

* [ ] **.dockerignore** précis (pas de `.git`, `node_modules`, artefacts build).
* [ ] **Multi-stage** si build d’app (runtime **léger**).
* [ ] **USER non-root**, pas de secrets en dur, **labels OCI** complets.
* [ ] **Pinning** des versions ; base image raisonnable (`alpine`/`distroless`/`debian-slim`).
* [ ] **History** cohérent ; caches nettoyés **dans la même couche**.
* [ ] **Tag SemVer** publié ; **digest** noté pour déploiement.
* [ ] Image poussée sur un **registre de confiance** ; espace local **nettoyé**.

---

### Aide-mémoire (condensé)

```bash
# Lister / filtrer
docker images --digests
docker images --filter reference='ghcr.io/org/*'

# Pull
docker pull --platform linux/arm64 alpine:3.20

# Build
docker build -t org/app:1.0 -f Dockerfile .

# Tag / Push
docker tag org/app:1.0 ghcr.io/org/app:1.0
docker login ghcr.io && docker push ghcr.io/org/app:1.0

# Inspect / History
docker inspect org/app:1.0
docker history org/app:1.0

# Save / Load
docker save org/app:1.0 > app.tar
docker load < app.tar

# Nettoyage
docker image prune -f
docker system df
```

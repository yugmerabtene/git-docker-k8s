# CHAPITRE-05 — REGISTRY (OCI Registry, Docker Hub/GHCR/GitLab, self-host, sécurité, CI/CD)

## 0) Objectifs & prérequis

**Objectifs**

* Comprendre le modèle **OCI Registry** (repository, manifest, tag, digest, layers, index multi-arch).
* Savoir **authentifier**, **taguer**, **pousser**, **tirer** des images et artefacts (SBOM, attestations).
* Gérer des **registres publics** (Docker Hub, GHCR, GitLab) et **privés** (registry:2, Harbor, Nexus).
* Mettre en place **TLS**, **auth**, **cache proxy**, **politiques d’immuabilité**, **GC** (garbage-collect).
* Intégrer la **signature (cosign)**, **SBOM** et **provenance** dans un pipeline **CI/CD**.

**Prérequis**

* Docker Engine/CLI ≥ 20.10 (BuildKit activé).
* Accès administrateur sur la machine (Linux/WSL2/Windows/macOS).
* Comptes sur les registres ciblés (Docker Hub, GitHub, GitLab…).

---

## 1) Modèle OCI Registry — concepts

* **Repository** : ensemble d’images/artefacts nommés (ex. `nginx`, `myorg/api`).
* **Tag** : étiquette **mutable** pointant vers un **manifest** (ex. `:1.2.3`, `:latest`).
* **Digest** : identifiant **immuable** (`@sha256:...`) d’un manifest ou d’une couche.
* **Manifest** : décrit les couches (layers) d’une image et ses métadonnées.
* **Index / Manifest list (multi-arch)** : pointe vers plusieurs manifests selon l’architecture (`amd64`, `arm64`).
* **Artefacts OCI** : autres types que des images (SBOM, attestations), stockés dans le même registre.

**Noms complets**

```
[REGISTRY[:PORT]/][NAMESPACE/]REPOSITORY[:TAG|@DIGEST]
# Exemples :
docker.io/library/alpine:3.20
ghcr.io/yug/monapp:1.0
registry.example.com/prod/web@sha256:abcd...
```

---

## 2) Authentification & configuration du client

### 2.1 Connexion / déconnexion

```bash
docker login                 # par défaut : docker.io
docker login ghcr.io         # GitHub Container Registry
docker login registry.gitlab.com
docker logout ghcr.io
```

**Fichiers**

* `~/.docker/config.json` : enregistre les credentials (ou références au **credential-helper**).

**Credential helpers**

* Linux/macOS : `credsStore`: `pass`, `osxkeychain`
* Windows : `wincred`

Exemple `config.json` (extrait) :

```json
{
  "auths": {},
  "credsStore": "desktop",
  "HttpHeaders": { "User-Agent": "Docker-Client/25.0" }
}
```

---

## 3) Tagging & publication — bonnes pratiques

### 3.1 Tagger localement avant push

```bash
docker build -t myapp:1.2.3 .
docker tag myapp:1.2.3 ghcr.io/yug/myapp:1.2.3
docker tag myapp:1.2.3 ghcr.io/yug/myapp:1.2
docker tag myapp:1.2.3 ghcr.io/yug/myapp:1
docker tag myapp:1.2.3 ghcr.io/yug/myapp:latest
```

### 3.2 Pousser / tirer

```bash
docker push ghcr.io/yug/myapp:1.2.3
docker pull ghcr.io/yug/myapp:1.2.3

# Par digest (immutabilité)
docker pull ghcr.io/yug/myapp@sha256:deadbeef...
```

### 3.3 Stratégies de tags

* **SemVer** : `MAJOR.MINOR.PATCH` + canaux `:rc`, `:stable`, `:latest`.
* Production : déployer par **digest** pour garantir le binaire exact.
* Éviter `latest` dans les manifests d’infra ; accepter en dev.

---

## 4) Options utiles `pull/push`

```bash
# PULL
docker pull --platform linux/amd64 node:20      # forcer arch
docker pull --all-tags ubuntu                    # tirer tous les tags (attention disque)
docker pull --quiet alpine

# PUSH
docker push --quiet ghcr.io/yug/myapp:1.2.3
```

**Erreurs fréquentes**

* `denied: requested access is denied` : pas logué, droits insuffisants, ou nom de repo invalide (majuscules interdites).
* `toomanyrequests: rate limit exceeded` (Docker Hub) : se connecter, payer, ou mettre en place un **proxy cache**.

---

## 5) Registres publics — connexions

### 5.1 Docker Hub (`docker.io`)

* Login :

  ```bash
  docker login
  ```
* **Rate limit** si non connecté. En entreprise : miroir/proxy recommandé.

### 5.2 GitHub Container Registry (GHCR) — `ghcr.io`

* PAT requis avec scope **`read:packages`** et **`write:packages`** (et `delete:packages` si besoin).

  ```bash
  echo $GHCR_PAT | docker login ghcr.io -u <github_user> --password-stdin
  docker tag myapp ghcr.io/<org>/<repo>/myapp:1.0
  docker push ghcr.io/<org>/<repo>/myapp:1.0
  ```

### 5.3 GitLab Container Registry — `registry.gitlab.com`

* Avec CI_JOB_TOKEN (CI) :

  ```bash
  docker login -u gitlab-ci-token -p "$CI_JOB_TOKEN" registry.gitlab.com
  docker push registry.gitlab.com/<group>/<project>/myapp:1.0
  ```

### 5.4 AWS ECR

```bash
aws ecr get-login-password --region eu-west-3 \
| docker login --username AWS --password-stdin <acct>.dkr.ecr.eu-west-3.amazonaws.com

docker tag myapp:1.0 <acct>.dkr.ecr.eu-west-3.amazonaws.com/myapp:1.0
docker push <acct>.dkr.ecr.eu-west-3.amazonaws.com/myapp:1.0
```

### 5.5 GCP Artifact Registry

```bash
gcloud auth configure-docker europe-docker.pkg.dev
docker tag myapp:1.0 europe-docker.pkg.dev/<proj>/<repo>/myapp:1.0
docker push europe-docker.pkg.dev/<proj>/<repo>/myapp:1.0
```

### 5.6 Azure ACR

```bash
az acr login -n <registryName>
docker tag myapp:1.0 <registryName>.azurecr.io/myapp:1.0
docker push <registryName>.azurecr.io/myapp:1.0
```

---

## 6) Construire & pousser multi-arch + SBOM/Provenance (Buildx)

```bash
# Créer un builder multi-arch une fois
docker buildx create --use --name multi
docker buildx inspect --bootstrap

# Build & push multi-arch + SBOM + provenance
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/yug/myapp:1.2.3 \
  --provenance=mode=max \
  --sbom=spdx \
  --push .
```

**Notes**

* `--provenance` et `--sbom` publient des **artefacts OCI** liés au manifest.
* Privilégier des registres compatibles OCI artefacts (GHCR, GitLab, Harbor…).

---

## 7) Signature & attestations (Chaîne d’approvisionnement)

### 7.1 Cosign (recommandé)

* Générer des clés :

  ```bash
  cosign generate-key-pair
  ```
* Signer :

  ```bash
  cosign sign --key cosign.key ghcr.io/yug/myapp:1.2.3
  ```
* Vérifier :

  ```bash
  cosign verify --key cosign.pub ghcr.io/yug/myapp:1.2.3
  ```
* Mode **keyless** (OIDC) possible en CI (GitHub Actions, GitLab).

### 7.2 Attestations & politiques

* Attacher une attestation (SLSA provenance) :

  ```bash
  cosign attest --predicate provenance.json --type slsaprovenance ghcr.io/yug/myapp:1.2.3
  ```
* Mettre en place des **policies d’admission** (OPA/Gatekeeper, Connaisseur, Kyverno) côté cluster pour n’autoriser que des images signées.

---

## 8) Héberger son registre privé (registry:2)

### 8.1 Déploiement minimal (sans TLS — labo)

```bash
docker run -d --name registry -p 5000:5000 registry:2
docker tag alpine:3.20 localhost:5000/alpine:3.20
docker push localhost:5000/alpine:3.20
docker pull localhost:5000/alpine:3.20
```

> En prod : **toujours** TLS + auth. Sans TLS, le démon Docker refusera (sauf `insecure-registries`).

### 8.2 TLS & reverse-proxy (NGINX + Let’s Encrypt) + Basic Auth

**1) Générer `htpasswd`**

```bash
# Install apache2-utils (ou httpd-tools)
htpasswd -Bbc auth/htpasswd admin 'motdepasse_fort'
```

**2) NGINX en frontal (exemple compose)**

```yaml
services:
  registry:
    image: registry:2
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry
      REGISTRY_HTTP_ADDR: :5000
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: '["*"]'
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    volumes:
      - ./registry:/var/lib/registry
    networks: [net]

  nginx:
    image: nginx:1.25
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - ./auth:/etc/nginx/auth:ro
    depends_on: [registry]
    networks: [net]

networks: { net: {} }
```

**3) Extrait `nginx.conf`**

```nginx
server {
  listen 443 ssl;
  server_name registry.example.com;
  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  location /v2/ {
    auth_basic "Registry";
    auth_basic_user_file /etc/nginx/auth/htpasswd;

    proxy_pass http://registry:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $remote_addr;
  }
}
server {
  listen 80;
  server_name registry.example.com;
  return 301 https://$host$request_uri;
}
```

**4) Test**

```bash
docker login registry.example.com
docker tag myapp:1.0 registry.example.com/myproj/myapp:1.0
docker push registry.example.com/myproj/myapp:1.0
```

### 8.3 UI facultative

* UI légère : `Joxit/docker-registry-ui` montée en reverse-proxy (lecture seule).

### 8.4 Stockage objet (S3) & backends

Configurer `config.yml` du registry :

```yaml
version: 0.1
storage:
  s3:
    region: eu-west-3
    bucket: my-registry-bucket
    accesskey: AKIA...
    secretkey: ...
http:
  addr: :5000
```

### 8.5 Suppression & Garbage-collect

1. Activer :

```yaml
storage:
  delete:
    enabled: true
```

2. Supprimer un tag (via API/outil), **arrêter** le registry, puis :

```bash
registry garbage-collect /etc/docker/registry/config.yml
```

3. Redémarrer.

> En pratique, préférer **Harbor** ou **Nexus/Artifactory** pour politiques de rétention & suppression sûres.

### 8.6 Pull-through cache (miroir Docker Hub)

`config.yml` :

```yaml
proxy:
  remoteurl: https://registry-1.docker.io
```

Puis configurer le démon Docker des clients pour utiliser ce miroir.

---

## 9) Alternatives riches : Harbor, Nexus/Artifactory

* **Harbor** : UI, RBAC fin, **scanners** (Trivy/Clair), **signatures**, **policies d’immuabilité**, **projets**, **proxy cache**, **replication** entre registres.
* **Nexus/Artifactory** : universel (packages variés), proxy, cache, policies d’entreprise.

---

## 10) Sécurité, conformité et politiques

* **TLS partout** ; bannir `insecure-registries` (sauf lab).
* **RBAC** et **PAT** aux scopes minimaux.
* **Immuabilité** des tags sensibles (`:release`, `:prod`) côté registry (Harbor).
* **Scanning vulnérabilités** automatisé (Trivy, Docker Scout, Harbor).
* **Signature cosign** + **attestations** (provenance SLSA).
* **SBOM** obligatoire (`--sbom=spdx`).
* **Rétention/GC** : nettoyer tags et blobs orphelins ; politiques d’expiration.
* **Mirroirs**/proxy pour réduire dépendance à Docker Hub & limiter les risques de supply-chain.

---

## 11) CI/CD — exemples

### 11.1 GitHub Actions → GHCR, multi-arch + SBOM + provenance + cosign

```yaml
name: build-and-push
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write   # pour cosign keyless
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - name: Login GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push (multi-arch + SBOM + provenance)
        run: |
          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            -t ghcr.io/${{ github.repository }}/myapp:${{ github.sha }} \
            -t ghcr.io/${{ github.repository }}/myapp:latest \
            --provenance=mode=max --sbom=spdx \
            --push .

      - name: Cosign sign (keyless)
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign ghcr.io/${{ github.repository }}/myapp:latest \
            --yes
```

### 11.2 GitLab CI → GitLab Registry (CI_JOB_TOKEN)

```yaml
build:
  image: docker:25
  services: [docker:25-dind]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  script:
    - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin registry.gitlab.com
    - docker build -t registry.gitlab.com/$CI_PROJECT_PATH/myapp:$CI_COMMIT_SHA .
    - docker push registry.gitlab.com/$CI_PROJECT_PATH/myapp:$CI_COMMIT_SHA
```

### 11.3 AWS ECR (GitHub Actions)

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-region: eu-west-3
    role-to-assume: arn:aws:iam::123:role/ci-ecr
- uses: aws-actions/amazon-ecr-login@v2
- run: |
    docker build -t $ECR_REGISTRY/myapp:$GITHUB_SHA .
    docker push $ECR_REGISTRY/myapp:$GITHUB_SHA
```

---

## 12) Dépannage

* **`denied: requested access is denied`**

  * Non authentifié, token expiré, repo inexistant, **nom en majuscules** (interdit).
* **`manifest unknown`** : le tag n’existe pas dans le repo.
* **`x509: certificate signed by unknown authority`**

  * Certificat invalide côté client ; ajouter CA à la trust-store, ou corriger TLS.
* **`insecure-registries`** (déconseillé) : `/etc/docker/daemon.json`

  ```json
  { "insecure-registries": ["registry.local:5000"] }
  ```

  puis `systemctl restart docker`.
* **Rate limit Docker Hub** : login, abonnement, ou **pull-through cache** (Harbor/registry proxy).
* **Problèmes de GC** : arrêter le registry avant `garbage-collect`, activer `delete.enabled`.

---

## 13) TP (exercices)

**TP-1 — Pousser sur GHCR**

1. Créer un PAT (write:packages).
2. `docker login ghcr.io`
3. `docker build -t ghcr.io/<org>/<repo>/app:1.0 .`
4. `docker push ghcr.io/<org>/<repo>/app:1.0`
5. `docker pull` depuis une autre machine.

**TP-2 — Registry privé avec NGINX + Basic Auth + TLS**

1. Générer `htpasswd`, certificats (Let’s Encrypt).
2. Lancer `registry:2` + `nginx` (compose ci-dessus).
3. Pousser une image signée cosign.
4. Vérifier accès, logs NGINX et `docker pull`.

**TP-3 — Buildx multi-arch + SBOM/Provenance + Cosign**

1. Créer builder.
2. Build & push `--platform amd64,arm64 --provenance --sbom`.
3. `cosign verify` sur le tag publié.

**TP-4 — Proxy cache Docker Hub**

1. Configurer `proxy.remoteurl` dans `registry:2`.
2. Configurer le démon Docker client pour utiliser ce miroir.
3. Tirer `alpine:3.20` plusieurs fois (observer les temps et logs).

---

## 14) Cheatsheet

```bash
# Login / Logout
docker login [REGISTRY] ; docker logout [REGISTRY]

# Tag / Push / Pull
docker tag src:1.0 ghcr.io/org/repo/app:1.0
docker push ghcr.io/org/repo/app:1.0
docker pull ghcr.io/org/repo/app@sha256:...

# Buildx multi-arch + SBOM + provenance
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.example.com/app:1.0 --provenance=mode=max --sbom=spdx --push .

# Cosign
cosign sign --key cosign.key registry.example.com/app:1.0
cosign verify --key cosign.pub registry.example.com/app:1.0

# Registry:2 rapide (labo)
docker run -d -p 5000:5000 --name registry registry:2
docker tag alpine:3.20 localhost:5000/alpine:3.20
docker push localhost:5000/alpine:3.20

# GC (registry:2)
registry garbage-collect /etc/docker/registry/config.yml
```


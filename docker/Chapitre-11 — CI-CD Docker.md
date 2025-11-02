# Chapitre-11 — CI/CD Docker

*(builds reproductibles, caches distants, promotion par digest, SBOM & signatures)*

## Objectifs d’apprentissage

* Concevoir des **pipelines Docker** fiables : lint/test → build → scan → signer/attester → push → **promote by digest**.
* Produire des images **reproductibles** (tags/digests pinés, BuildKit, multi-arch) avec **caches distants**.
* Générer et publier **SBOM** & **provenance** ; appliquer des **scans** (CVE/licences) bloquants.
* Intégrer la **signature** (cosign) et des **politiques d’admission** (deny si non signé/non scanné).
* Mettre en place une **stratégie de tags** (SemVer/canaux) et des **workflows de promotion** sans rebuild.

## Pré-requis

* Chap. 01–10 maîtrisés (Images, BuildKit, Registry, Sécurité, Perf).
* Accès à un registry (GHCR/Docker Hub/Harbor/ECR/GAR/ACR).

---

## 1) Principes d’architecture CI/CD Docker

**Étapes standardisées**

1. **Verify**: lint (Dockerfile/compose), SAST, tests unitaires.
2. **Build** (BuildKit/buildx) → **multi-arch**, **cache to/from**.
3. **Scan** (CVE/licences) → **fail-the-build** si seuil dépassé.
4. **Attest**: **SBOM + provenance**.
5. **Sign**: cosign (clé/“keyless” OIDC).
6. **Push**: tags SemVer + tag canal + **digest** (artefact).
7. **Promote**: copier **le digest validé** vers `staging`/`prod` (pas de rebuild).
8. **Policy**: admission par **digest signé** + SBOM présent.

**Règles d’or**

* **Jamais** de secrets dans l’image ou le repo.
* Déployer **par digest** (immutabilité).
* Tous les artefacts → **registry** (image, SBOM, attestation, signature).

---

## 2) Stratégie de versionning & tags

* **SemVer**: `1.4.2` (+ raccourcis `1.4`, `1`) **en build**, puis “gel” en prod (pas d’écrasement).
* **Canaux**: `-rc`, `-beta`, `-dev` pour branches de release.
* **Metadata**: tag **commit** (`sha-7`) + **branch** (ex. `main`) pour traçabilité.
* **Déploiement**: utiliser **`@sha256:<digest>`** côté infra (Compose/K8s).

---

## 3) Pipeline GitHub Actions — modèle complet

### 3.1 Variables utiles (exemple)

* `REGISTRY=ghcr.io`
* `IMAGE=ghcr.io/acme/web`
* `SEMVER` dérivé des tags Git (fallback `0.0.0`).

### 3.2 Workflow (multi-arch, scan, sbom, sign, push)

```yaml
name: ci-docker
on:
  push:
    branches: [ main ]
    tags:     [ 'v*.*.*' ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write          # push GHCR
      id-token: write          # OIDC (cosign keyless)
    env:
      REGISTRY: ghcr.io
      IMAGE: ghcr.io/acme/web
    steps:
      - uses: actions/checkout@v4

      # Métadonnées d'image (tags/labels OCI)
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=semver,pattern={{version}},prefix=v
            type=semver,pattern={{major}}.{{minor}},prefix=v
            type=ref,event=branch
            type=sha
          labels: |
            org.opencontainers.image.source=${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}

      # Buildx + QEMU multi-arch
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      # Login au registry (GHCR utilise GITHUB_TOKEN)
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Build & push multi-arch + caches + SBOM+provenance
      - uses: docker/build-push-action@v6
        id: build
        with:
          context: .
          file: ./Dockerfile
          target: runtime
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          provenance: true
          sbom: true
          cache-from: type=registry,ref=${{ env.IMAGE }}:buildcache
          cache-to:   type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max

      # Digest (manifest list) en sortie
      - name: Export digest
        run: echo "DIGEST=${{ steps.build.outputs.digest }}" >> $GITHUB_ENV

      # Scan (ex: Trivy). Échoue sur HIGH/CRITICAL
      - uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.IMAGE }}@${{ env.DIGEST }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'HIGH,CRITICAL'

      # Signature cosign (keyless via OIDC)
      - uses: sigstore/cosign-installer@v3.5.0
      - name: Cosign sign (keyless)
        env:
          COSIGN_EXPERIMENTAL: "true"
        run: |
          cosign sign --yes $IMAGE@${DIGEST}

      # Upload digest pour la promotion
      - name: Save digest artifact
        uses: actions/upload-artifact@v4
        with:
          name: image-digest
          path: digest.txt
        env:
          DIGEST_FILE: digest.txt
        shell: bash
        run: echo "${{ env.DIGEST }}" > digest.txt
```

### 3.3 Job de **promotion** (copie du digest)

```yaml
name: promote
on:
  workflow_dispatch:
    inputs:
      from:
        description: 'repo source (ex: acme/web)'
        required: true
        default: 'acme/web'
      to:
        description: 'repo cible (ex: acme/web-prod)'
        required: true
        default: 'acme/web-prod'
      tag:
        description: 'tag cible (ex: v1.4.2)'
        required: true

jobs:
  copy:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: imjasonh/setup-crane@v0.3
      - name: Download digest
        uses: actions/download-artifact@v4
        with: { name: image-digest, path: . }
      - name: Copy by digest (no rebuild)
        run: |
          SRC=ghcr.io/${{ github.event.inputs.from }}@$(cat digest.txt)
          DST=ghcr.io/${{ github.event.inputs.to }}:${{ github.event.inputs.tag }}
          crane copy "$SRC" "$DST"
```

**Points clés**

* **`build-push-action`** publie manifest list (amd64/arm64).
* **Caches** persos vers **tag `:buildcache`** dans le registry.
* **Trivy** échoue en cas de vulnérabilités sévères.
* **Cosign keyless** (OIDC) → signatures stockées comme artefacts OCI.
* **Promotion** = `crane copy` par **digest** (pas de rebuild/push binaire).

---

## 4) Pipeline GitLab CI — deux variantes

### 4.1 Docker-in-Docker (simple)

`.gitlab-ci.yml`

```yaml
stages: [ test, build, scan, sign, release ]

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""     # dind non-TLS (réseau privé runner)
  IMAGE: $CI_REGISTRY_IMAGE

services:
  - name: docker:dind

.test:
  stage: test
  image: docker:24-git
  script:
    - docker version
    - docker build --target=test -t test .
    - docker run --rm test ./run-tests.sh

build:
  stage: build
  image: docker:24-git
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker buildx create --use
    - docker buildx build \
        --platform linux/amd64,linux/arm64 \
        --provenance=true --sbom=true \
        --cache-to=type=registry,ref=$IMAGE:buildcache,mode=max \
        --cache-from=type=registry,ref=$IMAGE:buildcache \
        -t $IMAGE:$CI_COMMIT_SHORT_SHA \
        -t $IMAGE:${CI_COMMIT_TAG:-dev} \
        --push .

scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE:$CI_COMMIT_SHORT_SHA

sign:
  stage: sign
  image: ghcr.io/sigstore/cosign/cosign:v2.4.0
  script:
    - cosign login $CI_REGISTRY -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD"
    - cosign sign --yes $IMAGE:$CI_COMMIT_SHORT_SHA

release:
  stage: release
  image: gcr.io/go-containerregistry/crane:debug
  script:
    - DIGEST=$(crane digest $IMAGE:$CI_COMMIT_SHORT_SHA)
    - crane copy $IMAGE@$DIGEST $IMAGE:${CI_COMMIT_TAG:-rc}
```

### 4.2 Kaniko (sans daemon) + cosign

* Remplace `docker buildx build` par **`gcr.io/kaniko-project/executor`** (build sans démon).
* Signature cosign possible dans un job suivant avec `crane digest`.

---

## 5) Jenkins Pipeline (extrait)

```groovy
pipeline {
  agent { label 'docker' }
  environment {
    IMAGE = "registry.example.com/team/app"
  }
  stages {
    stage('Checkout'){ steps { checkout scm } }

    stage('Buildx'){
      steps {
        sh '''
          docker login registry.example.com -u $REG_USER -p $REG_PASS
          docker buildx create --use
          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            --provenance --sbom \
            --cache-from=type=registry,ref=$IMAGE:buildcache \
            --cache-to=type=registry,ref=$IMAGE:buildcache,mode=max \
            -t $IMAGE:${GIT_COMMIT:0:7} \
            --push .
        '''
      }
    }

    stage('Scan'){
      steps {
        sh 'trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE:${GIT_COMMIT:0:7}'
      }
    }

    stage('Sign'){
      steps { sh 'cosign sign --yes $IMAGE:${GIT_COMMIT:0:7}' }
    }

    stage('Promote'){
      steps {
        sh '''
          DIGEST=$(crane digest $IMAGE:${GIT_COMMIT:0:7})
          crane copy $IMAGE@$DIGEST $IMAGE:stable
        '''
      }
    }
  }
}
```

---

## 6) Scans, SBOM, signatures & politiques

* **Scans**: Trivy/Docker Scout en **pipeline** (bloquer **HIGH/CRITICAL**), rapport attaché.
* **SBOM**: `buildx --sbom` (SPDX/CycloneDX) → stocké comme artefact OCI au registry.
* **Provenance**: `buildx --provenance` → attestation SLSA-like.
* **Signatures**: cosign (clé locale ou **keyless OIDC**).
* **Policies** (pré-déploiement): OPA/Conftest sur Dockerfile/Compose/K8s Manifests pour refuser :

  * images non signées / sans SBOM,
  * tags `latest`,
  * conteneurs `--privileged`, sans `USER`, sans healthcheck, ports wildcard.

---

## 7) Caches distants & vitesse en CI

* **Cache registry** (`cache-to/from=type=registry,ref=:buildcache`).
* **Proxy cache** Docker Hub côté infra (voir Chap. 07).
* `.dockerignore` strict ; steps `deps`/`build` séparés (Node/Go/Maven/Python) pour maximiser le cache.

---

## 8) Gestion des secrets en pipeline

* **OIDC → Registry/Cloud** quand possible (pas de mots de passe).
* Secrets dans **store CI** (Actions/GitLab/Jenkins Credentials) ; **jamais** dans le repo.
* BuildKit `RUN --mount=type=secret` pour **ne pas** figer les secrets dans les couches.

---

## 9) Promotion entre environnements (sans rebuild)

**Principe**: promouvoir **le digest validé** (scan+sign) vers `staging`/`prod`.

Outils : `crane copy`, `skopeo copy`, ou API registry.

Exemple CLI:

```bash
DIGEST=$(crane digest ghcr.io/acme/web:v1.4.2)
crane copy ghcr.io/acme/web@${DIGEST} ghcr.io/acme/web-prod:v1.4.2
```

---

## 10) Déploiement par **digest** (rappel)

Compose/K8s doivent référencer l’image **immutably**:

```yaml
image: ghcr.io/acme/web@sha256:abcd...     # pas juste :v1.4.2
```

---

## 11) Aide-mémoire (commandes clés)

```bash
# Build multi-arch + push + sbom + provenance
docker buildx build --platform linux/amd64,linux/arm64 \
  --provenance --sbom \
  --cache-from=type=registry,ref=REG/IMG:buildcache \
  --cache-to=type=registry,ref=REG/IMG:buildcache,mode=max \
  -t REG/IMG:1.4.2 --push .

# Digests & copies
crane digest REG/IMG:1.4.2
crane copy REG/IMG@sha256:... REG/IMG-PROD:1.4.2

# Scan blocant
trivy image --severity HIGH,CRITICAL --exit-code 1 REG/IMG@sha256:...

# Signature & vérification
cosign sign --yes REG/IMG@sha256:...
cosign verify REG/IMG@sha256:...
```

---

## 12) Checklist de clôture (pipeline “prêt-prod”)

* **Build**

  * BuildKit/buildx activés ; **multi-arch** si requis.
  * `.dockerignore` strict ; **base pinée** (tag + digest recommandé).
  * **Caches distants** `cache-to/from` configurés.

* **Sécurité & conformité**

  * **Scan** bloquant (CVE HIGH/CRITICAL, licences).
  * **SBOM + provenance** générés et publiés au registry.
  * **Signature cosign** (clé ou keyless OIDC) vérifiée côté admission.

* **Registry & promotion**

  * **Push** tags SemVer/canal + **digest** collecté en artefact.
  * **Promotion par digest** (crane/skopeo), pas de rebuild.
  * Politiques d’**immutabilité** des tags prod (ou contrôles équivalents).

* **Déploiement**

  * Infra consomme **`@sha256`** (immutabilité).
  * Gates d’admission (OPA/Conftest) activés.

* **Opérations**

  * Logs/artefacts CI conservés (digest, rapports scans, attestations).
  * Runners avec ressources suffisantes ; proxy cache Docker Hub.
  * Secrets via OIDC/CI-secrets, **jamais** en clair.


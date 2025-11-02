# Chapitre-07 — Registry & Distribution

## Objectifs d’apprentissage

* Maîtriser l’**écosystème OCI registry** (pousser/tirer des images, authentification, namespaces, tags, digests).
* Déployer un **registry privé sécurisé** (TLS, authentification), configurer **proxy cache / mirroring**, et gérer **GC (garbage collect)**.
* Mettre en place des **politiques de gouvernance** (immutabilité, rétention, conventions de nommage), **scans**, **signatures** (cosign) et **SBOM/provenance**.
* Intégrer le registry à vos **pipelines CI/CD** (tokens, permissions minimales, promotion par digest).

## Pré-requis

* Compréhension des images (Ch.-01), du build (Ch.-05) et de Compose (Ch.-06).
* Nocions TLS/PKI et reverse-proxy (Nginx/Traefik/Caddy).

---

## 1) Rappels : identité d’une image & registres

* **Nom complet** : `[registry]/[namespace]/repo[:tag]` ou `@[digest]`
  Ex. `ghcr.io/acme/api:1.4.2`, `registry.example.com/prod/web@sha256:…`
* **Tag** = alias **mutable** ; **digest** = identifiant **immuable**.
* **Bonnes pratiques** : publier des **tags versionnés** (SemVer + canaux) et **déployer par digest** en prod.

---

## 2) Authentification côté client

```bash
# Login avec mot de passe ou token (recommandé)
docker login ghcr.io
docker login registry.example.com

# Pousser / tirer
docker push registry.example.com/team/app:1.0
docker pull registry.example.com/team/app@sha256:...
```

* Utilisez des **PAT / tokens CI** à portée **minimale** (scopes lecture/écriture par repo).
* Évitez de stocker des identifiants en clair ; préférez **OIDC** quand supporté par le CI.

---

## 3) Quick start : registry privé basique (filesystem)

### 3.1 En Compose (dev/lab)

```yaml
services:
  registry:
    image: registry:2
    container_name: registry
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry
      REGISTRY_HTTP_ADDR: :5000
      REGISTRY_STORAGE_DELETE_ENABLED: "true"   # nécessaire pour GC
    volumes:
      - registry-data:/var/lib/registry
    ports:
      - "5000:5000"
volumes:
  registry-data: {}
```

* En l’état : **HTTP** sans auth (lab uniquement).
* En production : **placer derrière un reverse-proxy TLS** + **authentification**.

---

## 4) Production : TLS + authentification

### 4.1 Registry config.yml (auth `htpasswd`, headers, proxy)

`config.yml`

```yaml
version: 0.1
log:
  fields: { service: registry }
storage:
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
  # Si derrière un reverse-proxy :
  relativeurls: true
  host: https://registry.example.com
auth:
  htpasswd:
    realm: basic-realm
    path: /auth/htpasswd
delete:
  enabled: true
```

### 4.2 Compose avec Nginx TLS (exemple)

```yaml
services:
  registry:
    image: registry:2
    environment:
      REGISTRY_CONFIGURATION_PATH: /etc/docker/registry/config.yml
    volumes:
      - ./config.yml:/etc/docker/registry/config.yml:ro
      - ./auth:/auth:ro
      - registry-data:/var/lib/registry
    networks: [ net ]
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro   # fullchain.pem / privkey.pem
    ports:
      - "443:443"
    depends_on: [ registry ]
    networks: [ net ]
networks: { net: {} }
volumes:   { registry-data: {} }
```

`nginx.conf` (extrait minimal TLS + proxy)

```nginx
events {}
http {
  server {
    listen 443 ssl;
    server_name registry.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    client_max_body_size 0;               # gros blobs
    chunked_transfer_encoding on;

    location /v2/ {
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-Proto https;
      proxy_pass http://registry:5000;
      add_header Docker-Distribution-Api-Version registry/2.0 always;
    }
  }
}
```

Créer l’**auth** :

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd httpd:2.4-alpine -Bbn user pass > auth/htpasswd
```

> Alternative facile : **Traefik/Caddy** avec ACME (Let’s Encrypt) automatique.

---

## 5) Proxy cache (pull-through cache) & mirrors

### 5.1 Registry comme proxy cache de Docker Hub

`config.yml` (extrait)

```yaml
proxy:
  remoteurl: https://registry-1.docker.io
```

* Le registry **cache** les blobs tirés.
* Configurez les clients Docker pour utiliser ce **miroir**.

### 5.2 Côté clients (`/etc/docker/daemon.json`)

```json
{
  "registry-mirrors": ["https://registry.example.com"],
  "insecure-registries": []
}
```

> Ne mettez **insecure-registries** que pour des lab **sans TLS** (à proscrire en prod).

---

## 6) Stockage : filesystem vs S3/objets

### 6.1 S3 backend (exemple)

`config.yml` (extrait)

```yaml
storage:
  s3:
    region: eu-west-3
    bucket: my-registry-bucket
    accesskey: ${S3_ACCESS_KEY}
    secretkey: ${S3_SECRET_KEY}
    encrypt: true
    secure: true
    rootdirectory: /registry
```

* Pour **HA/scalabilité** : répliquer plusieurs instances derrière un **LB**, stockage partagé (S3/Swift/GCS…), optionnellement un cache **Redis**.
* Pensez **versioning** côté bucket (rétention), chiffrement et **lifecycle**.

---

## 7) Suppression & Garbage Collect (GC)

* Activer `delete.enabled: true` (vu plus haut).
* Supprimer un **manifeste** (par **digest**, pas par tag) puis **GC**.

Avec **crane** :

```bash
# Lister digest
crane digest registry.example.com/team/app:1.0
# Supprimer par digest
crane delete registry.example.com/team/app@sha256:...
```

Lancer le **GC** (arrêt d’écriture recommandé) :

```bash
docker exec -it registry registry garbage-collect --delete-untagged=true /etc/docker/registry/config.yml
```

> Le GC libère le disque des blobs **orphelins**. Enregistrez une fenêtre de maintenance.

---

## 8) Gouvernance : nommage, immutabilité, rétention

* **Nommage** : `org/projet/service` ; préfixes d’environnements (`dev/`, `prod/`) au besoin.
* **Tags** : SemVer (`1.4.2`), canaux (`rc`, `stable`), politique **anti-écrasement** des tags prod.
* **Immutabilité** : **déployer par digest** ; activer immutabilité côté registry (Harbor/Nexus) si possible.
* **Rétention** : règles d’expiration (Harbor/Nexus) ou procédures de **GC** + nettoyage des tags.
* **Licences** : scans en CI, blocage si licence interdite.
* Documenter un **annuaire des images** (qui produit ? cadence ? PO ?).

*(Voir aussi **Annexe C** du syllabus pour une politique prête à l’emploi.)*

---

## 9) Sécurité supply-chain : scans, signatures, SBOM/provenance

### 9.1 Scans (CVE & licences)

* **Trivy** / **Docker Scout** en CI : échouer au-delà d’un **seuil** (ex. CVE HIGH).
* Harbor/Artifactory/Nexus : scans intégrés + **politiques** d’admission.

### 9.2 Signatures (cosign / Sigstore)

```bash
# Signer (clé locale)
cosign sign registry.example.com/team/app:1.4.2

# Vérifier (politique pull/CI)
cosign verify registry.example.com/team/app:1.4.2
```

* **Keyless** (OIDC) possible ; stocke signatures comme **artefacts OCI** reliés.

### 9.3 SBOM & provenance (buildx)

```bash
docker buildx build \
  --provenance=true --sbom=true \
  -t registry.example.com/team/app:1.4.2 --push .
```

* Les artefacts (SPDX/CycloneDX, provenance) sont **associés** à l’image dans le registry.
* Publiez et **conservez** ces preuves (audit/compliance).

---

## 10) Intégration CI/CD (promotion par digest)

* **Build & push** vers un registry **interne** (dev) ; scanner, signer, attester.
* **Promouvoir** l’**exact digest** validé vers staging/prod (retag/push contrôlé) :

```bash
# Récupérer le digest promu
DIGEST=$(crane digest registry.example.com/dev/app:1.4.2)
# Re-tagger (sans re-upload de blobs) vers prod
crane copy registry.example.com/dev/app@${DIGEST} registry.example.com/prod/app:1.4.2
```

* Déploiements **par digest** ; **deny** si non signé / non scanné / SBOM manquant.

---

## 11) Exemple complet : Registry privé + Traefik (ACME auto) + Auth

`compose.yaml`

```yaml
services:
  traefik:
    image: traefik:v3.1
    command:
      - "--providers.docker=true"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.email=admin@example.com"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    restart: unless-stopped

  registry:
    image: registry:2
    environment:
      REGISTRY_HTTP_ADDR: :5000
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: '[*]'
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_AUTH_HTPASSWD_REALM: "basic-realm"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.registry.rule=Host(`registry.example.com`)"
      - "traefik.http.routers.registry.entrypoints=websecure"
      - "traefik.http.routers.registry.tls.certresolver=le"
      - "traefik.http.services.registry.loadbalancer.server.scheme=http"
      - "traefik.http.services.registry.loadbalancer.server.port=5000"
    volumes:
      - registry-data:/var/lib/registry
      - ./auth:/auth:ro
    restart: unless-stopped

volumes:
  registry-data: {}
```

* Avantages : TLS **automatique**, config simple, **auth htpasswd** native du registry.

---

## 12) Outils pratiques autour des registries

* **crane / gcrane** (google/go-containerregistry) : `crane ls`, `crane digest`, `crane copy`, `crane delete`.
* **skopeo** : inspect/copy/sign (multi-backends).
* **oras** : gérer **artefacts OCI** (push/pull SBOM, attestions, charts, etc.).
* **regctl** : client Docker Distribution (delete/GC friendly).
* **Harbor/Nexus/Artifactory** : GUIs, RBAC, scans, immutabilité, réplication, rétention.

---

## 13) Dépannage courant

* **401 Unauthorized** : vérifier `docker login` (bon host), horloge (TLS/ACME), header `Host` côté proxy.
* **413 Request Entity Too Large** : augmenter `client_max_body_size` (Nginx) / `traefik.http.middlewares.compress` côté Traefik.
* **Manifest unknown** : tag absent, mauvais repo, ou **cache** pas encore rempli.
* **SSL / certificats** : chaîne incomplète → fournir **fullchain** ; horloge/NTP.
* **GC inefficace** : digest non supprimé, `delete.enabled=false`, ou writes toujours en cours.
* **Rate-limit Docker Hub** : configurer un **mirror/proxy cache** côté clients.

---

## 14) Do & Don’t

**Do**

* Toujours **TLS** + **auth** (même interne).
* **Déployer par digest** ; **signer** (cosign) ; **SBOM & provenance** activés.
* **Proxy cache** pour Docker Hub ; **mirrors** côté clients.
* **Politiques** : immutabilité des tags prod, rétention, nettoyage + **GC**.
* CI/CD : tokens **scopés**, **promotion par digest**, scans bloquants.

**Don’t**

* Pas d’**insecure registry** en prod.
* Ne pas **écraser** des tags stables publiés sans politique claire.
* Éviter d’exposer directement le registry sur Internet **sans** reverse-proxy durci.
* Ne stockez pas de **secrets** dans les images / tags publics.

---

## 15) Aide-mémoire (commandes clés)

```bash
# Auth & push
docker login registry.example.com
docker tag app:1.0 registry.example.com/team/app:1.0
docker push registry.example.com/team/app:1.0

# Digests & copies
crane digest registry.example.com/team/app:1.0
crane copy registry.example.com/team/app@sha256:... registry.example.com/prod/app:1.0

# Delete & GC
crane delete registry.example.com/team/app@sha256:...
docker exec -it registry registry garbage-collect --delete-untagged=true /etc/docker/registry/config.yml

# Signatures & vérif
cosign sign registry.example.com/team/app:1.0
cosign verify registry.example.com/team/app:1.0

# SBOM/provenance lors du build
docker buildx build --sbom --provenance -t registry.example.com/team/app:1.0 --push .
```

---

## 16) Checklist de clôture (qualité d’un registry)

* **TLS** valide (ACME/LE ou PKI interne), **auth** activée, en-têtes sûrs.
* **Delete.enabled** actif ; procédure **GC** documentée + fenêtre de maintenance.
* **Proxy cache** / **mirrors** configurés pour les clients ; **rate-limit** éliminé.
* **Politiques** : nommage, immutabilité des tags prod, **rétention** (Harbor/Nexus) ou GC programmé.
* **Sécurité** : scans CVE/licences en CI, **cosign** (sign & verify), **SBOM/provenance** publiés.
* **CI/CD** : tokens à **moindre privilège**, promotion **par digest**, traçabilité complète.
* **Supervision** : logs, capacité stockage, latence pushes/pulls, alertes expiration de certificats.


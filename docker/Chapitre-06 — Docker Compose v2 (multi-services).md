# Chapitre-06 — Docker Compose v2 (multi-services)

## Objectifs d’apprentissage

* Décrire une application **multi-services** avec `docker compose` : services, réseaux, volumes, secrets, configs.
* Piloter les **environnements** (profils, overrides, variables) et la **chaîne build→run** (buildx, cache, args, target).
* Exploiter les commandes **opérationnelles** : `up/down/ps/logs/exec/run/restart/pull/push/build/config/watch`.
* Mettre en œuvre des **dépendances fiables** (healthchecks + `depends_on`), la **séparation réseau**, et une **structure maintenable**.

## Pré-requis

* Docker Engine + CLI Compose v2 (`docker compose version`).
* Aisance avec YAML, Dockerfiles (chapitre 05) et réseau/stockage (ch. 03–04).

---

## 1) Fichiers Compose : repères essentiels

* **Nom par défaut** : `compose.yaml` (ou `docker-compose.yml` maintenu).
* **Plusieurs fichiers** possibles : `-f compose.yaml -f compose.prod.yaml`.
* **`.env`** au même niveau que le compose pour l’**interpolation** (`${VAR:-defaut}`).
* Le champ `version:` est **optionnel** (Compose v2 lit la spec sans).

**Exemple minimal**

```yaml
services:
  web:
    image: nginx:1.27
    ports: ["8080:80"]
```

---

## 2) Services : image, build, command, environment…

### 2.1 Image & build

```yaml
services:
  api:
    image: ghcr.io/acme/api:1.4.2   # ou :
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
      args:
        APP_VER: "1.4.2"
      ssh:        # BuildKit
        - default
      secrets:    # BuildKit (pas dans l'image)
        - npm_token
```

### 2.2 Commande, entrypoint, workdir, user

```yaml
  api:
    command: ["./server","--port","8080"]
    entrypoint: ["/usr/local/bin/wrapper"]
    working_dir: /app
    user: "10001:10001"
```

### 2.3 Variables d’environnement

```yaml
  api:
    environment:
      - TZ=Europe/Paris
      - LOG_LEVEL=info
    env_file:
      - ./.env             # valeurs par défaut
```

**Priorité** (du plus fort au plus faible) : variables shell > `environment` > `.env` > `env_file`.

---

## 3) Réseaux, ports, alias, IPAM

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "80:80"                   # publie sur l’hôte
    networks:
      - frontend
  api:
    image: ghcr.io/acme/api:1.4.2
    networks:
      frontend:
      backend:
        aliases: [ api-svc ]      # alias DNS sur ce réseau
  db:
    image: postgres:16
    networks:
      backend:
        ipv4_address: 172.31.0.10 # IP statique (si subnet défini)

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.31.0.0/24
          gateway: 172.31.0.1
```

* `internal: true` coupe l’**egress** (pas d’Internet) : idéal pour DB/queues.
* Attachez chaque service **au minimum de réseaux** nécessaires.

---

## 4) Volumes, binds et tmpfs

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - data_pg:/var/lib/postgresql/data        # volume nommé
  web:
    image: nginx:1.27
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro   # bind mount (RO)
  app:
    image: ghcr.io/acme/app:1.0
    read_only: true
    tmpfs:
      - /run
      - /tmp:size=64m,mode=1777

volumes:
  data_pg: {}
```

> Sous Windows, préférez des **chemins relatifs** et, si possible, WSL2 pour éviter les latences de montages sur `\\wsl$`.

---

## 5) Healthcheck, restart, ressources

```yaml
services:
  api:
    image: ghcr.io/acme/api:1.4.2
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
    restart: unless-stopped
    deploy:        # ⚠️ Swarm only (ignoré par docker compose)
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

* `deploy.*` est **ignoré** par `docker compose` (utile en Swarm).
* En **non-Swarm**, utilisez plutôt `cpus`, `mem_limit`, etc. via `docker run` équivalents… ou passez par `docker update` après coup.

---

## 6) `depends_on` et ordre de démarrage

```yaml
services:
  db:
    image: postgres:16
    healthcheck: { test: ["CMD","pg_isready","-U","postgres"] }
  api:
    image: ghcr.io/acme/api:1.4.2
    depends_on:
      db:
        condition: service_healthy   # attend la santé OK
```

* Les **dépendances fiables** exigent des **healthchecks** côté dépendances.
* N’utilisez pas `sleep` dans des entrypoints : préférez `service_healthy`.

---

## 7) Secrets & configs (fichiers montés)

### 7.1 Secrets (fichiers en lecture seule)

```yaml
services:
  api:
    image: ghcr.io/acme/api:1.4.2
    secrets:
      - jwt_secret
secrets:
  jwt_secret:
    file: ./secrets/jwt.key
```

### 7.2 Configs (fichiers non sensibles)

```yaml
services:
  web:
    image: nginx:1.27
    configs:
      - source: web_conf
        target: /etc/nginx/nginx.conf
configs:
  web_conf:
    file: ./nginx.conf
```

> Les **secrets** et **configs** de Compose sont montés en **fichiers** (pas en variables). Évitez de stocker des secrets en clair dans le dépôt.

---

## 8) Profils (activer/masquer des services)

```yaml
services:
  grafana:
    image: grafana/grafana:11
    profiles: ["observability"]

  loki:
    image: grafana/loki:2.9
    profiles: ["observability"]
```

* Lancer **avec profil** : `docker compose --profile observability up -d`.
* Sans ce profil, ces services sont **ignorés**.

---

## 9) Overrides & environnements

### 9.1 Fichiers multiples

```
compose.yaml             # base commune
compose.dev.yaml         # montages bind, hot-reload, logs verbeux
compose.prod.yaml        # images versionnées, read_only, ressources
```

Commande :

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

### 9.2 `docker compose config` (rendu final)

```bash
docker compose -f compose.yaml -f compose.prod.yaml config
# Affiche le YAML fusionné (utile pour valider)
```

---

## 10) Interpolation & .env

`.env`

```
APP_IMAGE=ghcr.io/acme/api
APP_TAG=1.4.2
```

`compose.yaml`

```yaml
services:
  api:
    image: "${APP_IMAGE}:${APP_TAG}"
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
```

* Valeur par défaut : `${VAR:-defaut}`.
* **Attention** : `.env` est **différent** de `env_file` (qui injecte des variables **dans le conteneur**).

---

## 11) Build Compose (buildx, caches)

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
      args: { BUILD_DATE: "${BUILD_DATE}" }
      cache_from:
        - type=registry,ref=ghcr.io/acme/cache:web
      cache_to:
        - type=registry,ref=ghcr.io/acme/cache:web,mode=max
```

Commandes typiques :

```bash
docker buildx create --name builder --use
docker buildx inspect --bootstrap

# Build & push via compose (si image et push configurés)
docker compose build
docker compose push
```

---

## 12) Cycle de vie : commandes opérationnelles

```bash
# Créer/lancer
docker compose up -d                # crée & démarre si absent
docker compose up --build -d        # reconstruit avant de lancer

# Arrêter/retirer
docker compose stop
docker compose down                 # + retire les réseaux
docker compose down --volumes       # + supprime volumes (prudent)
docker compose down --remove-orphans

# Inspection
docker compose ps
docker compose logs -f --tail=200
docker compose exec api sh          # shell dans "api"
docker compose run --rm job sh -c 'echo one-shot'

# Mises à jour
docker compose pull                 # tire les images taguées
docker compose restart api
docker compose up -d --no-deps api  # redémarrer un service sans toucher aux dépendances

# Échelle (non-Swarm)
docker compose up -d --scale web=3
```

---

## 13) Développement : `docker compose watch` (hot reload)

**Compose v2.22+**

```yaml
# compose.dev.yaml
services:
  api:
    build: { context: . }
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: package.json
```

Lancer :

```bash
docker compose -f compose.yaml -f compose.dev.yaml watch
```

* **sync** copie les fichiers en direct, **rebuild** relance un build sur changement.

---

## 14) Exemple “prod-like” complet (frontend/api/db + réseaux + health + secrets)

```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/pg_pwd
    secrets: [ pg_pwd ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 10
    networks:
      - backend
    volumes:
      - data_pg:/var/lib/postgresql/data
    restart: unless-stopped

  api:
    image: ghcr.io/acme/api:1.4.2@sha256:...   # déploiement par digest conseillé
    environment:
      - DB_HOST=db
      - DB_USER=postgres
      - DB_PASSWORD_FILE=/run/secrets/pg_pwd
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:8080/health || exit 1"]
      interval: 30s
    read_only: true
    tmpfs: [ /run, /tmp ]
    user: "10001:10001"
    networks:
      - frontend
      - backend
    restart: unless-stopped

  web:
    image: nginx:1.27
    ports: [ "80:80" ]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      api:
        condition: service_started
    networks:
      - frontend
    restart: unless-stopped

networks:
  frontend: { driver: bridge }
  backend:
    driver: bridge
    internal: true

volumes:
  data_pg: {}

secrets:
  pg_pwd:
    file: ./secrets/postgres.password
```

---

## 15) Réutilisation & factorisation (anchors, `x-`)

```yaml
x-health-fast: &health_fast
  interval: 10s
  timeout: 3s
  retries: 5

services:
  svc-a:
    image: ghcr.io/acme/a:1.0
    healthcheck:
      test: ["CMD","/health"]
      <<: *health_fast

  svc-b:
    image: ghcr.io/acme/b:1.0
    healthcheck:
      test: ["CMD","/ping"]
      <<: *health_fast
```

---

## 16) Dépannage & bonnes pratiques

**Dépannage**

* `docker compose config` : valider le YAML généré.
* `docker compose logs -f SERVICE` : voir les erreurs de démarrage.
* `depends_on` + **healthchecks** : diagnostiquer les enchaînements.
* `docker inspect` (conteneur) : confirmer réseaux/ports/volumes réellement montés.

**Do**

* Un **réseau interne** pour les backends, un **frontend** exposé.
* **Healthchecks** + `depends_on: condition: service_healthy`.
* **Secrets/configs** en fichiers (pas en variables) ; **read_only** + `tmpfs`.
* **Overrides** par environnement et **profils** pour options (observabilité, debug).
* Déployer par **digest** pour l’immutabilité.

**Don’t**

* Éviter `deploy.*` en pensant que Compose l’applique (c’est pour **Swarm**).
* Éviter `-P` (publication automatique) en prod ; mappez les ports **explicitement**.
* Éviter de mettre tous les services sur le même réseau par facilité.
* Ne jamais commit de **secrets** dans le dépôt.

---

## 17) Aide-mémoire (commandes clés)

```bash
# Lancer / arrêter
docker compose up -d
docker compose down --remove-orphans

# Inspection
docker compose ps
docker compose logs -f --tail=200 api
docker compose config

# Déboguer un service
docker compose exec api sh
docker compose run --rm api sh -lc 'curl -v http://localhost:8080/health'

# Mises à jour
docker compose pull
docker compose up -d --no-deps api
docker compose restart web

# Échelle
docker compose up -d --scale web=3

# Dev (watch)
docker compose -f compose.yaml -f compose.dev.yaml watch
```

---

## 18) Checklist de clôture (qualité d’une stack Compose)

* Réseaux **séparés** (frontend exposé, backend `internal` si possible).
* **Healthchecks** pertinents ; `depends_on` avec `service_healthy` pour l’ordre.
* **Secrets/configs** gérés en fichiers ; **rootfs read_only** + `tmpfs` déclarés.
* Volumes **nommés** pour la persistance (DB, state) ; binds **RO** pour configs.
* Variables/interpolation `.env` maîtrisées ; **overrides** prod/dev documentés.
* Commandes d’exploitation **standardisées** (up/down/logs/exec/restart).
* Images **versionnées** et idéalement déployées par **digest** ; buildx/caches configurés.
* `docker compose config` **propre** ; pas d’options Swarm critiques supposées actives.



# Chapitre-12 — Déploiement & Releases

*(Compose “prod-like”, blue/green, canary, rollbacks, migrations de base de données, runbooks)*

## Objectifs d’apprentissage

* Structurer un **déploiement Docker/Compose** prêt pour la prod (réseaux, LB, santé, secrets, immutabilité par digest).
* Exécuter des **releases sans interruption** : **blue/green**, **canary** pondéré, **rollback** rapide.
* Mettre en œuvre des **migrations de schéma** sûres (**expand-migrate-contract**).
* Outiller le processus : **runbooks**, **listes de contrôle**, artefacts versionnés (digest, SBOM, signatures).

## Pré-requis

* Chap. 01–11 (images, réseau, storage, Dockerfile, Compose, registry, sécurité, observabilité, perf, CI/CD).
* Notions reverse-proxy (Nginx/HAProxy/Traefik) et SQL (indexes, verrous, transactions).

---

## 1) Stratégies de déploiement : panorama

| Stratégie                  | Interruption |     Complexité | Idéal quand                                  |
| -------------------------- | -----------: | -------------: | -------------------------------------------- |
| **Recreate** (stop→start)  |          Oui |         Faible | Petites apps, maintenance planifiée          |
| **Blue/Green**             |          Non |        Moyenne | Changement rapide et **rollback instantané** |
| **Canary** (pondéré)       |          Non | Moyenne/Élevée | Valider en prod sur **x %** d’utilisateurs   |
| **A/B** (features toggles) |          Non |         Élevée | Expérimentations longues, UX/produit         |

> Compose ne gère pas “rolling update” natif comme K8s, mais **blue/green** + **canary via LB** couvre 95% des besoins.

---

## 2) Architecture “prod-like” de base (Compose)

Principes :

* **LB** en frontal (Nginx/HAProxy/Traefik), **frontend** exposé, **backend** interne.
* Services **read_only** + **tmpfs**, **healthchecks**, **digests** (pas de `:latest`).
* **Secrets** montés en fichiers, **volumes nommés** pour la persistance, **réseaux séparés**.

Extrait minimal :

```yaml
# compose.base.yaml
services:
  web:
    image: nginx:1.27
    ports: ["80:80"]
    networks: [ frontend ]
    depends_on: { api: { condition: service_started } }

  api:
    image: ghcr.io/acme/api@sha256:<digest>  # immuable
    read_only: true
    tmpfs: [ /run, /tmp ]
    healthcheck:
      test: ["CMD-SHELL","curl -fsS http://localhost:8080/health || exit 1"]
      interval: 20s
      retries: 5
    networks: [ backend ]

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/pg_pwd
    secrets: [ pg_pwd ]
    volumes: [ data_pg:/var/lib/postgresql/data ]
    networks: [ backend ]
    healthcheck: { test: ["CMD","pg_isready","-U","postgres"] }

networks:
  frontend: { driver: bridge }
  backend:  { driver: bridge, internal: true }

volumes:
  data_pg: {}

secrets:
  pg_pwd: { file: ./secrets/postgres.password }
```

---

## 3) Blue/Green avec Compose (2 *projects* parallèles)

Idée : exécuter **deux stacks** séparées (blue/green) derrière **un LB unique** et **basculer**.

### 3.1 Deux stacks « blue » et « green »

```
compose.blue.yaml   # mêmes services mais tag/digest BLUE
compose.green.yaml  # mêmes services mais tag/digest GREEN
```

Lancement :

```bash
docker compose -p app-blue  -f compose.base.yaml -f compose.blue.yaml  up -d
docker compose -p app-green -f compose.base.yaml -f compose.green.yaml up -d
```

> L’option `-p` isole réseaux/ressources : `app-blue_backend`, `app-green_backend`, etc.

### 3.2 LB HAProxy (pondération & bascule)

`haproxy.cfg` (extrait) :

```
global
  log stdout format raw local0
defaults
  mode http
  timeout client  30s
  timeout server  30s
  timeout connect 5s

frontend fe_http
  bind *:80
  default_backend be_api

backend be_api
  option httpchk GET /health
  server api_blue  api_blue:8080  check weight 100
  server api_green api_green:8080 check weight 0
```

Compose du LB :

```yaml
# compose.lb.yaml
services:
  haproxy:
    image: haproxy:lts
    ports: ["80:80"]
    volumes: [ "./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro" ]
    networks: [ lb, blue_net, green_net ]
networks:
  lb: {}
  blue_net:  { external: true, name: app-blue_backend }
  green_net: { external: true, name: app-green_backend }
```

Dans chaque stack, donnez aux services des **aliases** que le LB résout :

```yaml
# dans compose.blue.yaml
services:
  api:
    networks:
      backend:
        aliases: [ api_blue ]
# dans compose.green.yaml
services:
  api:
    networks:
      backend:
        aliases: [ api_green ]
```

**Basculer** (canary/green) = changer **weights** (10/90 → 50/50 → 100/0) dans `haproxy.cfg` puis :

```bash
docker kill -s HUP app-lb-haproxy   # reload sans downtime
```

> Variante Nginx : `upstream api { server api_blue:8080 weight=100; server api_green:8080 weight=0; }`

---

## 4) Scénario **Canary** (pondéré)

1. **Déployer** green (weight=10) → 10% du trafic.
2. **Observer** (erreurs, latence P95, logs, métriques).
3. **Augmenter** progressif (25 → 50 → 100).
4. **Figer** (weight=100 green / 0 blue), conserver blue comme **filet de sécurité** 24–48h.
5. **Désactiver** blue une fois confirmé (ou conserver pour rollback ultra-rapide).

---

## 5) **Rollback** en une commande

* Avec HAProxy/Nginx : remettre **weight=100** sur **blue**, **0** sur **green**, `HUP` LB → retour quasi instantané.
* Côté Compose, ne **supprimez** pas la stack ancienne tant que la nouvelle n’est pas validée.

**Bonnes pratiques rollback** :

* **Déployer par digest** → vous savez **exactement** à quoi revenir.
* Conserver le **runbook** et les **artefacts** (digest, SBOM, signatures) de la version précédente.
* Un **script** “switch-back” prêt à lancer (voir §10).

---

## 6) Migrations de base de données (expand → migrate → contract)

But : **aucun downtime** et **compatibilité bilatérale** entre ancien et nouveau code.

### 6.1 Étapes

1. **Expand** (pré-release)

   * Ajouter colonnes/tables/index **sans casser** l’existant.
   * Par ex. Postgres : `CREATE INDEX CONCURRENTLY`, éviter **locks** prolongés.
   * Déployer **green** qui sait lire **ancien + nouveau** schéma.

2. **Migrate/Backfill**

   * Tâche de **rétro-remplissage** (idempotente) qui copie/convertit les données.
   * Exécuter dans un **job** Compose dédié, monitoré.

3. **Switch** (canary/blue-green)

   * Basculer le trafic vers la version green.

4. **Contract** (post-validation)

   * Supprimer les champs/chemins **obsolètes** une fois la nouvelle version confirmée.

### 6.2 Jobs Compose pour migrations

```yaml
services:
  migrate:
    image: ghcr.io/acme/api-migrations@sha256:<digest>
    command: ["./migrate","up","--safe"]
    networks: [ backend ]
    environment: [ DB_URL=${DB_URL} ]
    depends_on:
      db: { condition: service_healthy }

  backfill:
    image: ghcr.io/acme/api@sha256:<digest>
    command: ["./tasks","backfill-new-column"]
    networks: [ backend ]
    environment: [ DB_URL=${DB_URL} ]
```

Exécution :

```bash
docker compose -p app-green run --rm migrate
docker compose -p app-green run --rm backfill
```

**Rappels DB** :

* **Transactions** pour petits changements, **batchs** pour gros volumes.
* Index **concurrently** (PG), éviter `ALTER` bloquants en heures pleines.
* **Feature flags** côté app pour écrire **double** (old+new) durant la transition.

---

## 7) Gestion des **configs & secrets** par environnement

* **Fichiers Compose superposés** : `compose.base.yaml` + `compose.prod.yaml`.
* `.env` par environnement (attention à l’interpolation, cf. Chap. 06).
* **Secrets** via `secrets:` (montés en **fichiers**), pas en variables.
* **Immutabilité** : images référencées **par digest** (CI/CD produit le digest validé).

---

## 8) Observabilité de release

Avant d’augmenter la part de trafic :

* **Healthchecks** OK, **logs** sans erreurs anormales.
* **Métriques** : taux d’erreur, latence P95, CPU/MEM, restart count.
* **Sondes actives** (synthetics) pointées sur la **green**.
* **Dash release** : panneaux comparatifs **blue vs green** (errors, latency, throughput).

---

## 9) Runbook de release (exécutable)

1. **Pré-flight**

   * Vérifier **digest** signé, SBOM/scans OK (CI).
   * Capacité disque `/var/lib/docker`, santé DB, LB disponible, NTP OK.

2. **Déploiement green**

   * `docker compose -p app-green -f compose.base.yaml -f compose.green.yaml up -d`
   * `docker compose -p app-green ps`, `logs -f` (API, web).

3. **Migrations**

   * `docker compose -p app-green run --rm migrate`
   * `docker compose -p app-green run --rm backfill` (si nécessaire)

4. **Canary 10%**

   * Modifier `haproxy.cfg` (weight green=10, blue=90), `kill -s HUP haproxy`
   * Observer 15–30 min (ou x requêtes / y erreurs max)

5. **Ramp-up**

   * 50% → 100% si métriques OK, alertes silencieuses.

6. **Stabilisation**

   * Laisser blue en réserve 24–48h.

7. **Contract**

   * Exécuter migrations “contract”, retirer écritures “double”.

8. **Nettoyage**

   * `docker compose -p app-blue down` quand validé.
   * Archiver artefacts (digest, cfg LB, logs release).

**Rollback** (à tout moment) :

* Remettre `weight green=0 / blue=100` + `HUP`.
* Si besoin, `docker compose -p app-green down`.

---

## 10) Automatisation (scripts)

### 10.1 Switch de poids HAProxy (bash simple)

```bash
#!/usr/bin/env bash
set -euo pipefail
CFG=haproxy.cfg
BLUE=$1   # 0..100
GREEN=$2  # 0..100
sed -i -E "s/(server api_blue .* weight )([0-9]+)/\1${BLUE}/"   "$CFG"
sed -i -E "s/(server api_green .* weight )([0-9]+)/\1${GREEN}/" "$CFG"
docker kill -s HUP app-lb-haproxy
echo "Switched weights: blue=${BLUE}, green=${GREEN}"
```

### 10.2 Verrouillage de version (digest pinning) en Compose

```bash
yq -i '.services.api.image = "ghcr.io/acme/api@sha256:'"$DIGEST"'"' compose.green.yaml
```

---

## 11) Exemples “prod-like” (synthèse)

### 11.1 Blue/Green via deux projects & HAProxy

* Deux stacks `app-blue` / `app-green` (digests différents).
* LB unique connecté aux deux **backend networks**.
* **Poids** HAProxy pour canary/bascule.
* **Jobs** `migrate`/`backfill` dans la stack candidate.
* **Rollback** = poids → blue 100%.

### 11.2 Canary fin (routes par header)

* Ajouter une **route** “X-Canary: 1” → envoyer 100% de ce trafic vers **green** pour tests E2E internes sans impacter tout le monde (règles HAProxy/Nginx).

---

## 12) Aide-mémoire (commandes clés)

```bash
# Démarrer/mettre à jour une stack nommée
docker compose -p app-green -f compose.base.yaml -f compose.green.yaml up -d

# Santé & logs
docker compose -p app-green ps
docker compose -p app-green logs -f --tail=200 api

# Jobs migrations
docker compose -p app-green run --rm migrate
docker compose -p app-green run --rm backfill

# Reload LB après changement de poids
docker kill -s HUP app-lb-haproxy

# Rollback immédiat = poids blue=100 / green=0 + reload
```

---

## 13) Checklist de clôture (release “prête-prod”)

**Avant**

* Images **signées** + **SBOM/provenance** publiés ; **digest** consigné.
* **Migrations expand** prêtes, jobs testés sur environnement miroir.
* LB & santé : **healthchecks** définis, dashboard comparatif prêt.

**Pendant**

* Déploiement **green** isolé, **canary 10%**, observation métriques & logs.
* **Backfill** terminé et idempotent, erreurs < seuil, latence P95 OK.

**Bascule**

* Poids LB → **100% green**, **0% blue**, surveillance rapprochée.
* **Rollback** scripté et **répété** en test (RTO court).

**Après**

* **Contract** effectué (schéma nettoyé), feature flags retirés.
* Stack blue retirée quand validé, **artefacts archivés** (digest, configs).
* Post-mortem / compte-rendu de release avec mesures avant/après.


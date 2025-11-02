# Chapitre-10 — Performance & Optimisation

## Objectifs d’apprentissage

* Mesurer avant d’optimiser : identifier les **goulets d’étranglement** (CPU, mémoire, I/O, réseau, démarrage).
* Réduire **taille** et **temps de build/pull** des images (couches, BuildKit, caches, registry).
* Améliorer les **temps de démarrage** (cold/warm start) et le **runtime** (cgroups, CPU pinning, NUMA, logs).
* Optimiser le **stockage** (overlay2 vs volumes), le **réseau** (MTU, NAT), et les **bind mounts** (WSL2/Windows).
* Mettre en place une **méthodologie** reproductible (benchmarks, budgets de perf, checklists CI/CD).

## Pré-requis

* Chap. 01–09 maîtrisés (images, conteneurs, storage, réseau, build, compose, sécurité, observabilité).
* Linux/WSL2 conseillé pour les mesures fines.

---

## 1) Mesurer d’abord (profilage rapide et budgets)

### 1.1 Indicateurs de base (par conteneur)

```bash
docker stats --no-stream NAME          # CPU/MEM/NET/IO instantané
docker inspect -f '{{.State.StartedAt}}' NAME
docker events --since=1h | grep NAME   # redémarrages, OOM, health
docker logs -f --tail=200 NAME         # latences et erreurs applicatives
```

### 1.2 Benchmarks synthétiques

* **Démarrage** : mesurer T0→Tready (port ouvert + healthcheck **OK**).
* **Throughput** : `wrk`, `ab`, `hey` sur endpoint critique.
* **Réseau** : `iperf3`, `curl -w`, `dig +trace` pour DNS.
* **I/O** : `fio` (sur volume), `dd` (indicatif), `iostat/iotop` côté hôte.

### 1.3 Budgets & objectifs

* Taille d’image **max** (ex. ≤ 150 MB runtime).
* Cold start **Tready** (ex. ≤ 2 s web, ≤ 10 s JVM/CDS).
* P95 latence & erreurs < SLO (suivi Grafana).
* Pull time (Région/proxy cache) ≤ X s.

---

## 2) Optimiser la **taille** d’image (build-time)

### 2.1 Multi-stage obligatoire

* **Builder → Runtime** : ne copier que l’artefact final.
* Bases **minimales** : `alpine`, `debian-slim`, **distroless** pour bins statiques.

### 2.2 Ordre des couches & cache

* Copier d’abord fichiers **stables** (ex. `package*.json`, `go.mod`) pour maximiser le cache.
* Grouper les `RUN` et nettoyer **dans la même couche** :

```dockerfile
RUN apt-get update && apt-get install -y curl \
 && rm -rf /var/lib/apt/lists/*
```

### 2.3 Dépôts & dépendances

* **Pinning** versions ; `npm ci` / `pip install --no-cache-dir` / `mvn -B -DskipTests`.
* Pour Go : `-ldflags="-s -w"` + `strip` pour binaire compact.

### 2.4 Choix de base (compat vs taille)

* `alpine` (musl) : petite, parfois moins perf/compat que glibc.
* `debian:bookworm-slim` : plus lourde, meilleure compatibilité.
* **distroless** : surface minimale (pas de shell) + très rapide au démarrage pour bins statiques.

---

## 3) BuildKit & caches (vitesse de build)

### 3.1 Activer BuildKit

```bash
export DOCKER_BUILDKIT=1
# ou daemon.json: { "features": { "buildkit": true } }
```

### 3.2 Caches persistants

```dockerfile
RUN --mount=type=cache,target=/root/.npm npm ci
RUN --mount=type=cache,target=/var/cache/apt apt-get update && ...
```

### 3.3 Caches distants (CI/CD)

```bash
docker buildx build \
  --cache-to=type=registry,ref=ghcr.io/acme/cache:web,mode=max \
  --cache-from=type=registry,ref=ghcr.io/acme/cache:web \
  -t ghcr.io/acme/web:1.4.3 .
```

### 3.4 Contexte de build minimal

* `.dockerignore` strict (éviter `.git`, artefacts lourds, secrets).
* `build --pull` pour ne pas traîner une base obsolète (sécurité + perf de diff).

---

## 4) Registry & distribution (vitesse de pull/push)

* **Proxy cache** local (Chap. 07) pour Docker Hub → réduit **latence** et **rate-limit**.
* **CDN/registres proches** de l’hôte (région).
* **Manifest list** multi-arch = un seul nom, tirage **arch** adaptée (pas d’échec QEMU).
* Si supporté par votre registry : couches **OCI + zstd** (réduction taille/temps). *Vérifier compatibilité avant d’activer.*

---

## 5) Démarrage rapide (cold/warm start)

### 5.1 Général

* **Images légères**, moins de couches, **ENTRYPOINT exec** (pas de shell).
* Préparer **répertoires en écriture** (tmpfs/volumes) et activer **read_only** si possible pour accélérer checks.
* **Healthcheck** rapide (HTTP local, délai court).

### 5.2 Langage/Runtime (exemples)

* **Go** : binaire statique → démarrage ~instantané.
* **Node.js** : `npm ci --omit=dev`, éviter transpile à l’exécution, pré-build.
* **Python** : pré-compiler (`python -m compileall`) ; serveurs **uvicorn+uvloop** ; `--workers` calibrés.
* **Java** : réduire classpath ; CDS (Class Data Sharing) / AppCDS ; JIT warmup à l’amorçage si besoin ; heap initial adapté (`-Xms`).
* **.NET** : trimming + ReadyToRun (AOT partiel).

---

## 6) Runtime CPU : quotas, pinning, NUMA

### 6.1 Quotas & poids (cgroups v2)

```bash
docker run --cpus=2.0 --cpu-shares=512 IMAGE
```

* `--cpus` = quota CFS (limite dure).
* `--cpu-shares` = **poids** relatif (meilleur effort).

### 6.2 Affinité CPU (réduction de migrations)

```bash
docker run --cpuset-cpus="0-3,6" IMAGE
```

### 6.3 NUMA (mémoire locale aux CPU)

```bash
docker run --cpuset-mems="0" --cpuset-cpus="0-7" IMAGE
```

* Utile sur serveurs bi-socket. **Mesurer** avant/après (latence).

---

## 7) Runtime Mémoire : limites & OOM

### 7.1 Limites & réservations

```bash
docker run --memory=1g --memory-swap=2g --memory-reservation=512m IMAGE
```

* `--memory` : plafond ; `--memory-swap` : mémoire+swap ; **éviter** OOMKill (137).
* `--memory-reservation` : limite **souple** (soft).

### 7.2 File descriptors & PIDs

```bash
docker run --ulimit nofile=65535:65535 --pids-limit=512 IMAGE
```

* Éviter FD épuisés sous haute charge.

---

## 8) Stockage & I/O (overlay2 vs volumes)

### 8.1 overlay2 (copy-on-write)

* **Écrire lourdement** dans le layer overlay = **lent** (COW).
  → Monter les répertoires **écrits** sur **volumes** (ext4/xfs) :

```bash
docker run -v data:/var/lib/app ...
```

### 8.2 Volumes sur disques rapides

* Placer `/var/lib/docker` (ou `data-root`) sur SSD/NVMe.
* Éviter NFS/SMB pour DBs critiques (latence). Si NFS requis : `nfsvers=4.1`, rsize/wsize adaptés.

### 8.3 Logs & rotation (I/O)

* `json-file` avec `max-size/max-file` pour réduire I/O disque (Chap. 09).
* Driver `local` (compaction) peut réduire footprint.

---

## 9) Réseau & latence

### 9.1 MTU & hairpin

* Adapter MTU des bridges si VLAN/PPPoE → éviter fragmentation.
* Tester l’accès **depuis un conteneur** à un service exposé via l’hôte (hairpin NAT).

### 9.2 Publication de ports

* Lier sur IP **spécifique** (`127.0.0.1:PORT`) si local uniquement (évite parcours réseau inutile).
* Pour **très** haut débit/latence minimale : `--network host` (à évaluer **sécurité**), sinon macvlan.

### 9.3 DNS

* Serveur DNS **proche** et fiable ; éviter timeouts → forte incidence perf.

---

## 10) Bind mounts & WSL2 (Windows 10)

* Sous Windows, les bind mounts **depuis NTFS** sont plus lents.
* **WSL2** : placez le projet **dans le FS Linux** (ex. `/home/…`), pas sous `\\wsl$` ni `C:\…` monté → gros gain I/O.
* Préférez **volumes nommés** pour caches/données ; limitez les binds aux **configs** RO.

---

## 11) Journaux & observabilité : coût minimal

* Logs **structurés** mais **pas verbeux** en prod (`info`), **debug** activé **temporairement**.
* **Échantillonnage** des traces (OTEL) à 1–10 % si trafic élevé.
* Export via **Fluent Bit** (léger) plutôt que stacks lourdes co-localisées.

---

## 12) Parallelisme & scale

* Préférer **horizontal scaling** (plusieurs réplicas) à un seul conteneur géant.
* **Load balancer** (Nginx/HAProxy/Traefik) en frontal ; stratégies **keep-alive** et **timeouts** adaptés.
* **Workers** app (gunicorn, node cluster) calibrés = `nb_cores × (1..2)` (mesurer).

---

## 13) Méthodologie d’optimisation (boucle)

1. **Mesurer** (baseline) → 2) **Formuler une hypothèse** →
2. **Modifier un seul paramètre** → 4) **Re-mesurer** →
3. **Valider/Revenir** → 6) **Documenter** (valeurs, gains, risques).

---

## 14) Exemples “prod-like” optimisés

### 14.1 Service web durci & performant

```bash
docker run -d --name web \
  -p 8080:8080 \
  --cpus=1.5 --cpu-shares=512 --cpuset-cpus="0-2" \
  --memory=512m --memory-swap=1g --memory-reservation=256m \
  --ulimit nofile=65535:65535 --pids-limit=512 \
  --read-only --tmpfs /tmp --tmpfs /run \
  -v data_web:/var/lib/app \
  --log-driver=json-file --log-opt max-size=10m --log-opt max-file=3 \
  ghcr.io/acme/web@sha256:<digest>
```

### 14.2 Compose avec réseaux séparés & caches de build

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
      cache_from: [ "type=registry,ref=ghcr.io/acme/cache:api" ]
      cache_to:   [ "type=registry,ref=ghcr.io/acme/cache:api,mode=max" ]
    deploy: {}            # ignoré par Compose, cf. Chap. 06
    read_only: true
    tmpfs: [ /run, /tmp ]
    ulimits:
      nofile: 65535
    networks: [ frontend, backend ]
  web:
    image: nginx:1.27
    ports: [ "80:80" ]
    networks: [ frontend ]
networks:
  frontend: { driver: bridge }
  backend:  { driver: bridge, internal: true }
```

---

## 15) Dépannage perf — cas fréquents & remèdes

* **CPU throttling** (latence en dents de scie) → réduire `--cpus` ? non : **augmenter** ou **retirer** le quota ; utiliser `--cpu-shares` pour priorité relative.
* **OOMKill (137)** → augmenter `--memory`, abaisser caches applicatifs, vérifier **fuites**.
* **I/O lents** → éviter overlay pour données, basculer sur **volumes**, déplacer `/var/lib/docker` sur SSD/NVMe.
* **Pull très long** → proxy cache, image trop grosse (multi-stage, distroless), réseau/MTU.
* **Cold start long** → pré-build (AOT, artefacts), healthcheck rapide, réduire init (migrations DB en **job séparé**).
* **DNS intermittents** → `--dns` stable, éviter résolveurs distants, TTL.

---

## 16) Aide-mémoire (commandes clés)

```bash
# Mesure
docker stats --no-stream NAME
time docker run --rm IMAGE true           # overhead démarrage brut
docker events --since 10m | grep NAME

# CPU/MEM limites
docker update --cpus=2 --memory=1g NAME

# FS & disque
docker system df
sudo du -xhd1 /var/lib/docker/
df -h

# Réseau
docker port NAME
docker run --rm -it --network NET nicolaka/netshoot

# BuildKit
docker buildx build --cache-from=type=registry,ref=... --cache-to=type=registry,ref=...,mode=max -t IMG .
```

---

## 17) Checklist de clôture (perf “prête-prod”)

**Images & build**

* Multi-stage, `.dockerignore` strict, base **minimale**, dépendances **pinnées**.
* Caches BuildKit **activés** (local/registry) ; proxy cache registry **en place**.
* Taille et couches **raisonnées** ; healthcheck **léger**.

**Runtime**

* Limites **cgroups** définies ; pinning CPU/NUMA si utile ; ulimits **nofile** suffisants.
* Répertoires écrits sur **volumes** ; **overlay** réservé au code/RO.
* **Logs** rotés et peu verbeux ; traces échantillonnées.

**Réseau**

* Réseaux **séparés** ; MTU et hairpin **testés** ; DNS local fiable.
* Publication de ports **explicite** et sur IP nécessaire uniquement.

**Plateforme**

* `/var/lib/docker` sur disque rapide ; monitoring de capacité.
* Benchmarks **avant/après** + budgets documentés ; runbooks prêts.


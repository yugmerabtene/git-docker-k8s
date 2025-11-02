# Chapitre-09 — Observabilité & Diagnostic

## Objectifs d’apprentissage

* Mettre en place une **observabilité** complète : **logs** (collecte/rotation/centralisation), **métriques** (ressources & services), **traces** (requêtes) et **événements** (lifecycle).
* Utiliser efficacement les **outils de diagnostic** Docker/OS : `logs`, `stats`, `events`, `inspect`, `top`, `diff`, `port`, `nsenter`, `tcpdump`, `strace`, `lsof`.
* Écrire des **runbooks** d’incident (CPU/MEM/IO, réseau, DNS, disque, permissions, healthchecks, crash loops) et des **alertes** (Prometheus).

## Pré-requis

* Chap. 01–08 maîtrisés (images, conteneurs, storage, réseau, build, compose, sécurité).
* Notions Linux (processus, cgroups, journaux, réseau).

---

## 1) Piliers de l’observabilité

* **Logs** : événements applicatifs structurés (JSON/logfmt), journaux système, logs conteneurs.
* **Métriques** : compteurs/jauges (CPU, MEM, IO, latence, erreurs), séries temporelles.
* **Traces** : cheminement d’une requête **end-to-end** (span/trace id).
* **Événements** : chronologie Docker (create, start, die, oom, health_status…).

---

## 2) Journaux Docker (drivers & rotation)

### 2.1 Drivers de logs

* `json-file` (par défaut) / `local` (compaction) — simples, locaux.
* `journald` / `syslog` — intégrés à l’OS.
* `fluentd` / `gelf` / `awslogs` / `splunk` — vers une stack externe.
* `none` — à proscrire sauf cas très particuliers.

### 2.2 Rotation & configuration globale

`/etc/docker/daemon.json` :

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

> Évite le remplissage disque par un conteneur trop verbeux.

### 2.3 Override par conteneur

```bash
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=20m --log-opt max-file=5 \
  --label app=web --env LOG_LEVEL=info \
  nginx:1.27
```

### 2.4 Bonnes pratiques de logs

* **Structurés** (JSON/logfmt), inclure `service`, `env`, `version`, **correlation/trace id**.
* Ne pas loguer de **secrets**.
* Limiter le verbiage par **niveau** (`info/warn/error`) + rotation.

---

## 3) Centraliser les logs (exemple minimal Fluent Bit)

`compose.logging.yaml`

```yaml
services:
  fluentbit:
    image: cr.fluentbit.io/fluent/fluent-bit:2
    volumes:
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    ports: ["24224:24224"]
    restart: unless-stopped
```

`fluent-bit.conf` (idée) : tail des `*.log` → sortie HTTP/ES/Loki.
Astuce : taguer par **labels** conteneur (`app`, `env`) pour filtres côté SIEM/observabilité.

---

## 4) Métriques (cAdvisor/Node Exporter → Prometheus/Grafana)

### 4.1 cAdvisor & Node Exporter (Compose extrait)

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.2
    ports: ["8088:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  nodeexporter:
    image: prom/node-exporter:v1.8.1
    pid: host
    network_mode: host
```

> cAdvisor : métriques par conteneur (CPU, mémoire, IO, restarts). Node Exporter : hôte.

### 4.2 Prometheus (scrape) & alertes de base (idées)

* **Alertes** utiles :

  * `container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9`
  * Restarts > N sur 10 min
  * Disque `/var/lib/docker` > 85 %
  * OOMKills > 0
  * Latence P95 service > S

---

## 5) Tracing (OpenTelemetry + Jaeger/Tempo)

### 5.1 Variables OTEL côté service

```bash
docker run -d --name api \
  -e OTEL_SERVICE_NAME=api \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317 \
  -e OTEL_TRACES_SAMPLER=parentbased_traceidratio \
  -e OTEL_TRACES_SAMPLER_ARG=0.1 \
  ghcr.io/acme/api:1.4.2
```

### 5.2 Compose (collector + jaeger, aperçu)

* `otel-collector` (recevoir OTLP, exporter vers Jaeger/Grafana Tempo).
* `jaeger` UI pour visualiser les traces.

---

## 6) CLI Docker pour diagnostiquer

```bash
docker ps -a
docker inspect NAME                    # métadonnées complètes
docker logs -f --tail=200 NAME         # journaux live
docker top NAME                        # processus
docker stats --no-stream NAME          # ressources
docker diff NAME                       # changements FS
docker port NAME                       # mapping ports
docker events --since=1h               # chronologie
docker system df                       # usage disque
```

**Formatage ciblé**

```bash
docker inspect -f '{{.State.Status}} {{.State.Health.Status}}' NAME
docker inspect -f '{{json .HostConfig.LogConfig}}' NAME | jq
docker inspect -f '{{json .Mounts}}' NAME | jq
```

---

## 7) Diagnostic réseau (interne & externe)

### 7.1 Container “trousse à outils” (netshoot)

```bash
docker run --rm -it --network app-net nicolaka/netshoot
# inside:
dig db
curl -sv http://api:8080/health
ss -lntp
traceroute 1.1.1.1
```

### 7.2 `tcpdump` ciblé

```bash
docker run --rm --net=host --privileged nicolaka/netshoot \
  tcpdump -nni any port 8080
```

### 7.3 `nsenter` (espace réseau du conteneur)

```bash
PID=$(docker inspect -f '{{.State.Pid}}' NAME)
sudo nsenter -t $PID -n sh -lc 'ip a; ip route; ss -lntp'
```

**Pièges courants**

* Service qui écoute sur `127.0.0.1` (au lieu de `0.0.0.0`).
* MTU/hairpin NAT : symptoms = timeouts aléatoires.
* DNS : mauvais `search`/serveur, TTLs trop longs, split-DNS.

---

## 8) Stockage & disque (overlay2/logs)

```bash
docker system df                      # vue Docker
df -h /var/lib/docker                 # capacité disque
sudo du -xhd1 /var/lib/docker/        # répertoires lourds
journalctl -u docker --since "1h ago" # erreurs dockerd
```

* Logs conteneurs : `/var/lib/docker/containers/<id>/<id>-json.log` (si `json-file`).
* Overlay2 : `upperdir` volumineux = écritures importantes (cache, build, temp).

---

## 9) CPU/Mémoire/PIDs (cgroups & OOMKill)

### 9.1 Détection OOMKill

```bash
docker inspect -f '{{.State.OOMKilled}}' NAME
docker events --since=2h | grep -i oom
```

### 9.2 Inspecter processus & FD

```bash
docker top NAME
docker exec NAME sh -lc 'ulimit -n; ls /proc/1/fd | wc -l'
```

### 9.3 Profilage rapide (selon image)

* `strace -fp <pid>` pour syscalls.
* `lsof -p <pid>` descripteurs ouverts.
* Applis instrumentées (pprof/py-spy/async-profiler) si disponible.

---

## 10) Healthchecks & crash loops

* Lire `State.Health.Log` via `inspect`, récupérer **les messages** d’échec.
* `depends_on: condition: service_healthy` (Compose) pour l’ordre de démarrage.
* Crash loop : vérifier **exit code** (`docker wait`), fichiers manquants, **port en usage**, **secret** non monté, **permissions**.

---

## 11) Runbooks d’incident (exemples concrets)

### 11.1 **Port déjà utilisé / service inaccessible**

1. `docker ps`, `docker port NAME`, `ss -lntp | grep :8080`
2. Vérifier que l’app écoute `0.0.0.0:8080` dans le conteneur.
3. Conflit : changer `-p`, arrêter le service fautif, ou lier sur `127.0.0.1`.

### 11.2 **CPU anormalement élevé**

1. `docker stats --no-stream NAME`, `top`/`htop` dans le conteneur.
2. Activer logs debug **temporairement** ; vérifier boucles/log spam.
3. Profiler (pprof/py-spy) si possible ; limiter via `--cpus`, alerter.

### 11.3 **Fuite mémoire / OOMKill (137)**

1. `inspect .State.OOMKilled`, `events`.
2. `stats` (MEM) & taille du **working set**.
3. Augmenter `--memory` si besoin **temporaire**, corriger la fuite.

### 11.4 **DNS / résolutions intermittentes**

1. `docker exec` → `cat /etc/resolv.conf`.
2. `dig name`, `nslookup`, vérifier `--dns`/`--dns-search`.
3. TTL, split-DNS, latence vers résolveur ; mettre un résolveur local fiable.

### 11.5 **Timeouts réseau / MTU**

1. `curl -v` vs `telnet host port`.
2. `tcpdump` voir SYN/SYN-ACK, ICMP frag needed, PMTU.
3. Ajuster **MTU** bridges/VM, vérifier hairpin NAT.

### 11.6 **Disque plein (logs ou overlay2)**

1. `df -h`, `docker system df`, `du -xhd1 /var/lib/docker`.
2. Rotation logs (`max-size/max-file`), `docker system prune` **maîtrisé**.
3. Agrandir partition, déplacer `data-root`, nettoyer images orphelines.

### 11.7 **Permission denied (SELinux/UID)**

1. SELinux : ajouter `:Z`/`:z` sur binds RHEL/Fedora.
2. UID/GID : aligner avec `-u` et `chown` le volume.
3. AppArmor : profil custom si nécessaire.

### 11.8 **TLS/Certificats (clients ou proxys)**

1. Horloge/NTP, chaîne **fullchain**, CN/SAN.
2. CAs dans trust store de l’image ; tester `openssl s_client`.
3. Renouvellement automatique (ACME) opérationnel.

---

## 12) Boîte à outils “diagnostic”

* **Images** : `nicolaka/netshoot` (réseau), `alpine`, `busybox`.
* **CLI utiles** : `curl`, `dig`, `nc`, `iproute2`, `ss`, `tcpdump`, `jq`, `strace`, `lsof`.
* **Hôte** : `journalctl`, `dmesg`, `iptables/nft`, `conntrack`, `iostat`, `iotop`, `pidstat`.

---

## 13) “Support bundle” (collecte standardisée)

Script (extrait bash) :

```bash
#!/usr/bin/env bash
set -euo pipefail
OUT="support_$(hostname)_$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$OUT"
docker info > "$OUT/docker_info.txt"
docker ps -a > "$OUT/docker_ps_a.txt"
docker system df > "$OUT/docker_system_df.txt"
journalctl -u docker --since "24 hours ago" > "$OUT/journal_docker_24h.log"
df -h > "$OUT/df_h.txt"
free -m > "$OUT/free_m.txt"
ss -lntp > "$OUT/ss_lntp.txt"
for c in $(docker ps --format '{{.Names}}'); do
  mkdir -p "$OUT/ct_$c"
  docker inspect "$c" > "$OUT/ct_$c/inspect.json"
  docker logs --since=12h "$c" > "$OUT/ct_$c/logs_12h.txt" || true
  docker stats --no-stream "$c" > "$OUT/ct_$c/stats.txt" || true
done
tar czf "$OUT.tgz" "$OUT"
echo "Bundle: $OUT.tgz"
```

---

## 14) Aide-mémoire (commandes clés)

```bash
# Vue rapide
docker ps -a
docker logs -f --tail=200 NAME
docker stats --no-stream NAME
docker inspect -f '{{.State.Status}} {{.State.Health.Status}}' NAME

# Événements & ports
docker events --since=1h
docker port NAME

# Réseau
docker run --rm -it --network NET nicolaka/netshoot
# Hôte & netns
PID=$(docker inspect -f '{{.State.Pid}}' NAME); sudo nsenter -t $PID -n ip a

# Disque
docker system df
sudo du -xhd1 /var/lib/docker

# OOM & exit codes
docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}' NAME
```

---

## 15) Checklist de clôture (observabilité “prête prod”)

* **Logs** : driver choisi, **rotation** configurée, **centralisation** opérationnelle, logs **structurés** + corrélation id.
* **Métriques** : cAdvisor + Node Exporter scrappés ; dashboards de base ; **alertes** CPU/MEM/IO, restarts, disque, latence/erreurs.
* **Traces** : OTEL activé vers collector (Jaeger/Tempo) sur les services critiques.
* **Événements** : procédure de lecture/archivage des `docker events`.
* **Runbooks** : incidents types documentés, commandes & critères de sortie clairs.
* **Support bundle** : script validé pour collecter preuves en 1 commande.
* **NTP** fiable, capacité `/var/lib/docker` surveillée, MTU/hairpin testés.
* **Sécurité** respectée pendant le diag (pas de `--privileged` par défaut, secrets masqués).


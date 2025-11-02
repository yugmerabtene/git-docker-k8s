# Chapitre-02 — Conteneurs (cycle de vie & exécution)

## Objectifs d’apprentissage

* Gérer **tout le cycle de vie** d’un conteneur : créer, démarrer, arrêter, redémarrer, supprimer.
* Exploiter les **modes d’exécution** (interactif/détaché), **journaux**, **exec**, **stats**, **healthchecks**.
* Appliquer des **limites de ressources** (CPU, mémoire, PIDs, ulimit) et des **politiques de redémarrage**.
* Savoir **diagnostiquer** (inspect/top/logs/events) et **paramétrer** l’arrêt propre (signaux, délais).

## Pré-requis

* Docker Engine/CLI opérationnel.
* Bases Linux (shell), notions de processus et signaux.

---

## 1) Cycle de vie : commandes essentielles

### 1.1 Créer vs exécuter

```bash
# Crée un conteneur stoppé (pas encore lancé)
docker create --name web nginx:1.27

# Lance un conteneur (crée + démarre)
docker run --name web -d -p 8080:80 nginx:1.27
```

### 1.2 Démarrer / arrêter / redémarrer / tuer / supprimer

```bash
docker start web                 # démarre un conteneur existant
docker stop -t 20 web            # SIGTERM puis SIGKILL après 20s
docker restart web               # stop + start
docker kill --signal SIGUSR1 web # envoie un signal immédiat
docker rm web                    # supprime (doit être arrêté)
docker rm -f web                 # force (kill + rm)
```

### 1.3 Lister / filtrer / formater

```bash
docker ps                        # conteneurs en cours
docker ps -a                     # tous (y compris arrêtés)
docker ps --filter "status=exited" --format '{{.Names}}\t{{.Status}}'
```

### 1.4 Pause / reprise / attente

```bash
docker pause web                 # gèle les processus (cgroup freezer)
docker unpause web
docker wait web                  # bloque jusqu’à l’arrêt et renvoie le code de sortie
```

---

## 2) Modes d’exécution & options courantes

### 2.1 Interactif vs détaché

```bash
docker run -it --name shell alpine:3.20 sh    # interactif avec TTY
docker run -d  --name job busybox sleep 3600  # détaché
```

* `-i` : STDIN ouvert ; `-t` : alloue un pseudo-TTY.
* `-d` : exécution en arrière-plan.

### 2.2 Nom, redémarrage, nettoyage

```bash
docker run --name api --restart=unless-stopped -d ghcr.io/acme/api:1.4.2
docker run --rm -it alpine sh   # supprimé automatiquement à l’arrêt
```

**Politiques `--restart`** :

* `no` (défaut), `on-failure[:N]`, `always`, `unless-stopped`.

### 2.3 Entrypoint / commande / répertoire / utilisateur

```bash
docker run --entrypoint /bin/mywrap    image ...
docker run --workdir /app              image ...
docker run -u 10001:10001              image ...
docker run image args...               # remplace CMD
```

* `--entrypoint` **remplace** ENTRYPOINT ; les “args” complètent/écrasent CMD.

### 2.4 Variables d’environnement

```bash
docker run -e APP_ENV=prod -e TZ=Europe/Paris image
docker run --env-file .env image
```

### 2.5 Réseau, ports, DNS, hosts (aperçu — détails au Chapitre Réseau)

```bash
docker run -p 8080:80 nginx
docker run --network my-net --ip 172.18.0.10 image
docker run --dns 1.1.1.1 --dns-search corp.local image
docker run --add-host db.internal:10.0.0.10 image
```

> Montages (volumes/bind/tmpfs) → Chapitre **Storage**.

---

## 3) Journaux, exec, stats, top, inspect, events

### 3.1 Logs

```bash
docker logs web
docker logs -f --since=10m --tail=100 web
docker logs --timestamps web
```

* Paramétrage du **driver de logs** par conteneur :

```bash
docker run --log-driver=json-file \
           --log-opt max-size=10m --log-opt max-file=3 \
           -d nginx
```

> Centralisation & métriques → Chapitre **Observabilité**.

### 3.2 Exec (ouvrir une session / lancer une commande)

```bash
docker exec -it web sh                 # shell interactif
docker exec -u 10001:10001 web id      # en tant qu’utilisateur spécifique
docker exec -e DEBUG=1 -w /app web ls  # avec env & répertoire de travail
```

### 3.3 Stats (ressources) & top (processus)

```bash
docker stats                   # stream des usages CPU/MEM/NET/IO
docker stats --no-stream web
docker top web                 # équivalent ps aux
```

### 3.4 Inspect (métadonnées) & formatage

```bash
docker inspect web | jq
docker inspect -f '{{.State.Status}} {{.State.Health.Status}}' web
docker inspect -f '{{json .NetworkSettings.Ports}}' web | jq
```

### 3.5 Events (chronologie des événements)

```bash
docker events --since 1h      # créer/kill/oom/kern events, etc.
```

---

## 4) Healthchecks (runtime)

### 4.1 Définir au lancement

```bash
docker run -d --name api \
  --health-cmd='curl -fsS http://localhost:8080/health || exit 1' \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  --health-start-period=20s \
  ghcr.io/acme/api:1.4.2
```

* Statut dans `.State.Health.Status` : `starting` → `healthy` / `unhealthy`.
* Sur échec, **Docker ne redémarre pas** automatiquement (sauf si votre superviseur/policy le fait).
* **Priorité** : un healthcheck défini en ligne de commande **écrase** celui du Dockerfile.

### 4.2 Lire l’état

```bash
docker inspect -f '{{.State.Health.Status}}' api
```

---

## 5) Limites de ressources & ulimit

### 5.1 CPU & mémoire (cgroups v2)

```bash
docker run --cpus=1.5 --memory=512m --memory-swap=1g image
docker run --cpuset-cpus="0,2" --cpu-shares=512 image
```

* `--cpus` : quota/period simplifié.
* `--memory` : limite stricte ; `--memory-swap` : mémoire + swap.
* `--cpu-shares` : poids relatif (meilleur effort).
* `--cpuset-cpus` : affinité CPU (ex: “0-3,6”).

### 5.2 Limites de PIDs & ulimit

```bash
docker run --pids-limit=256 image
docker run --ulimit nofile=4096:8192 --ulimit nproc=512:1024 image
```

### 5.3 Mettre à jour un conteneur en cours

```bash
docker update --cpus=2 --memory=1g --pids-limit=512 api
docker update --restart=always api
```

---

## 6) Signaux, arrêt propre & PID1

### 6.1 Arrêt propre

* `docker stop` envoie **SIGTERM**, puis **SIGKILL** après `--time` secondes.
* Si votre process PID1 **ignore SIGTERM**, il risque d’être tué (143 = TERM, 137 = KILL).

```bash
docker stop -t 30 api        # laisse 30s pour se terminer proprement
```

### 6.2 PID1 & init

* PID1 ne relaie pas toujours les signaux et ne “reap” pas les zombies.
* Utilisez un init léger :

```bash
docker run --init ... image  # Tini par défaut
```

### 6.3 Personnaliser le signal/timeout

```bash
docker run --stop-signal=SIGQUIT --stop-timeout=20 image
```

---

## 7) Copie de fichiers & renommage

```bash
docker cp ./config.yml api:/app/config.yml
docker cp api:/var/log/app.log ./app.log
docker rename api api-v1
```

> Pour déplacer des **données applicatives**, préférez les **volumes** (Chapitre Storage).

---

## 8) Sécurité d’exécution (rappels rapides)

*(Détails au Chapitre **Sécurité & Durcissement**)*

```bash
# Exécuter en utilisateur non-root
docker run -u 10001:10001 --read-only --tmpfs /tmp image

# Réduire les privilèges
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE image

# Empêcher l’élévation
docker run --security-opt no-new-privileges:true image
```

> Évitez `--privileged`. Montez uniquement ce qui est nécessaire (volumes/bind précis).

---

## 9) Dépannage & diagnostics rapides

### 9.1 Classiques

```bash
docker ps -a
docker logs -f --tail=200 nom
docker inspect nom | jq '.State, .HostConfig, .NetworkSettings'
docker top nom
docker stats --no-stream nom
docker events --since 30m
```

### 9.2 Cas fréquents

* **Port déjà utilisé** : vérifier `docker ps`, `ss -lntp`.
* **OOMKill (137)** : voir `docker inspect -f '{{.State.OOMKilled}}'`.
* **DNS** : tester `--dns`/`--add-host`, vérifier /etc/resolv.conf du conteneur.
* **Fichier introuvable (127/126)** : vérifier `ENTRYPOINT/CMD`, droits d’exécution, `WORKDIR`.
* **Healthcheck en échec** : exécuter la même commande avec `docker exec` pour diagnostiquer.

---

## 10) Exit codes utiles

* **0** : succès.
* **125** : erreur Docker (échec `docker run` lui-même).
* **126** : commande trouvée mais **non exécutable**.
* **127** : commande **introuvable**.
* **137** : **SIGKILL** (souvent OOMKill).
* **143** : **SIGTERM** (arrêt normal via `docker stop`).

```bash
docker wait job && echo $?
```

---

## 11) Bonnes pratiques (Do & Don’t)

**Do**

* Utiliser `--restart=unless-stopped` ou `always` pour les services.
* Préférer `--init` si votre process PID1 ne gère pas bien les signaux.
* Fixer **des limites** (`--cpus`, `--memory`, `--pids-limit`, `--ulimit`).
* Centraliser les **logs** et configurer la **rotation**.
* Lancer en **utilisateur non-root** et lecture seule (`--read-only` + `--tmpfs`).

**Don’t**

* Ne pas dépendre d’un **shell** comme PID1 si votre app peut être PID1 directement.
* Éviter `--privileged` (sauf cas exceptionnel et maîtrisé).
* Éviter de monter `-v /:/host` ou des chemins hôte sensibles.
* Ne pas ignorer un **healthcheck** rouge ; investiguer d’abord.

---

## 12) Exemples synthèse

### 12.1 Service web “prod-like” (sécurisé & limité)

```bash
docker run -d --name web \
  -p 8080:8080 \
  --restart=unless-stopped \
  --cpus=1.0 --memory=512m --pids-limit=256 \
  --read-only --tmpfs /tmp --tmpfs /run \
  -u 10001:10001 \
  --health-cmd='curl -fsS http://localhost:8080/health || exit 1' \
  --health-interval=30s --health-timeout=5s --health-retries=3 \
  ghcr.io/acme/web:1.4.2
```

### 12.2 Conteneur de debug éphémère

```bash
docker run --rm -it --network app-net --entrypoint sh alpine:3.20
```

### 12.3 Ajuster à chaud une limite

```bash
docker update --memory=768m --cpus=1.5 web
```

---

## 13) Aide-mémoire (cheat-sheet minimal)

```bash
# Lister / filtrer
docker ps -a
docker ps --filter "status=exited"

# Démarrer / arrêter / supprimer
docker start NAME
docker stop -t 20 NAME
docker rm -f NAME

# Logs / Exec
docker logs -f --tail=200 NAME
docker exec -it NAME sh

# Ressources / Process / État
docker stats --no-stream NAME
docker top NAME
docker inspect NAME

# Health
docker inspect -f '{{.State.Health.Status}}' NAME

# Update / Restart policy
docker update --cpus=2 --memory=1g NAME
docker update --restart=always NAME
```

---

## 14) Checklist de clôture (qualité d’exécution d’un service)

* Politique `--restart` définie et adaptée au rôle du service.
* **Healthcheck** pertinent ; état **healthy** observé.
* **Limites** cgroups définies (CPU/MEM/PIDs/ulimit).
* Journaux lisibles et **rotation** configurée.
* **Arrêt propre** testé (`stop -t` suffisant ; pas de KILL systématique).
* **Utilisateur non-root**, système de fichiers **read-only** + `tmpfs` nécessaires.
* Commandes de dépannage documentées (logs/exec/inspect/stats/top/events).


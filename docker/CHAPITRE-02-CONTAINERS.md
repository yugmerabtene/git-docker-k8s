# CHAPITRE-02 — CONTAINERS DOCKER (théorie + pratique approfondie)

## 0) Objectifs & prérequis

**Objectifs**

* Comprendre précisément ce qu’est un conteneur (couches, writable layer, isolation).
* Maîtriser le cycle de vie d’un conteneur (create/run/start/stop/restart/kill/pause/rm).
* Savoir exécuter des conteneurs avec `docker run` et toutes ses options importantes.
* Superviser, diagnostiquer, limiter, sécuriser et persister les données des conteneurs.

**Prérequis**

* Docker Engine/CLI 20.10+ installé (ou Docker Desktop).
* Connaissances de base du système (réseau, fichiers, permissions, variables d’environnement).

---

## 1) Notions clefs : image vs conteneur

* **Image** : modèle immuable (read-only), assemblage de **layers** (couches) content-adressées (SHA256).
* **Conteneur** : **instance en exécution** d’une image, avec une **couche writable** au sommet (copy-on-write).
* **Isolation** : noms d’espace (namespaces) + cgroups (limitation CPU/RAM/PIDs) + options de sécurité (seccomp, AppArmor, capabilities).
* **Réseau** par défaut : `bridge` (NAT) ; autres modes : `host`, `none`, ou réseaux utilisateur.
* **Processus PID 1** du conteneur : reçoit les signaux (SIGTERM/SIGKILL). Un mauvais PID1 peut ignorer/propager mal les signaux.

---

## 2) Cycle de vie et commandes de base

### 2.1 Lister les conteneurs — `docker ps` / `docker container ls`

```bash
docker ps                 # conteneurs RUNNING
docker ps -a              # tous (y compris stoppés)
docker ps -q              # IDs seulement
docker ps --format '{{.ID}} {{.Image}} {{.Names}} {{.Status}}'
```

Filtres utiles (`--filter` ou `-f`) :

* `status=running|exited|paused|created|restarting`
* `name=motif` ; `ancestor=image[:tag]` ; `before=id` ; `since=id`
* `label=key[=value]`

### 2.2 Créer / démarrer

```bash
docker create [OPTIONS] IMAGE [CMD ...]   # crée sans démarrer
docker start [OPTIONS] CONTAINER          # démarre un conteneur créé ou stoppé
docker run   [OPTIONS] IMAGE [CMD ...]    # create + start en une commande
```

### 2.3 Arrêter / redémarrer / tuer

```bash
docker stop  CONTAINER         # envoie SIGTERM puis SIGKILL après timeout
docker restart CONTAINER       # stop puis start
docker kill   CONTAINER        # envoie SIGKILL (ou --signal=SIGUSR1)
```

Options utiles :

```bash
docker stop -t 30 CONTAINER    # délai d’attente (seconds) avant SIGKILL
docker kill --signal=SIGINT CONTAINER
```

### 2.4 Pause / reprise

```bash
docker pause   CONTAINER    # freeze cgroups
docker unpause CONTAINER
```

### 2.5 Suppression

```bash
docker rm CONTAINER              # supprimer un conteneur stoppé
docker rm -f CONTAINER           # stop + remove (force)
docker container prune           # supprimer tous les conteneurs stoppés
```

---

## 3) `docker run` en profondeur (lancer un conteneur)

**Syntaxe générale**

```bash
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
```

### 3.1 Mode d’exécution & nommage

* Détaché vs interactif :

```bash
docker run -d IMAGE                  # détaché (en arrière-plan)
docker run -it IMAGE bash            # interactif + pseudo-TTY
docker run -it --rm IMAGE sh         # auto-nettoyage à la sortie
```

* Nom, hostname, labels :

```bash
docker run --name web1 nginx
docker run --hostname app01 --domainname prod.local alpine
docker run --label tier=frontend --label owner=yug nginx
```

* Redémarrage automatique (politiques) :

```bash
docker run --restart=no|on-failure[:N]|unless-stopped|always IMAGE
# Exemples
docker run --restart=on-failure:5 myjob
docker run --restart=unless-stopped nginx
```

* Entrypoint/Cmd override :

```bash
docker run --entrypoint /bin/sh alpine -c 'echo hello'
docker run -w /workdir alpine pwd
docker run -u 1000:1000 alpine id
```

### 3.2 Variables d’environnement et fichiers

```bash
docker run -e KEY=VALUE -e TZ=Europe/Paris alpine env
docker run --env-file .env myapp
```

* `--env-file` : fichier texte `KEY=VALUE` (une par ligne).
* Masquer des secrets en runtime : préférer variables de runtime (et non dans l’image). Pour le build, préférer `--secret` (BuildKit).

### 3.3 Volumes et montages

Deux syntaxes : `-v` (historique) et `--mount` (recommandée pour la clarté).

#### a) `-v` / `--volume` : `SRC:DST[:OPTIONS]`

```bash
# Bind mount (host path -> container path)
docker run -v /host/data:/data nginx

# Named volume (géré par Docker)
docker volume create appdata
docker run -v appdata:/var/lib/app myapp

# Options :ro (lecture seule), :Z / :z (SELinux), :rshared/...
docker run -v /host:/container:ro alpine
```

#### b) `--mount` : plus explicite

```bash
# Bind mount
docker run --mount type=bind,src=/host/data,dst=/data,ro nginx

# Named volume
docker run --mount type=volume,src=appdata,dst=/var/lib/app myapp

# tmpfs (en RAM)
docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=64m alpine
```

#### c) Rootfs read-only + tmpfs pour écriture éphémère

```bash
docker run --read-only --tmpfs /run --tmpfs /tmp myapp
```

Bonnes pratiques pour réduire la surface d’attaque.

### 3.4 Réseau, ports et DNS

* Publier des ports :

```bash
docker run -p 8080:80 nginx           # host:container
docker run -p 127.0.0.1:8080:80 nginx # bind sur loopback uniquement
docker run -P nginx                   # publie automatiquement les EXPOSE
```

* Choisir le réseau :

```bash
docker network create -d bridge netapp
docker run --network netapp --name web nginx
```

* Modes spéciaux :

```bash
docker run --network host nginx   # partage la pile réseau de l’hôte (Linux)
docker run --network none alpine  # pas de réseau
```

* DNS et résolutions :

```bash
docker run --dns 1.1.1.1 --dns-search corp.local --dns-option ndots:1 alpine
docker run --add-host db.local:10.0.0.10 myapp
```

* Alias dans un réseau utilisateur :

```bash
docker network connect --alias api netapp backend1
```

### 3.5 Limitation de ressources (cgroups)

* CPU :

```bash
docker run --cpus="1.5" myapp               # 1,5 CPU logiques
docker run --cpu-shares=512 myapp           # priorité relative
docker run --cpuset-cpus="0,2" myapp        # cores autorisés
```

* Mémoire :

```bash
docker run --memory=512m --memory-swap=1g myapp
docker run --memory-reservation=256m myapp  # soft limit
```

* PIDs, ulimits :

```bash
docker run --pids-limit=256 myapp
docker run --ulimit nofile=65536:65536 myapp
```

### 3.6 Sécurité (principales options)

* Capabilities & privilèges :

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp
docker run --privileged myapp               # très large, à éviter
```

* Profils de confinement :

```bash
docker run --security-opt seccomp=unconfined myapp
docker run --security-opt apparmor=profile_name myapp
```

* Utilisateur et userns :

```bash
docker run -u 1000:1000 myapp               # éviter root si possible
# user namespaces (Docker daemon config) pour mappage UID host≠container
```

* Périphériques et GPU :

```bash
docker run --device /dev/ttyUSB0 myapp
docker run --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04
```

* IPC/PID namespace :

```bash
docker run --pid=host myapp                 # partage table des PIDs (diagnostic)
docker run --ipc=host myapp
```

### 3.7 Journaux et log drivers

* Journaux d’un conteneur :

```bash
docker logs CONTAINER
docker logs -f --tail=100 CONTAINER
docker logs --since=10m CONTAINER
```

* Choisir le driver de log :

```bash
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 myapp
docker run --log-driver local myapp         # efficace, rotation intégrée
# Autres : syslog, journald, gelf, fluentd, awslogs, splunk, etc.
```

### 3.8 Healthcheck (surveillance de santé)

* Déclaré dans l’image (Dockerfile `HEALTHCHECK`) ou surchargé au run :

```bash
docker run \
  --health-cmd='curl -fsS http://localhost:8080/health || exit 1' \
  --health-interval=10s \
  --health-timeout=3s \
  --health-retries=3 \
  --health-start-period=15s \
  myapp
```

* Statut visible dans `docker ps` (`(healthy)`, `(unhealthy)`).

### 3.9 Autres options utiles

```bash
docker run --init myapp                      # min-init pour gérer les zombies
docker run --rm myapp                        # supprime le conteneur à l’arrêt
docker run --workdir /app myapp              # répertoire courant
docker run --tty --interactive alpine sh     # équivalent -it
docker run --ip 172.18.0.10 --network netapp myapp  # IP statique (bridge user)
docker run --mac-address 02:42:ac:11:00:0a myapp
```

---

## 4) Gestion, inspection, exécution dans un conteneur existant

### 4.1 Inspecter et superviser

```bash
docker inspect CONTAINER                  # JSON complet (network, mounts, args…)
docker top CONTAINER                      # processus en cours
docker stats                              # CPU/MEM/NET/IO live (tous)
docker stats CONTAINER1 CONTAINER2        # sélection
docker port CONTAINER                     # mapping de ports effectif
docker diff CONTAINER                     # changements sur le FS (A/C/D)
```

### 4.2 Exécuter une commande à l’intérieur — `docker exec`

```bash
docker exec CONTAINER ls /                # non interactif
docker exec -it CONTAINER /bin/sh         # shell interactif
docker exec -u 1000:1000 CONTAINER whoami # en tant qu’utilisateur spécifique
```

### 4.3 Attacher / détacher

```bash
docker attach CONTAINER
# Détacher sans tuer le process (si supporté) : Ctrl-p Ctrl-q
```

### 4.4 Copie de fichiers — `docker cp`

```bash
docker cp host.txt CONTAINER:/data/host.txt
docker cp CONTAINER:/var/log/app.log ./app.log
```

### 4.5 Mise à jour à chaud de limites — `docker update`

```bash
docker update --cpus=2 --memory=1g CONTAINER
```

### 4.6 Événements Docker — `docker events`

```bash
docker events                              # flux temps réel (start/stop/oom…)
```

### 4.7 Attendre / codes de sortie

```bash
docker wait CONTAINER                      # bloque jusqu’à l’arrêt, retourne exit code
docker inspect --format='{{.State.ExitCode}}' CONTAINER
```

### 4.8 Export / commit (à utiliser avec discernement)

* Exporter le rootfs d’un conteneur :

```bash
docker export CONTAINER > rootfs.tar
```

* Faire une image à partir d’un conteneur (non recommandé en CI/CD) :

```bash
docker commit CONTAINER repo/image:tag
```

Bonne pratique : **préférer un Dockerfile** pour la traçabilité.

---

## 5) Exemples complets

### 5.1 NGINX en frontal, volumétrie et logs

```bash
docker run -d --name web \
  -p 80:80 \
  --restart=unless-stopped \
  --mount type=bind,src=/srv/www,dst=/usr/share/nginx/html,ro \
  --log-driver local \
  nginx:1.25
```

### 5.2 PostgreSQL avec persistance, locale et healthcheck

```bash
docker volume create pgdata
docker run -d --name pg \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  -e TZ=Europe/Paris \
  --mount type=volume,src=pgdata,dst=/var/lib/postgresql/data \
  --health-cmd="pg_isready -U postgres || exit 1" \
  --health-interval=10s --health-timeout=3s --health-retries=5 \
  --restart=unless-stopped \
  postgres:16
```

### 5.3 Service applicatif durci (read-only + tmpfs + capabilities minimales)

```bash
docker run -d --name app \
  -p 8080:8080 \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  -u 1000:1000 \
  --restart=on-failure:3 \
  myorg/myapp:1.0
```

### 5.4 Outil shell éphémère dans le réseau de l’app

```bash
docker run --rm -it --network netapp alpine sh
# diagnostic tcpdump/nslookup/curl, etc.
```

### 5.5 GPU (NVIDIA)

```bash
docker run --gpus all --rm nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

---

## 6) Dépannage rapide

* **Port déjà utilisé** : “address already in use”

  * Trouver qui écoute : `ss -ltnp | grep :8080` (Linux) / `netstat -ano` (Windows).
  * Changer le host-port : `-p 8081:80` ou arrêter le service conflictuel.

* **Permission denied sur bind mount** (Linux SELinux) :

  * Ajouter `:Z` (relabel) ou `:z` sur `-v` / `--mount`, ex. `,Z`.

* **Conteneur quitte immédiatement** :

  * Vérifier : `docker logs` et `docker inspect .State.ExitCode`.
  * S’assurer que le CMD/Entrypoint ne termine pas aussitôt (long-running).

* **Impossible d’arrêter proprement** :

  * Le processus PID1 ignore SIGTERM → utiliser `--init` ou corriger l’image.
  * Augmenter timeout : `docker stop -t 30`.

* **Fichiers root dans volume** :

  * Démarrer avec `-u 1000:1000` ou fixer permissions côté host.

---

## 7) Bonnes pratiques

* Conteneurs **éphémères** ; persistance via **volumes**.
* Ne pas utiliser `docker commit` ; préférer un **Dockerfile**.
* **Tags précis** (éviter `latest` en prod) ; utiliser **digests** pour l’immutabilité.
* Mettre des **healthchecks** exploitables par l’orchestrateur.
* Activer **log rotation** (`--log-driver local` ou options `json-file`).
* **Principe du moindre privilège** : `--cap-drop ALL`, `-u non-root`, `--read-only`, `tmpfs`.
* Limiter les ressources (`--cpus`, `--memory`) pour éviter l’overcommit.
* Isoler via **réseaux utilisateurs** et noms DNS locaux (network aliases).

---

## 8) TP guidé (corrigé attendu)

1. Démarrage d’un NGINX exposé localement et diagnostic :

```bash
docker run -d --name web -p 8080:80 nginx:1.25
curl -I http://localhost:8080
docker logs web --tail=20
docker inspect web --format '{{.NetworkSettings.IPAddress}}'
docker stop web && docker rm web
```

Résultats attendus : `HTTP/1.1 200 OK`, logs d’accès, IP bridge.

2. Application stateful :

```bash
docker volume create data
docker run -d --name db -p 5432:5432 \
  -e POSTGRES_PASSWORD=pass \
  --mount type=volume,src=data,dst=/var/lib/postgresql/data \
  postgres:16
docker exec -it db psql -U postgres -c "select 1;"
docker stop db && docker start db     # vérifier persistance
```

3. Conteneur durci :

```bash
docker run -d --name hard \
  --read-only --tmpfs /run --tmpfs /tmp \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  -p 8081:8080 myorg/myapp:1.0
docker exec -it hard sh -c 'touch /etc/test'  # doit échouer (read-only)
```

4. Nettoyage :

```bash
docker ps -a
docker container prune -f
docker volume ls
```

---

## 9) Référence rapide (cheatsheet)

* Lister : `docker ps [-a] [-f ...]`, `docker container ls`
* Créer/démarrer : `docker create`, `docker start`, `docker run`
* Arrêter/Tuer : `docker stop [-t N]`, `docker kill [--signal=...]`
* Logs/Exec : `docker logs [-f --tail=...]`, `docker exec [-it]`
* Infos : `docker inspect`, `docker top`, `docker stats`, `docker port`, `docker diff`
* Réseau : `docker network create|ls|inspect|connect|disconnect`
* Volumes : `docker volume create|ls|inspect|rm`
* Limites : `--cpus`, `--memory`, `--pids-limit`, `--ulimit`
* Sécurité : `--cap-drop/--cap-add`, `--security-opt`, `--read-only`, `-u`
* Santé : `--health-*`
* Nettoyage : `docker container prune`, `docker system df`, `docker system prune`


Souhaites-tu que je prépare **CHAPITRE-03 — Réseaux Docker** (ponts, `iptables`, DNS interne, multi-réseaux, isolation, patterns de prod) ou **CHAPITRE-03 — Volumes & persistance** avec le même niveau de détail ?

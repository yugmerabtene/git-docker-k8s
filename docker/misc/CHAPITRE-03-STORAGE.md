# CHAPITRE-03 — STORAGE (Volumes, Bind Mounts, Tmpfs, Drivers, Sauvegardes)

## 0) Objectifs et prérequis

**Objectifs**

* Comprendre le modèle de stockage Docker (couche writable, overlay2) et ses limites.
* Maîtriser les trois types de montages: **bind mounts**, **volumes nommés**, **tmpfs**.
* Savoir créer, monter, inspecter, sauvegarder, restaurer et nettoyer les données persistantes.
* Connaître les options avancées: SELinux `:Z/:z`, propagation, NFS/CIFS, quotas, sécurité.

**Prérequis**

* Docker Engine/CLI ≥ 20.10 (ou Docker Desktop).
* Notions de base Linux (permissions, UID/GID, montage de FS).
* Droits administrateur (sudo).

---

## 1) Rappels d’architecture: image, couche writable et driver

* **Image:** couches read-only content-addressées (SHA256).
* **Conteneur:** ajoute une **couche writable** au sommet (copy-on-write).
* **Driver de stockage:** généralement **overlay2** (Linux). Il gère la couche writable et le montage des couches read-only.
* **Limite:** la couche writable d’un conteneur est **éphémère** (supprimée avec le conteneur). Toute donnée à conserver doit être sur un **montage**: volume nommé, bind mount, ou tmpfs (éphémère RAM).

---

## 2) Les trois types de montages

### 2.1 Bind mounts (chemin hôte → chemin conteneur)

* Monte un **répertoire/fichier de l’hôte** dans le conteneur.
* Rapide et pratique en **développement** (hot-reload).
* Dépend des permissions et du FS de l’hôte (UID/GID, ACL, SELinux).

**Syntaxes**

```bash
# ancien -v
docker run -v /host/path:/container/path IMAGE

# moderne --mount
docker run --mount type=bind,src=/host/path,dst=/container/path,ro IMAGE
```

**Options courantes**

* `ro` (lecture seule) pour réduire la surface d’attaque.
* SELinux (bind mount uniquement) : `:Z` (relabel privé), `:z` (partagé). Ex:

  ```bash
  docker run -v /data:/data:Z nginx
  ```
* Propagation (cas avancés de montages imbriqués) : `rprivate` (défaut), `rshared`, `rslave`.

  ```bash
  docker run --mount 'type=bind,src=/mnt,dst=/mnt,bind-propagation=rshared' IMAGE
  ```

**Avantages / Inconvénients**

* * Simple, pas de gestion Docker supplémentaire.
* − Couplage fort à l’hôte, risques de permissions, dépend des chemins exacts.

---

### 2.2 Volumes nommés (gérés par Docker)

* Stockage **géré par Docker** (répertoire sous `/var/lib/docker/volumes/...` pour driver local).
* Idéal en **production**: portable entre conteneurs, décorrélé de l’arborescence hôte.
* Pré-population: au **premier montage** d’un **volume nommé** sur un chemin qui contient des fichiers dans l’image, Docker **copie** le contenu initial dans le volume (utile pour bases de config par défaut).

**Commandes de base**

```bash
docker volume create appdata
docker volume ls
docker volume inspect appdata
docker volume rm appdata
docker volume prune   # supprime volumes non utilisés par un conteneur
```

**Montage**

```bash
# -v
docker run -d --name db -v appdata:/var/lib/postgresql/data postgres:16

# --mount (recommandé pour la clarté)
docker run -d --name db \
  --mount type=volume,src=appdata,dst=/var/lib/postgresql/data \
  postgres:16
```

**Avantages / Inconvénients**

* * Découplage de l’arborescence hôte, facilité d’inspection via Docker.
* * Compatible avec **plugins de volumes** (NetApp, Portworx, etc.).
* − Moins trivial à éditer directement via le FS hôte (mais possible).

---

### 2.3 Tmpfs (mémoire volatile)

* Monte un répertoire **en RAM** dans le conteneur. Éphémère, très rapide.
* Idéal pour **fichiers temporaires**, caches, sockets runtime, sans persistance.

**Exemples**

```bash
# -v historico: type=tmpfs
docker run --tmpfs /tmp:rw,size=64m,mode=1777 alpine

# --mount
docker run --mount type=tmpfs,dst=/run,tmpfs-size=32m,tmpfs-mode=1777 alpine
```

---

## 3) Comparatif rapide

| Type       | Persistance | Perf                 | Sécurité                         | Cas d’usage                          |
| ---------- | ----------- | -------------------- | -------------------------------- | ------------------------------------ |
| Bind mount | Oui         | Très bon (direct FS) | Dépend du FS hôte (SELinux, ACL) | Dev, hot-reload, fichiers hôte       |
| Volume     | Oui         | Bon                  | Isolé, géré Docker, plugins      | Prod, DB, données applicatives       |
| Tmpfs      | Non (RAM)   | Excellent            | Volatile                         | Caches, secrets temporaires, runtime |

---

## 4) Cas pratiques essentiels

### 4.1 PostgreSQL avec volume nommé

```bash
docker volume create pgdata
docker run -d --name pg -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  --mount type=volume,src=pgdata,dst=/var/lib/postgresql/data \
  --health-cmd="pg_isready -U postgres || exit 1" \
  --health-interval=10s --health-retries=5 \
  postgres:16
```

### 4.2 NGINX static avec bind mount en lecture seule

```bash
docker run -d --name web -p 8080:80 \
  --mount type=bind,src=/srv/www,dst=/usr/share/nginx/html,ro \
  nginx:1.25
```

### 4.3 Application durcie: rootfs read-only + tmpfs pour le runtime

```bash
docker run -d --name app -p 8080:8080 \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  --mount type=volume,src=appdata,dst=/var/lib/app \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  -u 1000:1000 \
  myorg/myapp:1.0
```

---

## 5) Volumes avancés

### 5.1 Volumes “local” avec `--opt`

Le driver **local** supporte des options utiles:

* Simuler un bind via volume:

  ```bash
  docker volume create \
    --driver local \
    --opt type=none \
    --opt device=/data/apps \
    --opt o=bind \
    appdata
  ```
* NFS (souvent préférable de monter NFS sur l’hôte puis bind-mount):

  ```bash
  docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=10.0.0.5,nolock,soft,rw,vers=4 \
    --opt device=:/exports/prod \
    nfsdata
  ```
* CIFS/SMB (selon noyau, paquets):

  ```bash
  docker volume create \
    --driver local \
    --opt type=cifs \
    --opt device=//fileserver/share \
    --opt o=username=user,password=pass,uid=1000,gid=1000,file_mode=0644,dir_mode=0755 \
    smbdata
  ```

> Remarque: selon les distributions, certains `type=` et `o=` requièrent modules/paquets. En pratique prod, **monter le FS sur l’hôte** (systemd fstab) puis **bind-mount** est souvent plus maîtrisable.

### 5.2 Plugins de volumes

* Fournisseurs: Portworx, NetApp Trident, RexRay, etc.
* Permettent snapshots, réplication, chiffrement, QoS, haute dispo.
* Installation et options spécifiques au plugin.

### 5.3 Pré-population et masquage

* **Pré-population (volumes nommés uniquement)**: au **premier montage** sur un chemin contenant des fichiers dans l’image, Docker copie ces fichiers dans le volume.
* **Bind mount masque** toujours le contenu du chemin dans l’image.
  Exemple: si l’image a `/usr/share/nginx/html/index.html`, un bind mount sur `/usr/share/nginx/html` **cache** ce contenu.

---

## 6) Permissions, UID/GID, SELinux, Windows/WSL

### 6.1 UID/GID

* Les fichiers écrits dans un volume prennent l’UID/GID **du processus dans le conteneur**.
* Solutions si “Permission denied”:

  * Démarrer avec un utilisateur adéquat: `-u 1000:1000`.
  * `chown` côté hôte sur le répertoire bind-mounté.
  * Init script dans l’image pour `chown` au démarrage (avec prudence).

### 6.2 SELinux (RHEL/CentOS/Fedora)

* Sur bind mounts, utiliser `:Z` (relabel privé) ou `:z` (partagé) à la fin de `-v host:ctr:Z`.
* `:Z` est le plus courant (confinement strict).

### 6.3 Windows / Docker Desktop / WSL2

* Linux containers via WSL2: chemins Windows mappés sous `/mnt/c/...`.
* Bind mounts:

  ```bash
  docker run -v C:\data:C:\data mcr.microsoft.com/windows/nanoserver:1809   # conteneurs Windows
  docker run -v /mnt/c/data:/data alpine                                    # conteneurs Linux via WSL2
  ```
* Performance: éviter d’innombrables petits fichiers via bind mounts Windows↔Linux; préférer volumes nommés dans WSL2 pour de meilleures perfs.

---

## 7) Sauvegarde et restauration de volumes

### 7.1 Sauvegarder un volume nommé en tar

```bash
# Sauvegarder 'pgdata' dans backup.tar.gz du dossier courant
docker run --rm \
  -v pgdata:/data:ro \
  -v "$PWD":/backup \
  alpine sh -c "cd /data && tar -czf /backup/pgdata_$(date +%F).tar.gz ."
```

### 7.2 Restaurer un volume nommé

```bash
# Créer le volume si besoin
docker volume create pgdata

# Restaurer
docker run --rm \
  -v pgdata:/data \
  -v "$PWD":/backup \
  alpine sh -c "cd /data && tar -xzf /backup/pgdata_2025-11-01.tar.gz"
```

### 7.3 Copier depuis/vers conteneur

```bash
docker cp CONTAINER:/path/in/container ./localdir
docker cp ./localfile CONTAINER:/path/in/container
```

> Pour des gros volumes, préférer la méthode tar via `docker run` avec un conteneur utilitaire (alpine/busybox) comme ci-dessus.

---

## 8) Compose: volumes déclaratifs

**compose.yaml (extraits)**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    # ou bind mount:
    # - ./dbdata:/var/lib/postgresql/data

  web:
    image: nginx:1.25
    ports: ["8080:80"]
    volumes:
      - ./site:/usr/share/nginx/html:ro

volumes:
  pgdata:
    driver: local
    # driver_opts:
    #   type: none
    #   device: /data/pg
    #   o: bind
```

Commandes:

```bash
docker compose up -d
docker compose down           # ne supprime pas les volumes par défaut
docker compose down -v        # supprime aussi les volumes
docker volume ls
docker volume inspect <name>
```

---

## 9) Quotas, limites d’espace, chiffrement

* **Quotas direct via Docker volume (driver local)**: non géré nativement pour limiter la taille.
  Stratégies:

  * **FS sous-jacent** avec quotas (XFS project quotas), LVM, ZFS/Btrfs avec quotas natifs.
  * Volumes fournis par des **plugins** supportant quotas.
* **Tmpfs** supporte `size=`:

  ```bash
  docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=64m alpine
  ```
* **Chiffrement au repos**:

  * Couche hôte: LUKS/LVM chiffré, ZFS natif, eCryptfs.
  * Plug-ins de stockage supportant le chiffrement.

---

## 10) Nettoyage et diagnostic

* Utilisation disque:

  ```bash
  docker system df       # résumé images/containers/volumes
  docker system df -v    # détails
  ```
* Nettoyage volumes non utilisés:

  ```bash
  docker volume prune
  ```
* Inspection d’un volume:

  ```bash
  docker volume inspect appdata
  # "Mountpoint" indique le chemin réel sur l’hôte (driver local)
  ```
* Diff du FS d’un conteneur (couche writable, pas volumes):

  ```bash
  docker diff CONTAINER
  ```

---

## 11) Sécurité et bonnes pratiques

* Monter les données en **lecture seule** si possible: `ro`.
* Utiliser `--read-only` + **tmpfs** pour répertoires d’écriture éphémère (`/run`, `/tmp`).
* Exécuter en **utilisateur non-root**: `-u 1000:1000` et aligner les permissions.
* En SELinux, ajouter `:Z` sur bind mounts.
* Éviter de monter des chemins sensibles de l’hôte (`/`, `/etc`, `/var/run/docker.sock`) sauf cas contrôlé.
* Pour bases de données:

  * Utiliser **volumes nommés** dédiés.
  * Sauvegardes régulières via **dump logique** (pg_dump, mysqldump) ou snapshot FS (selon plugin/FS).
* Documenter un **plan de sauvegarde/restauration** (tests réguliers).

---

## 12) Exercices (TP) avec résultats attendus

### TP-1: Bind mount NGINX en lecture seule

1. Créer un dossier hôte:

   ```bash
   sudo mkdir -p /srv/www && echo "OK" | sudo tee /srv/www/index.html
   ```
2. Lancer:

   ```bash
   docker run -d --name web -p 8080:80 \
     --mount type=bind,src=/srv/www,dst=/usr/share/nginx/html,ro \
     nginx:1.25
   ```
3. Vérifier:

   ```bash
   curl -sI http://localhost:8080 | grep "200"
   ```

   Attendu: `HTTP/1.1 200 OK`

### TP-2: PostgreSQL + volume nommé + persistance

1. Créer volume:

   ```bash
   docker volume create pgdata
   ```
2. Lancer DB:

   ```bash
   docker run -d --name pg -p 5432:5432 \
     -e POSTGRES_PASSWORD=secret \
     --mount type=volume,src=pgdata,dst=/var/lib/postgresql/data \
     postgres:16
   ```
3. Insérer une donnée, redémarrer et vérifier la persistance.
   Attendu: les données survivent au `docker restart pg`.

### TP-3: Sauvegarde et restauration

1. Sauvegarder pgdata:

   ```bash
   docker run --rm -v pgdata:/data:ro -v "$PWD":/backup \
     alpine sh -c "cd /data && tar -czf /backup/pgdata.tar.gz ."
   ```
2. Restaurer dans un nouveau volume `pgdata2` et monter sur un nouveau conteneur:

   ```bash
   docker volume create pgdata2
   docker run --rm -v pgdata2:/data -v "$PWD":/backup \
     alpine sh -c "cd /data && tar -xzf /backup/pgdata.tar.gz"
   ```

   Attendu: même contenu restauré.

### TP-4: Rootfs read-only + tmpfs

1. Démarrer un conteneur applicatif:

   ```bash
   docker run -d --name app \
     --read-only \
     --tmpfs /run --tmpfs /tmp \
     -p 8081:8080 myorg/myapp:1.0
   ```
2. Tenter une écriture sous `/` dans le conteneur:

   ```bash
   docker exec app sh -c 'touch /root/deny || echo "WRITE DENIED"'
   ```

   Attendu: échec d’écriture (rootfs read-only).

---

## 13) Référence rapide (cheatsheet)

* Créer/lister/inspecter/supprimer volumes:

  ```bash
  docker volume create NAME
  docker volume ls
  docker volume inspect NAME
  docker volume rm NAME
  docker volume prune
  ```
* Monter un **volume nommé**:

  ```bash
  docker run --mount type=volume,src=NAME,dst=/path IMAGE
  ```
* Monter un **bind**:

  ```bash
  docker run --mount type=bind,src=/host,dst=/ctr,ro IMAGE
  ```
* **tmpfs**:

  ```bash
  docker run --mount type=tmpfs,dst=/run,tmpfs-size=64m IMAGE
  ```
* Sauvegarde/restauration:

  ```bash
  docker run --rm -v VOL:/data:ro -v "$PWD":/bkp alpine sh -c "cd /data && tar -czf /bkp/vol.tgz ."
  docker run --rm -v VOL:/data -v "$PWD":/bkp alpine sh -c "cd /data && tar -xzf /bkp/vol.tgz"
  ```


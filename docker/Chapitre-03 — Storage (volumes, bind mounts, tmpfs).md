# Chapitre-03 — Storage (volumes, bind mounts, tmpfs)

## Objectifs d’apprentissage

* Choisir et configurer le **bon type de montage** (bind, volume, tmpfs) selon le besoin.
* Maîtriser les **permissions** (UID/GID), **SELinux** (`:Z/:z`) et les options de montage (`ro`, propagation).
* Savoir **sauvegarder/restaurer/migrer** des données de conteneurs proprement.
* Durcir l’exécution avec **rootfs en lecture seule** (`--read-only`) et points d’écriture contrôlés.

## Pré-requis

* Docker Engine/CLI opérationnel, bases Linux (permissions, chemins).
* Notions réseau si NFS/SMB sont utilisés.

---

## 1) Panorama : bind vs volume vs tmpfs

| Type       | Déclaration (exemples)                                                            | Cas d’usage typiques                                  | Avantages                              | Points d’attention                         |
| ---------- | --------------------------------------------------------------------------------- | ----------------------------------------------------- | -------------------------------------- | ------------------------------------------ |
| **Bind**   | `-v /host/path:/ctr/path[:opts]` ou `--mount type=bind,src=/host,dst=/ctr[:opts]` | Dev, config partagée, logs, fichiers hôte ↔ conteneur | Simple, direct                         | Permissions hôte, masquage, sécurité       |
| **Volume** | `-v volname:/ctr/path` ou `--mount type=volume,src=volname,dst=/ctr`              | Données applicatives (DB, blobs)                      | Isolé de l’arbo hôte, lifecycle Docker | Visible via Docker, pas de chemin “humain” |
| **tmpfs**  | `--tmpfs /ctr/path[:opts]` ou `--mount type=tmpfs,dst=/ctr/path,tmpfs-size=64m`   | Caches, fichiers temporaires sensibles/perf           | En RAM, rapide, disparaît à l’arrêt    | Volatile, limite mémoire                   |

**Règle d’or :**

* **Volume** pour **données persistantes** (DB, state applicatif).
* **Bind** pour **fichiers du projet** (dev) ou dépôts de **configs/logs**.
* **tmpfs** pour **temporaire** et **performances** (cache, `/tmp`, `/run`).

---

## 2) Syntaxe `-v` vs `--mount`

* **Court (`-v`)** : historique, compact, pratique.
  `-v SRC:DST[:OPTIONS]`
* **Verbeux (`--mount`)** : lisible, options explicites, recommandé en prod.
  `--mount type=bind,source=/src,destination=/dst,ro,bind-propagation=rshared`

**Synonymes** : `source` = `src`, `destination` = `dst` = `target`.

---

## 3) Volumes nommés (cycle de vie complet)

### 3.1 Créer / lister / inspecter / supprimer

```bash
docker volume create --label app=db data_pg
docker volume ls --filter label=app=db
docker volume inspect data_pg
docker volume rm data_pg               # échoue si utilisé
docker volume prune -f                 # supprime volumes orphelins
```

### 3.2 Monter un volume

```bash
# Syntaxe courte
docker run -d --name pg -v data_pg:/var/lib/postgresql/data postgres:16

# Syntaxe --mount
docker run -d --name pg \
  --mount type=volume,src=data_pg,dst=/var/lib/postgresql/data \
  postgres:16
```

### 3.3 Pré-population d’un volume

Un volume **vide** monté sur un chemin contenant des fichiers **copie** le contenu de l’image **dans le volume** au premier montage.

```bash
# Exemple : si l'image contient /app/defaults
docker run --rm -v myvol:/app ghcr.io/acme/app:1.0
# myvol est désormais pré-populé avec /app depuis l'image
```

### 3.4 Initialiser depuis un répertoire local

```bash
# Copie /host/seed → volume data_app via un conteneur utilitaire
docker run --rm \
  -v data_app:/dest \
  -v $PWD/seed:/seed:ro \
  alpine sh -c "cp -a /seed/. /dest/"
```

---

## 4) Bind mounts (montages hôte ↔ conteneur)

### 4.1 Base et options

```bash
# Lecture/écriture (par défaut)
docker run -v $PWD/config:/app/config ghcr.io/acme/app:1.0

# Lecture seule
docker run -v $PWD/config:/app/config:ro ghcr.io/acme/app:1.0

# Version --mount équivalente
docker run --mount type=bind,src=$PWD/config,dst=/app/config,ro ghcr.io/acme/app:1.0
```

### 4.2 Propagation (bind-propagation)

* **Par défaut** : `rprivate` (pas de propagation).
* Pour propager des montages enfants (cas avancés) :

```bash
docker run --mount \
  type=bind,src=/mnt/parent,dst=/mnt/parent,bind-propagation=rshared \
  ghcr.io/acme/app:1.0
```

> Le point **hôte** doit lui-même être monté en **rshared** (côté système). Cas rares hors orchestrateur.

### 4.3 Masquage (gotcha)

Un bind sur `/ctr/path` **masque** ce que l’image avait à cet emplacement.
→ **Vérifier** que le conteneur ne dépend pas de fichiers de l’image à ce chemin.

---

## 5) tmpfs (mémoire)

### 5.1 Montage tmpfs simple

```bash
docker run --tmpfs /tmp -d ghcr.io/acme/app:1.0
```

### 5.2 Taille et mode

```bash
docker run --mount type=tmpfs,dst=/run,tmpfs-size=64m,tmpfs-mode=1777 \
  -d ghcr.io/acme/app:1.0
```

* **Volatile** : disparaît à l’arrêt.
* **Idéal** pour caches/temp (rapide, pas sur disque).

---

## 6) Permissions & UID/GID

### 6.1 Alignement hôte ↔ conteneur

Le noyau applique les **UID/GID** : si l’app tourne en `10001:10001`, l’hôte doit **autoriser** ces UID/GID.

```bash
# Exécuter l'app en user non-root
docker run -u 10001:10001 -v data_app:/data ghcr.io/acme/app:1.0
```

### 6.2 Ajuster l’ownership

```bash
# Initialiser le volume avec le bon owner
docker run --rm -v data_app:/data alpine chown -R 10001:10001 /data
```

### 6.3 userns-remap (aperçu)

La remap d’UID/GID hôte/ctr change la vue des IDs. À planifier **avant** l’exploitation (impacts sur permissions et sauvegardes).

---

## 7) SELinux (`:Z` / `:z`) sur Fedora/CentOS/RHEL

* `:Z` : étiquette **privée** (confinée à ce conteneur).
* `:z` : étiquette **partagée** (plusieurs conteneurs y accèdent).

```bash
docker run -v /srv/data:/var/lib/mysql:Z -d mysql:8
```

> Sur distributions **sans SELinux** (ex. Ubuntu), ces suffixes sont ignorés.
> Avec **AppArmor**, gérer plutôt des profils/abstractions (pas d’option `:Z`).

---

## 8) Réseaux de fichiers : NFS & SMB (via driver `local`)

### 8.1 NFS (exemple)

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.50,nfsvers=4.1,rw \
  --opt device=:/export/data \
  nfs_data

docker run -d --name app \
  --mount type=volume,src=nfs_data,dst=/data \
  ghcr.io/acme/app:1.0
```

### 8.2 SMB/CIFS (exemple)

```bash
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt device=//10.0.0.60/share \
  --opt o=username=user,password=secret,uid=10001,gid=10001 \
  smb_data
```

> Performance & permissions dépendent du serveur distant ; privilégier **volumes** locaux pour DBs critiques (latence/fiabilité).

---

## 9) Rootfs en lecture seule (`--read-only`) + points d’écriture

### 9.1 Patron sécurisé

```bash
docker run -d --name web \
  --read-only \
  --tmpfs /tmp --tmpfs /run \
  -v data_web:/var/lib/app \
  ghcr.io/acme/web:1.4.2
```

* Le rootfs devient **RO** ; seuls **tmpfs** et **volumes** restent écrits.
* Identifier **tous** les chemins que l’app veut écrire (`/tmp`, `/run`, `/var/lib/app`…).

### 9.2 Variante bind RO + volume RW

```bash
docker run -d \
  --read-only \
  -v $PWD/config:/app/config:ro \
  -v data_app:/app/data \
  ghcr.io/acme/app:1.0
```

---

## 10) Sauvegarde / Restauration de volumes

### 10.1 Sauvegarder un volume (tar)

```bash
# Archive .tar.gz du volume data_app
docker run --rm -v data_app:/data -v $PWD:/backup \
  alpine sh -c "cd /data && tar czf /backup/data_app_$(date +%F).tgz ."
```

### 10.2 Restaurer un volume

```bash
docker run --rm -v data_app:/data -v $PWD:/backup \
  alpine sh -c "cd /data && tar xzf /backup/data_app_2025-11-01.tgz"
```

### 10.3 Migration volume → volume

```bash
# Copier data_app → data_new via un conteneur utilitaire
docker run --rm -v data_app:/src -v data_new:/dst alpine sh -c "cp -a /src/. /dst/"
```

> **DBs** : préférer les **outils natifs** (ex. `pg_dump`, `mysqldump`) pour cohérence logique.

---

## 11) Compose : déclarer volumes/bind/tmpfs

### 11.1 Volumes nommés

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - data_pg:/var/lib/postgresql/data

volumes:
  data_pg: {}
```

### 11.2 Bind mounts

```yaml
services:
  web:
    image: ghcr.io/acme/web:1.4.2
    volumes:
      - ./config:/app/config:ro
```

### 11.3 tmpfs & rootfs RO

```yaml
services:
  app:
    image: ghcr.io/acme/app:1.0
    read_only: true
    tmpfs:
      - /run
      - /tmp:size=64m,mode=1777
    volumes:
      - data_app:/var/lib/app
volumes:
  data_app: {}
```

---

## 12) Diagnostic & bonnes pratiques

### 12.1 Vérifier les montages effectifs

```bash
docker inspect -f '{{json .Mounts}}' app | jq
```

### 12.2 Pièges courants

* **Masquage** d’un chemin important par un bind → l’app ne trouve plus ses fichiers d’image.
* **Permissions** : UID/GID non alignés → `EACCES`.
* **SELinux** : oubli de `:Z/:z` sur Fedora/CentOS/RHEL → `permission denied`.
* **Net FS** : NFS/SMB non disponibles → montée en erreur au démarrage.
* **Nettoyage agressif** : `docker volume prune` supprime un volume **non référencé** (attention au Compose down avec `-v`).

### 12.3 Do & Don’t

**Do**

* Préférer **volumes** pour la **persistance** applicative.
* Définir des **UID/GID** cohérents, initialiser l’ownership.
* Utiliser `--read-only` + `tmpfs` pour durcir.
* Sauvegarder via **tar** ou **outils natifs DB**, tester la **restauration**.

**Don’t**

* Éviter de monter tout `/` : `-v /:/host` (dangereux).
* Ne stockez pas de secrets sur volume partagé sans contrôle.
* N’exposez pas des sockets/chemins sensibles au conteneur inutilement.

---

## 13) Aide-mémoire (cheat-sheet)

```bash
# Volumes
docker volume create --label app=db data_pg
docker volume ls --filter label=app=db
docker volume inspect data_pg
docker volume rm data_pg
docker volume prune -f

# Bind
docker run -v $PWD/config:/app/config:ro ghcr.io/acme/app:1.0

# tmpfs
docker run --mount type=tmpfs,dst=/run,tmpfs-size=64m ghcr.io/acme/app:1.0

# SELinux
docker run -v /srv/data:/data:Z ghcr.io/acme/app:1.0

# NFS
docker volume create --driver local \
  --opt type=nfs --opt o=addr=10.0.0.50,nfsvers=4.1,rw \
  --opt device=:/export/data nfs_data

# Backup volume
docker run --rm -v data_app:/data -v $PWD:/backup \
  alpine sh -c "cd /data && tar czf /backup/data_app_$(date +%F).tgz ."

# Inspecter les montages d’un conteneur
docker inspect -f '{{json .Mounts}}' app | jq
```

---

## 14) Checklist de clôture (qualité du stockage)

* Type de montage **adapté** (volume/bind/tmpfs) et **documenté**.
* **Permissions** correctes (UID/GID alignés), SELinux `:Z/:z` si nécessaire.
* Rootfs **read-only** quand possible, **tmpfs** pour `/tmp` et `/run`.
* **Sauvegardes** testées + plan de **restauration** documenté.
* Éviter le **masquage** involontaire de chemins critiques.
* Compose : volumes nommés déclarés, pas de `-v` absolu non maîtrisé.

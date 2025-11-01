# CHAPITRE-04 — RÉSEAU DOCKER (théorie + pratique approfondie)

## 0) Objectifs & prérequis

**Objectifs**

* Comprendre l’architecture réseau Docker (NAT, bridge, DNS interne).
* Maîtriser `docker run -p`, `docker network *`, les drivers (`bridge`, `host`, `none`, `macvlan`, `ipvlan`).
* Savoir créer des sous-réseaux, attribuer des IP statiques, utiliser des alias DNS, connecter/déconnecter des conteneurs.
* Diagnostiquer (ports, iptables/nftables, hairpin NAT, conflits) et appliquer des bonnes pratiques de segmentation.

**Prérequis**

* Docker Engine/CLI ≥ 20.10.
* Notions réseau (IP, CIDR, routes, NAT, DNS).
* Accès root/administrateur.

---

## 1) Concepts fondamentaux

* **Bridge par défaut (`docker0`)** : un switch virtuel Linux + NAT vers l’hôte (iptables MASQUERADE). Les conteneurs reçoivent des IP privées (ex. `172.17.0.0/16`).
* **User-defined bridge** : réseau bridge créé par l’utilisateur (isolation par défaut entre réseaux distincts, **DNS interne** avec noms/aliases).
* **Port publishing (`-p`/`-P`)** : règle DNAT (iptables) pour exposer un port conteneur sur l’hôte.
* **Service discovery interne** : chaque réseau user-defined a son **DNS embarqué** (nom du service = nom du conteneur ou alias).
* **Plusieurs réseaux** : un conteneur peut être membre de **N** réseaux (front/back, DMZ).
* **Drivers spéciaux** :

  * `host` : partage la pile réseau de l’hôte (pas d’isolation, pas de `-p`).
  * `none` : aucun réseau.
  * `macvlan`/`ipvlan` : conteneur visible **directement** sur le LAN (IP L2/L3 dédiée).
  * (`overlay` pour multi-hôte via Swarm, hors scope single-node).

---

## 2) Commandes de base

```bash
docker network ls
docker network inspect <network>
docker network create [OPTIONS] <name>
docker network rm <name>
docker network connect <network> <container>
docker network disconnect <network> <container>
```

**Création de réseaux (bridge user-defined)** :

```bash
# Réseau bridge avec plage dédiée + gateway + IPAM
docker network create \
  --driver bridge \
  --subnet 172.18.0.0/16 \
  --gateway 172.18.0.1 \
  --ip-range 172.18.5.0/24 \
  netapp
```

> Avantages d’un **user-defined bridge** : DNS interne, isolation inter-réseaux, IPAM propre, noms/aliases stables.

---

## 3) Lancer des conteneurs avec réseau

### 3.1 Bridge par défaut vs réseau dédié

```bash
# Bridge par défaut (docker0)
docker run -d --name web nginx

# Réseau dédié (recommandé)
docker network create appnet
docker run -d --name web --network appnet nginx
docker run -d --name api --network appnet myorg/api:1.0
```

* Dans `appnet`, `web` peut résoudre `api` via DNS → `http://api:PORT`.

### 3.2 Alias DNS & IP statique

```bash
# Alias DNS sur le réseau
docker run -d --name backend --network appnet --network-alias api myorg/api:1.0

# IP statique (nécessite --subnet à la création du réseau)
docker run -d --name db --network appnet --ip 172.18.5.10 postgres:16
```

### 3.3 Se connecter à plusieurs réseaux

```bash
docker run -d --name gateway --network frontnet myorg/gw:1.0
docker network connect backnet gateway
# 'gateway' a maintenant 2 interfaces (frontnet + backnet)
```

---

## 4) Publication de ports (DNAT) : `-p` et `-P`

**Syntaxes :**

```bash
# hostPort:containerPort (TCP par défaut)
docker run -d -p 8080:80 nginx

# Spécifier protocole
docker run -d -p 514:514/udp rsyslog:latest

# Lier à une IP d’écoute spécifique (loopback)
docker run -d -p 127.0.0.1:8080:80 nginx

# Publier automatiquement tous les EXPOSE (ports) de l'image
docker run -d -P myapp
```

**Points clés :**

* `-p` crée des règles `iptables` (DNAT vers l’IP conteneur).
* Conflits de port hôte → “address already in use”.
* Inside container, **bind sur 0.0.0.0** est requis pour écouter sur l’IP conteneur (ne pas binder sur 127.0.0.1 dans le conteneur si vous voulez publier).

**Multi-ports :**

```bash
docker run -d \
  -p 80:80 -p 443:443/tcp -p 514:514/udp \
  --name reverse-proxy myorg/nginx:tls
```

---

## 5) Drivers réseau

### 5.1 `bridge` (par défaut / user-defined)

* NAT vers l’hôte
* DNS interne (user-defined)
* Isolation entre réseaux
* **Cas d’usage** : microservices sur même hôte, dev/prod simple

### 5.2 `host`

```bash
docker run --network host myapp
```

* Pas de NAT, **mêmes ports** que l’hôte, plus performant.
* Pas de `-p`, pas de DNS Docker.
* **Cas d’usage** : outils réseau, haute perf, ou services déjà gérant ports.

### 5.3 `none`

```bash
docker run --network none myapp
```

* Aucune interface (sauf loopback).
* **Cas** : jobs isolés, sécurité maximale, traitement offline.

### 5.4 `macvlan` (IP L2 sur le LAN)

```bash
# 1) Créer un réseau macvlan parent=NIC hôte reliée au LAN
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan_net

# 2) Donner une IP LAN au conteneur
docker run -d --name iot \
  --network macvlan_net \
  --ip 192.168.1.50 \
  myorg/collector:1.0
```

* Le conteneur apparaît comme **une machine du LAN** (nouvelle MAC & IP).
* **Attention** : par défaut, l’hôte **ne** peut pas joindre le conteneur macvlan (il faut un **macvlan “bridge” sur l’hôte** ou une route). Idem pour hairpin.

### 5.5 `ipvlan`

* Variante L3 (pas d’adresse MAC unique par conteneur).
* Utile dans environnements avec restrictions L2.

> **Note** : macvlan/ipvlan nécessitent compréhension L2/L3 de votre infra (switchs, VLANs, routes). Idéal pour donner des IP “réelles” aux conteneurs.

---

## 6) DNS interne & service discovery

* Sur un **user-defined bridge**, Docker fournit un **DNS** qui résout :

  * nom du conteneur (`--name`)
  * `--network-alias`
* Exemple :

  ```bash
  docker network create appnet
  docker run -d --name db --network appnet postgres:16
  docker run --rm --network appnet alpine ping -c1 db
  ```
* Résolution via `/etc/resolv.conf` dans le conteneur pointant vers DNS Docker.

---

## 7) IPAM & options avancées (bridge)

**Créer un réseau avec IPAM personnalisé :**

```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/16 \
  --ip-range 10.10.10.0/24 \
  --gateway 10.10.0.1 \
  appnet
```

**Attribution d’IP fixe :**

```bash
docker run -d --network appnet --ip 10.10.10.25 --name cache redis:7
```

**Options bridge utiles :**

* `--internal` : réseau **sans sortie** vers l’hôte/Internet (pas de NAT).

  ```bash
  docker network create --driver bridge --internal backnet
  ```
* `--attachable` (surtout overlay/Swarm) : attacher conteneurs non gérés par le cluster.

---

## 8) Diagnostics, outils & dépannage

### 8.1 Informations & états

```bash
docker ps
docker network ls
docker network inspect appnet
docker inspect <container> --format '{{json .NetworkSettings.Networks}}' | jq
docker port <container>
```

### 8.2 Tests applicatifs

```bash
# Dans le même réseau que l’app :
docker run --rm -it --network appnet alpine sh
apk add --no-cache curl bind-tools
curl -I http://web:80
dig web
```

### 8.3 Ports & conflits (hôte)

* Linux : `ss -ltnp | grep :8080` ou `ss -lunp | grep :514`
* Windows : `netstat -ano | findstr :8080`

### 8.4 iptables/nftables (Linux)

* DNAT créé par `-p` :

  ```bash
  sudo iptables -t nat -S | grep DOCKER
  sudo iptables -t nat -L -n -v
  ```
* Hairpin NAT (accès du conteneur à l’IP publique de l’hôte renvoyée vers lui-même) : activé par défaut sur user-defined bridge ; tester via `curl http://host_ip:8080`.

### 8.5 “Connection refused” – checklist

* Le service **écoute dans le conteneur** (`ss -ltnp` inside).
* **Bind 0.0.0.0** dans le conteneur, pas uniquement `127.0.0.1`.
* Le **port est bien publié** (`docker port`, `-p host:container`).
* Pas de **pare-feu** hôte bloquant (UFW/Firewalld/Windows Defender).
* Bon **protocole** (`/tcp` vs `/udp`).

### 8.6 Sniffer un trafic

* Sniffer dans le **namespace réseau** d’un conteneur :

  ```bash
  # Sniffer via un sidecar réseau-partagé
  docker run --rm --net=container:<container_id_or_name> nicolaka/netshoot tcpdump -i any -n port 80
  ```

---

## 9) Sécurité & bonnes pratiques

* **Segmentation** : séparer front/back via **réseaux différents** ; n’exposez pas la DB sur le réseau “front”.
* **Réseaux `--internal`** pour services qui ne doivent **pas sortir**.
* **Principe du moindre privilège** : seuls les services qui doivent publier des ports utilisent `-p`; tout le reste reste interne.
* Centraliser l’exposition via un **reverse-proxy** (Nginx/Traefik/HAProxy).
* **Logs & métriques** : exporter événements (drop, DNAT) côté hôte si nécessaire.
* **macvlan/ipvlan** : maîtriser l’impact L2/L3 (VLAN, DHCP, ARP).
* **Éviter `--network host`** sauf besoin explicite (diminue l’isolation).

---

## 10) Exemples complets

### 10.1 Front ↔ API sur réseau dédié + publication front

```bash
docker network create appnet

docker run -d --name api --network appnet myorg/api:1.0
docker run -d --name web --network appnet -p 8080:80 myorg/web:1.0

# Dans 'web', l’API est accessible via http://api:PORT
```

### 10.2 Reverse proxy unique exposé, services internes isolés

```bash
docker network create frontnet
docker network create --internal backnet

docker run -d --name api --network backnet myorg/api:2.0
docker run -d --name auth --network backnet myorg/auth:1.0

docker run -d --name proxy \
  --network frontnet \
  --network backnet \
  -p 80:80 -p 443:443 \
  myorg/nginx:tls
# NGINX route vers http://api et http://auth sur backnet
```

### 10.3 macvlan : donner une IP LAN au conteneur

```bash
docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=eth0 macvlan_net

docker run -d --name cam-collector \
  --network macvlan_net --ip 192.168.10.50 \
  myorg/collector:1.0
```

> Pour joindre `cam-collector` **depuis l’hôte**, créer une interface macvlan **aussi sur l’hôte** (ex. `ip link add macvlan0 link eth0 type macvlan mode bridge` + IP hôte dans le même subnet) et ajouter la route correspondante.

---

## 11) Docker Compose — Réseaux déclaratifs

**compose.yaml**

```yaml
version: "3.9"
services:
  web:
    image: nginx:1.25
    ports: ["8080:80"]
    networks:
      - front
      - back
    depends_on: [api]

  api:
    image: myorg/api:1.0
    networks:
      - back
    # alias DNS dans 'back'
    network_mode: # (ne pas définir ici si vous utilisez 'networks:')

networks:
  front:
    driver: bridge
  back:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.22.0.0/16
          ip_range: 172.22.5.0/24
          gateway: 172.22.0.1
```

**Commandes :**

```bash
docker compose up -d
docker compose ps
docker compose down
```

---

## 12) TP (exercices guidés)

### TP-1 : Réseau dédié + service discovery

1. Créer réseau :

   ```bash
   docker network create appnet
   ```
2. Lancer API + front :

   ```bash
   docker run -d --name api --network appnet hashicorp/http-echo -text="API OK"
   docker run -d --name web --network appnet -p 8080:80 nginx
   ```
3. Tester depuis un conteneur utilitaire :

   ```bash
   docker run --rm --network appnet alpine sh -c "apk add curl -q && curl -s api:5678"
   ```

   **Attendu** : `API OK`

### TP-2 : Réseau interne + reverse proxy

1. Réseaux :

   ```bash
   docker network create --internal backnet
   docker network create frontnet
   ```
2. Services internes :

   ```bash
   docker run -d --name svc1 --network backnet hashicorp/http-echo -text="SVC1"
   docker run -d --name svc2 --network backnet hashicorp/http-echo -text="SVC2"
   ```
3. NGINX multi-réseau :

   ```bash
   docker run -d --name proxy --network frontnet nginx
   docker network connect backnet proxy
   ```
4. Montez un `nginx.conf` custom via bind pour router `svc1`/`svc2`.
   **Attendu** : accès à `svc1`/`svc2` via `localhost:80`.

### TP-3 : IP statique & alias

1. Réseau IPAM :

   ```bash
   docker network create --subnet 10.200.0.0/16 appnet2
   ```
2. Lancer DB avec IP fixe :

   ```bash
   docker run -d --name db --network appnet2 --ip 10.200.5.10 postgres:16
   ```
3. Lancer app avec alias :

   ```bash
   docker run -d --name app --network appnet2 --network-alias database myorg/app:1.0
   ```
4. Résoudre `database` depuis `app` (via logs/test).
   **Attendu** : `database` → `10.200.5.10`.

---

## 13) Référence rapide (cheatsheet)

* Lister/inspecter réseaux :

  ```bash
  docker network ls
  docker network inspect <net>
  ```
* Créer réseau bridge :

  ```bash
  docker network create --driver bridge --subnet 172.20.0.0/16 appnet
  ```
* Lancer avec réseau + ports :

  ```bash
  docker run -d --name web --network appnet -p 8080:80 nginx
  ```
* Connexion / déconnexion :

  ```bash
  docker network connect appnet web
  docker network disconnect appnet web
  ```
* Alias / IP statique :

  ```bash
  docker run -d --network appnet --network-alias api myapp
  docker run -d --network appnet --ip 172.20.0.10 mydb
  ```
* Diagnostics :

  ```bash
  docker port <ctr> ; docker logs <ctr>
  ss -ltnp | grep :8080
  docker run --rm --net=container:<ctr> nicolaka/netshoot tcpdump -i any -n
  ```

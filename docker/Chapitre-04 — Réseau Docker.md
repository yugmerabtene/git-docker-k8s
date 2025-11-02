# Chapitre-04 — Réseau Docker

## Objectifs d’apprentissage

* Concevoir et administrer des réseaux Docker **user-defined** (isolation, DNS interne, IPAM).
* Maîtriser la **publication de ports** (`-p/-P`, TCP/UDP, bind d’IP), l’attachement **multi-réseaux**, et les **alias** DNS.
* Appliquer des **contrôles egress/ingress** simples (réseaux `--internal`, séparation front/back, règles pare-feu).
* Comprendre les **modes réseau** (`bridge`, `host`, `none`, notions `macvlan/ipvlan`, IPv6) et diagnostiquer les problèmes (NAT, MTU, hairpin).

## Pré-requis

* Docker Engine/CLI opérationnel.
* Notions IP/CIDR/ports/DNS/iptables.

---

## 1) Les drivers réseau Docker (panorama)

| Driver      | Usage principal                                            | Particularités                                                    |
| ----------- | ---------------------------------------------------------- | ----------------------------------------------------------------- |
| **bridge**  | Par défaut (Linux). Réseaux L2 isolés avec NAT vers l’hôte | DNS embarqué, IPAM configurable, port-mapping `-p`                |
| **host**    | Partage la pile réseau de l’hôte                           | Pas de `-p` (tout est “sur l’hôte”), risques de conflits de ports |
| **none**    | Isolation complète                                         | Pas d’accès réseau, pas de DNS interne                            |
| **macvlan** | Donner une IP **de votre LAN** au conteneur                | Besoin d’un plan d’adressage, mode promisc/commutateur, pièges L2 |
| **ipvlan**  | Variante L3/L2 plus simple que macvlan selon cas           | Moins “promisc” que macvlan, exigences routeur                    |

> Les **overlays** (réseaux distribués multi-hôtes) relèvent de **Swarm/K8s** (voir Chapitre 13).

---

## 2) Réseaux `bridge` user-defined : le standard quotidien

### 2.1 Créer / lister / inspecter

```bash
docker network create app-net
docker network ls
docker network inspect app-net | jq '.[0].IPAM, .[0].Containers, .[0].Internal'
```

* Les **user-defined bridges** offrent un **DNS interne** : les **noms de conteneurs** deviennent résolvables *sur le même réseau*.

### 2.2 Attacher / détacher un conteneur

```bash
docker run -d --name web --network app-net nginx:1.27
docker network connect app-net another-container
docker network disconnect app-net another-container
```

### 2.3 IPAM (adresses/IP statiques)

```bash
# Crée un réseau avec plan d’adressage précis
docker network create \
  --subnet 172.18.0.0/16 \
  --ip-range 172.18.5.0/24 \
  --gateway 172.18.0.1 app-net

# Attribuer une IP fixe au conteneur
docker run -d --name db \
  --network app-net --ip 172.18.5.10 \
  postgres:16
```

* `--subnet` et `--ip-range` doivent **ne pas** chevaucher d’autres réseaux.
* `aux-addresses` (via `--ipam-opt`) réserve des IPs (ex. pour le gateway virtuel).

### 2.4 Multi-réseaux (front/back)

```bash
docker network create frontend
docker network create backend --internal   # egress bloqué (pas d'accès Internet)

docker run -d --name api \
  --network frontend \
  --network backend \
  --network-alias api-svc \
  ghcr.io/acme/api:1.4.2
```

* Un conteneur peut être **sur plusieurs réseaux**.
* Un réseau `--internal` **n’a pas d’egress** : utile pour cacher une DB/queue.

---

## 3) DNS interne, alias, hosts & nommage

### 3.1 Résolution par nom de conteneur

* Sur un **user-defined bridge**, `ping db` (ou `curl http://db:5432`) fonctionne **entre conteneurs** du même réseau.

### 3.2 Alias DNS

```bash
docker run -d --name payments \
  --network backend \
  --network-alias pay-svc \
  ghcr.io/acme/payments:2.1
```

* `pay-svc` devient un **alias** DNS (en plus de `payments`).

### 3.3 Paramètres DNS & hosts

```bash
docker run --dns 1.1.1.1 --dns-search corp.local --dns-option ndots:1 image
docker run --add-host redis.internal:10.0.0.10 image
docker run --hostname api-01 --domainname lab.local image
```

> Le DNS interne **ne traverse pas** les réseaux : si deux conteneurs ne partagent **aucun** réseau, la résolution par nom échoue.

---

## 4) EXPOSE vs publication de ports

* **`EXPOSE`** (Dockerfile) = **métadonnée** (port(s) sur lesquels l’app écoute).
* **Publication** réelle = option `-p` (ou `-P`) à `docker run`.

### 4.1 `-p` (port mapping explicite)

```bash
# TCP par défaut
docker run -d -p 8080:80 nginx

# UDP explicite
docker run -d -p 53:53/udp coredns/coredns

# Lier sur une IP précise de l’hôte
docker run -d -p 127.0.0.1:8080:80 nginx

# Plage de ports
docker run -d -p 3000-3005:3000-3005 myapp
```

* `HOST_IP:HOST_PORT:CONTAINER_PORT[/proto]`
* Lier sur `127.0.0.1` limite l’accès **au seul hôte** (sécurité).

### 4.2 `-P` (publish all)

```bash
docker run -d -P myimage
docker port <container>   # affiche les ports exposés et leur mapping aléatoire
```

* Publie **tous** les ports présents dans **EXPOSE** (ou détectés par l’image).

### 4.3 Vérifier les ports

```bash
docker port web
ss -lntp | grep dockerd
```

**Piège courant** : dans le conteneur, si votre service écoute **seulement sur `127.0.0.1`**, il **ne sera pas atteignable** par le NAT externe. Il doit écouter sur `0.0.0.0` (ou l’IP du conteneur).

---

## 5) Modes réseau spéciaux

### 5.1 `host`

```bash
docker run --network host ghcr.io/acme/agent:latest
```

* Le conteneur **partage l’interface** de l’hôte : **pas** de `-p`.
* **Pros** : latence minimale, découverte multicast, accès ports host.
* **Cons** : conflits de ports, **surface d’attaque** plus large, moins d’isolation.
* (Sur macOS/Windows Desktop, le support diffère car VM sous-jacente.)

### 5.2 `none`

```bash
docker run --network none ghcr.io/acme/job:latest
```

* Aucun accès réseau. Utile pour lots offline/forensic.

### 5.3 `macvlan` (aperçu)

```bash
# Donne une IP L2 sur le LAN au conteneur
docker network create -d macvlan \
  --subnet 192.168.10.0/24 \
  --gateway 192.168.10.1 \
  -o parent=eth0 pub-net

docker run -d --network pub-net --ip 192.168.10.50 nginx
```

* **Contraintes** : commutateur/routeur, mode promisc, l’hôte **ne voit** pas toujours bien le conteneur (besoin d’un macvlan “bridge” ou route). À tester **en lab** avant prod.

### 5.4 `ipvlan` (aperçu)

* Plus “route-centric”. Moins de promisc ; dépend de la topologie L3.
* Utilisation similaire à `macvlan` avec driver `ipvlan`.

---

## 6) IPv6 (aperçu utile)

### 6.1 Activer IPv6 côté démon

`/etc/docker/daemon.json` :

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/48"
}
```

Redémarrer Docker, puis créer un réseau **IPv6** :

```bash
docker network create --ipv6 --subnet fd00:dead:beef:1::/64 v6net
docker run -d --network v6net myimage
```

* Vérifier MTU et pare-feu sur IPv6 (filtrage `ip6tables/nftables`).

---

## 7) Contrôles egress/ingress (sans orchestrateur)

### 7.1 Réseaux internes (egress off)

```bash
docker network create --internal backend
# Rien sur Internet depuis ce réseau (mais inter-conteneurs OK)
```

### 7.2 Séparation front/back

* **frontend** (exposé par `-p`) ↔ **backend** (`--internal`).
* Les services partagent **uniquement** les réseaux nécessaires.

### 7.3 Pare-feu hôte (rappels)

* Docker gère des règles **NAT** automatiques (POSTROUTING MASQUERADE).
* Si vous **désactivez** `--iptables=false`, **vous devez** gérer toutes les règles.

> Pour de la micro-segmentation plus fine : séparer par réseaux, puis ajouter des règles iptables/nftables **par interface bridge** (`br-xxxx`).

---

## 8) MTU, NAT & hairpin (dépannage réseau)

* **MTU** : décalages (ex. VLAN/PPPoE/VM) ⇒ paquets fragmentés/perdus.
  → Ajuster MTU du **bridge** Docker ou des interfaces hôte.
* **Hairpin NAT** : accéder à `localhost:HOST_PORT` **depuis un conteneur**.
  → En général OK, mais parfois bloqué par pare-feu/conntrack. Tester.
* **Connexions refusées vs timeouts** :

  * **Refused** = rien n’écoute au port cible (ou listen `127.0.0.1` seulement).
  * **Timeout** = filtrage/routage/pare-feu/DNS/MTU.

---

## 9) Diagnostic pratique

### 9.1 Inspecter réseau & conteneur

```bash
docker network inspect app-net | jq
docker inspect -f '{{.NetworkSettings.Networks}}' web
docker port web
```

### 9.2 Netshoot (trousse à outils réseau)

```bash
docker run --rm -it --network app-net nicolaka/netshoot
# depuis ce shell :
dig db
curl -v http://web:80
ss -lntp
traceroute 1.1.1.1
```

### 9.3 Sur l’hôte

```bash
ip a | grep -E 'docker0|br-'
iptables -t nat -S | grep DOCKER
ss -lntp | grep -E 'dockerd|nginx|java'
```

---

## 10) Exemples synthèse

### 10.1 Stack front/back avec réseau interne

```bash
# Réseaux
docker network create frontend
docker network create --internal backend

# Backend (DB), non exposée
docker run -d --name db --network backend \
  -e POSTGRES_PASSWORD=secret postgres:16

# API sur les deux réseaux
docker run -d --name api \
  --network frontend \
  --network backend \
  --network-alias api-svc \
  ghcr.io/acme/api:1.4.2

# Reverse-proxy exposé
docker run -d --name web \
  --network frontend -p 80:80 \
  -v $PWD/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:1.27
```

### 10.2 Bind sur IP locale (sécuriser l’accès)

```bash
# Accessible seulement depuis l’hôte
docker run -d -p 127.0.0.1:8080:8080 ghcr.io/acme/admin-ui:latest
```

### 10.3 IP statique pour une DB

```bash
docker network create --subnet 172.22.0.0/24 --gateway 172.22.0.1 dbnet
docker run -d --name db --network dbnet --ip 172.22.0.10 postgres:16
```

---

## 11) Compose : réseaux, alias, internal, IPAM

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    networks:
      backend:
        aliases: [ db, pg ]
  api:
    image: ghcr.io/acme/api:1.4.2
    networks:
      frontend:
      backend:
        aliases: [ api-svc ]
  web:
    image: nginx:1.27
    ports:
      - "80:80"
    networks:
      frontend:

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

> Pour IPv6 en Compose : ajouter `enable_ipv6: true` + `ipam` v6 si Docker l’autorise.

---

## 12) Do & Don’t

**Do**

* Utiliser des **user-defined bridges** (isolation + DNS) plutôt que `docker0`.
* Séparer **frontend** (exposé) et **backend** (`--internal` si possible).
* Lier les ports sur une **IP spécifique** (ex. `127.0.0.1`) quand c’est local-only.
* Documenter les **plans d’adressage** (`--subnet`, IP statiques pour DB/queues).
* Consommer les services **par nom DNS** (alias), pas par IP.

**Don’t**

* Éviter de tout mettre sur le **même réseau** (risque de couplage & fuites).
* Ne pas publier de ports **inutiles** ; bannir les `-P` non maîtrisés.
* Éviter `--network host` en prod sauf besoin prouvé (et maîtrisé).
* Ne pas compter sur des **liens** (`--link`) : **déprécié** ; préférez DNS user-defined.
* Ne pas désactiver iptables Docker sans plan pare-feu **équivalent**.

---

## 13) Aide-mémoire (cheat-sheet)

```bash
# Réseaux
docker network create app-net
docker network ls
docker network inspect app-net | jq
docker network rm app-net

# IPAM & internal
docker network create --subnet 172.18.0.0/16 app-net
docker network create --internal backend

# Attacher / détacher
docker network connect app-net ct
docker network disconnect app-net ct

# DNS & alias
docker run -d --network app-net --network-alias api-svc ghcr.io/acme/api:1.4.2

# Publication de ports
docker run -d -p 8080:80 nginx
docker run -d -p 127.0.0.1:8080:80 nginx
docker port nginx

# Modes spéciaux
docker run --network host image
docker run --network none image

# Netshoot
docker run --rm -it --network app-net nicolaka/netshoot
```

---

## 14) Checklist de clôture (qualité réseau)

* Réseaux **user-defined** utilisés ; DNS interne fonctionnel par **noms**/alias.
* **Plans d’adressage** et **IPAM** documentés ; IP statiques réservées si besoin.
* Séparation **front/back** ; **backend** en `--internal` si compatible.
* Ports publiés **explicitement** (`-p`) et **au strict nécessaire** ; binds sur IP d’écoute voulues.
* **Pare-feu** de l’hôte cohérent (Docker iptables actif ou règles équivalentes).
* MTU/hairpin **testés** ; diagnostics (`netshoot`, `docker port`, `iptables -t nat -S`) connus.
* Pas d’usage de `--link` ; pas de `--network host` non justifié.


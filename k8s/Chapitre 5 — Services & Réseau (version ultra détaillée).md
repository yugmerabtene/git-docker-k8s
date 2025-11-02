# Chapitre 5 — Services & Réseau (version **ultra détaillée**)

*(Services L4, Endpoints/EndpointSlice, kube-proxy iptables/IPVS, DNS CoreDNS, Ingress L7, TLS, ExternalTrafficPolicy, sessionAffinity, hints topologiques, CNI & MTU, MetalLB, runbooks de debug — avec décomposition **commande par commande**)*

---

## 0) Préambule (méthode d’étude)

Pour **chaque** ressource (Service, Ingress, etc.) :

1. lire le **YAML** ; 2) **expliquer chaque champ** ; 3) exécuter les **commandes kubectl** ; 4) comprendre **les flags** ; 5) diagnostiquer avec **événements/logs** ; 6) corriger.

---

## 1) Services (L4) — anatomie et **décomposition champ par champ**

### 1.1 Service ClusterIP (interne)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: demo
  labels:
    app.kubernetes.io/name: web
    app.kubernetes.io/part-of: demo
spec:
  type: ClusterIP               # VIP interne (par défaut si omis)
  selector:                     # DOIT matcher les labels des Pods ciblés
    app.kubernetes.io/name: web
  ports:
    - name: http
      port: 80                  # Port “virtuel” du Service (exposé aux clients in-cluster)
      targetPort: 8080          # Port réel du conteneur (peut être un nom, ex: "http")
      protocol: TCP             # TCP | UDP | SCTP
```

**Décomposition :**

* `apiVersion: v1` : famille “core” (stable).
* `kind: Service` : objet de couche **L4**.
* `metadata.name/namespace/labels` : identifiants + **labels** pour sélection/filtrage.
* `spec.type: ClusterIP` : crée une **IP virtuelle** accessible **uniquement** depuis le cluster.
* `spec.selector` : **critique** ; alimente automatiquement les **Endpoints** (ou EndpointSlice) à partir des Pods.
* `spec.ports[*].port` : port du **Service** (côté client interne).
* `spec.ports[*].targetPort` : port **dans** le conteneur (peut référencer un **nom** défini dans `containerPort.name`).
* `protocol` : la plupart du temps **TCP**.

**Commandes (avec explications)**

```bash
kubectl get svc web-svc -n demo -o wide
# -o wide : colonnes supplémentaires (cluster IP, sélecteurs, etc.)

kubectl describe svc web-svc -n demo
# 'describe' : YAML “commenté” + Events final — très utile pour diagnostiquer endpoints

kubectl get endpoints web-svc -n demo -o yaml
# Montre les adresses (IP Pods + ports) sélectionnées par le Service

kubectl get endpointslices -n demo -l kubernetes.io/service-name=web-svc -o yaml
# EndpointSlice, format scalable (adresses, conditions, zone/topology hints)

kubectl port-forward -n demo svc/web-svc 8080:80
# Port-forward depuis votre machine -> Service (test rapide sans Ingress/LB)
# 8080 local -> 80 du Service (VIP)
```

**Pièges fréquents et corrections**

* **0 endpoints** : labels/selector incohérents, **namespace** mauvais, Pods **notReady**.
  → `kubectl get pods -n demo --show-labels`, `kubectl describe svc`, `kubectl get events`.
* `targetPort` incorrect (mauvais **numéro** ou **nom** absent) → 503/connexion refusée.
* `protocol` incompatible (rarement UDP/SCTP par erreur).

---

### 1.2 Service NodePort (exposition simple depuis les nœuds)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: demo
spec:
  type: NodePort
  selector: { app.kubernetes.io/name: web }
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 31080              # 30000-32767 (ou auto si omis)
```

**Idée :** ouvre un **port haut** sur **chaque nœud**.
**Accès :** `http://<IP_NOEUD>:31080`.

**Commandes**

```bash
kubectl get svc web-nodeport -n demo -o wide
# montre le nodePort ; utilisez n'importe quelle IP de nœud

kubectl describe svc web-nodeport -n demo
# vérifie le mapping (port->targetPort) + endpoints
```

**Attention aux flags côté Service**

* `externalTrafficPolicy: Local|Cluster` (détaillé §1.4).
* `sessionAffinity: ClientIP` (afin de “coller” un client au même Pod).

---

### 1.3 Service LoadBalancer (cloud/MetalLB)

```yaml
apiVersion: v1
kind: Service
metadata: { name: web-lb, namespace: demo }
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # conserve l'IP source (voir §1.4)
  selector: { app.kubernetes.io/name: web }
  ports: [ { name: http, port: 80, targetPort: 80 } ]
```

**Idée :** provisionne un **LB externe** (Cloud) ou, **on-prem**, nécessite **MetalLB**.
**Commandes**

```bash
kubectl get svc web-lb -n demo -o wide
# Sur Cloud : une EXTERNAL-IP doit apparaître
# On-prem : EXTERNAL-IP vient de MetalLB (pool défini)
```

---

### 1.4 Champs avancés (très importants)

#### a) `externalTrafficPolicy`

* `Cluster` *(défaut)* : la VIP LB/NodePort **répartit** sur **tous** les nœuds → **SNAT** → **perte d’IP source** côté Pod.
* `Local` : **pas de cross-node** ; ne route **que** vers des Pods **sur le même nœud** → **préserve l’IP source**.
  **Attention** : si aucun Pod sur un nœud exposé, ce nœud **ne répond pas**.

```yaml
spec:
  externalTrafficPolicy: Local
```

#### b) `sessionAffinity`

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800   # 3h (valeur max typique)
```

* Évite qu’un client change de Pod (utile pour sessions non stateless).

#### c) Dual-stack

```yaml
spec:
  ipFamilyPolicy: PreferDualStack     # SingleStack | PreferDualStack | RequireDualStack
  ipFamilies: [ IPv4, IPv6 ]
```

#### d) Headless & publishNotReadyAddresses

```yaml
apiVersion: v1
kind: Service
metadata: { name: db-headless, namespace: demo }
spec:
  clusterIP: None               # Headless → pas de VIP ; DNS renvoie IP des Pods
  selector: { app.kubernetes.io/name: db }
  publishNotReadyAddresses: false
```

* Très utile avec **StatefulSets**, drivers qui gèrent shards/replicas côté client.

---

## 2) kube-proxy — **comment** la VIP en L4 devient du trafic vers les Pods

### 2.1 Rôles & modes

* **Programme** dynamiquement les **règles** de translation L4 : Service VIP → Endpoints (Pods).
* **Modes** :

  * `iptables` (universel) : utilise les chaînes **KUBE-** dans la table **nat**.
  * `IPVS` (performant) : utilise les tables IPVS (réglages via `ipvsadm`).
  * (Cilium eBPF peut remplacer kube-proxy ; hors périmètre ici.)

**Commandes d’inspection**

```bash
kubectl -n kube-system get ds kube-proxy -o wide
# Vérifie le DaemonSet (un pod kube-proxy par nœud)

kubectl -n kube-system logs -f ds/kube-proxy
# Logs : (re)programmation des règles, erreurs

# Mode iptables : voir les chaînes programmées
sudo iptables -t nat -L -n | sed -n '/KUBE-SVC/p' | head
sudo iptables -S | grep KUBE- | head

# Mode IPVS : voir services/real servers
sudo ipvsadm -ln | head
```

**Pièges**

* “**Flush** iptables” à l’aveugle → casse le routage Services.
* `conntrack` saturé → timeouts ; surveiller `nf_conntrack_*` (sysctls).

---

## 3) Endpoints & EndpointSlice — **la** liste des Pods réellement éligibles

**Endpoints (legacy)**

```bash
kubectl get endpoints web-svc -n demo -o yaml
# addresses[*].ip = IP de Pods prêts (readiness OK)
```

**EndpointSlice (scalable)**

```bash
kubectl get endpointslices -n demo -l kubernetes.io/service-name=web-svc -o yaml
# .endpoints[*].addresses : IPs
# .endpoints[*].conditions.ready : readiness
# .endpoints[*].topology.zone : info de zone pour hints topologiques
```

**Extraire les IPs via jsonpath**

```bash
kubectl get endpoints web-svc -n demo -o jsonpath='{.subsets[*].addresses[*].ip}{"\n"}'
kubectl get endpointslices -n demo -l kubernetes.io/service-name=web-svc \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[*]}{"\n"}{end}'
```

---

## 4) DNS in-cluster (CoreDNS) — **comment** on résout les Services

### 4.1 FQDN et requêtes

* FQDN : `<svc>.<ns>.svc.cluster.local`
* Même namespace : `http://<svc>` suffit.
* Headless + **ports nommés** → enregistrements **SRV**.

**Commandes (dans un Pod “netshoot”)**

```bash
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- sh

# Dans le pod:
dig A web-svc.demo.svc.cluster.local +short
# A : IPv4 ; AAAA : IPv6 ; +short : sorties compactes

dig SRV _http._tcp.web-svc.demo.svc.cluster.local +short
# Pour un headless, SRV peut lister les cibles (avec port nommés)

curl -I http://web-svc.demo.svc.cluster.local
# -I : HEAD (entêtes seulement) ; utile pour tester rapidement le code HTTP
```

### 4.2 NodeLocal DNS Cache (aperçu)

* Déploie un **cache DNS** local à chaque nœud → **réduit la latence** et la pression sur CoreDNS.
* Utile si `kubectl top pods -n kube-system` montre CoreDNS **haut** ou si DNS ressenti lent.

---

## 5) Ingress (L7 HTTP/HTTPS) — **règles** host/path → Service

### 5.1 Pré-requis

* **Ingress Controller** installé (Nginx, Traefik, HAProxy, Cilium LB…).
* `spec.ingressClassName` doit correspondre à la classe fournie par le contrôleur (souvent `nginx`).

### 5.2 Ingress **HTTP** (sans TLS) — avec **explication des champs**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"  # timeout HTTP côté proxy
spec:
  ingressClassName: nginx   # lie cet Ingress au contrôleur Nginx
  rules:
    - host: web.local       # Nom hôte que le client demande (Host:)
      http:
        paths:
          - path: /         # préfixe d’URL à router
            pathType: Prefix  # Prefix | Exact
            backend:
              service:
                name: web-svc     # service cible
                port:
                  number: 80
```

**Tests**

```bash
# Ajoutez dans /etc/hosts (ou Windows hosts) : <IP_INGRESS> web.local
curl -I -H 'Host: web.local' http://web.local/
# -H 'Host: ...' : force l’en-tête Host (utile quand on cible l’IP)
```

**Diagnostics**

```bash
kubectl get ingress -n demo -o wide
kubectl describe ingress web -n demo
kubectl -n ingress-nginx get pods -o wide
kubectl -n ingress-nginx logs -f deploy/ingress-nginx-controller
```

### 5.3 Ingress **HTTPS** (TLS)

Créer un **secret** TLS (cert + key) :

```bash
kubectl create secret tls tls-web \
  --cert=fullchain.pem --key=privkey.pem -n demo
# --cert : chaîne complète (cert + intermédiaires)
# --key  : clé privée correspondante
```

Activer TLS dans l’Ingress :

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts: [ "web.local" ]
      secretName: tls-web      # secret TLS à utiliser
  rules:
    - host: web.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: web-svc, port: { number: 80 } }
```

**Annotations utiles (Nginx)**

* `nginx.ingress.kubernetes.io/rewrite-target: /` (rewrite des chemins)
* `nginx.ingress.kubernetes.io/ssl-redirect: "true"` (force HTTPS)
* `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` (upload max)
* **Canary** :

  * `nginx.ingress.kubernetes.io/canary: "true"`
  * `nginx.ingress.kubernetes.io/canary-weight: "10"`

> En **prod**, gérer les certificats via **cert-manager** (ACME Let’s Encrypt).

---

## 6) CNI & MTU — **connectivité Pod↔Pod** et soucis “qui pendouillent”

### 6.1 Overlay vs Routed

* **Overlay** (VXLAN/GRE) : facile, mais **MTU réduite** (souvent ~1450) → attention aux gros paquets / HTTP/2/TLS.
* **Routed** (Calico BGP/Direct, Cilium eBPF) : meilleures perfs, moins d’overhead.

### 6.2 Diagnostiquer la **MTU** (expliquer chaque option)

Sur l’hôte :

```bash
ip a           # liste interfaces + MTU de chacune
ip route       # routes ; vérifier passerelles & chemins
```

Depuis un Pod **netshoot** :

```bash
ping -s 1472 <IP_DEST> -M do
# -s 1472 : taille payload ICMP (1472 + 28 bytes headers = 1500 → MTU Ethernet)
# -M do   : "do not fragment" (PMTUD). Si ça échoue, MTU effective < 1500.
# Ajustez à la baisse (ex : 1420) jusqu’à succès pour estimer la MTU overlay.
```

**Ajuster MTU côté CNI (selon plugin)**

* Calico (ConfigMap `calico-config`) → `veth_mtu: "1440"` (à adapter).
* Cilium (values Helm) → `--mtu=…`.

### 6.3 Hairpin NAT, hostPort/hostNetwork

* **Hairpin** : un Pod atteint **son propre Service** ; dépend du support CNI/kube-proxy.
* `hostPort` / `hostNetwork: true` : exposition directe via l’hôte → **couplage fort** aux nœuds ; préférer Service/Ingress.

---

## 7) MetalLB (on-prem) — donner des **EXTERNAL-IP** à vos Services

**Idée** : en bare-metal, MetalLB annonce des IP (Layer-2 ARP/NDP ou BGP).

**CR d’exemple (L2)**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: public-pool, namespace: metallb-system }
spec:
  addresses: [ "192.168.1.240-192.168.1.250" ]

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: { name: l2adv, namespace: metallb-system }
spec:
  ipAddressPools: [ "public-pool" ]
```

Ensuite, un Service `type: LoadBalancer` reçoit une IP dans ce **pool**.

**Commandes**

```bash
kubectl -n metallb-system get all
kubectl get ipaddresspool -n metallb-system
kubectl get svc -A | grep LoadBalancer
```

---

## 8) **Runbooks** de debug — scénarios & **commandes décortiquées**

### 8.1 “Le Service n’a pas d’Endpoints”

**Symptômes** : trafic 503/timeout ; `kubectl get endpoints web-svc` → **vide**.

**À faire**

```bash
kubectl get pods -n demo --show-labels
# Vérifie que les pods ont bien app.kubernetes.io/name=web

kubectl describe svc web-svc -n demo
# Voit le selector, les ports, et s’il liste des Endpoints

kubectl get endpoints web-svc -n demo -o yaml
kubectl get endpointslices -n demo -l kubernetes.io/service-name=web-svc -o yaml
# Regarde .addresses, .conditions.ready

kubectl get events -n demo --sort-by=.lastTimestamp | tail -n 20
# Événements récents (readiness failing, etc.)
```

**Correctifs** : aligner **selector/labels** ; vérifier **readinessProbe** ; bon **namespace**.

---

### 8.2 “Ingress renvoie 404 ou ignore mon host”

**À faire**

```bash
kubectl get ingress -n demo -o wide
# Vérifie host/path & ingressClassName

kubectl describe ingress web -n demo
# Détail du backend lié, erreurs éventuelles

kubectl -n ingress-nginx get pods -o wide
kubectl -n ingress-nginx logs -f deploy/ingress-nginx-controller
# Voir les logs du contrôleur (parsing d’annotations, erreurs)

# Test en forçant l’en-tête Host, utile si on cible l'IP:
curl -I -H 'Host: web.local' http://<IP_INGRESS>/
```

**Correctifs** : corriger `ingressClassName`, `host`/DNS, `serviceName/port`, annotations.

---

### 8.3 “Mon app ne voit plus l’IP source”

**Cause** : `externalTrafficPolicy: Cluster` (SNAT).
**Fix** :

```yaml
spec:
  externalTrafficPolicy: Local
```

**Attention** : le trafic n’aboutit que sur les nœuds ayant des Pods **Ready**.

---

### 8.4 “Connexions HTTP qui pendent / resets sporadiques”

**Pistes** : **MTU overlay** trop élevée, `conntrack` saturé, timeouts Nginx/Ingress.
**À faire**

```bash
# Tester MTU
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- sh -lc \
  "ping -s 1472 <IP> -M do || ping -s 1420 <IP> -M do"

# kube-proxy logs
kubectl -n kube-system logs -f ds/kube-proxy

# (hôte) conntrack
sudo sysctl net.netfilter.nf_conntrack_max
sudo conntrack -S | head   # si disponible
```

**Correctifs** : baisser MTU côté CNI ; augmenter `proxy-read-timeout` via annotation Ingress ; tuner `nf_conntrack_max`.

---

### 8.5 “DNS lent / épars”

**À faire**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide  # CoreDNS
kubectl top pod -n kube-system
# Charge CPU/mémoire

# Inside netshoot:
dig +trace web-svc.demo.svc.cluster.local
```

**Fix** : déployer **NodeLocal DNS Cache**, vérifier policies réseau, réduire chattiness.

---

### 8.6 “kube-proxy/iptables cassés”

**À faire**

```bash
kubectl -n kube-system logs -f ds/kube-proxy
sudo iptables -t nat -L -n | grep KUBE | head
sudo ipvsadm -ln | head
```

**Fix** : ne **jamais** flush iptables sans comprendre ; vérifier ServiceCIDR/PodCIDR ; re-déployer kube-proxy si cassé.

---

## 9) Aide-mémoire **enrichi** — commandes & flags utiles

### 9.1 Services & Endpoints

```bash
# Listing & formats
kubectl get svc -A -o wide
kubectl get svc web-svc -n demo -o yaml

# Endpoints & EndpointSlice (IPs & readiness)
kubectl get endpoints web-svc -n demo -o jsonpath='{.subsets[*].addresses[*].ip}{"\n"}'
kubectl get endpointslices -n demo -l kubernetes.io/service-name=web-svc -o yaml

# Describe (détails + Events)
kubectl describe svc web-svc -n demo

# Port-forward rapide (dev)
kubectl port-forward -n demo svc/web-svc 8080:80
```

### 9.2 Ingress

```bash
kubectl get ingress -A
kubectl describe ingress web -n demo
kubectl -n ingress-nginx get pods -o wide
kubectl -n ingress-nginx logs -f deploy/ingress-nginx-controller

# Test d’hôte virtuel
curl -I -H 'Host: web.local' http://<IP_INGRESS>/
```

### 9.3 kube-proxy

```bash
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs -f ds/kube-proxy
sudo iptables -t nat -L -n | grep KUBE | head
sudo ipvsadm -ln | head
```

### 9.4 DNS

```bash
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- sh
# Puis:
dig A web-svc.demo.svc.cluster.local +short
dig SRV _http._tcp.web-svc.demo.svc.cluster.local +short
curl -I http://web-svc.demo.svc.cluster.local
```

### 9.5 CNI & MTU

```bash
ip a
ip route
# Test MTU (ne pas fragmenter)
ping -s 1472 <IP> -M do || ping -s 1420 <IP> -M do
```

---

## 10) Bonnes pratiques “réseau”

* **Labels normalisés** (`app.kubernetes.io/*`) cohérents entre Pods/Services/Ingress.
* **Pas** d’images `:latest` ; readiness **sérieuse** pour éviter d’ajouter des Pods non prêts aux Endpoints.
* Choisir `externalTrafficPolicy` selon le besoin de **source IP** côté app.
* **IngressClass** explicite ; annotations **documentées** et **versionnées**.
* **TLS** partout vers l’extérieur ; en prod, **cert-manager**.
* **CNI/MTU** : **documenter** et **tester** (HTTP/2, payloads volumineux).
* Surveiller **CoreDNS** & **kube-proxy** (métriques/logs) ; éviter **NodePort** en prod si possible (préférer LB/Ingress).

---

## 11) Mini-lab guidé (avec explications des flags)

### Étape 1 — Déployer l’app et le Service

```bash
kubectl create ns demo
kubectl config set-context --current --namespace=demo

kubectl create deploy web --image=nginx:1.27 --replicas=2
# create deploy : générateur impératif → Deployment apps/v1 avec 2 Pods

kubectl expose deploy web --name=web-svc --port=80 --target-port=80
# expose : crée un Service depuis un objet existant (selector auto)
# --port : port du Service ; --target-port : port conteneur

kubectl get svc web-svc -o wide
kubectl get endpoints web-svc -o yaml
```

### Étape 2 — Tester DNS interne & HTTP

```bash
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- \
  sh -lc "dig +short web-svc.demo.svc.cluster.local && curl -I http://web-svc"
# -lc "cmds" : exécute plusieurs commandes via shell
```

### Étape 3 — Ingress (Nginx)

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web-svc, port: { number: 80 } } }
YAML
```

**Tester :**

* Ajouter `web.local` dans `/etc/hosts` → `<IP_INGRESS> web.local`.
* `curl -I -H 'Host: web.local' http://web.local/`.

### Étape 4 — NodePort + ExternalTrafficPolicy

```bash
kubectl patch svc web-svc -p '{"spec":{"type":"NodePort","externalTrafficPolicy":"Local"}}'
# patch JSON merge : modifie en place la spec
kubectl get svc web-svc -o wide
# Tenter : http://<IP_NOEUD>:<NODEPORT>
```

### Étape 5 — Headless + SRV

```bash
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: Service
metadata: { name: web-headless, namespace: demo }
spec:
  clusterIP: None
  selector: { app.kubernetes.io/name: web }
  ports: [ { name: http, port: 80, targetPort: 80 } ]
YAML

kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- \
  sh -lc "dig A web-headless.demo.svc.cluster.local +short"
```

**Nettoyage**

```bash
kubectl delete ingress web
kubectl delete svc web-svc web-headless
kubectl delete deploy web
```




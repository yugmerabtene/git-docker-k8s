# Chapitre 2 — Architecture détaillée

*(control plane, nœuds, réseau, stockage, sécurité, commandes d’inspection & diagnostics)*

---

## 1) Objectifs d’apprentissage

* Comprendre **qui fait quoi** dans Kubernetes (control plane vs nœuds).
* Savoir **inspecter**, **diagnostiquer** et **sécuriser** les composants clés : **etcd**, **kube-apiserver**, **kube-controller-manager**, **kube-scheduler**, **kubelet**, **runtime CRI**, **CNI**, **kube-proxy**.
* Maîtriser les **flux internes** (authN/authZ/admission), l’**ordonnancement**, la **santé** et les **journaux**.
* Connaître les **bonnes pratiques** (HA, sauvegardes etcd, RBAC, TLS, audit, évictions) et les **pièges fréquents** (quorum etcd, MTU CNI, dérive d’horloge).

---

## 2) Vue d’ensemble (schéma mental)

```
[Client kubectl] 
    ↓ (TLS, AuthN/AuthZ/Admission)
[kube-apiserver] <→ [etcd] (KV distribué, quorum Raft)
    ↓ watch/notify
[kube-controller-manager]     [kube-scheduler]
    ↓                                 ↓
          (Objets K8s) ←——————— [API Server] ———————→ (événements)
                                    ↓
                              [kubelet sur chaque nœud]
                                    ↓ (CRI)
                           [containerd / CRI-O (pods/containers)]
                                    ↓ (CNI)
                              [réseau Pod ↔ Service (kube-proxy/IPVS/iptables)]
```

---

## 3) Control plane : composants & commandes d’inspection

### 3.1 etcd — base de données clé/valeur (Raft)

* **Rôle** : vérité unique de l’état du cluster (objets API sérialisés en etcd).
* **Ports** : 2379 (client), 2380 (cluster peer).
* **Quorum** : nombre impair de membres (3/5). **Perdre le quorum = cluster bloqué** en écriture.
* **Entretien** : compaction, defrag, **sauvegardes régulières**.

**Fichiers & pods (kubeadm)**

* Manifeste statique : `/etc/kubernetes/manifests/etcd.yaml`
* Data dir : souvent `/var/lib/etcd`

**Commandes clés (sur un master)**

```bash
# 0) Préparer l’outil etcdctl (avec kubeadm, exécutez depuis le conteneur etcd)
export ETCDCTL_API=3

# 1) Vérifier la santé (via TLS) : adaptez les chemins certs selon votre installation
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# 2) Lister les membres
etcdctl --endpoints=https://127.0.0.1:2379 ... member list

# 3) Snapshot (sauvegarde)
etcdctl --endpoints=https://127.0.0.1:2379 ... snapshot save /root/etcd-$(date +%F-%H%M).db
etcdctl snapshot status /root/etcd-*.db

# 4) Maintenance (compaction + defrag)
etcdctl --endpoints=https://127.0.0.1:2379 ... compact $(etcdctl ... endpoint status --write-out=json | jq -r '.[0].Status.header.revision')
etcdctl --endpoints=https://127.0.0.1:2379 ... defrag
```

**Bonnes pratiques**

* **HA** : 3 nœuds etcd dédiés, disques rapides, **sauvegardes** testées (restore).
* **Sécurité** : TLS obligatoire, certificats avec **rotation**, accès restreint 2379/2380.
* **Surveillance** : taille DB, compactions régulières, alerte **perte de membre**.

---

### 3.2 kube-apiserver — unique point d’entrée

* **Rôle** : front-door de l’API, validation des requêtes, **authN → authZ → admission**.
* **Flux** :

  1. **AuthN** (cert client, token, ServiceAccount, OIDC…)
  2. **AuthZ** (**RBAC** typiquement)
  3. **Admission** (plugins & webhooks : mutating/validating)
  4. **Etcd** (lecture/écriture) + **watch**.
* **Extensibilité** : **CRDs**, **API Aggregation**.

**Commandes d’inspection**

```bash
kubectl cluster-info
kubectl -n kube-system get pods -l component=kube-apiserver -o wide
kubectl -n kube-system logs -f kube-apiserver-<node>
```

**Points d’attention (flags)**

* `--authorization-mode=Node,RBAC` (éviter ABAC en prod)
* `--disable-admission-plugins` / `--enable-admission-plugins` (ex. PodSecurity, NodeRestriction)
* **Audit** : `--audit-policy-file`, `--audit-log-path`, `--audit-log-maxage`
* **TLS** seulement (pas de port HTTP non sécurisé)

**Diagnostics rapides**

* Requêtes qui échouent ?
  `kubectl auth can-i get pods --as system:serviceaccount:ns:sa`
* Admission refusée ? Lire les **Events** et les logs API server.

---

### 3.3 kube-controller-manager — boucles de réconciliation

* **Rôle** : exécute les contrôleurs “de base” (Deployment→ReplicaSet→Pods, Node, Service, Endpoints, Job, CronJob, Namespace GC, CSR…).
* **Leader election** : un seul actif, les autres standby.
* **Inspection**

```bash
kubectl -n kube-system get pods -l component=kube-controller-manager -o wide
kubectl -n kube-system logs -f kube-controller-manager-<node>
kubectl -n kube-system get lease | grep controller
kubectl -n kube-system describe lease kube-controller-manager
```

**Pièges** : perte de leader (clock skew), droits manquants (RBAC), API server lent → backlog d’events.

---

### 3.4 kube-scheduler — placement des Pods

* **Rôle** : sélectionne un **nœud** pour chaque Pod en **Pending**.
* **Étapes** : **filtrage** (taints/tolerations, ressources, affinities) → **scorage** (préférence) → **liaison** (bind).
* **Fonctionnalités** : **affinity/anti-affinity**, `topologySpreadConstraints`, **priorités & préemption**.
* **Inspection**

```bash
kubectl -n kube-system get pods -l component=kube-scheduler -o wide
kubectl -n kube-system logs -f kube-scheduler-<node>
kubectl get events --sort-by=.lastTimestamp | egrep -i "Scheduled|FailedScheduling" | tail
```

**Diagnostics** : “FailedScheduling” → manque de CPU/Mem, taints non tolérés, contraintes d’affinité trop strictes, volumes introuvables.

---

## 4) Nœuds : kubelet, runtime (CRI), CNI & kube-proxy

### 4.1 kubelet — agent du nœud

* **Rôle** : s’enregistre au control plane, **exécute** les Pods via CRI, publie l’état (`NodeStatus`), gère **probes**, **cgroups**, **evictions** (DiskPressure/MemPressure).
* **Fichiers** : `/var/lib/kubelet/config.yaml`, `/var/lib/kubelet/`, certificats bootstrap (`/var/lib/kubelet/pki/`).
* **Journaux & état**

```bash
systemctl status kubelet
journalctl -u kubelet -f
kubectl get nodes -o wide
kubectl describe node <nœud>           # conditions, capacity, allocatable, taints
```

* **Evictions** (ex.) : `--eviction-hard=memory.available<500Mi,nodefs.available<10%`
* **TLS bootstrap** + **NodeRestriction** admission plugin (recommandé).

### 4.2 Runtime CRI — containerd / CRI-O

* **Rôle** : crée/supprime conteneurs & pods sandboxes, gère images, logs, cgroups.
* **Outils** : `crictl` (client CRI), `ctr` (bas niveau containerd).

```bash
crictl info
crictl ps -a
crictl images
crictl logs <container-id>
crictl inspectp <pod-id>
```

* **RuntimeClass** : choisir un runtime alternatif (gVisor, Kata) par Pod.

### 4.3 CNI — réseau Pod ↔ Pod

* **Rôle** : attacher une **interface** au Pod, attribuer IP (CIDR), configurer routage.
* **Plugins** : Flannel, Calico, Cilium (eBPF), Weave…
* **Pièges** : **MTU** (fragmentation), routes manquantes, conflits CIDR.
* **Debug**

```bash
ip a
ip route
# Pod de debug
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- sh
# depuis le pod:
curl -I http://web-svc.default.svc.cluster.local
```

### 4.4 kube-proxy — Services (iptables / IPVS)

* **Rôle** : programme les règles de translation pour Services/Endpoints (VIP → Pods).
* **Modes** : `iptables` (compatibilité), `ipvs` (performances). Cilium peut remplacer kube-proxy (eBPF).
* **Inspection**

```bash
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs -f ds/kube-proxy
iptables -S | grep KUBE- | head
ipvsadm -ln | head   # si mode ipvs
```

* **Pièges** : flush iptables imprudent, conntrack saturé, mismatch CIDR Service/Pod.

---

## 5) Authentification, Autorisation, Admission (chaîne AAA)

1. **AuthN** (qui ?) : certificats client, tokens (SA/JWT), OIDC (SSO), Webhook.
2. **AuthZ** (a-t-il le droit ?) : **RBAC** (recommandé), Node, ABAC (éviter).
3. **Admission** (doit-on autoriser la création ? peut-on muter l’objet ?)

   * **Plugins** intégrés (ex. **PodSecurity**, **NodeRestriction**, LimitRanger, ResourceQuota…)
   * **Webhooks** (mutating/validating) — ex. politiques OPA/Kyverno.

**Vérifier un droit**

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:viewer -n demo
```

**API Aggregation & CRD** : ajout d’API (metrics-server, services API externes), **CRDs** = nouveaux “types d’objets”.

---

## 6) Santé, métriques & journaux

**Santé (HTTP)**

* kube-apiserver : `/healthz`, `/livez`, `/readyz`
* scheduler/controller : `/healthz`
* **Accès** via port-forward si nécessaire :

```bash
kubectl -n kube-system port-forward deploy/kube-apiserver-<node> 8443:6443
# puis curl -k https://localhost:8443/readyz?verbose
```

**Métriques**

* Exposées en **Prometheus** format sur `/metrics` (selon composants).
* **metrics-server** fournit CPU/Mem Pods/Nodes pour HPA/`kubectl top`.

**Journaux**

```bash
kubectl -n kube-system logs -f <pod-composant>
journalctl -u kubelet -f
```

---

## 7) Réseau : CIDR, Services, Ingress (vision archi)

* **Pod CIDR** : plage IP Pods (ex. 10.244.0.0/16).
* **Service CIDR** : VIP Services (ex. 10.96.0.0/12).
* **Hairpin NAT** : Pod→Service→Pod local (à surveiller).
* **Ingress** : couche L7 (HTTP/HTTPS) via contrôleur (Nginx/Traefik/HAProxy).
* **LoadBalancer** : IP externe (Cloud) ou **MetalLB** on-prem.

**Diagnostics**

```bash
kubectl get svc,ep -A
kubectl describe svc <name>
kubectl get ingress -A
```

---

## 8) Stockage : CSI (aperçu archi)

* **CSI** = interface standard pour pilotes de stockage (EBS, Ceph, NFS, …).
* **PV/PVC** : abstraction K8s, **StorageClass** pour provisionnement dynamique.
* **StatefulSet** : identité stable + PVC par réplique.
* **Diagnostics**

```bash
kubectl get sc
kubectl get pv,pvc -A
kubectl describe pvc <name>
```

---

## 9) Sécurité & durcissement (liens archi)

* **PKI** : toutes les coms en **TLS** (certs kubelet, apiserver, etcd).
* **RBAC** : principe du moindre privilège, **groupes** & **ServiceAccounts**.
* **Admission** : **Pod Security** (baseline/restricted), **NodeRestriction**.
* **Kubelet** : `readOnlyPort` désactivé, authentification/autorisation activées.
* **Audit** : politique et journalisation, rotation des logs.
* **Chiffrement at rest** (secrets) côté API server (en prod).

---

## 10) Atelier guidé — inspection complète d’un cluster “kubeadm”

> Objectif : lire l’**état** et les **relations** entre composants, vérifier la **santé** et diagnostiquer.

### A) Control plane

```bash
# Contexte & nœuds
kubectl config current-context
kubectl get nodes -o wide

# Pods système
kubectl -n kube-system get pods -o wide

# API server
kubectl -n kube-system logs -f kube-apiserver-$(hostname)
kubectl get --raw '/readyz?verbose' | jq -r .  # (via apiserver si autorisé)

# Controller-manager & scheduler (leader + logs)
kubectl -n kube-system get lease | egrep 'controller|scheduler'
kubectl -n kube-system logs -f kube-controller-manager-$(hostname)
kubectl -n kube-system logs -f kube-scheduler-$(hostname)

# Events récents (triés)
kubectl get events --sort-by=.lastTimestamp | tail -n 30
```

### B) etcd

```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint health

# snapshot (test sauvegarde)
etcdctl ... snapshot save /root/etcd-snap.db && etcdctl snapshot status /root/etcd-snap.db
```

### C) Nœud, kubelet, runtime, réseau

```bash
# kubelet & conditions
kubectl describe node $(hostname)
journalctl -u kubelet -n 200

# runtime CRI (containerd)
crictl info
crictl ps -a
crictl images

# kube-proxy
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs -f ds/kube-proxy

# CNI (routes/MTU)
ip a; ip route
```

---

## 11) Pièges fréquents & correctifs

**(1) Perte de quorum etcd**

* *Symptômes* : API “read-only”, erreurs timeouts/etcd.
* *Correctifs* : restaurer membre perdu, vérifier **latence réseau**, refaire **snapshot restore** si nécessaire, **disques**.

**(2) kube-apiserver lent / saturé**

* *Symptômes* : latence `kubectl`, throttling.
* *Correctifs* : auditer charge, limiter webhooks lents, tuner `--max-requests-inflight`, scaler control plane.

**(3) “FailedScheduling”**

* *Causes* : taints non tolérés, ressources insuffisantes, anti-affinité stricte, volumes non dispo.
* *Commandes* :

  ```bash
  kubectl get events --sort-by=.lastTimestamp | grep -i FailedScheduling | tail
  kubectl describe node <nœud> | sed -n '/Taints:/,/Events:/p'
  ```

  Ajouter **tolerations**, **resources**, ou assouplir **affinity**.

**(4) Réseau Pod cassé (CNI/MTU)**

* *Symptômes* : pings perdus, connexions HTTP qui “pendouillent”.
* *Correctifs* : vérifier **MTU** (VXLAN ↓), routes et policies, logs du CNI. Tester via **netshoot**.

**(5) kube-proxy / Services (IPVS/iptables)**

* *Symptômes* : Service sans routage, DNS ok mais pas de retour.
* *Correctifs* : `iptables -S | grep KUBE` ou `ipvsadm -ln`, vérifier **Endpoints** et **readiness** Pods.

**(6) Dérive d’horloge (NTP)**

* *Effets* : leader election instable, certificats invalides.
* *Correctifs* : NTP strict sur tous les nœuds.

**(7) Certificats expirés (clusters kubeadm)**

* *Commande* :

  ```bash
  kubeadm certs check-expiration
  kubeadm certs renew all
  systemctl restart kubelet
  ```

  (Planifier la rotation à l’avance, surveiller les dates d’expiration.)

---

## 12) Bonnes pratiques architecture

* **HA control plane** : au moins **3** control planes derrière un **load balancer** TCP (6443).
* **etcd dédié** (ou managé) : **odd** members, sauvegardes testées, disques rapides, isolation réseau.
* **RBAC strict** + **NodeRestriction** + **Pod Security** (baseline/restricted).
* **Chiffrement at rest** des **Secrets** côté API server.
* **Audit** activé avec **policy minimale** (éviter logs trop volumineux en prod).
* **Version skew** supporté (±1 mineur entre components), upgrades planifiés.
* **Observabilité** : Prometheus/Grafana, logs centralisés, alertes (quorum etcd, NotReady nodes, failure rate API).

---

## 13) Aide-mémoire (commandes clés “archi”)

```bash
# Topologie & composants
kubectl cluster-info
kubectl -n kube-system get pods -o wide
kubectl get nodes -o wide
kubectl get events --sort-by=.lastTimestamp | tail -n 30

# Control plane
kubectl -n kube-system logs -f kube-apiserver-$(hostname)
kubectl -n kube-system logs -f kube-controller-manager-$(hostname)
kubectl -n kube-system logs -f kube-scheduler-$(hostname)
kubectl -n kube-system get lease | grep -E 'controller|scheduler'

# etcd (kubeadm)
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key endpoint health
etcdctl ... snapshot save /root/etcd-$(date +%F).db

# Nœuds & kubelet
kubectl describe node $(hostname)
journalctl -u kubelet -f
crictl ps -a; crictl images

# Réseau & Services
kubectl get svc,ep -A
kubectl -n kube-system logs -f ds/kube-proxy
iptables -S | grep KUBE  # ou ipvsadm -ln
```


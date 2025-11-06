# Chapitre 2 — Architecture de Kubernetes

*(Control Plane, Nœuds, Réseau, Stockage, Sécurité, Diagnostic)*

---

## 1. Objectifs d’apprentissage

À la fin de ce chapitre, vous serez capable de :

* Expliquer l’**architecture interne** de Kubernetes (Control Plane et Nœuds).
* Identifier les **composants clés** du cluster et leur rôle.
* Diagnostiquer le **fonctionnement global** du système à l’aide de `kubectl`.
* Comprendre les **flux internes de communication et de sécurité**.
* Mettre en place un **cluster local complet** pour le projet fil rouge.

---

## 2. Vue d’ensemble de l’architecture Kubernetes

Kubernetes est une plateforme **distribuée** composée de deux ensembles logiques :

1. **Le Control Plane**
   Regroupe les composants responsables de l’**API**, du **stockage d’état**, de la **réconciliation** et de l’**ordonnancement** :

   * **kube-apiserver** : point d’entrée unique. Valide les requêtes, applique AuthN/AuthZ/Admission, et persiste/lecture l’état.
   * **etcd** : base **clé/valeur** distribuée (consensus **Raft**) stockant l’état source de vérité.
   * **kube-controller-manager** : exécute des **boucles de contrôle** assurant la convergence vers l’état souhaité (Deployments, Nodes, Jobs, GC…).
   * **kube-scheduler** : **assigne** chaque Pod en attente à un nœud selon ressources/contraintes.

2. **Les Nœuds (Workers)**
   Exécutent les **Pods** et hébergent :

   * **kubelet** : agent local qui reçoit les ordres du Control Plane et orchestre les conteneurs.
   * **Container Runtime** (via **CRI**, ex. containerd/CRI-O) : crée/détruit les conteneurs.
   * **kube-proxy** : programme les règles **L4** (iptables/IPVS) pour la translation des Services.
   * **CNI** : plugin réseau (Calico/Cilium/Flannel/Weave) qui attache des interfaces et attribue des IP aux Pods.

### Flux internes (schéma mental)

```
[kubectl/clients] → (TLS, AuthN/AuthZ/Admission) → [kube-apiserver] ↔ [etcd]
                                        │                  ▲   watch
                              controllers/scheduler  ──────┘
                                        │
                                    [kubelet] → (CRI) → [runtime] → containers
                                        │
                                       (CNI) réseau Pod↔Pod, (kube-proxy) Services
```

Points clés :

* **Modèle déclaratif** : on publie un état souhaité (YAML). Les contrôleurs assurent la convergence.
* **Découplage fort** : API centrale, nœuds remplaçables, Pods éphémères.
* **Observabilité** : tout passe par l’API → **events**, **logs** et **metrics** accessibles.

---

## 3. Control Plane : les composants principaux

### 3.1 etcd — base clé/valeur (consensus Raft)

* **Rôle** : stocke l’état complet du cluster (objets API sérialisés).
* **Ports** : 2379 (client API server), 2380 (peer cluster).
* **Quorum** : nombre impair (3/5). Perte de quorum → **écritures impossibles**.
* **Maintenance** : compaction, **defrag**, **sauvegardes régulières** (et tests de restauration).

Avec kubeadm (emplacements usuels) :

* Manifeste statique : `/etc/kubernetes/manifests/etcd.yaml`
* Données : `/var/lib/etcd`

Commandes typiques (sur un nœud control-plane) :

```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

etcdctl --endpoints=https://127.0.0.1:2379 ... member list
etcdctl --endpoints=https://127.0.0.1:2379 ... snapshot save /root/etcd-$(date +%F-%H%M).db
etcdctl snapshot status /root/etcd-*.db
```

Bonnes pratiques :

* 3 nœuds etcd dédiés (ou etcd managé), disques rapides, **sauvegardes testées**.
* Sécurité : **TLS partout**, accès ports 2379/2380 restreints, rotation de certificats.

### 3.2 kube-apiserver — front-door et cœur de l’API

* **Rôle** : reçoit/valide chaque requête, applique **AuthN → AuthZ → Admission**, lit/écrit dans etcd, diffuse via **watch**.
* **Extensibilité** : **CRDs** (nouvelles ressources), **API Aggregation**.

Inspection :

```bash
kubectl cluster-info
kubectl -n kube-system get pods -l component=kube-apiserver -o wide
kubectl -n kube-system logs -f kube-apiserver-<node_name>
```

Paramètres à connaître (selon déploiement) :

* `--authorization-mode=Node,RBAC`
* `--enable-admission-plugins=PodSecurity,NodeRestriction,...`
* `--audit-policy-file`, `--audit-log-path`
* TLS obligatoire (pas de port HTTP ouvert)

### 3.3 kube-controller-manager — boucles de réconciliation

* **Rôle** : orchestre les contrôleurs natifs (Deployment→ReplicaSet→Pods, Node, Service, Job, CronJob, Namespace GC, CSR…).
* **Leader Election** : un actif, les autres en attente (évite double exécution).

Inspection :

```bash
kubectl -n kube-system get pods -l component=kube-controller-manager -o wide
kubectl -n kube-system logs -f kube-controller-manager-<node_name>
kubectl -n kube-system get lease | grep controller
kubectl -n kube-system describe lease kube-controller-manager
```

### 3.4 kube-scheduler — placement des Pods

* **Rôle** : choisit un **nœud** pour chaque Pod en **Pending**.
* **Processus** : **filtrage** (taints/tolerations, ressources, affinities) → **notation** (préférences, spread) → **binding**.
* **Fonctionnalités** : affinities, `topologySpreadConstraints`, **priorités & préemption**.

Inspection :

```bash
kubectl -n kube-system get pods -l component=kube-scheduler -o wide
kubectl -n kube-system logs -f kube-scheduler-<node_name>
kubectl get events --sort-by=.lastTimestamp | egrep -i "Scheduled|FailedScheduling" | tail
```

---

## 4. Nœuds et exécution des conteneurs

### 4.1 kubelet — agent du nœud

* **Rôle** : enregistre le nœud auprès de l’API, applique les PodSpecs via **CRI**, publie l’état (`NodeStatus`), gère **cgroups** et **probes**.
* **Fichiers** : `/var/lib/kubelet/config.yaml`, certificats bootstrap `/var/lib/kubelet/pki/`.

Diagnostics :

```bash
systemctl status kubelet
journalctl -u kubelet -f
kubectl get nodes -o wide
kubectl describe node <node_name>     # capacity, allocatable, conditions, taints
```

Points d’attention :

* **Evictions** : mémoire/disque (ex. `--eviction-hard=memory.available<500Mi,...`).
* Sécurité : **NodeRestriction** activé, `readOnlyPort` désactivé.

### 4.2 Container Runtime via CRI — containerd / CRI-O

* **Rôle** : cycle de vie des conteneurs, images, logs, cgroups.
* **Outils** : `crictl` (client CRI), `ctr` (bas niveau containerd).

Exemples :

```bash
crictl info
crictl ps -a
crictl images
crictl logs <container_id>
crictl inspectp <pod_sandbox_id>
```

**RuntimeClass** : sélectionner un runtime alternatif (gVisor/Kata) par Pod.

### 4.3 CNI — réseau des Pods

* **Rôle** : rattacher une interface, attribuer IP **Pod CIDR**, configurer routes.
* **Plugins** : Flannel, Calico, Cilium (eBPF), Weave.
* **Pièges** : **MTU** (VXLAN), routes manquantes, conflits CIDR.

Debug :

```bash
ip a
ip route
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- sh
# Dans le Pod :
curl -I http://kubernetes.default.svc
```

### 4.4 kube-proxy — Services (iptables/IPVS)

* **Rôle** : programme la translation L4 pour les Services (VIP → Endpoints).
* **Modes** : `iptables` (compat) ou `ipvs` (perf). Cilium peut remplacer kube-proxy (eBPF).

Inspection :

```bash
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs -f ds/kube-proxy
iptables -S | grep KUBE- | head
# si IPVS
ipvsadm -ln | head
```

---

## 5. Sécurité et flux AAA

La **chaîne AAA** (Authentication → Authorization → Admission) s’applique à **toute requête** au `kube-apiserver`.

1. **Authentification (AuthN)** — *Qui ?*
   Certificats **X.509**, Tokens (ServiceAccount/JWT), **OIDC**, Webhook.

2. **Autorisation (AuthZ)** — *A-t-il le droit ?*
   **RBAC** (recommandé), Node, Webhook (ABAC à proscrire en prod).

3. **Admission** — *Doit-on l’autoriser / modifier ?*
   **Mutating/Validating Admission** (plugins intégrés, ex. **PodSecurity**, **NodeRestriction**) ou webhooks externes (**OPA Gatekeeper**, **Kyverno**).

Vérifier un droit :

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:viewer -n demo
```

Audit et chiffrement :

* **Audit logs** via `--audit-policy-file`, `--audit-log-path` sur l’API server.
* **Chiffrement at-rest** des **Secrets** via `--encryption-provider-config`.

---

## 6. **TP – Projet Fil Rouge (Phase 1)**

### **Installation complète de Docker + Kubernetes (Minikube) + kubectl**

Ce TP constitue la **base du projet fil rouge**.
À la fin, vous disposerez d’un **cluster Kubernetes fonctionnel**, capable d’exécuter vos futures applications conteneurisées.

---

### 6.1 Objectif

Installer pas à pas un environnement Kubernetes local (sous **Windows 10/11** ou **Linux Ubuntu/Debian**)
et en vérifier le bon fonctionnement.

---

## **6.2 Prérequis généraux**

| Élément                    | Description                                      | Détails                             |
| -------------------------- | ------------------------------------------------ | ----------------------------------- |
| **Système d’exploitation** | Windows 10/11 (avec WSL2) ou Linux Ubuntu/Debian | Mode administrateur requis          |
| **CPU**                    | Minimum 4 cœurs                                  | Recommandé : 8                      |
| **RAM**                    | Minimum 8 Go                                     | Recommandé : 16 Go                  |
| **Stockage**               | 25 Go libres minimum                             | SSD recommandé                      |
| **Connexion Internet**     | Stable                                           | Nécessaire pour les téléchargements |
| **Comptes**                | Accès administrateur (sudo / PowerShell Admin)   | —                                   |

---

## **6.3 Installation étape par étape**

### **Étape 1 — Installer Docker**

#### Sous **Linux (Ubuntu/Debian)**

```bash
# 1. Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# 2. Installer les dépendances nécessaires
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y

# 3. Ajouter le dépôt officiel Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list

# 4. Installer Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

# 5. Démarrer et activer Docker
sudo systemctl enable docker
sudo systemctl start docker

# 6. Vérifier la version
docker --version
```

#### Sous **Windows 10/11**

1. Téléchargez **Docker Desktop** depuis [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).
2. Lancez l’installation, puis redémarrez votre machine.
3. Activez **WSL2** lors du premier démarrage (obligatoire).
4. Vérifiez ensuite que Docker est actif dans la barre de tâches.
5. Testez avec :

   ```powershell
   docker version
   ```

---

### **Étape 2 — Installer kubectl**

#### Sous **Linux**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

#### Sous **Windows (PowerShell Admin)**

```powershell
choco install kubernetes-cli -y
kubectl version --client
```

---

### **Étape 3 — Installer Minikube**

#### Sous **Linux**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### Sous **Windows**

```powershell
choco install minikube -y
```

---

### **Étape 4 — Démarrer votre premier cluster**

```bash
minikube start --driver=docker --cpus=4 --memory=8192
```

> Le paramètre `--driver=docker` lance le cluster directement dans un conteneur Docker, sans besoin de machine virtuelle.

**Vérifications :**

```bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
```

Sortie attendue :

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.30.0
```

---

### **Étape 5 — Inspection des composants système**

Lister les composants du Control Plane :

```bash
kubectl -n kube-system get pods -o wide
```

Visualiser les logs du kube-apiserver :

```bash
kubectl -n kube-system logs -f kube-apiserver-minikube
```

Lister les Services internes :

```bash
kubectl get svc -A
```

---

### **Étape 6 — Premier Pod de test**

Créer un Pod simple :

```bash
kubectl run nginx-demo --image=nginx --port=80
kubectl get pods
kubectl describe pod nginx-demo
kubectl logs nginx-demo
```

Résultat attendu :

* Le Pod est en statut `Running`.
* Le conteneur NGINX écoute sur le port 80.

---

### **Étape 7 — Nettoyage du TP**

```bash
kubectl delete pod nginx-demo
minikube stop
```

---

### **Étape 8 — Validation du TP**

| Vérification                  | Commande                          | Résultat attendu        |
| ----------------------------- | --------------------------------- | ----------------------- |
| Cluster opérationnel          | `kubectl get nodes`               | STATUS = Ready          |
| Composants kube-system actifs | `kubectl -n kube-system get pods` | Tous Running            |
| Pod test NGINX actif          | `kubectl get pods`                | Running                 |
| Journal NGINX                 | `kubectl logs nginx-demo`         | Affichage des logs HTTP |

---

## **6.4 Résultat et continuité du projet fil rouge**

Vous disposez désormais d’un **cluster Kubernetes complet** exécuté localement.
Ce cluster servira de **plateforme de déploiement** pour le projet fil rouge, où :

* Chapitre 3 : **Configuration et déploiement applicatif (YAML)**
* Chapitre 4 : **Administration et supervision**
* Chapitre 5 : **Sécurité et durcissement**

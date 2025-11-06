# **Chapitre 2 — Architecture de Kubernetes**

*(Control Plane, Nœuds, Réseau, Stockage, Sécurité, Diagnostic)*

---

## **1. Objectifs d’apprentissage**

À la fin de ce chapitre, vous serez capable de :

* Expliquer l’**architecture interne** de Kubernetes (Control Plane et Nœuds).
* Identifier les **composants clés** du cluster et leur rôle.
* Diagnostiquer le **fonctionnement global** du système à l’aide de `kubectl`.
* Comprendre les **flux internes de communication et de sécurité**.
* Mettre en place un **cluster local complet** pour le projet fil rouge.

---

## **2. Vue d’ensemble de l’architecture Kubernetes**

Kubernetes est une plateforme **distribuée** composée de deux ensembles logiques :

1. **Le Control Plane**
   Regroupe les composants responsables de l’**API**, du **stockage d’état**, de la **réconciliation** et de l’**ordonnancement** :

   * **kube-apiserver** : point d’entrée unique. Valide les requêtes, applique AuthN/AuthZ/Admission, et persiste/lit l’état.
   * **etcd** : base **clé/valeur** distribuée (consensus **Raft**) stockant l’état source de vérité.
   * **kube-controller-manager** : exécute des **boucles de contrôle** assurant la convergence vers l’état souhaité (Deployments, Nodes, Jobs, GC…).
   * **kube-scheduler** : **assigne** chaque Pod en attente à un nœud selon ressources et contraintes.

2. **Les Nœuds (Workers)**
   Exécutent les **Pods** et hébergent :

   * **kubelet** : agent local qui reçoit les ordres du Control Plane et orchestre les conteneurs.
   * **Container Runtime** (via **CRI**, ex. containerd/CRI-O) : crée et détruit les conteneurs.
   * **kube-proxy** : programme les règles **L4** (iptables/IPVS) pour la translation des Services.
   * **CNI** : plugin réseau (Calico, Cilium, Flannel, Weave) qui attache des interfaces et attribue des IP aux Pods.

### **Flux internes (schéma mental)**

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
* **Observabilité** : tout passe par l’API → **events**, **logs**, **metrics**.

---

## **3. Control Plane : les composants principaux**

### **3.1 etcd — base clé/valeur (consensus Raft)**

* **Rôle** : stocke l’état complet du cluster (objets API sérialisés).
* **Ports** : 2379 (client API server), 2380 (peer cluster).
* **Quorum** : nombre impair (3/5). Perte de quorum → **écritures impossibles**.
* **Maintenance** : compaction, **defrag**, **sauvegardes régulières** (et tests de restauration).

Avec kubeadm :

* Manifeste : `/etc/kubernetes/manifests/etcd.yaml`
* Données : `/var/lib/etcd`

Commandes :

```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

Bonnes pratiques :

* 3 nœuds etcd dédiés, **disques rapides**, sauvegardes testées.
* **TLS partout**, ports 2379/2380 restreints, rotation des certificats.

---

### **3.2 kube-apiserver — cœur de Kubernetes**

* **Rôle** : reçoit/valide chaque requête, applique **AuthN → AuthZ → Admission**, lit/écrit dans etcd.
* **Extensibilité** : **CRDs**, **API Aggregation**.
* **Audit** : via `--audit-policy-file` et `--audit-log-path`.

Exemples :

```bash
kubectl cluster-info
kubectl -n kube-system get pods -l component=kube-apiserver -o wide
```

---

### **3.3 kube-controller-manager — boucles de réconciliation**

* Orchestre les contrôleurs (Deployment → ReplicaSet → Pods, Node, Service, Job…)
* **Leader Election** : un actif, les autres en attente.

Diagnostic :

```bash
kubectl -n kube-system logs -f kube-controller-manager-<node_name>
kubectl -n kube-system get lease | grep controller
```

---

### **3.4 kube-scheduler — placement des Pods**

* **Rôle** : choisit un nœud pour chaque Pod en `Pending`.
* **Étapes** :

  * Filtrage → contraintes (`taints`, `affinities`, ressources)
  * Notation → scores (`spread`, `preemption`)

Vérification :

```bash
kubectl get events --sort-by=.lastTimestamp | grep -i scheduling
```

---

## **4. Nœuds et exécution des conteneurs**

### **4.1 kubelet — agent du nœud**

* **Rôle** : enregistre le nœud auprès de l’API, gère Pods via CRI.
* **Diagnostics** :

```bash
systemctl status kubelet
kubectl get nodes -o wide
kubectl describe node <node_name>
```

* **Evictions** : mémoire/disque insuffisants.
* **Sécurité** : plugin `NodeRestriction` activé.

---

### **4.2 Container Runtime — containerd / CRI-O**

* Gère les images, conteneurs et journaux.
* Outils : `crictl`, `ctr`.

Exemples :

```bash
crictl ps -a
crictl images
```

---

### **4.3 CNI — réseau des Pods**

* Attribue IP, configure routes et isolation.
* Plugins : Flannel, Calico, Cilium, Weave.

Diagnostic :

```bash
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- sh
```

---

### **4.4 kube-proxy — routage des Services**

* Met en place la translation L4.
* Modes : `iptables` (compat), `ipvs` (performance).

---

## **5. Sécurité et flux AAA**

Toute requête au `kube-apiserver` passe par 3 étapes :

1. **Authentification (AuthN)** — *Qui ?*

   * Certificats X.509, tokens JWT, OIDC.
2. **Autorisation (AuthZ)** — *A-t-il le droit ?*

   * RBAC, Node, Webhook.
3. **Admission** — *Doit-on autoriser ?*

   * Plugins (Mutating/Validating), ex. **PodSecurity**, **OPA Gatekeeper**.

Vérification :

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:viewer -n demo
```

---

## **6. TP – Projet Fil Rouge (Phase 1)**

### **Objectif**

Installer un environnement Kubernetes local complet avec Docker + Minikube + kubectl.
Tester le bon fonctionnement avant de passer au Chapitre 3.

---

### **6.1 Prérequis**

| Élément  | Description                           | Détails           |
| -------- | ------------------------------------- | ----------------- |
| OS       | Windows 10/11 (WSL2) ou Ubuntu/Debian | Mode admin requis |
| CPU      | 4 cœurs min.                          | 8 recommandés     |
| RAM      | 8 Go min.                             | 16 Go recommandés |
| Stockage | 25 Go libres                          | SSD recommandé    |
| Internet | Connexion stable                      | —                 |

---

### **6.2 Étapes d’installation**

#### **Étape 1 — Installer Docker**

**Sous Linux :**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
sudo systemctl enable docker && sudo systemctl start docker
docker --version
```

**Sous Windows :**

Télécharger **Docker Desktop** → installer → activer **WSL2** → vérifier avec :

```powershell
docker version
```

---

#### **Étape 2 — Installer kubectl**

**Linux :**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client
```

**Windows :**

```powershell
choco install kubernetes-cli -y
kubectl version --client
```

---

#### **Étape 3 — Installer Minikube**

**Linux :**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Windows :**

```powershell
choco install minikube -y
```

---

#### **Étape 4 — Démarrer le cluster**

```bash
minikube start --driver=docker --cpus=4 --memory=8192
```

Vérifications :

```bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
```

---

#### **Étape 5 — Premier Pod de test**

```bash
kubectl run nginx-demo --image=nginx --port=80
kubectl get pods
kubectl logs nginx-demo
```

---

#### **Étape 6 — Nettoyage**

```bash
kubectl delete pod nginx-demo
minikube stop
```

---

### **6.3 Validation du TP**

| Vérification  | Commande                          | Résultat attendu   |
| ------------- | --------------------------------- | ------------------ |
| Cluster actif | `kubectl get nodes`               | Ready              |
| kube-system   | `kubectl -n kube-system get pods` | Running            |
| Pod nginx     | `kubectl get pods`                | Running            |
| Logs nginx    | `kubectl logs nginx-demo`         | Logs HTTP visibles |

---

## **7. Dépannage Minikube — erreurs fréquentes**

### **Problème :**

```
X Fermeture en raison de DRV_AS_ROOT : Le pilote "docker" ne doit pas être utilisé avec les privilèges root.
```

### **Cause :**

Le driver Docker ne supporte pas `sudo`.
Tu dois exécuter Minikube avec ton utilisateur normal.

---

### **Problème :**

```
permission denied while trying to connect to the Docker daemon socket
```

### **Cause :**

Ton utilisateur n’a pas accès au daemon Docker.

### **Solution :**

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Puis vérifier :

```bash
docker ps
```

et relancer :

```bash
minikube start --driver=docker
```

---

### **Problème :**

```
PROVIDER_DOCKER_NEWGRP : permission denied while trying to connect to the Docker daemon socket
```

### **Cause :**

Tu as ajouté ton utilisateur au groupe docker mais la session n’a pas été rechargée.

### **Solution :**

Ferme ta session, reconnecte-toi, ou exécute `newgrp docker` avant de relancer `minikube`.

---

### **Problème :**

```
DRV_NOT_HEALTHY : aucun pilote en fonctionnement
```

### **Cause :**

Docker n’est pas actif.

### **Solution :**

Redémarre Docker :

```bash
sudo systemctl restart docker
```

Puis relance :

```bash
minikube delete --all
minikube start --driver=docker
```

---

### **Vérifications finales**

```bash
minikube status
kubectl get nodes
```

Sortie attendue :

```
minikube   Ready    control-plane   2m    v1.30.0
```

---

## **8. Conclusion**

Ce chapitre a permis de :

* Comprendre l’**architecture interne de Kubernetes** (Control Plane + Nœuds).
* Installer un **cluster local fonctionnel** (Docker + Minikube).
* Diagnostiquer les problèmes courants liés aux permissions Docker.
* Préparer la suite du **projet fil rouge** :

  * **Chapitre 3 :** déploiement et configuration d’applications.
  * **Chapitre 4 :** supervision et administration du cluster.

# Chapitre 3 — Installation & environnements de labo

*(minikube, kind, kubeadm — Windows/macOS/Linux — prérequis, installation, vérifications, pièges)*

---

## 1) Objectifs d’apprentissage

* Choisir un **environnement de labo** adapté (minikube, kind, kubeadm) et connaître leurs forces/faiblesses.
* Installer **kubectl** proprement et gérer plusieurs **contexts**.
* Monter un **cluster local** reproductible, avec **CNI**, **Ingress Controller** et **metrics-server**.
* Savoir **vérifier**, **diagnostiquer** et **réinitialiser** un cluster de labo.

---

## 2) Cartographie des options (quand utiliser quoi)

| Option                           | Cas d’usage                                 | Avantages                                                                                | Limites                                                             |
| -------------------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **minikube** (1 nœud par défaut) | Démarrer vite, tests Ingress/Storage, démos | Addons (ingress, dashboard, metrics), drivers variés (Docker, Hyper-V, VirtualBox, None) | Mono-nœud par défaut (multi-nœud possible), moins proche de la prod |
| **kind** (Kubernetes in Docker)  | CI/CD, multi-nœud local simple              | Rapide, fichiers cluster YAML, facile à jeter/recréer                                    | Pas d’add-ons auto, Ingress à installer, nécessite Docker           |
| **kubeadm** (réaliste)           | Lab “comme en prod”, multi-master/worker    | Contrôle fin (CNI/CSI, HA), processus d’init standard                                    | Plus verbeux, demande Linux, swap off, pare-feu/sysctl              |

> Règle simple : **minikube** pour démarrer, **kind** pour CI/multi-nœud rapide, **kubeadm** pour apprendre “le vrai” cluster.

---

## 3) Prérequis système & réseau

### 3.1 Matériel & OS (minima confortables)

* CPU : **4 vCPU** (8+ idéal), RAM : **8 Go** (16+ idéal), Disque : **20 Go** libres pour images.
* OS : Linux, macOS, **Windows 10/11** (Docker Desktop/WSL2 recommandés).

### 3.2 Virtualisation & drivers

* **Windows 10** : activer **WSL2** et/ou **Hyper-V** (Pro/Enterprise).
* **Drivers minikube** : `docker` (recommandé), `hyperv`, `virtualbox`, `none` (root, Linux).
* **kind** : nécessite Docker.

### 3.3 Réseau & pare-feu

* Ouvrir localement les ports du **driver** (Docker Desktop, Hyper-V, VirtualBox).
* Ingress Controller (NGINX) expose un **NodePort** (30000-32767) ou via `minikube tunnel`.

### 3.4 Kernel & sysctl (kubeadm)

* Linux :

  ```bash
  sudo modprobe overlay
  sudo modprobe br_netfilter
  cat <<'SYS' | sudo tee /etc/sysctl.d/99-kubernetes.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  SYS
  sudo sysctl --system
  ```
* **swap off** (kubeadm) : `sudo swapoff -a` et commenter la ligne `swap` dans `/etc/fstab`.

---

## 4) Installer kubectl (et outils utiles)

### 4.1 kubectl (toutes plateformes)

* **Windows** (PowerShell admin) :

  ```powershell
  winget install -e --id Kubernetes.kubectl
  kubectl version --client --output=yaml
  ```
* **macOS** :

  ```bash
  brew install kubectl
  kubectl version --client --output=yaml
  ```
* **Linux (deb-based)** :

  ```bash
  sudo apt-get update && sudo apt-get install -y ca-certificates curl
  curl -fsSL https://dl.k8s.io/release/stable.txt
  VER=$(curl -L -s https://dl.k8s.io/release/stable.txt)
  curl -LO https://dl.k8s.io/release/${VER}/bin/linux/amd64/kubectl
  sudo install -m 0755 kubectl /usr/local/bin/kubectl
  kubectl version --client --output=yaml
  ```

### 4.2 kubectx/kubens (switch rapide)

```bash
# macOS
brew install kubectx
# Linux
git clone https://github.com/ahmetb/kubectx ~/.kubectx && sudo ln -sf ~/.kubectx/kubectx /usr/local/bin/kubectx && sudo ln -sf ~/.kubectx/kubens /usr/local/bin/kubens
```

### 4.3 krew (plugins kubectl) — optionnel, très utile

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS=$(uname | tr '[:upper:]' '[:lower:]'); ARCH=$(uname -m)
  case $ARCH in x86_64) ARCH=amd64;; aarch64) ARCH=arm64;; armv*) ARCH=arm;; *) ARCH=amd64;; esac
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-${OS}_${ARCH}.tar.gz"
  tar zxvf krew-${OS}_${ARCH}.tar.gz
  ./"krew-${OS}_${ARCH}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew install ctx ns neat images sniff df-pv
```

---

## 5) Option A — **minikube** (recommandé pour démarrer)

### 5.1 Installation

* **Windows 10** :

  ```powershell
  winget install -e --id Docker.DockerDesktop   # Docker Desktop (WSL2)
  winget install -e --id Kubernetes.minikube
  minikube version
  ```
* **macOS** :

  ```bash
  brew install --cask docker
  brew install minikube
  ```
* **Linux** :

  ```bash
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  sudo install minikube-linux-amd64 /usr/local/bin/minikube
  ```

### 5.2 Créer le cluster

* **Driver Docker** (le plus simple) :

  ```bash
  minikube start --driver=docker --cpus=4 --memory=8192 --kubernetes-version=stable
  kubectl get nodes -o wide
  ```
* **Multi-nœud** :

  ```bash
  minikube start --driver=docker --nodes=3 --cpus=2 --memory=4096
  ```

### 5.3 Addons utiles

```bash
minikube addons enable metrics-server
minikube addons enable ingress           # NGINX Ingress Controller
minikube addons enable dashboard         # (optionnel)
```

### 5.4 Test rapide

```bash
kubectl create deploy web --image=nginx:1.27 --replicas=2
kubectl expose deploy web --port=80 --target-port=80 --type=NodePort
minikube service web --url
```

> Si vous avez activé `ingress`, créez un Ingress et utilisez `minikube tunnel` pour une IP locale.

### 5.5 Nettoyage

```bash
kubectl delete deploy web svc web
minikube delete
```

**Pièges minikube**

* Driver absent → installer Docker Desktop/Hyper-V/VirtualBox.
* `minikube tunnel` nécessite sudo (création d’interface réseau).

---

## 6) Option B — **kind** (multi-nœud rapide pour CI/Dev)

### 6.1 Installation

* **Windows** :

  ```powershell
  winget install -e --id Docker.DockerDesktop
  iwr -useb https://kind.sigs.k8s.io/dl/latest/kind-windows-amd64 | Out-File -FilePath $env:TEMP\kind.exe -Encoding byte
  Move-Item $env:TEMP\kind.exe $env:ProgramFiles\kind.exe
  $env:Path += ";$env:ProgramFiles"
  kind --version
  ```
* **macOS** : `brew install kind`
* **Linux** :

  ```bash
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
  chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
  ```

### 6.2 Cluster multi-nœud avec Ingress

Créez `kind.yaml` :

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

Créer le cluster :

```bash
kind create cluster --config kind.yaml
kubectl get nodes -o wide
```

Installer **NGINX Ingress** (version “bare-metal” pour kind) :

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller
```

Déployer un test :

```bash
kubectl create deploy web --image=nginx:1.27
kubectl expose deploy web --port=80 --target-port=80
cat <<'YAML' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations: { nginx.ingress.kubernetes.io/rewrite-target: / }
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: web, port: { number: 80 } }
YAML
```

Ajoutez à `/etc/hosts` (ou `C:\Windows\System32\drivers\etc\hosts`) :

```
127.0.0.1 web.local
```

Test : `curl -I http://web.local/`

### 6.3 Nettoyage

```bash
kind delete cluster
```

**Pièges kind**

* Oublier le `extraPortMappings` → pas d’accès 80/443 depuis l’hôte.
* Attendre le rollout complet d’ingress-nginx avant les tests.

---

## 7) Option C — **kubeadm** (cluster réaliste)

> Chemin Linux. Idéal en VM(s) locales (Proxmox/VirtualBox/Hyper-V) ou VPS.

### 7.1 Préparation (tous les nœuds)

* **Désactiver swap** :

  ```bash
  sudo swapoff -a
  sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
  ```

* **Modules & sysctl** (cf. §3.4)

* **Container runtime** (containerd recommandé) :

  ```bash
  # Debian/Ubuntu (exemple rapide)
  sudo apt-get update && sudo apt-get install -y containerd
  sudo mkdir -p /etc/containerd
  containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
  sudo systemctl enable --now containerd
  ```

  Vérifier **cgroup driver** : `SystemdCgroup = true` pour alignement kubelet →
  dans `/etc/containerd/config.toml`, section `plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options`.

* **kubeadm, kubelet, kubectl** :

  ```bash
  sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
  sudo mkdir -p /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
  sudo systemctl enable --now kubelet
  ```

### 7.2 Init control-plane (nœud master)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# (CIDR compatible Flannel ; cliquez selon votre CNI)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 7.3 Installer un **CNI**

* **Flannel** (simple) :

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml
  ```
* **Calico** (features réseau avancées) :

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
  ```

> Attendre que `coredns` et les pods CNI soient `Running`.

### 7.4 Joindre des **workers** (sur chaque worker)

Utilisez la commande join fournie par `kubeadm init` (exemple) :

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Vérifiez :

```bash
kubectl get nodes -o wide
```

### 7.5 Ingress Controller (ex. nginx)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller
```

Exposez via LoadBalancer (cloud) ou **MetalLB** on-prem :

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
# Configurez un pool d'adresses IP locales (Layer2) -> CR "IPAddressPool" + "L2Advertisement"
```

### 7.6 metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top nodes
kubectl top pods -A
```

### 7.7 Réinitialiser (wipe)

```bash
sudo kubeadm reset -f
sudo systemctl restart containerd
sudo rm -rf ~/.kube
```

**Pièges kubeadm**

* Oublier **swap off** → kubelet refuse de démarrer.
* **CNI** absent → Pods en `Pending` (No CNI).
* MTU **VXLAN** non adapté → connexions qui “pendouillent” (voir CNI doc).
* **Horloge** non synchronisée → problèmes de certificats/élection de leader.

---

## 8) Vérifications post-installation (toutes options)

```bash
# 1) Nœuds et pods système
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide

# 2) DNS interne
kubectl run -it netshoot --image=nicolaka/netshoot --rm --restart=Never -- \
  sh -lc "dig A kubernetes.default.svc.cluster.local +short && nslookup kubernetes"

# 3) Deploiement minimal + Service
kubectl create deploy demo --image=nginx:1.27 --replicas=2
kubectl expose deploy demo --port=80 --target-port=80
kubectl get svc demo
kubectl get endpoints demo

# 4) Ingress (si installé) + /etc/hosts (web.local -> IP Ingress)
```

---

## 9) Gestion des contexts & namespaces

```bash
# Lister les contexts / utiliser un context
kubectl config get-contexts
kubectl config use-context <ctx>

# Fixer un namespace par défaut
kubectl config set-context --current --namespace=demo

# Plugins (krew)
kubectl ctx      # switch de context
kubectl ns       # switch de namespace
```

---

## 10) Dépannage (symptômes → commandes → corrections)

* **Pods bloqués Pending** → CNI absent/KO, quotas, Scheduling :

  ```bash
  kubectl describe pod <name>
  kubectl get events --sort-by=.lastTimestamp | tail -n 30
  ```

  *Fix* : installer/soigner CNI, libérer ressources, corriger affinity/taints.

* **ImagePullBackOff** → image introuvable/privée :

  ```bash
  kubectl describe pod <name> | sed -n '/Events:/,$p'
  ```

  *Fix* : tag correct, `imagePullSecrets`.

* **Node NotReady** → kubelet/CRI/cni :

  ```bash
  kubectl describe node <node>
  journalctl -u kubelet -f
  crictl ps -a; crictl images
  ```

* **Ingress ne répond pas** :

  ```bash
  kubectl -n ingress-nginx get pods -o wide
  kubectl describe ingress <name>
  ```

  *Fix* : attendre rollout, vérifier hosts, `minikube tunnel` (minikube), MetalLB (kubeadm).

---

## 11) Aide-mémoire (création & suppression rapides)

### minikube

```bash
minikube start --driver=docker --cpus=4 --memory=8192
minikube addons enable ingress
minikube addons enable metrics-server
# ...
minikube delete
```

### kind

```bash
kind create cluster --config kind.yaml
kubectl apply -f deploy-ingress-nginx.yaml
# ...
kind delete cluster
```

### kubeadm

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
kubectl apply -f <CNI>.yaml
# ...
sudo kubeadm reset -f
```

---

## 12) Bonnes pratiques de labo

* **Scriptable** : gardez vos commandes/manifestes dans un repo (reproductibilité).
* **Versions** : figez les versions Kubernetes et des add-ons (ingress, metrics).
* **Nettoyage régulier** : `kind delete cluster` / `minikube delete` / `kubeadm reset` entre grosses manipulations.
* **Observabilité** : activez **metrics-server**, testez `kubectl top`, regardez les **Events**.
* **Sécurité** : même en labo, évitez `:latest`, utilisez des **namespaces**, **labels** propres.


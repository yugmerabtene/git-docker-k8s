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

*(Section identique à la version précédente, conservée intégralement pour cohérence de cours)*

---

## 3. Control Plane : les composants principaux

*(Section identique à la version précédente, détaillant etcd, kube-apiserver, controller-manager, scheduler.)*

---

## 4. Nœuds et exécution des conteneurs

*(Section identique à la version précédente, détaillant kubelet, runtime, CNI, kube-proxy.)*

---

## 5. Sécurité et flux AAA

*(Section identique à la version précédente.)*

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

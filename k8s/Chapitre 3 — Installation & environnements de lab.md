# Chapitre 3 — Installation et configuration de Kubernetes

*(Outils, manifests YAML, déploiement d’applications, gestion des ressources)*

---

## 1. Objectifs d’apprentissage

À la fin de ce chapitre, l’apprenant sera capable de :

* Installer, configurer et administrer Kubernetes dans un environnement local.
* Créer, appliquer et comprendre les **manifests YAML** (Pods, Deployments, Services).
* Gérer les ressources des conteneurs (CPU, mémoire, stockage).
* Utiliser `kubectl` pour déployer, inspecter et mettre à jour les applications.
* Étendre le projet fil rouge en déployant une **application web complète** (frontend + backend).

---

## 2. Présentation des différentes solutions d’installation

Kubernetes peut être installé de plusieurs manières selon le contexte :

### 2.1 Environnement local (développement)

* **Minikube** : cluster à nœud unique, léger, parfait pour l’apprentissage.
* **Kind (Kubernetes in Docker)** : exécute un cluster Kubernetes dans des conteneurs Docker.
* **MicroK8s** : distribution légère développée par Canonical (Ubuntu).

### 2.2 Environnement sur site (on-premise)

* **kubeadm** : outil officiel pour installer un cluster “from scratch”.
* **Rancher** : interface graphique complète pour clusters multi-nœuds.
* **OpenShift** : distribution commerciale de Red Hat basée sur Kubernetes.

### 2.3 Environnements Cloud (managés)

* **EKS (AWS)**, **AKS (Azure)**, **GKE (Google Cloud)** : infrastructure et plan de contrôle gérés automatiquement.
* Ces solutions sont prêtes à l’emploi mais souvent payantes.

---

## 3. Installation des outils de base

*(Si vous avez déjà terminé le TP du Chapitre 2, votre environnement est prêt. Sinon, voici la procédure complète pour installation de zéro.)*

### 3.1 Docker

*(Même installation que Chapitre 2, section Docker, à répéter si besoin.)*
Docker sert de **runtime de conteneur** utilisé par Minikube ou Kind.

### 3.2 kubectl

* Client en ligne de commande pour communiquer avec le cluster.
* Fonctionne en HTTPS via **API Server**.

Vérifiez la configuration :

```bash
kubectl config view
kubectl cluster-info
```

### 3.3 Minikube

Si vous souhaitez recréer un environnement propre :

```bash
minikube delete
minikube start --driver=docker --cpus=4 --memory=8192
```

---

## 4. Configuration et manipulation de base

### 4.1 Syntaxe YAML

Kubernetes utilise des **fichiers YAML** pour décrire l’état souhaité du système.
Structure de base :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monpod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Explication :

* **apiVersion** : version de l’API (ex. `v1`, `apps/v1`).
* **kind** : type d’objet (Pod, Service, Deployment…).
* **metadata** : nom, labels, namespace.
* **spec** : définition fonctionnelle (conteneurs, volumes, ports…).

---

### 4.2 Les principaux objets Kubernetes

| Objet                           | Description                                                      |
| ------------------------------- | ---------------------------------------------------------------- |
| **Pod**                         | Unité minimale d’exécution contenant un ou plusieurs conteneurs. |
| **Deployment**                  | Gère les Pods, permet les mises à jour continues et le scaling.  |
| **Service**                     | Expose des Pods sur le réseau interne ou externe.                |
| **ConfigMap / Secret**          | Stockage de configuration et d’informations sensibles.           |
| **PersistentVolumeClaim (PVC)** | Réservation de stockage persistant.                              |

---

### 4.3 Commandes `kubectl` essentielles

| Action             | Commande                         |
| ------------------ | -------------------------------- |
| Créer un objet     | `kubectl apply -f fichier.yaml`  |
| Supprimer un objet | `kubectl delete -f fichier.yaml` |
| Lister les objets  | `kubectl get pods,svc,deploy -A` |
| Inspecter un objet | `kubectl describe pod <nom>`     |
| Voir les logs      | `kubectl logs <nom>`             |
| Accéder à un Pod   | `kubectl exec -it <nom> -- bash` |

---

## 5. Gestion des ressources

### 5.1 Allocation CPU et mémoire

Exemple dans un conteneur :

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

* **requests** : ressources minimales nécessaires.
* **limits** : ressources maximales autorisées.
* Kubernetes planifie les Pods en fonction des *requests* pour éviter la saturation.

### 5.2 Gestion du stockage

Les volumes peuvent être :

* **éphémères** : supprimés avec le Pod.
* **persistants (PV/PVC)** : conservés entre les redémarrages.

---

## 6. **TP – Projet Fil Rouge (Phase 2)**

### Déploiement complet d’une application web (frontend + backend simulé)

---

### 6.1 Objectif

Poursuivre le projet du Chapitre 2 :

* Déployer une application composée de deux services :

  1. **Frontend (Nginx)** simulant l’interface utilisateur.
  2. **Backend (API simulée avec httpd)**.
* Connecter les deux à l’aide de Services internes.
* Exposer le frontend via un **Ingress HTTP** local.

---

### 6.2 Prérequis

* Avoir le cluster Minikube fonctionnel (Chapitre 2).
* Docker, kubectl et Minikube opérationnels.
* Avoir 8 Go de RAM et 4 vCPU libres minimum.

---

### 6.3 Étape 1 — Vérification du cluster

```bash
minikube start --driver=docker
kubectl get nodes -o wide
kubectl get pods -A
```

---

### 6.4 Étape 2 — Créer un namespace dédié au projet (si ce n'est pas déja fait au chapitre-01)

```bash
kubectl create namespace projet-fil-rouge
kubectl config set-context --current --namespace=projet-fil-rouge
```

---

### 6.5 Étape 3 — Créer le backend (API simulée)

Créer le fichier `backend.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: httpd:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

Appliquer :

```bash
kubectl apply -f backend.yaml
kubectl get pods,svc
```

---

### 6.6 Étape 4 — Créer le frontend (Nginx)

Créer le fichier `frontend.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_URL
          value: "http://backend"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
```

Appliquer :

```bash
kubectl apply -f frontend.yaml
```

---

### 6.7 Étape 5 — Exposer le frontend avec Ingress

Activer Ingress :

```bash
minikube addons enable ingress
```

Créer `ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: local.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

Appliquer :

```bash
kubectl apply -f ingress.yaml
```

Tester :

```bash
minikube tunnel
curl -I http://local.dev
```

---

### 6.8 Étape 6 — Vérification du déploiement

```bash
kubectl get all -n projet-fil-rouge
kubectl describe ingress web-ingress
kubectl logs -l app=frontend
kubectl logs -l app=backend
```

**Résultat attendu :**

* Les deux Pods sont en statut **Running**.
* L’Ingress répond sur `http://local.dev`.
* Les logs du backend montrent les requêtes du frontend.

---

### 6.9 Étape 7 — Nettoyage (facultatif)

```bash
kubectl delete namespace projet-fil-rouge
```

---

## 7. Résumé pour diapo

### 1️⃣ Objectifs

* Installer et configurer un cluster Kubernetes complet.
* Créer et appliquer des **manifests YAML**.
* Déployer une **application web complète**.

### 2️ Outils

* Docker, Minikube, kubectl.
* API Server, Controller, Scheduler.
* YAML pour la configuration déclarative.

### 3️ Concepts clés

* **Pod, Deployment, Service, Ingress**.
* **requests/limits** pour la gestion des ressources.
* Configuration centralisée via **namespaces**.

### 4️ TP Fil Rouge – Phase 2

* Création d’un **namespace dédié**.
* Déploiement d’un **frontend Nginx** et d’un **backend HTTPD**.
* Exposition du frontend via un **Ingress HTTP local**.
* Résultat : **application web fonctionnelle** sur `http://local.dev`.


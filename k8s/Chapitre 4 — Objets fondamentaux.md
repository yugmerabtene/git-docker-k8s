# Chapitre 4 — Administration et supervision de Kubernetes

*(Supervision, logs, métriques, scaling, mises à jour continues)*

---

## 1. Objectifs d’apprentissage

À la fin de ce chapitre, l’apprenant sera capable de :

* Administrer un cluster Kubernetes en production (inspection, supervision, journalisation).
* Analyser les **logs**, surveiller les **métriques** et détecter les anomalies.
* Gérer les **mises à jour continues** des applications (Rolling Update / Rollback).
* Mettre en œuvre le **scaling automatique** des Pods selon la charge.
* Poursuivre le **projet fil rouge** avec un environnement supervisé et auto-scalable.

---

## 2. Introduction à l’administration du cluster

### 2.1 Notions fondamentales

* Kubernetes repose sur un **contrôle continu** : chaque objet est surveillé par un contrôleur.
* L’administration consiste à :

  * Vérifier l’état du cluster et des nœuds.
  * Superviser les Pods et les Services.
  * Gérer les ressources (CPU, mémoire, stockage).
  * Surveiller la santé des applications.

### 2.2 Commandes de base d’administration

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl top nodes
kubectl top pods
kubectl describe pod <nom>
kubectl get events --sort-by=.lastTimestamp | tail
```

---

## 3. Supervision et journalisation

### 3.1 Les métriques

* Les métriques représentent l’utilisation **CPU**, **mémoire**, **réseau** et **stockage**.
* Outil intégré : **metrics-server**.
* Les métriques alimentent :

  * les commandes `kubectl top`,
  * les règles d’autoscaling (HPA),
  * les tableaux de bord (Prometheus, Grafana).

**Installation du metrics-server :**

```bash
minikube addons enable metrics-server
```

Vérification :

```bash
kubectl top nodes
kubectl top pods
```

---

### 3.2 Les journaux (logs)

* Chaque Pod produit des logs consultables avec :

  ```bash
  kubectl logs <nom_du_pod>
  ```
* Pour des applications multi-conteneurs :

  ```bash
  kubectl logs <nom_du_pod> -c <nom_du_conteneur>
  ```
* Pour les composants système :

  ```bash
  kubectl -n kube-system logs <pod>
  ```
* Pour un suivi continu :

  ```bash
  kubectl logs -f <nom_du_pod>
  ```

---

### 3.3 Les événements

* Kubernetes enregistre les événements du cluster :

  ```bash
  kubectl get events --sort-by=.metadata.creationTimestamp
  ```
* Types d’événements :

  * `Scheduled` : Pod assigné à un nœud.
  * `Pulled` : image téléchargée.
  * `Started` / `Killing` : cycle de vie.
  * `FailedScheduling` : ressource indisponible.

---

## 4. Rolling Update et Rollback

### 4.1 Principe

* Les **Rolling Updates** permettent de **mettre à jour une application sans interruption** :

  * les nouveaux Pods démarrent progressivement,
  * les anciens sont arrêtés une fois les nouveaux disponibles.
* En cas d’échec, un **Rollback** restaure la version précédente.

### 4.2 Paramètres importants

Dans un `Deployment` :

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

### 4.3 Commandes associées

```bash
# Mettre à jour une image
kubectl set image deployment/frontend nginx=nginx:1.25

# Suivre la progression
kubectl rollout status deployment/frontend

# Historique des déploiements
kubectl rollout history deployment/frontend

# Revenir à la version précédente
kubectl rollout undo deployment/frontend
```

---

## 5. Autoscaling (mise à l’échelle automatique)

### 5.1 Types de scalabilité

* **HPA (Horizontal Pod Autoscaler)** : augmente le nombre de Pods.
* **VPA (Vertical Pod Autoscaler)** : ajuste les ressources (CPU/RAM) d’un Pod.
* **Cluster Autoscaler** : ajoute ou retire des nœuds.

### 5.2 Création d’un HPA

```bash
kubectl autoscale deployment frontend --cpu-percent=50 --min=1 --max=5
```

### 5.3 Vérification

```bash
kubectl get hpa
kubectl describe hpa frontend
```

### 5.4 Simulation de charge

```bash
kubectl run loader --image=busybox --restart=Never -- /bin/sh -c \
"while true; do wget -q -O- http://frontend; done"
```

> Kubernetes crée automatiquement de nouveaux Pods quand la charge CPU dépasse 50 %.

---

## 6. LAB – Projet Fil Rouge (Phase 3)

### Supervision, mise à jour et autoscaling de l’application web

---

### 6.1 Objectif

* Étendre l’application déployée au **Chapitre 3**.
* Ajouter :

  * la **supervision (metrics-server)**,
  * une **mise à jour continue** du frontend,
  * un **autoscaling dynamique**.

---

### 6.2 Prérequis

* Cluster Minikube opérationnel avec :

  * **frontend (Nginx)**,
  * **backend (HTTPD)**,
  * **Ingress activé**.
* Docker, kubectl, et metrics-server installés :

  ```bash
  minikube addons enable metrics-server
  ```

---

### 6.3 Étape 1 — Vérification initiale

```bash
kubectl get all -n projet-fil-rouge
kubectl top pods -n projet-fil-rouge
```

---

### 6.4 Étape 2 — Mise à jour continue (Rolling Update)

Mettre à jour le **frontend** vers une nouvelle version :

```bash
kubectl set image deployment/frontend nginx=nginx:1.27
kubectl rollout status deployment/frontend
kubectl rollout history deployment/frontend
```

> Vérifiez que le service reste accessible sur `http://local.dev`.

---

### 6.5 Étape 3 — Création d’un autoscaler

```bash
kubectl autoscale deployment frontend --cpu-percent=50 --min=1 --max=5
```

Vérification :

```bash
kubectl get hpa -n projet-fil-rouge
```

---

### 6.6 Étape 4 — Simulation de charge

Lancer un Pod générant du trafic :

```bash
kubectl run loadtest --image=busybox --restart=Never -- /bin/sh -c \
"while true; do wget -q -O- http://frontend; done"
```

Observer l’évolution :

```bash
kubectl get hpa -w
kubectl get pods -l app=frontend
```

> Vous verrez progressivement apparaître de nouveaux Pods lorsque la charge CPU augmente.

---

### 6.7 Étape 5 — Observation et supervision

```bash
kubectl top pods
kubectl get events --sort-by=.metadata.creationTimestamp | tail
```

Pour afficher graphiquement :

```bash
minikube dashboard
```

---

### 6.8 Étape 6 — Nettoyage

```bash
kubectl delete pod loadtest
kubectl delete hpa frontend
```

---

### 6.9 Résultats attendus

* L’application supporte les montées en charge sans interruption.
* Les Pods se créent et se détruisent automatiquement.
* Le Rolling Update s’effectue sans perte de disponibilité.
* Les métriques et événements sont consultables.

---

## 7. Bonnes pratiques d’administration

* Mettre à jour régulièrement Kubernetes et les images.
* Surveiller les quotas et limites de ressources.
* Utiliser des **namespaces** pour isoler les environnements.
* Sauvegarder les manifests YAML versionnés dans Git.
* Mettre en place un **système d’alerte** (Prometheus, Grafana, Alertmanager).

---

## 8. Résumé pour diapo

### 1. Objectifs

* Administrer et superviser un cluster Kubernetes.
* Analyser les logs, métriques et événements.
* Gérer les mises à jour continues et l’autoscaling.

### 2. Outils utilisés

* `kubectl`, `metrics-server`, `minikube dashboard`.
* Composants : API Server, Controller Manager, Scheduler.

### 3. Concepts clés

* **Rolling Update** : mise à jour sans interruption.
* **Rollback** : retour à la version précédente.
* **HPA** : autoscaling horizontal.
* **Metrics Server** : collecte des métriques CPU/mémoire.

### 4. LAB – Phase 3 du projet fil rouge

* Ajout du **metrics-server**.
* Mise à jour du **frontend Nginx** sans coupure.
* Activation du **scaling automatique** selon la charge CPU.
* Résultat : application auto-scalable, supervisée et tolérante aux montées en charge.



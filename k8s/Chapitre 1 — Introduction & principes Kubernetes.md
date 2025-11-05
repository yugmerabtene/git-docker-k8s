# Chapitre 1 – Introduction à Kubernetes

---

## 1. Présentation de Kubernetes et origine du projet

### 1.1 Définition générale

* **Kubernetes** (abrégé **K8s**) est une **plateforme open-source d’orchestration de conteneurs**, conçue pour **automatiser le déploiement, la mise à l’échelle et la maintenance** d’applications exécutées dans des conteneurs.
* Le terme **orchestration** fait référence à la **coordination automatique** de multiples conteneurs répartis sur plusieurs serveurs physiques ou virtuels.
* Kubernetes agit comme un **système d’exploitation pour le cloud** : il gère les ressources, le réseau, le stockage et la résilience des applications.

### 1.2 Origine et historique

* Créé par **Google** en **2014**, Kubernetes est issu d’un projet interne nommé **Borg**, développé au début des années 2000 pour gérer les millions de conteneurs de services comme Gmail, YouTube et Google Search.
* En **2015**, Google confie la gestion du projet à la **Cloud Native Computing Foundation (CNCF)**, une organisation à but non lucratif soutenue par la **Linux Foundation**.
* La CNCF est chargée de garantir la **neutralité**, la **pérennité** et la **gouvernance communautaire** du projet.
* Depuis, Kubernetes est devenu **le standard mondial de l’orchestration de conteneurs**, adopté par toutes les grandes entreprises technologiques.

### 1.3 Objectif fondamental

Kubernetes vise à :

* **Simplifier** la gestion des applications conteneurisées.
* **Automatiser** le déploiement et la maintenance.
* **Optimiser** l’utilisation des ressources matérielles (CPU, mémoire, stockage).
* **Assurer la résilience** et la haute disponibilité des services.
* **Offrir une portabilité** entre les environnements (on-premise, cloud public, cloud privé).

### 1.4 Signification du nom

* Le mot **Kubernetes** vient du grec *κυβερνήτης* (*kubernêtês*) qui signifie **“pilote de navire”** ou **“gouvernail”**.
* Ce terme illustre le rôle de Kubernetes : **piloter et diriger les conteneurs** pour maintenir le cap (l’état souhaité du système).
* Le **logo** de Kubernetes représente une **barre de gouvernail à sept branches**, chacune symbolisant un aspect fondamental du système.

---

## 2. Fonctionnalités : automatisation des déploiements et de la maintenance des applications en conteneurs

Kubernetes propose un ensemble de fonctionnalités avancées destinées à **automatiser la gestion complète du cycle de vie des applications**.

### 2.1 Automatisation du déploiement

* Définir des **fichiers de configuration déclaratifs (YAML)** qui décrivent l’état souhaité du système.
* Kubernetes compare en permanence **l’état réel** du cluster à **l’état souhaité**, et exécute les actions nécessaires pour les aligner.
* Cela permet des **déploiements reproductibles** et **cohérents**, sans intervention manuelle.

### 2.2 Gestion des mises à jour (Rolling Updates)

* Kubernetes gère les **mises à jour progressives** des applications sans interruption de service.
* Il remplace les anciens conteneurs par de nouveaux de manière contrôlée, garantissant la **continuité du service**.
* Possibilité de **rollback** (retour arrière) en cas d’erreur lors d’un déploiement.

### 2.3 Mise à l’échelle automatique (Autoscaling)

* Ajustement automatique du nombre de conteneurs en fonction de la **charge CPU, mémoire ou métriques personnalisées**.
* Trois types de scalabilité :

  * **Horizontal (HPA – Horizontal Pod Autoscaler)** : ajuste le nombre de Pods.
  * **Vertical (VPA – Vertical Pod Autoscaler)** : ajuste les ressources allouées à chaque Pod.
  * **Cluster Autoscaler** : ajoute ou retire des nœuds du cluster selon la demande.

### 2.4 Auto-réparation (Self-Healing)

* Kubernetes **redémarre automatiquement** les conteneurs ou Pods en échec.
* Si un nœud devient indisponible, les Pods sont **réassignés** à un autre nœud disponible.
* L’objectif est d’assurer une **disponibilité maximale** sans intervention humaine.

### 2.5 Répartition de charge et découverte de services

* Kubernetes fournit un **Service Discovery** interne : chaque application est accessible via un **nom DNS interne** et un **load balancer** intégré.
* Les requêtes des utilisateurs sont automatiquement distribuées entre les différents Pods disponibles.

### 2.6 Gestion de la configuration et des secrets

* Utilisation de **ConfigMaps** pour stocker des configurations applicatives non sensibles.
* Utilisation de **Secrets** pour stocker des informations confidentielles (mots de passe, clés API).
* Ces éléments sont injectés dans les conteneurs à l’exécution, sans avoir à modifier les images.

### 2.7 Gestion du stockage persistant

* Kubernetes permet de rattacher des **volumes persistants (Persistent Volumes – PV)** aux Pods.
* Les données sont conservées même si le Pod est supprimé ou redémarré.
* Utilisation de **Persistent Volume Claims (PVC)** pour demander de l’espace de stockage dynamiquement.

### 2.8 Observabilité et supervision

* Intégration native avec des outils de **monitoring (Prometheus, Metrics-Server)** et de **logging (Fluentd, Elasticsearch, Kibana)**.
* Suivi des métriques système (CPU, RAM, réseau) et applicatives pour une **supervision en temps réel**.

---

## 3. Conteneurs supportés et plateformes utilisant Kubernetes

### 3.1 Runtimes compatibles

Kubernetes est indépendant du moteur de conteneur utilisé grâce à la **CRI (Container Runtime Interface)**, une interface standard qui permet de communiquer avec différents environnements d’exécution.

Principaux runtimes pris en charge :

* **containerd** : runtime léger et rapide (issu du projet Docker).
* **CRI-O** : runtime développé spécifiquement pour Kubernetes, conforme à la spécification **OCI (Open Container Initiative)**.
* **Docker Engine** : longtemps utilisé, mais remplacé par containerd depuis la version 1.24 de Kubernetes.
* **gVisor / Kata Containers** : runtimes sécurisés utilisant la virtualisation légère pour isoler les conteneurs.

### 3.2 Plateformes d’exécution

Kubernetes fonctionne sur plusieurs environnements, qu’ils soient **locaux**, **on-premise**, ou **dans le cloud** :

* **Local :**

  * **Minikube** : outil pour créer un cluster Kubernetes local sur une seule machine (idéal pour les tests).
  * **kind (Kubernetes IN Docker)** : exécute un cluster Kubernetes à l’intérieur de conteneurs Docker.
  * **MicroK8s** : distribution légère de Canonical (Ubuntu) pour les environnements de développement.

* **On-Premise (infrastructure interne) :**

  * Installation manuelle via **kubeadm**.
  * Outils de gestion comme **Rancher**, **OpenShift** ou **VMware Tanzu**.

* **Cloud :**

  * **AWS EKS (Elastic Kubernetes Service)**
  * **Azure AKS (Azure Kubernetes Service)**
  * **Google GKE (Google Kubernetes Engine)**
  * **IBM Cloud Kubernetes Service**, **Oracle OKE**, **Alibaba ACK**, etc.

Ces services cloud offrent des **clusters managés** : l’infrastructure et la maintenance sont prises en charge par le fournisseur.

---

## 4. Composants de Kubernetes

Kubernetes repose sur une **architecture distribuée** et se compose de plusieurs éléments essentiels interconnectés.
Chaque composant joue un rôle précis dans la gestion, le contrôle et l’exécution des applications.

### 4.1 Les composants du plan de contrôle (Control Plane)

* **API Server** : point d’entrée du cluster. Reçoit les requêtes `kubectl` ou API REST, valide les configurations et interagit avec les autres composants.
* **etcd** : base de données clé/valeur distribuée qui stocke l’état et la configuration du cluster.
* **Controller Manager** : surveille l’état du cluster et applique les actions nécessaires pour atteindre l’état souhaité.
* **Scheduler** : assigne les Pods aux nœuds en fonction des ressources disponibles.
* **Cloud Controller Manager** : interface entre Kubernetes et les services cloud (EBS, Load Balancer, etc.).

### 4.2 Les composants des nœuds (Node Components)

* **Kubelet** : agent installé sur chaque nœud, responsable du lancement et du suivi des conteneurs.
* **Kube-Proxy** : gère la connectivité réseau et la répartition du trafic entre Pods.
* **Container Runtime** : environnement d’exécution des conteneurs (Docker, containerd, CRI-O).

---

## 5. Définitions essentielles

### 5.1 Pod

* Un **Pod** est la plus petite unité déployable de Kubernetes.
* Il contient un ou plusieurs **conteneurs** partageant :

  * le même **espace réseau** (même adresse IP),
  * le même **stockage temporaire** (volumes partagés),
  * et le même **cycle de vie**.
* Chaque Pod représente une **instance unique d’une application**.

### 5.2 Label

* Un **Label** est une **clé/valeur attachée à un objet Kubernetes** (Pod, Service, Deployment…).
* Il sert à identifier, filtrer et regrouper des objets.
* Exemple d’usage : sélectionner tous les Pods ayant `app=frontend`.

### 5.3 Controller

* Un **Controller** est un composant qui **surveille en permanence l’état du cluster** et agit pour maintenir l’état souhaité.
* Exemples de contrôleurs :

  * **Deployment** : gère les déploiements d’applications.
  * **ReplicaSet** : maintient un nombre fixe de Pods.
  * **DaemonSet** : exécute un Pod sur chaque nœud.
  * **StatefulSet** : gère les applications avec état (bases de données, stockage).

### 5.4 Service

* Un **Service** est un objet qui **expose un ensemble de Pods sur le réseau**.
* Il fournit un **point d’accès stable (adresse IP fixe et nom DNS)** malgré la nature éphémère des Pods.
* Types de Services :

  * **ClusterIP** : accessible uniquement à l’intérieur du cluster.
  * **NodePort** : exposé via un port de chaque nœud.
  * **LoadBalancer** : utilise un équilibreur de charge externe (cloud).

---

## 6. Conclusion du Chapitre 1

À la fin de ce chapitre, les apprenants doivent être capables de :

* Expliquer ce qu’est Kubernetes et ses objectifs.
* Décrire son origine et sa gouvernance (Google, CNCF).
* Identifier ses principales fonctionnalités d’automatisation.
* Reconnaître les environnements et conteneurs supportés.
* Citer et définir les principaux composants (Pod, Label, Controller, Service).

---

Souhaites-tu que je rédige maintenant le **Chapitre 2 – Architecture** dans le même format (listes détaillées, explications complètes, sans code) pour poursuivre la continuité du cours ?

# Présentation & Origine de Kubernetes

---

## 1. Définition générale

* **Kubernetes** (souvent abrégé en **K8s**, car il y a 8 lettres entre le "K" et le "s") est une **plateforme open-source d’orchestration de conteneurs**.
* Le terme **orchestration** désigne la **gestion automatisée et coordonnée** du cycle de vie d’applications exécutées dans des conteneurs.
* L’objectif fondamental de Kubernetes est de **déployer, gérer et maintenir des applications conteneurisées** de manière **automatique**, **scalable** (extensible), et **résiliente** (capable de se rétablir en cas d’incident).

---

## 2. Origine du projet

### a. Création et contexte

* Développé initialement par **Google** en **2014**, Kubernetes est le **successeur conceptuel du projet interne “Borg”**, utilisé en interne chez Google depuis les années 2000.
* Borg était un **système d’orchestration propriétaire** qui permettait à Google de gérer **des millions de conteneurs** à travers ses datacenters.
* Kubernetes a été conçu à partir des **principes et enseignements** tirés de Borg, mais réécrit en **open-source** pour être utilisé par toute la communauté.

### b. Chronologie synthétique

* **2003-2013 :** Google utilise Borg en interne pour orchestrer ses propres services (Gmail, YouTube, etc.).
* **2014 :** Publication du projet Kubernetes en open-source sur GitHub.
* **2015 :** Transfert officiel de la gouvernance du projet à la **Cloud Native Computing Foundation (CNCF)**, organisme à but non lucratif chargé de garantir son développement neutre et communautaire.
* **Depuis 2016 :** Adoption massive par l’industrie, intégration dans les grands fournisseurs de cloud (AWS, Azure, GCP).

---

## 3. Organisation et gouvernance

### a. La CNCF (Cloud Native Computing Foundation)

* La **CNCF**, ou *Cloud Native Computing Foundation*, est une **fondation à but non lucratif** créée en 2015 par la **Linux Foundation**.
* Sa mission principale est de **promouvoir, normaliser et soutenir** l’écosystème des technologies **cloud-native**, c’est-à-dire :

  * conçues pour fonctionner de manière **élastique et distribuée** dans le cloud,
  * reposant sur des **conteneurs**, des **microservices**, et des **API (Application Programming Interfaces)**.
* La CNCF héberge et maintient plusieurs projets majeurs du monde cloud, notamment :

  * **Kubernetes** (orchestration de conteneurs)
  * **Prometheus** (monitoring et alerting)
  * **Envoy** (proxy et service mesh)
  * **Helm** (gestionnaire de packages Kubernetes)
  * **containerd** (moteur de conteneurs léger)
  * **etcd** (base de données clé/valeur distribuée utilisée par Kubernetes pour stocker sa configuration)

### b. Gouvernance ouverte

* Kubernetes est gouverné par une **communauté internationale** d’entreprises, d’ingénieurs et de contributeurs indépendants.
* Les décisions sont prises de manière **collégiale et transparente** via :

  * Des **Special Interest Groups (SIGs)** : groupes spécialisés (réseau, sécurité, stockage, etc.).
  * Des **KEPs (Kubernetes Enhancement Proposals)** : documents formels de proposition d’évolution du projet.
  * Des **releases régulières** (environ 3 à 4 par an) supervisées par des équipes de la CNCF.

---

## 4. Objectifs et philosophie du projet

### a. Objectif principal

Kubernetes vise à **automatiser le déploiement, la gestion et la maintenance** d’applications exécutées dans des conteneurs, tout en assurant :

* **Disponibilité continue** : aucune interruption lors des mises à jour ou des pannes.
* **Scalabilité horizontale et verticale** : ajustement dynamique des ressources selon la charge.
* **Portabilité** : fonctionnement identique sur un poste local, un datacenter ou un cloud public.
* **Indépendance vis-à-vis du moteur de conteneurs** grâce à la **CRI (Container Runtime Interface)**.

### b. Philosophie “Cloud-Native”

Le terme **Cloud-Native** signifie que les applications sont :

* **Construites pour le cloud**, et non simplement hébergées dessus.
* **Distribuées par conception**, c’est-à-dire composées de plusieurs microservices indépendants.
* **Éphémères**, donc redéployables automatiquement en cas de défaillance.
* **Observables**, grâce à des outils intégrés de logs, métriques et traçabilité.

---

## 5. Problématiques résolues par Kubernetes

Avant Kubernetes, la gestion de conteneurs reposait sur des scripts manuels ou des orchestrateurs limités (comme Docker Swarm).
Kubernetes résout plusieurs défis majeurs :

* **Planification (Scheduling)** : choisir automatiquement sur quel serveur exécuter chaque conteneur.
* **Scalabilité (Scalability)** : ajuster le nombre de conteneurs selon la charge de travail.
* **Résilience (Resilience)** : redémarrer ou remplacer les conteneurs défaillants.
* **Réseau (Networking)** : permettre la communication entre conteneurs et services de manière standardisée.
* **Stockage (Storage)** : associer des volumes persistants indépendants du cycle de vie des conteneurs.
* **Déploiement continu (Continuous Deployment)** : assurer des mises à jour progressives sans interruption de service.
* **Observabilité (Observability)** : fournir des métriques et des journaux pour surveiller l’état du cluster.
* **Sécurité (Security)** : définir des règles d’accès, d’authentification et d’isolation réseau entre applications.

---

## 6. Pourquoi le nom “Kubernetes” ?

* Le mot **Kubernetes** vient du grec ancien *κυβερνήτης* (*kubernêtês*), qui signifie **“pilote de navire”** ou **“gouvernail”**.
* Ce nom symbolise parfaitement la **fonction de pilotage et de contrôle** des applications conteneurisées.
* Le **logo** de Kubernetes représente une **barre de gouvernail à sept branches**, chaque branche correspondant à une composante majeure du système.

---

## 7. Acteurs et écosystème

### a. Principaux contributeurs

* **Google** : créateur du projet et contributeur historique.
* **Red Hat / IBM** : intégration dans OpenShift.
* **Microsoft** : via Azure Kubernetes Service (AKS).
* **Amazon Web Services (AWS)** : via Elastic Kubernetes Service (EKS).
* **VMware**, **Canonical**, **SUSE**, **Oracle**, **Alibaba Cloud** : intégrations et extensions spécifiques.

### b. Outils de l’écosystème Kubernetes

Kubernetes s’intègre dans un écosystème complet de logiciels open-source :

* **Docker / containerd / CRI-O** : moteurs d’exécution de conteneurs.
* **Helm** : gestionnaire de packages applicatifs.
* **Prometheus / Grafana** : supervision et métrologie.
* **Istio / Linkerd** : gestion du trafic et sécurité réseau (Service Mesh).
* **Kubectl / K9s / Lens** : outils d’administration et d’observation du cluster.


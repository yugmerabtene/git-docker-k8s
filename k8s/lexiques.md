# Lexique Kubernetes

## A

* **Admission Controller** — Chaîne de plugins du **kube-apiserver** qui modifient (mutating) et/ou valident (validating) les objets avant persistance. Sert à imposer des politiques (quotas, sécurité, labels obligatoires).
* **Affinity / Anti-Affinity** — Contraintes de placement des Pods : co-localiser ou éloigner des Pods selon des labels (node/pod affinity).
* **API Aggregation** — Mécanisme pour **étendre l’API** K8s en agrégeant des APIs tierces sous le même endpoint.
* **API Server (kube-apiserver)** — Point d’entrée unique du cluster. Gère AuthN/AuthZ/Admission et persiste l’état dans **etcd**.
* **AuthN (Authentication)** — Vérifie l’identité (certificats X.509, tokens, OIDC).
* **AuthZ (Authorization)** — Vérifie les autorisations (RBAC, Node, Webhook, ABAC – à éviter en prod).
* **Autoscaling** — Mise à l’échelle automatique: **HPA (Horizontal Pod Autoscaler)**, **VPA (Vertical Pod Autoscaler)**, **Cluster Autoscaler**.

## B

* **Baseline (Pod Security)** — Niveau de sécurité “par défaut” des Pods selon les Pod Security Standards (plus permissif que “restricted”).
* **Borg** — Orchestrateur interne historique de Google, inspiration de Kubernetes.

## C

* **Calico** — Plugin **CNI (Container Network Interface)** offrant réseau + **NetworkPolicies** (option eBPF).
* **Cluster** — Ensemble **Control Plane + Nodes** gérés comme une seule plate-forme.
* **Cluster Autoscaler** — Ajuste le **nombre de nœuds** (cloud/on-prem) selon la pression des Pods en attente.
* **ClusterIP (Service)** — Type de Service exposé **uniquement** à l’intérieur du cluster (VIP).
* **CNI (Container Network Interface)** — Spécification pour brancher une **stack réseau** aux Pods (Calico, Cilium, Flannel, Weave).
* **ConfigMap** — Objet contenant de la **configuration non sensible** injectée dans les Pods (fichiers, variables d’environnement).
* **Container Runtime** — Moteur d’exécution (ex. **containerd**, **CRI-O**). Interface via **CRI (Container Runtime Interface)**.
* **controller-manager (kube-controller-manager)** — Exécute les boucles de réconciliation (Deployment, RS, Node, Job, GC, etc.).
* **CRI (Container Runtime Interface)** — API gRPC standard entre kubelet et runtime (implémentations : containerd, CRI-O).
* **CRI-O** — Runtime léger, conforme **OCI (Open Container Initiative)**, parlant CRI (image service, runtime service).
* **CronJob** — Planifie des **Jobs** périodiques (exécution selon un cron).
* **CSI (Container Storage Interface)** — Interface standard pour pilotes de stockage (EBS, Ceph, NFS…).

## D

* **DaemonSet** — Garantit **un Pod par nœud** (ou par sous-ensemble de nœuds) pour agents système (logs, CNI, monitoring).
* **Deployment** — Contrôleur de déploiement **déclaratif** (rolling update, rollback, scaling).
* **DNS (CoreDNS)** — Service DNS interne pour la **découverte de services** et noms `*.svc.cluster.local`.

## E

* **eBPF (extended Berkeley Packet Filter)** — Tech noyau Linux pour réseau/observabilité haute perf (Cilium).
* **Endpoint / EndpointsSlice** — Liste des adresses Pod cibles derrière un Service.
* **etcd** — Base **clé/valeur distribuée** (Raft) contenant l’état source de vérité du cluster.
* **Egress** — **Trafic sortant** des Pods vers l’extérieur du cluster; contrôlé via **NetworkPolicies** ou **Egress Gateway**.

## F

* **Flannel** — Plugin réseau **CNI** simple (overlay VXLAN/host-gw).
* **gRPC** — Protocole RPC moderne (utilisé par CRI, CSI).

## G

* **GKE/EKS/AKS** — Services managés Kubernetes (Google/AWS/Azure).

## H

* **HPA (Horizontal Pod Autoscaler)** — Ajuste le **nombre de Pods** selon des métriques (CPU, mémoire, personnalisées).
* **Helm** — Gestionnaire de **charts** (packages K8s) pour déploiements paramétrables.
* **Health Probes (Liveness/Readiness/Startup)** — Vérifications de santé des containers pour redémarrage, admission au Service, délais de démarrage.

## I

* **Ingress** — Objet L7 (HTTP/HTTPS) définissant des **règles de routage** vers des Services; implémenté par un **Ingress Controller** (NGINX, Traefik, HAProxy).
* **Init Container** — Conteneur(s) exécuté(s) **avant** les containers applicatifs, pour préparer l’environnement.
* **IPVS / iptables** — Modes de **kube-proxy** pour la translation de Services (IPVS plus performant).

## J

* **Job** — Exécution **finie** (batch), réussit quand N complétions atteintes.

## K

* **kubeadm** — Outil officiel d’**installation** de clusters K8s.
* **kubectl** — CLI pour manipuler l’API. Contexte via `~/.kube/config`.
* **kubelet** — Agent de **chaque nœud** : lance les Pods via CRI, publie l’état, applique cgroups/probes/evictions.
* **kube-proxy** — Programme les règles L4 (iptables/IPVS) pour Services.
* **Kustomize** — Personnalisation déclarative de manifests (overlays) sans templates.

## L

* **Label / Selector** — Paires clé/valeur pour **sélectionner** des objets (routing, Services, RS, affinities).
* **LimitRange / ResourceQuota** — Cadre les **requests/limits** et quotas par namespace.
* **LoadBalancer (Service)** — Type de Service exposé avec IP publique via un **LB** (cloud) ou **MetalLB** on-prem.
* **Logs (kubectl logs)** — Journaux de containers/Pods; pour les composants, namespace `kube-system`.

## M

* **MetalLB** — Équilibreur de charge **on-prem** (L2/L3) pour Services `LoadBalancer`.
* **Metrics Server** — Source de métriques **CPU/RAM** pour `kubectl top` et HPA.
* **Minikube** — Cluster local **mono-nœud** pour apprentissage/tests.
* **MTU (Maximum Transmission Unit)** — Taille max trame; erreurs MTU → pertes/frags, surtout avec overlays (VXLAN).

## N

* **Namespace** — **Isolation logique** (quotas, RBAC, cycles de vie).
* **NetworkPolicy** — Règles **Ingress/Egress** par Pod/NS (nécessite CNI compatible).
* **Node** — Machine (VM/physique) exécutant les **Pods**; conditions `Ready`, pressions mémoire/disque.
* **Node Affinity / Taints & Tolerations** — Mécanismes de **placement** / **repoussement** des Pods vis-à-vis des nœuds.

## O

* **OCI (Open Container Initiative)** — Spécifications d’images/containers.
* **OIDC (OpenID Connect)** — Authentification déléguée/SSO vers l’API (utilisateurs).
* **Operator** — Contrôleur spécialisé gérant une **application avec état** via **CRD (CustomResourceDefinition)**.

## P

* **PersistentVolume (PV) / PersistentVolumeClaim (PVC)** — Abstractions stockage persistant; **StorageClass** pour le provisionnement dynamique.
* **Pod** — Plus petite **unité déployable** (un ou plusieurs containers partageant IP/namespace réseau et volumes).
* **PodDisruptionBudget (PDB)** — Garantit un **minimum de Pods disponibles** pendant des évictions/maintenance.
* **Pod Security Standards (Baseline/Restricted/Privileged)** — Normes CNCF remplaçant PSP; contrôlées via Pod Security Admission ou Kyverno/Gatekeeper.
* **PreStop / PostStart Hooks** — Hooks d’exécution au cycle de vie du container.
* **Probe (Liveness/Readiness/Startup)** — Voir “Health Probes”.
* **Prometheus / Grafana** — Collecte/visualisation **métriques**; Alertmanager pour alertes.
* **Pull Policy (`IfNotPresent/Always/Never`)** — Stratégie de téléchargement d’image.
* **PVC Access Modes (`RWO/RWX/ROX`)** — Modes d’accès: lecture/écriture exclusive, partagée, lecture seule.

## Q

* **Quorum (etcd)** — Majorité requise (N impairs 3/5) pour écrire. Perte de quorum → cluster bloqué en écriture.

## R

* **RBAC (Role-Based Access Control)** — Contrôle d’accès par **Role/ClusterRole** et liaisons **RoleBinding/ClusterRoleBinding**.
* **Readiness/Liveness/Startup** — Voir Probes.
* **ReplicaSet (RS)** — Maintient un **nombre de Pods** à l’identique (utilisé par Deployment).
* **Requests / Limits** — Réservations et plafonds **CPU/Mémoire** par container.
* **Rolling Update / Rollback** — Mise à jour progressive d’un Deployment / retour arrière.
* **RuntimeClass** — Associe un **runtime alternatif** (gVisor/Kata) à un Pod.
* **Runtime Security** — Sécurité à l’exécution (ex. **Falco**, Sysdig) pour détecter des comportements anormaux.

## S

* **SA (ServiceAccount)** — Identité **applicative** d’un Pod pour appeler l’API (token JWT monté par défaut).
* **Scheduler (kube-scheduler)** — Assigne les Pods aux nœuds (filtrage + scoring + bind).
* **Secret** — Donnée **sensible** (clé API, mot de passe); encodage Base64 et **chiffrement at-rest** recommandé côté API server.
* **Selector** — Filtrage par labels (Services, RS, Deployments).
* **Service** — IP virtuelle stable (ClusterIP/NodePort/LoadBalancer) exposant un **groupe de Pods**.
* **Sidecar Pattern** — Container auxiliaire (logs/proxy/agent) co-localisé dans le Pod.
* **StatefulSet (STS)** — Déploiement **avec identité stable** et volumes persistants par réplique (bases de données).
* **StorageClass (SC)** — Définit une **classe de stockage** (provisionnement dynamique, politiques de reprise).
* **Startup Probe** — Délai de démarrage long; empêche des redémarrages prématurés.
* **Supervisor / Controller** — Boucle de réconciliation assurant l’état désiré.

## T

* **Taints & Tolerations** — Marquage repoussant des Pods non tolérants; utile pour nœuds spéciaux (GPU, infra).
* **TLS / mTLS** — Chiffrement des communications (API server, etcd, kubelets; mTLS entre sidecars en service mesh).
* **TopologySpreadConstraints** — Répartition des Pods entre zones/nœuds pour haute disponibilité.

## U

* **Upgrade / Version Skew** — Compatibilité **±1 version mineure** entre composants (planifier les mises à niveau).
* **UUID (K8s UID)** — Identifiant unique d’objet (différent du `metadata.name`).

## V

* **VPA (Vertical Pod Autoscaler)** — Ajuste **requests/limits** d’un Pod selon la consommation.
* **Volume** — Montage dans un Pod (emptyDir, hostPath, configMap, secret, PVC, projected…).

## W

* **Watch** — Abonnement aux changements d’objets via l’API (stream).
* **Webhook Admission (Mutating/Validating)** — Webhooks externes d’admission (ex. **OPA Gatekeeper**, **Kyverno**).
* **Workload** — Ressource gérant des Pods (Deployment, StatefulSet, DaemonSet, Job/CronJob).

## Y

* **YAML** — Format déclaratif des manifests (apiVersion/kind/metadata/spec).

---

# Mini-lexique Sécurité/DevSecOps (compléments)

* **Kyverno** — Moteur de **politiques déclaratives** K8s (YAML natif) pour valider/muter/générer des objets.
* **OPA Gatekeeper (Open Policy Agent)** — Politiques en **Rego** pour Admission Control.
* **Falco** — Détection **runtime** (règles sur syscalls/événements).
* **Trivy / Clair / Anchore / Snyk** — **Image scanning** pour vulnérabilités CVE.
* **SealedSecrets** — Chiffre les Secrets côté Git; déchiffrés dans le cluster.
* **Audit Logging** — Journalisation des appels API pour conformité (ISO 27001, RGPD).
* **Pod Security Admission** — Contrôle natif des Pod Security Standards (remplace PSP).
* **Network Policy Egress/Ingress** — Filtrage L3/L4 par namespace/labels/ipBlock.

---

# Mini-lexique Réseau

* **CoreDNS** — DNS interne du cluster.
* **Egress Gateway (mesh)** — Point de sortie centralisé pour le trafic sortant (Istio).
* **Ingress Controller** — Implémentation des règles Ingress (NGINX, Traefik, HAProxy).
* **NodePort (Service)** — Expose un Service sur **un port** de **chaque nœud**.
* **Headless Service (`clusterIP: None`)** — Pas de VIP; résolution DNS directe vers les Pods (utile STS).

---

# Mini-lexique Stockage

* **AccessModes (RWO/RWX/ROX)** — Modes d’accès des volumes.
* **ReclaimPolicy (Retain/Delete/Recycle)** — Politique de recyclage d’un PV après libération.
* **VolumeMode (Filesystem/Block)** — Mode d’exposition du volume au Pod.

---

# Mini-lexique Opérations

* **Drain / Cordon** — Vidage/gel d’un nœud pour maintenance.
* **Eviction** — Expulsion d’un Pod (pression mémoire/disque, PDB pris en compte).
* **Events** — Chronologie des changements; clé pour le diagnostic (`kubectl get events`).
* **Dashboard** — UI web (souvent via `minikube dashboard`).


# Chapitre-13 — Orchestration avec Swarm/Kubernetes

*(passerelles depuis Compose, HA, déploiements, réseau, stockage, sécurité, observabilité)*

## Objectifs d’apprentissage

* Comprendre **quand** et **pourquoi** passer de Docker Compose à un orchestrateur (**Swarm** ou **Kubernetes**).
* Savoir modéliser une application en **services répliqués**, gérer **mises à jour progressives**, **rollbacks**, **placement** des workloads et **haute disponibilité**.
* Maîtriser les bases **réseau**, **stockage**, **sécurité**, **observabilité** et **CI/CD** propres à Swarm et Kubernetes.
* Disposer d’une **cartographie Compose → Swarm/K8s** et de **patrons de déploiement** (rolling, blue/green, canary) réutilisables.

## Pré-requis

* Chap. 01–12 (images, conteneurs, réseau, stockage, Dockerfile/BuildKit, Compose, registry, sécurité, perf, CI/CD, releases).
* Connaissances Linux réseau/FS. Pour Kubernetes : notions YAML, RBAC.

---

## 1) Quand quitter Compose ?

* **Scale & HA** : besoin de **réplicas** auto-gérés, redémarrage sur panne, **auto-healing**.
* **Rolling updates** et **rollbacks** atomiques.
* **Planification** (anti-affinité, contraintes par nœud), **autoscaling**, **quotas**.
* **Sécurité** (RBAC, isolement réseau), **multi-tenant**.
* **Add-ons** : ingress contrôleurs, service mesh, secrets KMS/cloud, opérateurs DB.

> **Swarm** : chemin le plus court depuis Compose, simple, natif Docker.
> **Kubernetes** : écosystème riche, standard de facto, plus verbeux mais plus puissant.

---

## 2) Compose → Swarm : concepts & commandes

### 2.1 Concepts clés

* **Cluster** Swarm = **managers** (Raft) + **workers**.
* **Service** = définition d’un conteneur répliqué (**tasks**).
* **Stack** = groupe de services issu d’un fichier Compose v3+ (`deploy.*` **pris en charge**).
* **Réseau overlay** chiffré multi-hôtes ; **routing mesh** (LB L4 intégré).
* **Secrets/Configs** gérés par Swarm (montés en fichiers).
* **Update/Rollback** paramétrables (parallélisme, délais, action on-failure).

### 2.2 Initialiser un cluster & joindre des nœuds

```bash
docker swarm init                         # sur le premier manager
docker swarm join --token <worker-token> <manager-ip>:2377
docker node ls
```

### 2.3 Créer un service répliqué

```bash
docker service create --name web --replicas 3 -p 80:80 nginx:1.27
docker service ls
docker service ps web
```

### 2.4 Déployer une **stack** depuis Compose (v3+)

```bash
docker stack deploy -c compose.yaml myapp
docker stack services myapp
docker stack ps myapp
docker stack rm myapp
```

> Contrairement à `docker compose`, les champs **`deploy.*`** (réplicas, ressources, update) **sont actifs** en Swarm.

### 2.5 Mises à jour & rollback

```bash
docker service update --image nginx:1.27.2 --update-parallelism 1 \
  --update-delay 10s --update-monitor 30s --update-failure-action rollback web

docker service rollback web
```

### 2.6 Placement & ressources

```bash
# Contraintes (exécuter sur les nœuds tagués)
docker node update --label-add role=frontend <node>
docker service create --constraint 'node.labels.role == frontend' ...

# Limites
docker service create --limit-cpu 1 --limit-memory 512M ...
```

### 2.7 Réseau overlay & modes de résolution

* **Routing mesh** : publication L4 sur tous les nœuds (`-p 80:80`).
* **DNS VIP** (par défaut) vs **DNSRR** (round-robin sans VIP) :

```bash
docker service create --name api --replicas 3 --endpoint-mode dnsrr ...
```

### 2.8 Secrets & configs

```bash
echo 'supersecret' | docker secret create jwt -
docker service create --name api --secret jwt ghcr.io/acme/api:1.4.2
docker config create webconf ./nginx.conf
```

### 2.9 Patron Compose→Swarm (extrait `compose.yaml`)

```yaml
version: "3.9"
services:
  web:
    image: nginx:1.27
    ports: [ "80:80" ]
    deploy:
      replicas: 3
      update_config: { parallelism: 1, delay: 10s, order: start-first }
      rollback_config: { parallelism: 1, delay: 5s }
      placement:
        constraints: [ "node.labels.role == frontend" ]
    networks: [ front ]

  api:
    image: ghcr.io/acme/api@sha256:...
    deploy:
      replicas: 4
      resources:
        limits: { cpus: "1.0", memory: 512M }
      endpoint_mode: dnsrr
    networks: [ front, back ]
    secrets: [ jwt ]

networks:
  front: { driver: overlay }
  back:  { driver: overlay, attachable: true }

secrets:
  jwt: { external: true }
```

**Limites Swarm (à connaître)** : écosystème plus restreint (ingress avancés, opérateurs), autoscaling natif limité, communauté moindre vs K8s.

---

## 3) Compose → Kubernetes : concepts & ressources

### 3.1 Objets fondamentaux

* **Pod** : plus petite unité d’exécution (1+ conteneurs).
* **Deployment** : gère les **réplicas**, **rolling updates** et **rollbacks** (ReplicaSets).
* **Service** : L4 stable (ClusterIP/NodePort/LoadBalancer) avec **DNS** interne.
* **Ingress** : L7 (HTTP) via **Ingress Controller** (Nginx, Traefik, HAProxy…).
* **ConfigMap / Secret** : configuration et secrets **montés** (fichiers/env).
* **StatefulSet** : apps **stateful** (DB, queue) avec **volumes persistants** stables.
* **Job/CronJob** : traitements batch/planifiés.
* **Namespace/RBAC** : multi-tenant et permissions.
* **HorizontalPodAutoscaler (HPA)** : autoscaling **CPU/mémoire** (et metrics custom).

### 3.2 Mapping Compose → K8s (repères)

| Compose (v2)    | Swarm           | Kubernetes                             |
| --------------- | --------------- | -------------------------------------- |
| service         | service         | Deployment (+ Service)                 |
| ports           | publish         | Service (NodePort/LB) + Ingress        |
| volumes         | volumes         | PersistentVolumeClaim (+ StorageClass) |
| configs/secrets | configs/secrets | ConfigMap / Secret                     |
| networks        | overlay         | CNI (pod network) + Service/Ingress    |
| healthcheck     | update monitor  | probes (readiness/liveness/startup)    |

### 3.3 Manifeste minimal (api + web)

```yaml
# deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, labels: { app: demo } }
spec:
  replicas: 3
  selector: { matchLabels: { app: demo, tier: api } }
  template:
    metadata: { labels: { app: demo, tier: api } }
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/api@sha256:...      # déployer par digest
          ports: [ { containerPort: 8080 } ]
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "1",    memory: "512Mi" }
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 20
---
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: demo, tier: api }
  ports: [ { port: 80, targetPort: 8080 } ]
  type: ClusterIP
```

```yaml
# web + ingress
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, labels: { app: demo, tier: web } }
spec:
  replicas: 2
  selector: { matchLabels: { app: demo, tier: web } }
  template:
    metadata: { labels: { app: demo, tier: web } }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports: [ { containerPort: 80 } ]
---
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: demo, tier: web }
  ports: [ { port: 80, targetPort: 80 } ]
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web, port: { number: 80 } } }
```

### 3.4 Stockage & données persistantes

* **StorageClass** (provisionneur dynamique cloud/CSI), **PVC** par Pod/StatefulSet.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data-pg }
spec:
  accessModes: [ ReadWriteOnce ]
  resources: { requests: { storage: 20Gi } }
  storageClassName: fast-ssd
```

### 3.5 Sécurité (bases)

* **ServiceAccount** + **RBAC** (roles/rolebindings) par namespace.
* **SecurityContext** : `runAsUser`, `runAsNonRoot`, `readOnlyRootFilesystem`, `capabilities`.
* **Pod Security Admission** (baseline/restricted).
* **NetworkPolicy** (isoler le trafic).
* **imagePullSecrets** pour registries privés.
* **OPA Gatekeeper/Kyverno** : politiques d’admission (refuser `:latest`, pods non-root, etc.).
* **cosign** + **policy-controller** (vérif. de signatures) — selon stack choisie.

### 3.6 Observabilité

* `kubectl logs -f`, `kubectl describe`, `kubectl top pods/nodes`.
* **metrics-server**, Prometheus/Grafana pour métriques, Loki/ELK pour logs.
* Traces : OTEL Collector + Jaeger/Tempo.

### 3.7 Mises à jour, rollbacks & autoscaling

```bash
kubectl rollout status deploy/api
kubectl rollout undo deploy/api
kubectl set image deploy/api api=ghcr.io/acme/api@sha256:...   # update par digest
kubectl autoscale deploy/api --min=3 --max=10 --cpu-percent=70
```

* Probes **readiness** = portes de trafic ; **liveness** = redémarrage en échec.

### 3.8 Blue/Green & Canary (K8s)

* **Blue/Green** : deux Deployments (labels `version=blue|green`) + **Service** pointant vers `version=green` à la bascule.
* **Canary** :

  * Ingress Nginx (annotations canary/poids) **ou**
  * Deux Services (stable/canary) & règles L7 pondérées **ou**
  * **Service mesh** (Istio/Linkerd) pour controler le pourcentage fin, avec traces mTLS.

### 3.9 Gestion des manifests

* **Helm** (charts, valeurs) pour factoriser.
* **Kustomize** (patch/overlay) pour variantes env sans templating.
* **Kompose** pour convertir un Compose en manifests (point de départ, **à revoir** manuellement).

---

## 4) Réseau : Swarm vs Kubernetes (repères)

| Thème          | Swarm                            | Kubernetes                                       |
| -------------- | -------------------------------- | ------------------------------------------------ |
| L4 interne     | VIP/DNSRR                        | Service ClusterIP (kube-proxy/IPVS)              |
| L4 externe     | Routing mesh (`-p`)              | Service NodePort/LoadBalancer                    |
| L7             | Nginx/Traefik externes           | Ingress Controller (Nginx/Traefik/HAProxy/Envoy) |
| DNS            | Interne service name             | CoreDNS                                          |
| Network policy | Basique (isolation via networks) | **NetworkPolicy** (CNI must support)             |

---

## 5) Stockage & données

|          | Swarm                                | Kubernetes                                           |
| -------- | ------------------------------------ | ---------------------------------------------------- |
| Volumes  | `local`, NFS, drivers tiers          | **PVC/PV/StorageClass** (CSI)                        |
| Stateful | Réplicas simples, pas de StatefulSet | **StatefulSet** (volumes par Pod, identités stables) |
| Backups  | scripts/volumes                      | opérateurs/sidecars & jobs dédiés                    |

---

## 6) Sécurité (résumé opérationnel)

**Swarm**

* Secrets/Configs natifs, chiffrés en transit & au repos (Raft).
* Contrainte placement, pas de RBAC riche.
* Exposition API Docker à restreindre (TLS, pare-feu).

**Kubernetes**

* **RBAC** fin, **ServiceAccounts**, **NetworkPolicy**, PodSecurity (baseline/restricted).
* **SecurityContext** non-root, capabilities minimales, **readOnlyRootFilesystem**.
* Admission policies (Gatekeeper/Kyverno), signatures cosign, **imagePolicyWebhook** (selon stack).

---

## 7) CI/CD & déploiements

* **Swarm** : `docker stack deploy` depuis CI, paramètres `deploy.*` dans Compose ; promotion **par digest**.
* **K8s** : `kubectl apply`/`helm upgrade`/`kustomize` ; `rollout status/undo`; promotion **par digest** ; gates (OPA), scanners (Trivy/Starboard), **HPA**.

---

## 8) Exemples “prod-like” synthèse

### 8.1 Swarm — stack web/api/db

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    volumes: [ data_pg:/var/lib/postgresql/data ]
    networks: [ back ]
    deploy:
      placement: { constraints: [ "node.labels.role == backend" ] }

  api:
    image: ghcr.io/acme/api@sha256:...
    secrets: [ pg_pwd ]
    networks: [ front, back ]
    deploy:
      replicas: 4
      resources: { limits: { cpus: "1", memory: 512M } }
      update_config: { parallelism: 1, delay: 10s, order: start-first }

  web:
    image: nginx:1.27
    ports: [ "80:80" ]
    networks: [ front ]
    deploy:
      replicas: 3
      placement: { constraints: [ "node.labels.role == frontend" ] }

networks:
  front: { driver: overlay }
  back:  { driver: overlay }
volumes:
  data_pg: {}
secrets:
  pg_pwd: { external: true }
```

### 8.2 Kubernetes — api avec HPA & NetworkPolicy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, labels: { app: demo } }
spec:
  replicas: 3
  selector: { matchLabels: { app: demo } }
  template:
    metadata: { labels: { app: demo } }
    spec:
      serviceAccountName: api
      containers:
        - name: api
          image: ghcr.io/acme/api@sha256:...
          ports: [ { containerPort: 8080 } ]
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            capabilities: { drop: [ "ALL" ] }
---
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: demo }
  ports: [ { port: 80, targetPort: 8080 } ]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-allow-web, namespace: default }
spec:
  podSelector: { matchLabels: { app: demo } }
  ingress:
    - from:
        - podSelector: { matchLabels: { tier: web } }
      ports: [ { protocol: TCP, port: 8080 } ]
  policyTypes: [ Ingress ]
```

---

## 9) Aide-mémoire (cheat-sheet)

**Swarm**

```bash
docker swarm init
docker node ls
docker service create --name web --replicas 3 -p 80:80 nginx
docker service update --image nginx:1.27.2 web
docker service rollback web
docker stack deploy -c compose.yaml app
docker stack ps app
```

**Kubernetes**

```bash
kubectl get nodes,pods,svc,ingress
kubectl logs -f deploy/api
kubectl describe pod <name>
kubectl rollout status deploy/api
kubectl set image deploy/api api=REG/IMG@sha256:...
kubectl rollout undo deploy/api
kubectl apply -k overlays/prod        # Kustomize
helm upgrade --install web ./chart     # Helm
```

---

## 10) Checklist de clôture (prêt pour l’orchestration)

**Modélisation**

* Services stateless en **Deployments**/services ; stateful en **StatefulSets** + **PVC**.
* **Digests** partout (immutabilité), **secrets** montés en fichiers.

**Réseau**

* Entrées via **Ingress/LB** ; politiques de flux (**NetworkPolicy**).
* Probes **readiness/liveness** correctes ; rolling updates testés.

**Sécurité**

* **Non-root**, capabilities minimales, FS **read-only**.
* **RBAC** par namespace, **Pod Security** baseline/restricted.
* Admission policies (Gatekeeper/Kyverno), **signatures cosign** vérifiées.

**Observabilité**

* Logs centralisés, métriques Prometheus, traces OTEL.
* Dashboards de release (blue/green/canary) et alertes SLO.

**Opérations**

* Runbooks de **rollout/rollback**, migrations **expand→migrate→contract**.
* Sauvegardes/restores des **PVC** testés.
* Pipelines CI/CD : **scan → sbom → sign → push → promote by digest**.


# Annexes (supports fournis)

## 1) Glossaire Kubernetes (sélection essentielle)

* **Cluster** : ensemble nœuds + plan de contrôle (API Server, etcd, Scheduler, Controller-Manager).
* **Nœud (Node)** : machine (VM/physique) qui exécute les Pods (kubelet, runtime, kube-proxy).
* **Espace de noms (Namespace)** : partition logique d’isolement et de quotas.
* **Pod** : plus petite unité déployable ; 1..N conteneurs partageant réseau/IPC/volumes.
* **Container** : processus isolé packagé par une image OCI.
* **Image** : artefact OCI immuable (référence par digest `@sha256:...` recommandée).
* **Label / Annotation** : paires clé/valeur (sélection/organisation vs métadonnées non-indexées).
* **Selector** : requête sur labels (Services, Deployments, Policies, etc.).
* **Taint/Toleration** : mécanisme d’exclusion/réservation de nœuds.
* **Affinity/Anti-Affinity** : contraintes de placement (avec `topologyKey`).
* **QoS** : `Guaranteed`, `Burstable`, `BestEffort` selon requests/limits CPU/Mem.
* **Deployment** : contrôleur déclaratif pour Pods stateless (gère ReplicaSet & rollouts).
* **ReplicaSet** : maintient un nombre de Pods identiques (utilisé par Deployment).
* **StatefulSet** : Pods à identité stable + stockage persistant ordonné.
* **DaemonSet** : un Pod par nœud (logs/monitoring/agents).
* **Job / CronJob** : exécution finie / planifiée.
* **Service** : accès stable aux Pods (types : ClusterIP, NodePort, LoadBalancer, Headless).
* **EndpointSlice** : liste scalable des endpoints d’un Service.
* **Ingress** : exposition HTTP/HTTPS (L7) via un contrôleur (NGINX, Traefik…).
* **Gateway API** : évolution d’Ingress (objets Gateway/Route, fonctionnalités L7 avancées).
* **ConfigMap / Secret** : configuration non sensible / sensible.
* **Volume / PVC / PV / StorageClass** : stockage persistant, provisionnement CSI.
* **VolumeSnapshot** : snapshot de PV (via driver CSI).
* **CNI** : plugin réseau (Calico, Cilium…).
* **NetworkPolicy** : pare-feu L3/L4 inter-Pods (deny/allow).
* **RBAC** : droits (Role/ClusterRole) + liaisons (RoleBinding/ClusterRoleBinding).
* **ServiceAccount** : identité des Pods vis-à-vis de l’API ; token monté (désactivable).
* **PSA** : Pod Security Admission (niveaux `baseline`/`restricted`).
* **Admission (Validating/Mutating)** : garde-fous (OPA Gatekeeper, Kyverno, Sigstore).
* **Probes** : `liveness` (auto-heal), `readiness` (gating trafic), `startup` (grâce au boot).
* **HPA / VPA / KEDA** : autoscaling horizontal, vertical, par événements externes.
* **PDB** : PodDisruptionBudget (disponibilité lors d’évictions/rollouts).
* **PriorityClass** : priorité/preemption de Pods.
* **kube-proxy** : routage Service (iptables/ipvs).
* **Metrics Server** : source CPU/Mem pour `kubectl top` et HPA.

---

## 2) Cheat-sheet `kubectl` (opérations courantes)

### Base & affichage

```bash
kubectl version --short                 # versions client/serveur
kubectl config get-contexts             # contextes Kubeconfig
kubectl config use-context <ctx>        # bascule de contexte
kubectl get ns                          # namespaces
kubectl get pod,svc,deploy -A -o wide   # vue rapide multi-ns
kubectl get all -n <ns>                 # ressources usuelles d’un ns
kubectl describe pod/<name> -n <ns>     # détails + Events
kubectl explain deployment.spec.strategy --recursive  # documentation schéma
```

### Sélection & formats

```bash
kubectl get pods -n <ns> -l app=api
kubectl get pods --field-selector=status.phase!=Running
kubectl get pods -o jsonpath='{.items[*].status.qosClass}{"\n"}'
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

### Appliquer / valider / supprimer

```bash
kubectl diff -f deploy.yaml                       # diff déclaratif
kubectl apply -f deploy.yaml                      # appliquer
kubectl delete -f deploy.yaml                     # supprimer
kubectl apply -k kustomize/overlays/prod          # Kustomize
```

### Rollout & scale

```bash
kubectl rollout status deploy/api -n app
kubectl rollout history deploy/api -n app
kubectl rollout undo deploy/api -n app
kubectl scale deploy/api -n app --replicas=5
kubectl set image deploy/api api=repo@sha256:... -n app
```

### Logs / Exec / Port-forward

```bash
kubectl logs pod/<name> -n app --tail=200
kubectl logs pod/<name> -n app --previous         # crash précédent
kubectl exec -it pod/<name> -n app -- sh          # shell
kubectl port-forward svc/api -n app 8080:8080     # local→cluster
```

### Ressources & nœuds

```bash
kubectl top nodes ; kubectl top pods -A
kubectl cordon <node> ; kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

### Secrets / ConfigMap

```bash
kubectl create secret generic db-creds -n app \
  --from-literal=DB_USER=app --from-literal=DB_PASS='s3cret!'
kubectl create configmap api-config -n app --from-literal=APP_ENV=prod
```

---

## 3) Modèles YAML (prêts à copier)

### 3.1 Deployment (stateless) + Probes + Ressources

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: app
  labels: { app: api, tier: backend }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 }
  selector: { matchLabels: { app: api } }
  template:
    metadata:
      labels: { app: api, tier: backend }
    spec:
      automountServiceAccountToken: false         # désactive si inutile
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      containers:
      - name: api
        image: ghcr.io/org/api@sha256:REPLACER_ME # déploiement par digest
        imagePullPolicy: IfNotPresent
        ports: [ { name: http, containerPort: 8080 } ]
        envFrom:
          - configMapRef: { name: api-config }
        env:
          - name: DB_USER
            valueFrom: { secretKeyRef: { name: db-creds, key: DB_USER } }
          - name: DB_PASS
            valueFrom: { secretKeyRef: { name: db-creds, key: DB_PASS } }
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "1",    memory: "512Mi" }
        readinessProbe:
          httpGet: { path: /healthz, port: http }
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /livez, port: http }
          initialDelaySeconds: 15
          periodSeconds: 10
```

### 3.2 Service (ClusterIP) + Headless (si besoin)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: app
  labels: { app: api }
spec:
  type: ClusterIP
  selector: { app: api }
  ports:
    - name: http
      port: 8080
      targetPort: http
---
# Headless (si discovery DNS granulaire nécessaire)
apiVersion: v1
kind: Service
metadata: { name: api-headless, namespace: app, labels: { app: api } }
spec:
  clusterIP: None
  selector: { app: api }
  ports: [ { name: http, port: 8080, targetPort: http } ]
```

### 3.3 Ingress (NGINX)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: app
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port: { number: 8080 }
  tls:
    - hosts: [ api.example.com ]
      secretName: api-tls
```

### 3.4 PVC + StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: fast-retain }
provisioner: csi.example.com
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-api
  namespace: app
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: fast-retain
  resources: { requests: { storage: 10Gi } }
```

### 3.5 StatefulSet (avec Service headless)

```yaml
apiVersion: v1
kind: Service
metadata: { name: db, namespace: app }
spec:
  clusterIP: None
  selector: { app: db }
  ports: [ { name: psql, port: 5432 } ]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db, namespace: app }
spec:
  serviceName: db
  replicas: 3
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports: [ { name: psql, containerPort: 5432 } ]
        volumeMounts: [ { name: data, mountPath: /var/lib/postgresql/data } ]
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-retain
      resources: { requests: { storage: 50Gi } }
```

### 3.6 NetworkPolicy (deny-all + allow front→api)

```yaml
# Par défaut : tout bloquer dans le ns "app"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
# Autoriser le frontend (ns=frontend) à joindre l'API sur 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend-to-api, namespace: app }
spec:
  podSelector: { matchLabels: { app: api } }
  ingress:
  - from:
    - namespaceSelector: { matchLabels: { name: frontend } }
    ports: [ { protocol: TCP, port: 8080 } ]
```

---

## 4) Checklists (sécurité, release, incident)

### 4.1 Sécurité (avant mise en prod)

* [ ] **Images** : base pinée par **digest** ; **pas de `:latest`** ; labels OCI complets.
* [ ] **User** : `runAsNonRoot`, UID dédié ; `readOnlyRootFilesystem` si possible.
* [ ] **Capabilities** : `drop: ["ALL"]` puis ajouter au besoin (pas de `NET_ADMIN`/`SYS_ADMIN` par défaut).
* [ ] **Seccomp/AppArmor/SELinux** : `seccompProfile: RuntimeDefault` (ou profil strict).
* [ ] **ServiceAccount** : minimal ; `automountServiceAccountToken: false` si inutile.
* [ ] **Secrets** : montés en **fichiers** ; rotation documentée ; jamais en clair dans l’image.
* [ ] **Réseau** : `NetworkPolicy` deny-all + règles explicites ; egress contrôlé.
* [ ] **Ressources** : requests/limits posés ; PDB présent ; probes correctes.
* [ ] **Supply-chain** : SBOM générée ; **Trivy** bloquant ; **Cosign** signature ; policies d’admission (digest + signature).
* [ ] **Audit** : logs cluster/registre centralisés ; traces d’accès & de changement.

### 4.2 Release (qualité & traçabilité)

* [ ] **Versionning** SemVer ; release tag `vX.Y.Z`.
* [ ] **CI** : build multi-arch, tests unit/int, scan, SBOM, signature passés.
* [ ] **Artefact** : digest enregistré ; chart Helm validé (`helm lint`, `kubeconform`).
* [ ] **Stratégie** : Rolling (`maxUnavailable=0`) ou Blue-Green/Canary documentée.
* [ ] **Migrations** : hooks Helm `pre-install/upgrade` idempotents.
* [ ] **Observabilité** : dashboards, alertes, logs/trace prêts ; smoke test post-deploy.
* [ ] **Rollback** : procédure testée (`helm rollback` ou GitOps revert).
* [ ] **CHANGELOG** : notes livrées + risques connus.

### 4.3 Incident (triage & réponse)

* [ ] **Détection** : alerte → classer severité ; assigner responsable (astreinte).
* [ ] **Triage 5–10 min** : `kubectl get events -A`, `top`, état ns/app, endpoints/probes.
* [ ] **Classes** : Image/Pull, Crash/OOM, Pending/Scheduling, Réseau/DNS, PV/Permissions, Ingress/LB, Policies.
* [ ] **Confinement** : réduire trafic (canary→0), basculer Service (blue→green), rollback.
* [ ] **Collecte preuves** : logs (`--previous`), events, manifest rendu, horodatage NTP.
* [ ] **Communication** : statut interne/externe, ticketing, time-line.
* [ ] **Remédiation** : correctif, tests, remise en service contrôlée.
* [ ] **Post-mortem** : cause racine, actions durables (policies, tests, alertes), échéances suivies.


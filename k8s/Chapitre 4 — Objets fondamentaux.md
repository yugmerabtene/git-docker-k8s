# Chapitre 4 — Objets fondamentaux

*(Pods, ReplicaSets, Deployments, Jobs/CronJobs, DaemonSets — initContainers, sidecars, probes, ressources, cycle de vie, stratégies de rollout, PDB, commandes d’inspection)*

---

## 1) Objectifs d’apprentissage

* Modéliser correctement des **workloads** stateless/batch : **Pod**, **ReplicaSet**, **Deployment**, **Job**, **CronJob**, **DaemonSet**.
* Comprendre le **cycle de vie** d’un Pod (création → exécution → arrêt) et ses hooks (**postStart**, **preStop**).
* Maîtriser les **probes** (**liveness**, **readiness**, **startup**) et l’allocation **CPU/RAM** (**requests/limits**, QoS).
* Savoir **déployer**, **mettre à jour** (rolling update) et **revenir en arrière** (rollback) un Deployment.
* Lancer des traitements **batch** avec **Job/CronJob** (parallélisme, retries, deadlines).
* Déployer des agents **node-wide** avec **DaemonSet** (logs/monitoring/IDS).
* Appliquer les bonnes pratiques : labels/selector, immutabilité, PDB (PodDisruptionBudget), stop gracieux.

---

## 2) Rappels rapides (API & structure)

Tout objet suit la structure :

```yaml
apiVersion: <groupe/vers>   # v1, apps/v1, batch/v1...
kind: <Type>                # Pod, Deployment, Job, ...
metadata:                    # nom, namespace, labels (fondamentaux !)
  name: ...
  labels: { app.kubernetes.io/name: ... }
spec:                        # état souhaité par vous
  ...
status:                      # état observé (rempli par K8s)
  ...
```

**Labels/Selectors** lient les ressources (p. ex. un **Service** ou un **ReplicaSet** cible des **Pods** via labels).

---

## 3) Pod — l’unité d’exécution

### 3.1 Pod minimal (commenté)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels: { app.kubernetes.io/name: hello }
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports: [ { containerPort: 80 } ]   # informatif
      # command: ["nginx","-g","daemon off;"]  # override ENTRYPOINT/CMD
      # args: ["-c","/etc/nginx/nginx.conf"]
      # env: [ { name: ENV, value: "prod" } ]
      # volumeMounts: [ { name: data, mountPath: /data } ]
  # volumes: [ { name: data, emptyDir: {} } ]
  # restartPolicy: Always   # (Pod “standalone”) Always/OnFailure/Never
  # terminationGracePeriodSeconds: 30
```

**Notes clés**

* **command/args** remplacent **ENTRYPOINT/CMD** de l’image (toujours tester !).
* **restartPolicy** : pour un Pod « seul » ; dans un **Deployment**, K8s force `Always`.
* **emptyDir**: volume éphémère (option `medium: Memory` pour tmpfs).

### 3.2 initContainers & sidecars

* **initContainers** : s’exécutent **avant** les containers applicatifs, **séquentiellement**, **doivent réussir**.
* **Sidecar** : deuxième conteneur coopérant (proxy, log shipper).

Exemple (init + sidecar) :

```yaml
spec:
  initContainers:
    - name: wait-db
      image: busybox:1.36
      command: [ "sh","-c","until nslookup db-svc; do sleep 2; done" ]
  containers:
    - name: app
      image: ghcr.io/acme/app:1.0.0
      ports: [ { containerPort: 8080 } ]
    - name: log-shipper
      image: fluent/fluent-bit:2.2
      volumeMounts: [ { name: varlog, mountPath: /var/log/app } ]
  volumes:
    - name: varlog
      emptyDir: {}
```

### 3.3 Hooks (postStart/preStop) & arrêt gracieux

* **postStart** : juste après le démarrage (non bloquant).
* **preStop** : avant l’envoi de **SIGKILL** (K8s envoie **SIGTERM**, attend *grace period*, puis **SIGKILL**).

```yaml
lifecycle:
  preStop:
    exec: { command: ["sh","-c","sleep 5"] }  # laisser le LB drainer
```

> Ajustez `terminationGracePeriodSeconds` pour que l’app puisse fermer proprement ses connexions.

---

## 4) Probes — liveness / readiness / startup

* **liveness** : “le process est-il vivant ?” → sinon **restart** du conteneur.
* **readiness** : “le Pod peut-il recevoir du trafic ?” → sinon **retiré** des Endpoints.
* **startup** : “le process a-t-il correctement démarré ?” → suspend l’évaluation de liveness/readiness tant qu’elle n’est pas OK (utile pour gros boots).

### 4.1 Exemple HTTP (classique)

```yaml
readinessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
livenessProbe:
  httpGet: { path: /live, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 10
startupProbe:
  httpGet: { path: /started, port: 8080 }
  failureThreshold: 30
  periodSeconds: 2       # ~60s max pour démarrer
```

Alternatives : `tcpSocket`, `exec`.

**Bonnes pratiques**

* **Toujours** une **readinessProbe** sur les services exposés.
* `startupProbe` pour les boots longs (évite redémarrages intempestifs).
* Attention aux timeouts trop courts (pics latence ≠ crash).

---

## 5) Ressources CPU/RAM & classes QoS

### 5.1 Requests & Limits

```yaml
resources:
  requests: { cpu: "200m", memory: "256Mi" }   # réservation (scheduling)
  limits:   { cpu: "500m", memory: "512Mi" }   # plafond (throttling / OOMKill)
```

* **CPU** : 1000m = 1 vCPU.
* **RAM** : valeurs formatées (`Mi`, `Gi`).

### 5.2 QoS (qualité de service)

* **Guaranteed** : *requests = limits* pour **tous** les conteneurs → protégés en pression.
* **Burstable** : requests définis mais pas égaux aux limits.
* **BestEffort** : **aucun** request/limit → sacrifiés en premier.

**Recommandations**

* Fixez **requests** réalistes (évite overscheduling).
* Visualisez la conso (`kubectl top pods`) et ajustez au fil du temps.

---

## 6) ReplicaSet & Deployment — le duo stateless

### 6.1 Deployment (de base)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels: { app.kubernetes.io/name: web }
spec:
  replicas: 3
  selector: { matchLabels: { app.kubernetes.io/name: web } }  # Immuable !
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 1 pod peut manquer pendant la MAJ
      maxSurge: 1        # 1 pod supplémentaire temporaire
  template:
    metadata: { labels: { app.kubernetes.io/name: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [ { containerPort: 80 } ]
          readinessProbe: { httpGet: { path: /, port: 80 }, initialDelaySeconds: 3 }
          resources:
            requests: { cpu: 100m, memory: 64Mi }
            limits:   { cpu: 300m, memory: 128Mi }
```

**Immuable** : `spec.selector` (matchLabels) **ne doit pas changer**. Changer les labels du template **cassera** l’association avec le ReplicaSet.

### 6.2 Commandes clés (rollout)

```bash
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pod -l app.kubernetes.io/name=web -o wide

kubectl set image deploy/web nginx=nginx:1.27.1   # déclenche un rolling update
kubectl rollout status deploy/web                # suivre la progression
kubectl rollout history deploy/web               # historique
kubectl rollout undo deploy/web                  # rollback

kubectl scale deploy/web --replicas=5            # scale out
```

### 6.3 Stratégies

* **RollingUpdate** (par défaut) : paramétrer **maxUnavailable**/**maxSurge**.
* **Recreate** : stop *tous* les Pods → start nouveaux (downtime, rare).
* **Canary** simple : créer **2 Deployments** (90/10 via Ingress/Service/mesh).

### 6.4 PodDisruptionBudget (PDB) — éviter les “drains” brutaux

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app.kubernetes.io/name: web } }
```

Empêche les **évictions volontaires** (drain, MAJ nœud) de descendre sous 2 Pods disponibles.

---

## 7) DaemonSet — un Pod par nœud (agents)

Exemples : logs, monitoring, sécurité (Fluent Bit, Promtail, Node Exporter, Falco…).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: node-exporter }
spec:
  selector: { matchLabels: { app.kubernetes.io/name: node-exporter } }
  template:
    metadata: { labels: { app.kubernetes.io/name: node-exporter } }
    spec:
      hostNetwork: true
      containers:
        - name: ne
          image: quay.io/prometheus/node-exporter:v1.8.1
          ports: [ { containerPort: 9100 } ]
```

* **hostNetwork**/montages hostPath possibles (attention aux droits).
* Tient compte des **taints/tolerations** (ajoutez des tolerations si vos nodes sont “tainted”).

---

## 8) Job — traitements batch

Exécuter un conteneur **jusqu’au succès**, avec **retries**.

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: etl-once }
spec:
  backoffLimit: 3               # nb de tentatives en cas d’échec
  activeDeadlineSeconds: 600    # deadline globale
  template:
    spec:
      restartPolicy: OnFailure  # (Job) OnFailure/Never (≠ Deployment)
      containers:
        - name: etl
          image: python:3.12-slim
          command: ["python","/app/run_etl.py"]
          resources:
            requests: { cpu: 200m, memory: 256Mi }
```

**Comportements**

* `completions` & `parallelism` permettent d’exécuter **N** tâches (éventuellement parallèles).

---

## 9) CronJob — planifier des Jobs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: prune-cache }
spec:
  schedule: "*/5 * * * *"           # cron (minute heure jour mois jour-semaine)
  timeZone: "Europe/Paris"          # (si supporté par version/cluster)
  concurrencyPolicy: Forbid         # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 120
  suspend: false
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: prune
              image: alpine:3.20
              command: [ "sh", "-lc", "echo prune && sleep 2" ]
```

**Bonnes pratiques**

* **concurrencyPolicy** : `Forbid` pour éviter le chevauchement.
* **History limits** pour ne pas polluer etcd.
* **startingDeadlineSeconds** en cas de raté temporaire.

---

## 10) Objets & ownership (garbage collector)

* Un **Deployment** **possède** un **ReplicaSet**, qui **possède** des **Pods** (OwnerReferences).
* Supprimer le **Deployment** enlève tout (selon `--cascade`).

```bash
kubectl delete deploy/web --cascade=background
kubectl get rs --show-owner
kubectl get pod -o jsonpath='{.metadata.ownerReferences}'
```

---

## 11) Diagnostics & commandes d’expert

### 11.1 Inspecter en profondeur

```bash
# Objets liés par labels
kubectl get deploy,rs,pod,svc -l app.kubernetes.io/name=web -o wide

# JSONPath utiles
kubectl get pod -l app.kubernetes.io/name=web -o jsonpath='{range .items[*]}{.metadata.name} {.status.podIP}{"\n"}{end}'

# Voir les classes QoS
kubectl get pod -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# Tri des events
kubectl get events --sort-by=.lastTimestamp | tail -n 30
```

### 11.2 `kubectl rollout` (tout le cycle)

```bash
kubectl set image deploy/web nginx=nginx:1.27.2
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=2
```

### 11.3 Patching fin

```bash
# Monter/descendre les replicas
kubectl patch deploy/web -p '{"spec":{"replicas":5}}'

# JSON Patch : changer la stratégie
kubectl patch deploy/web --type=json \
  -p='[{"op":"replace","path":"/spec/strategy/rollingUpdate/maxUnavailable","value":"25%"}]'
```

---

## 12) Pièges fréquents & correctifs

* **Selector ≠ labels du template** → le ReplicaSet **ne gère** aucun Pod.
  *Fix* : aligner `spec.selector.matchLabels` et `spec.template.metadata.labels`.

* **Rolling update bloqué** (readiness jamais OK) → Service n’a **pas** d’Endpoints.
  *Fix* : inspecter **readinessProbe** (timeouts/trop stricts), regarder `kubectl describe pod`.

* **OOMKill** (mémoire) / **CPU throttling** → requests/limits mal dimensionnés.
  *Fix* : `kubectl top`, ajuster les valeurs, tester sous charge.

* **Jobs qui bouclent** → `backoffLimit` insuffisant, **logs** masquent la vraie erreur.
  *Fix* : `kubectl logs job/<name>`, monter le niveau de logs app, `activeDeadlineSeconds`.

* **CronJob en retard** → pas de **timeZone** gérée/plus ancienne version, `startingDeadlineSeconds` trop bas.
  *Fix* : vérifier la version K8s, ajuster `startingDeadlineSeconds`, surveiller le contrôleur.

* **DaemonSet qui n’apparaît pas sur tous les nœuds** → taints non tolérés.
  *Fix* : ajouter **tolerations** dans le template Pod.

---

## 13) Bonnes pratiques (dès ce chapitre)

* **Labels normalisés** (`app.kubernetes.io/*`) partout (Deploy/RS/Pod/Svc/PDB).
* **Readiness** AVANT exposition ; **startupProbe** pour boots longs.
* **Requests/limits** **toujours** définis ; viser **Guaranteed** pour workloads critiques.
* **Déploiement par digest** `@sha256` (évite les surprises) et **pas `:latest`**.
* **PDB** sur services critiques (évite les downtime lors de drains).
* **Logs structurés** (clé/valeurs), **exit codes** cohérents pour Jobs.

---

## 14) Aide-mémoire (cheat-sheet)

```bash
# Déplois, listes, inspectes
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pod,svc -l app.kubernetes.io/name=web -o wide
kubectl describe deploy/web

# Rollout
kubectl set image deploy/web nginx=nginx:1.27.2
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web

# Scale & PDB
kubectl scale deploy/web --replicas=5
kubectl apply -f pdb.yaml

# Jobs & CronJobs
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/etl-once
kubectl apply -f cronjob.yaml
kubectl get cj,jobs
```


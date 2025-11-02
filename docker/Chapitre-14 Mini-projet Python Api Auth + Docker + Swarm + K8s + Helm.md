# Mini-projet Python — Web API + DB (Register/Login) 

Ce dépôt mono-fichier regroupe **tout le nécessaire** pour :
- une API Python (FastAPI) avec **register/login JWT** et `/users/me` ;
- un **PostgreSQL** packagé via **Docker Compose** ;
- des manifestes **Swarm** (stack) & **Kubernetes** ;
- un **Chart Helm minimal** prêt à publier ;
- des **Annexes** (Dockerfile patterns, Makefile, CI, sécurité, observabilité, runbooks…).

> Copier/Coller ce document dans votre éditeur puis scinder par fichiers selon l’arborescence ci-dessous.

---

## 0) Arborescence cible

```
mini-api/
  app/
    main.py
    auth.py
    database.py
    models.py
    schemas.py
    security.py
    __init__.py
    requirements.txt
    Dockerfile
  .env.example
  compose.yaml
  compose.dev.yaml
  compose.swarm.yaml
  k8s/
    namespace.yaml
    secret.yaml
    configmap.yaml
    postgres-statefulset.yaml
    postgres-service.yaml
    api-deployment.yaml
    api-service.yaml
    ingress.yaml
  chart/mini-api/
    Chart.yaml
    values.yaml
    templates/
      deployment.yaml
      service.yaml
      secret.yaml
      configmap.yaml
      ingress.yaml
      postgres-statefulset.yaml
      postgres-service.yaml
  annexes/
    Makefile
    dockerfile-patterns.md
    github-actions-ci.yml
    compose.logging.yaml
    runbook-release.md
    security-hardening.compose.yaml
```

---

## 1) Code API (FastAPI + SQLAlchemy + JWT)

### `app/requirements.txt`
```
fastapi==0.115.2
uvicorn[standard]==0.32.0
SQLAlchemy==2.0.36
psycopg2-binary==2.9.10
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
pydantic-settings==2.4.0
python-multipart==0.0.12
```

### `app/database.py`
```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DB_URL = os.getenv("DATABASE_URL")
if not DB_URL:
    # Fallback variables unitaires
    DB_USER = os.getenv("DB_USER", "app")
    DB_PWD = os.getenv("DB_PASSWORD", "app")
    DB_HOST = os.getenv("DB_HOST", "db")
    DB_NAME = os.getenv("DB_NAME", "app_db")
    DB_URL = f"postgresql+psycopg2://{DB_USER}:{DB_PWD}@{DB_HOST}:5432/{DB_NAME}"

engine = create_engine(DB_URL, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
Base = declarative_base()

# Utilitaires de session
from contextlib import contextmanager
@contextmanager
def session_scope():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except:
        session.rollback()
        raise
    finally:
        session.close()
```

### `app/models.py`
```python
from sqlalchemy import Column, Integer, String, DateTime, func, UniqueConstraint
from .database import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    full_name = Column(String(255), nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    __table_args__ = (
        UniqueConstraint('email', name='uq_users_email'),
    )
```

### `app/schemas.py`
```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)
    full_name: str | None = None

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class UserOut(BaseModel):
    id: int
    email: EmailStr
    full_name: str | None = None
    class Config:
        from_attributes = True

class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"
```

### `app/security.py`
```python
import os, datetime
from jose import jwt
from passlib.context import CryptContext

pwd_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")

# JWT secret: priorité au fichier si présent
JWT_SECRET = None
_secret_file = os.getenv("JWT_SECRET_FILE")
if _secret_file and os.path.exists(_secret_file):
    JWT_SECRET = open(_secret_file, 'rb').read().decode('utf-8').strip()
if not JWT_SECRET:
    JWT_SECRET = os.getenv("JWT_SECRET", "change-me-in-prod")
JWT_ALG = os.getenv("JWT_ALG", "HS256")
JWT_EXPIRE_MIN = int(os.getenv("JWT_EXPIRE_MIN", "60"))


def hash_password(p: str) -> str:
    return pwd_ctx.hash(p)


def verify_password(p: str, hashed: str) -> bool:
    return pwd_ctx.verify(p, hashed)


def create_access_token(sub: str) -> str:
    now = datetime.datetime.utcnow()
    payload = {
        "sub": sub,
        "iat": now,
        "exp": now + datetime.timedelta(minutes=JWT_EXPIRE_MIN),
    }
    return jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALG)


def decode_token(token: str) -> dict:
    return jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALG])
```

### `app/auth.py`
```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from .database import SessionLocal
from . import models, schemas
from .security import hash_password, verify_password, create_access_token, decode_token

router = APIRouter(prefix="/auth", tags=["auth"])
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("/register", response_model=schemas.UserOut, status_code=201)
def register(payload: schemas.UserCreate, db: Session = Depends(get_db)):
    if db.query(models.User).filter(models.User.email == payload.email).first():
        raise HTTPException(status_code=409, detail="Email already registered")
    user = models.User(
        email=payload.email,
        password_hash=hash_password(payload.password),
        full_name=payload.full_name,
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

@router.post("/login", response_model=schemas.Token)
def login(payload: schemas.UserLogin, db: Session = Depends(get_db)):
    user = db.query(models.User).filter(models.User.email == payload.email).first()
    if not user or not verify_password(payload.password, user.password_hash):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid credentials")
    token = create_access_token(str(user.id))
    return {"access_token": token, "token_type": "bearer"}

# Dépendance utilisateur courant
from fastapi import Request

def current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)) -> models.User:
    try:
        payload = decode_token(token)
        uid = int(payload.get("sub"))
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
    user = db.query(models.User).get(uid)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

### `app/main.py`
```python
import os
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import text
from .database import Base, engine, session_scope
from . import models
from .auth import router as auth_router, current_user

app = FastAPI(title="Mini API", version="1.0.0")

# CORS (dev par défaut)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Créer tables si absentes (démo simple)
Base.metadata.create_all(bind=engine)

@app.get("/health")
def health():
    with session_scope() as s:
        s.execute(text("SELECT 1"))
    return {"status": "ok"}

app.include_router(auth_router)

@app.get("/users/me")
def me(user=Depends(current_user)):
    return {"id": user.id, "email": user.email, "full_name": user.full_name}
```

### `app/__init__.py`
```python
# empty
```

### `app/Dockerfile`
```dockerfile
# ---- build stage (optionnel si besoin de deps natives) ----
FROM python:3.12-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app
COPY requirements.txt ./
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && pip install -r requirements.txt \
    && apt-get purge -y build-essential \
    && rm -rf /var/lib/apt/lists/*

# ---- runtime ----
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN adduser --disabled-password --gecos "" appuser
WORKDIR /app
COPY --from=base /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=base /usr/local/bin /usr/local/bin
COPY . .
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 2) Docker Compose

### `.env.example`
```
# Application
JWT_SECRET=change-me
JWT_EXPIRE_MIN=60

# Postgres
DB_USER=app
DB_PASSWORD=app
DB_NAME=app_db
DB_HOST=db
```

### `compose.yaml` (prod-like)
```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 3s
      retries: 10
    volumes:
      - data_pg:/var/lib/postgresql/data
    networks: [ backend ]

  api:
    build: ./app
    environment:
      - DATABASE_URL=postgresql+psycopg2://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - JWT_SECRET=${JWT_SECRET}
      - JWT_EXPIRE_MIN=${JWT_EXPIRE_MIN}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8000/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
    ports:
      - "8000:8000"
    read_only: true
    tmpfs: [ /tmp, /run ]
    networks: [ frontend, backend ]

networks:
  frontend: { driver: bridge }
  backend:  { driver: bridge, internal: true }

volumes:
  data_pg: {}
```

### `compose.dev.yaml` (développement rapide)
```yaml
services:
  api:
    build:
      context: ./app
    volumes:
      - ./app:/app
    environment:
      - DATABASE_URL=postgresql+psycopg2://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - JWT_SECRET=${JWT_SECRET}
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

> Lancer en dev : `docker compose -f compose.yaml -f compose.dev.yaml --env-file .env up -d`

---

## 3) Swarm (stack)

> Swarm lit `deploy.*`, secrets/configs managés par le cluster. Exemple minimal :

### `compose.swarm.yaml`
```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=app
      - POSTGRES_DB=app_db
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    deploy:
      placement:
        constraints: ["node.role == worker"]
    volumes: [ data_pg:/var/lib/postgresql/data ]
    networks: [ backend ]

  api:
    image: mini-api:1.0 # remplacez par l'image poussée au registry
    environment:
      - DATABASE_URL=postgresql+psycopg2://app:$(cat /run/secrets/db_password)@db:5432/app_db
      - JWT_SECRET_FILE=/run/secrets/jwt_secret
    secrets: [ jwt_secret, db_password ]
    deploy:
      replicas: 3
      update_config: { parallelism: 1, delay: 10s, order: start-first }
      rollback_config: { parallelism: 1, delay: 5s }
      resources:
        limits: { cpus: "1.0", memory: 512M }
    ports: [ "8000:8000" ]
    networks: [ frontend, backend ]

secrets:
  db_password: { external: true }
  jwt_secret:  { external: true }

networks:
  frontend: { driver: overlay }
  backend:  { driver: overlay }

volumes:
  data_pg: {}
```

**Commandes** :
```
docker swarm init
printf 'app\n' | docker secret create db_password -
printf 'change-me\n' | docker secret create jwt_secret -
# Construire & pousser l'image avant (ou utiliser une image publique)
# docker build -t REG/mini-api:1.0 ./app && docker push REG/mini-api:1.0

docker stack deploy -c compose.swarm.yaml mini
```

> Note : interpolation `$(cat /run/secrets/...)` n’est pas supportée nativement dans `environment`. Alternative : wrapper d’entrypoint qui lit les fichiers de secrets et exporte les variables avant de lancer Uvicorn.

---

## 4) Kubernetes (manifests YAML)

> Namespace `demo`, PostgreSQL en **StatefulSet** + PVC, API en **Deployment** + **Service** + **Ingress**.

### `k8s/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

### `k8s/secret.yaml` (DEV: non chiffré — base64 required)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mini-secrets
  namespace: demo
 type: Opaque
stringData:
  DB_USER: app
  DB_PASSWORD: app
  DB_NAME: app_db
  JWT_SECRET: change-me
```

### `k8s/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mini-config
  namespace: demo
 data:
  DB_HOST: postgres
```

### `k8s/postgres-statefulset.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: demo
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels: { app: postgres }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports: [ { containerPort: 5432 } ]
          env:
            - name: POSTGRES_USER
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_USER } }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_PASSWORD } }
            - name: POSTGRES_DB
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_NAME } }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources: { requests: { storage: 5Gi } }
```

### `k8s/postgres-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: demo
spec:
  ports: [ { port: 5432, targetPort: 5432 } ]
  selector: { app: postgres }
  clusterIP: None  # Headless pour StatefullSet
```

### `k8s/api-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, namespace: demo, labels: { app: mini, tier: api } }
spec:
  replicas: 2
  selector: { matchLabels: { app: mini, tier: api } }
  template:
    metadata: { labels: { app: mini, tier: api } }
    spec:
      containers:
        - name: api
          image: mini-api:1.0 # Remplacez par REG/mini-api@sha256:...
          ports: [ { containerPort: 8000 } ]
          env:
            - name: DB_USER
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_USER } }
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_PASSWORD } }
            - name: DB_NAME
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_NAME } }
            - name: DB_HOST
              valueFrom: { configMapKeyRef: { name: mini-config, key: DB_HOST } }
            - name: DATABASE_URL
              value: "postgresql+psycopg2://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):5432/$(DB_NAME)"
            - name: JWT_SECRET
              valueFrom: { secretKeyRef: { name: mini-secrets, key: JWT_SECRET } }
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 15
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
```

### `k8s/api-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata: { name: api, namespace: demo }
spec:
  selector: { app: mini, tier: api }
  ports: [ { port: 80, targetPort: 8000 } ]
  type: ClusterIP
```

### `k8s/ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mini
  namespace: demo
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: mini.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port: { number: 80 }
```

> Déploiement : `kubectl apply -f k8s/ -n demo` (créer d’abord le namespace). Pour du **digest pinning** : remplacez `image:` par `REG/mini-api@sha256:...`.

---

## 5) Chart Helm minimal

### `chart/mini-api/Chart.yaml`
```yaml
apiVersion: v2
name: mini-api
version: 0.1.0
appVersion: "1.0.0"
description: Mini API FastAPI + Postgres
```

### `chart/mini-api/values.yaml`
```yaml
image:
  repository: mini-api
  tag: "1.0"
  pullPolicy: IfNotPresent

db:
  user: app
  password: app
  name: app_db

ingress:
  enabled: true
  host: mini.local
```

### `chart/mini-api/templates/secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata: { name: mini-secrets }
type: Opaque
stringData:
  DB_USER: {{ .Values.db.user | quote }}
  DB_PASSWORD: {{ .Values.db.password | quote }}
  DB_NAME: {{ .Values.db.name | quote }}
  JWT_SECRET: {{ randAlphaNum 32 | quote }}
```

### `chart/mini-api/templates/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: mini-config }
data:
  DB_HOST: postgres
```

### `chart/mini-api/templates/postgres-statefulset.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: postgres }
spec:
  serviceName: postgres
  replicas: 1
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_USER
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_USER } }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_PASSWORD } }
            - name: POSTGRES_DB
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_NAME } }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources: { requests: { storage: 5Gi } }
```

### `chart/mini-api/templates/postgres-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata: { name: postgres }
spec:
  ports: [ { port: 5432, targetPort: 5432 } ]
  selector: { app: postgres }
  clusterIP: None
```

### `chart/mini-api/templates/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 2
  selector: { matchLabels: { app: mini, tier: api } }
  template:
    metadata: { labels: { app: mini, tier: api } }
    spec:
      containers:
        - name: api
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports: [ { containerPort: 8000 } ]
          env:
            - name: DB_USER
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_USER } }
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_PASSWORD } }
            - name: DB_NAME
              valueFrom: { secretKeyRef: { name: mini-secrets, key: DB_NAME } }
            - name: DB_HOST
              valueFrom: { configMapKeyRef: { name: mini-config, key: DB_HOST } }
            - name: DATABASE_URL
              value: "postgresql+psycopg2://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):5432/$(DB_NAME)"
            - name: JWT_SECRET
              valueFrom: { secretKeyRef: { name: mini-secrets, key: JWT_SECRET } }
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 5
          livenessProbe:
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 15
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
```

### `chart/mini-api/templates/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: mini, tier: api }
  ports: [ { port: 80, targetPort: 8000 } ]
  type: ClusterIP
```

### `chart/mini-api/templates/ingress.yaml`
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mini
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port: { number: 80 }
{{- end }}
```

> Installation : `helm upgrade --install mini chart/mini-api -n demo --create-namespace`.

---

## 6) Annexes

### `annexes/Makefile`
```makefile
SHELL := /bin/bash
ENV ?= .env

.PHONY: dev up down logs build test

dev:
	docker compose -f compose.yaml -f compose.dev.yaml --env-file $(ENV) up -d

up:
	docker compose --env-file $(ENV) up -d

down:
	docker compose down --remove-orphans

logs:
	docker compose logs -f --tail=200 api

build:
	docker build -t mini-api:1.0 ./app

test:
	curl -s localhost:8000/health | jq .
```

### `annexes/dockerfile-patterns.md`
```md
# Patterns Dockerfile (Python)
- Multi-stage pour réduire la taille.
- `pip install --no-cache-dir`, `.dockerignore` strict.
- USER non-root, FS read-only + tmpfs.
- Healthcheck au niveau Compose/K8s plutôt que dans l’image.
- Déployer par **digest** en prod.
```

### `annexes/github-actions-ci.yml`
```yaml
name: ci
on: { push: { branches: [ main ] } }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: ./app
          tags: mini-api:1.0
          load: true
      - name: Test
        run: |
          docker run -d -p 8000:8000 --name api mini-api:1.0
          sleep 3
          curl -f http://localhost:8000/health
```

### `annexes/security-hardening.compose.yaml`
```yaml
services:
  api:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs: [ /tmp, /run ]
    cap_drop: [ "ALL" ]
    cap_add: [ "NET_BIND_SERVICE" ]
```

### `annexes/compose.logging.yaml`
```yaml
services:
  fluentbit:
    image: cr.fluentbit.io/fluent/fluent-bit:2
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
    ports: [ "24224:24224" ]
```

### `annexes/runbook-release.md`
```md
# Runbook Release (Compose)
1. Pre-flight: digest signé, DB OK, NTP OK, disque OK.
2. Déployer green: `docker compose -p app-green ... up -d`.
3. Migrations: `compose run --rm migrate`.
4. Canary 10% via LB, observer métriques/logs 15–30 min.
5. 50%→100%, puis garder blue 24h.
6. Contract, nettoyage, post-mortem.
Rollback : poids LB → blue=100%.
```

---

## 7) Tests rapides

- **Register** :
  ```bash
  curl -X POST http://localhost:8000/auth/register \
    -H 'Content-Type: application/json' \
    -d '{"email":"a@b.com","password":"secret123","full_name":"Alice"}'
  ```
- **Login** :
  ```bash
  TOKEN=$(curl -s -X POST http://localhost:8000/auth/login -H 'Content-Type: application/json' -d '{"email":"a@b.com","password":"secret123"}' | jq -r .access_token)
  ```
- **/users/me** :
  ```bash
  curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/users/me
  

---

## Annexes (compléments demandés)

### annexes/annexe-a-air-gapped.md
```md
# Annexe A — Air-gapped & mirroring (environnements contraints)

## Objectifs
- Utiliser Docker et distribuer des images **sans accès direct à Internet**.
- Mettre en place un **miroir/proxy cache** et une **chaîne de synchronisation** d’artefacts OCI (images, SBOM, signatures).
- Garantir la **sécurité** (signatures, vérification offline) et une politique claire de **publication interne**.

## 1) Registry proxy/cache (pull‑through) avec `registry:2`
### 1.1 Exemple Compose (proxy Docker Hub)
```yaml
services:
  regcache:
    image: registry:2
    ports: ["5000:5000"]
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:5000
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
      # (optionnel) auth basique
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - regdata:/var/lib/registry
      - ./auth:/auth:ro
volumes:
  regdata: {}
```
**Client (daemon.json)** :
```json
{
  "registry-mirrors": ["http://<ip_du_registry>:5000"],
  "insecure-registries": ["<ip_du_registry>:5000"]
}
```
> **Prod** : placez le proxy derrière TLS (reverse‑proxy Nginx/HAProxy) + authentification.

### 1.2 Harbor comme miroir/registry d’entreprise
- Déploie : **registry**, **chartmuseum**, **notary**, **trivy** (scan), **GC** et **politiques**.
- Activer **proxy cache** vers Docker Hub/Quay/GHCR ; définir **rétention** et **immutabilité**.
- AuthN : LDAP/OIDC ; RBAC par projet.

## 2) Synchronisation offline (images & artefacts)
### 2.1 Copier images entre registries
- **skopeo** (rootless possible) :
```bash
skopeo copy docker://docker.io/library/nginx:1.27 \
            docker://reg.local:5000/cache/nginx:1.27
```
- **crane** (go-containerregistry) :
```bash
crane copy docker.io/library/nginx:1.27 reg.local:5000/cache/nginx:1.27
```

### 2.2 Artefacts OCI (SBOM, attestations)
- **oras** (OCI Artifacts) pour pousser un SBOM :
```bash
oras cp ghcr.io/acme/web:1.4.2 \
        reg.local:5000/acme/web:1.4.2 --include-subject
```
- **cosign** pour signatures :
```bash
cosign sign --key cosign.key reg.local:5000/acme/web:1.4.2
cosign attach sbom --sbom sbom.spdx.json reg.local:5000/acme/web:1.4.2
```

### 2.3 Stratégie air‑gap (workflow)
1. **Zone connectée** : builder → scanner → générer **SBOM**/**provenance** → **signer**.
2. **Exporter** : `skopeo/crane` + `oras/cosign` → bundle (images + artefacts) vers disque/clé.
3. **Importer** en zone air‑gapped → pousser vers **registry interne** (Harbor/registry:2).
4. **Déployer** depuis le **registry interne** uniquement.

## 3) Politique de publication interne (SSoT)
- **Single Source of Truth** : `registry.intra/ORG/PROJET/SERVICE`.
- **Promotion par digest** uniquement (pas de rebuild).
- **Tags** : `vX.Y.Z`, `X.Y`, `latest` **interdit en prod**.
- **Labels OCI** obligatoires : source, revision, builddate.

## 4) Sécurité (offline)
- **Clés cosign** stockées en HSM/KMS interne ; **CRL**/rotation planifiées.
- **Vérification offline** : 
```bash
cosign verify --key cosign.pub registry.intra/acme/web@sha256:...
```
- **GC**/rétention : nettoyer caches et tags périmés selon Annexe C.
```

### annexes/annexe-b-forensic-ir.md
```md
# Annexe B — Forensic & réponse à incident (IR)

## Objectifs
- **Préserver les preuves** (intégrité, horodatage) en environnement Docker.
- Capturer images, couches, logs, volumes, événements, traces réseau.
- Alignement **ISO 27001** (registre de preuves, chaîne de possession).

## 1) Principes clés
- **Ne pas altérer** la scène : éviter redémarrages, `docker commit` ; préférer **export**/**save**.
- **Horodatage fiable** (NTP), **hash** (SHA‑256) après chaque capture.
- **Chaîne de possession** : qui, quoi, quand, comment ; journaliser.

## 2) Collecte rapide (scriptable)
```bash
# Liste & états
docker ps -a > evidence/ps_a.txt

docker events --since=24h > evidence/events_24h.txt
journalctl -u docker --since "24 hours ago" > evidence/dockerd_24h.log

# Inspect + logs (chaque conteneur)
for c in $(docker ps -a --format '{{.ID}}'); do
  mkdir -p evidence/$c
  docker inspect $c > evidence/$c/inspect.json
  docker logs --since=24h $c > evidence/$c/logs_24h.txt || true
  docker diff $c > evidence/$c/diff.txt || true
  # FS conteneur (sans volumes)
  docker export $c | gzip -c > evidence/$c/export.tar.gz
  sha256sum evidence/$c/export.tar.gz >> evidence/hashes.txt
done

# Images utilisées
docker images --digests > evidence/images.txt
for i in $(docker images --format '{{.Repository}}@{{.Digest}}' | grep -v "<none>"); do
  name=$(echo $i | tr '/:@' '___')
  docker pull $i || true
  docker save $i | gzip -c > evidence/images/${name}.tar.gz
  sha256sum evidence/images/${name}.tar.gz >> evidence/hashes.txt
done
```

## 3) Volumes & données persistantes
```bash
# Archive d’un volume nommé (lecture seule via conteneur utilitaire)
VOL=data_pg
CID=$(docker create -v $VOL:/v alpine:3.20 sh)
docker cp $CID:/v - > evidence/vol_${VOL}.tar
sha256sum evidence/vol_${VOL}.tar >> evidence/hashes.txt
docker rm $CID
```
> Si LVM/Btrfs : préférer un **snapshot** filesystem du host, monté en RO pour l’archivage.

## 4) Réseau & processus
```bash
# Capture réseau ciblée
sudo tcpdump -nni any port 8000 -w evidence/capture_8000.pcap

# Espace netns du conteneur
PID=$(docker inspect -f '{{.State.Pid}}' <NAME>)
sudo nsenter -t $PID -n ss -lntp > evidence/net_sockets.txt

# Syscalls (court, ciblé)
sudo strace -fp $PID -o evidence/strace.txt -tt -T -e trace=network -r -f -qq & sleep 10; kill %1
```

## 5) Intégrité & emballage
```bash
find evidence -type f -exec sha256sum {} + > evidence/sha256_manifest.txt
tar --mtime='UTC 2025-01-01' --sort=name -czf evidence_bundle.tgz evidence/
sha256sum evidence_bundle.tgz > evidence_bundle.tgz.sha256
```

## 6) Bonnes pratiques
- **Gel** des preuves : copie immuable (WORM), coffre chiffré.
- **Pas de nettoyage** avant décision (management/IR lead).
- **Corrélation** : horodater avec UTC, consigner fuseau.

## 7) Cartographie ISO/IEC 27001:2022 (exemples)
- A.5.24 **Gestion des incidents** (processus documenté, registres).
- A.8.16 **Surveillance** (journaux dockerd/containers conservés).
- A.5.29 **Sécurité des informations pour l’utilisation des services cloud** (registre d’artefacts OCI, signatures).
- A.8.28 **Collecte de preuves** (intégrité, chaîne de possession).
```

### annexes/annexe-c-gouvernance-images.md
```md
# Annexe C — Politique de nommage, immutabilité & rétention d’images

## Objectifs
- Standardiser le **nomenclature** des images, leurs **versions**, et le **cycle de vie**.
- Empêcher les erreurs (tags mouvants), maîtriser coûts stockage via **rétention**.

## 1) Nommage & labels
- Chemin : `registry.intra/ORG/PROJET/SERVICE` (minuscule, `-` séparateur).
- Exemples :
  - `registry.intra/clairfact/pdp/api`
  - `ghcr.io/acme/shop/web`
- **Labels OCI** obligatoires (Dockerfile):
```dockerfile
LABEL org.opencontainers.image.source="https://git.example.com/acme/web" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.created="2025-11-01T00:00:00Z"
```

## 2) Versionning & tags
- **SemVer**: `v1.4.2` + canaux : `-rc`, `-beta`, `-dev`.
- Tags de confort : `v1.4`, `v1` autorisés **dans le registry**, pas utilisés par l’infra.
- **Interdit en prod** : `latest`.
- **Déploiement** : toujours **`@sha256:digest`**.

## 3) Immutabilité & promotion
- Tags **prod** immuables (Harbor règle d’immutabilité, ou webhook qui refuse repush).
- **Promotion par digest** :
```bash
DIGEST=$(crane digest registry.intra/acme/web:v1.4.2)
crane copy registry.intra/acme/web@${DIGEST} registry.intra/acme/web-prod:v1.4.2
```
- Admission (OPA/Conftest) : refuser `:latest`, exigence `USER` non-root, healthcheck, signature cosign.

## 4) Rétention & GC
- **Dev** : conserver 20 versions / 30 jours.
- **RC/QA** : 10 versions / 90 jours.
- **Prod** : conserver 10 versions **immuables** / 2 ans (ou selon exigence légale).
- **GC planifié** : hebdo (Harbor : tasks) ; sauvegarde métadonnées (digest, SBOM) avant purge.

## 5) Conformité & auditabilité
- **SBOM** & **signatures** conservés (artefacts OCI) et référencés dans le changelog release.
- Revue **licences** en CI (Trivy, FOSSA/Syft) ; blocage si non conforme.
- Traçabilité : mapping **commit → digest → release** (CI écrit un registre d’artefacts).

## 6) Modèle de politique (extrait)
```md
- Les images déployées en production DOIVENT être référencées par digest.
- Les tags de release (`vX.Y.Z`) sont immuables après publication.
- L’utilisation du tag `latest` est interdite sur les environnements QA/Prod.
- Toute image doit porter les labels OCI `source`, `revision`, `created`.
- Les images DOIVENT être signées (cosign) et accompagnées d’un SBOM.
- Les règles de rétention et GC suivantes s’appliquent : dev (20/30j), rc (10/90j), prod (10/2ans).
```
```


---

# COURS — Annexes A à C (liées au mini‑projet FastAPI/PostgreSQL)

> Format pédagogique : **Objectifs**, **Prérequis**, **Théorie**, **Ateliers pas‑à‑pas**, **Livrables**, **Critères d’évaluation**. Chaque annexe s’appuie sur le mini‑projet livré plus haut (`mini-api/`).

## Annexe A — Air‑gapped & mirroring (environnements contraints)

### Objectifs
- Déployer un **miroir/proxy cache** d’images pour accélérer/fiabiliser les builds du mini‑projet sans dépendre d’Internet.
- **Synchroniser** images & artefacts OCI (SBOM, signatures) entre un réseau connecté et une **zone air‑gapped**.
- **Signer** et **vérifier** des images hors‑ligne ; publier selon une **politique interne** (SSoT, promotion par digest).

### Prérequis
- Docker installé (hôte Linux/VM ou Docker Desktop). Port libre : `5000` (ou HTTPS derrière un reverse proxy).
- Espace disque ≥ 20 Go pour le cache d’images. Accès sudo sur l’hôte.
- Mini‑projet cloné et fonctionnel (`mini-api/`).

### Théorie (synthèse)
- **Registry proxy/cache** : `registry:2` en mode pull‑through (ou **Harbor** avec proxy + rétention + scan).
- **OCI Artifacts** : une image peut être accompagnée d’artefacts liés (SBOM, attestations) référencés par **digest**.
- **Outils** : `skopeo`/`crane` (copie images/digests), `oras` (artefacts OCI), `cosign` (signatures & vérification).
- **Flux air‑gap** : Build+scan+sig → **export** (zone connectée) → **import** (zone fermée) → déploiement depuis **registry interne**.

### Ateliers pas‑à‑pas
**Atelier A1 — Déployer un proxy cache Docker Hub**
1. Créez un répertoire `airgap/` et un `compose.yaml` proxy (exemple déjà fourni dans *annexes/annexe-a-air-gapped.md*).
2. Lancez : `docker compose up -d`. Vérifiez `docker logs -f regcache`.
3. Côté client, ajoutez au `daemon.json` la clé `"registry-mirrors"` pointant sur `http://<ip>:5000` puis redémarrez le daemon Docker.
4. Testez un pull : `docker pull python:3.12-slim` puis `postgres:16` et **mesurez** le temps au 1er et 2e pull (cache).

**Atelier A2 — Builder le mini‑projet en passant par le miroir**
1. Dans `mini-api/`, lancez : `docker compose build` (ou `docker build` dans `app/`).
2. Observez que les bases (`python`, `postgres`) proviennent du miroir (`regcache`).
3. Lancez l’app : `docker compose up -d` et testez `/health`.

**Atelier A3 — Synchronisation vers un registry interne (air‑gapped)**
1. Sur la **zone connectée**, préparez un bundle :
   - `crane copy docker.io/library/postgres:16 reg.local:5000/cache/postgres:16`
   - `crane copy docker.io/library/python:3.12-slim reg.local:5000/cache/python:3.12-slim`
   - `crane copy ghcr.io/<vous>/mini-api:1.0 reg.local:5000/mini/mini-api:1.0`
2. **Export** des artefacts : `cosign attach sbom ...` puis `oras cp ... --include-subject` (voir modèles fournis).
3. **Transférez** (disque/clé) → **Importez** vers le registry interne en zone air‑gapped.
4. **Retaguez** vos manifests (Compose/K8s/Helm) pour pointer **uniquement** sur `reg.local:5000/...`.

**Atelier A4 — Signatures & vérification hors‑ligne**
1. Générez une paire cosign (ou utilisez OIDC côté CI en zone connectée et **exportez la clé publique**).
2. Signez l’image : `cosign sign reg.local:5000/mini/mini-api:1.0`.
3. Vérifiez : `cosign verify --key cosign.pub reg.local:5000/mini/mini-api@sha256:...`.

### Livrables
- `compose.yaml` du proxy, captures des temps de pulls (avant/après cache).
- Transcript des commandes `crane/skopeo/oras/cosign` et liste des **digests** importés.
- Manifests (Compose/K8s/Helm) **retagués** vers le registry interne.

### Critères d’évaluation
- Build et run du mini‑projet **100 % via le miroir** (aucun pull externe pendant l’exécution).
- **Intégrité** : déploiement par **digest** + **signature vérifiée**.
- Dossier de preuves (scripts, logs, digests, SBOM) clair et réutilisable.

---

## Annexe B — Forensic & réponse à incident (IR)

### Objectifs
- Savoir **geler** un environnement Docker en incident sans détruire les preuves.
- Capturer images, conteneurs, volumes, logs, réseau pour la **chaîne de possession**.
- Produire un **bundle d’évidence** reproductible et vérifiable (hashes).

### Prérequis
- Accès root/sudo sur l’hôte, espace disque suffisant (`evidence/`). NTP activé.
- Mini‑projet en cours d’exécution (`docker compose up -d`).

### Théorie (synthèse)
- **Do no harm** : éviter `docker commit`/redémarrages — préférer *export/save* + copies RO.
- **Hashing** systématique (SHA‑256) et horodatage UTC.
- **Couverture** : états Docker, inspect, logs 24 h, images/digests, FS conteneur, **volumes**, réseau, syscalls courts.

### Scénario
- Suspicion de brute‑force sur `/auth/login` de l’API : pics d’erreurs 401, latence anormale.

### Ateliers pas‑à‑pas
**Atelier B1 — Collecte d’évidence du mini‑projet**
1. Créez `evidence/` et capturez : `docker ps -a`, `docker events --since=24h`, journaux `dockerd`.
2. Pour chaque conteneur : `docker inspect`, `docker logs --since=24h`, `docker diff`, `docker export` (+ `sha256sum`).
3. Images utilisées : `docker images --digests` puis `docker save` de chaque digest.

**Atelier B2 — Volumes & base de données**
1. Archivez le volume `data_pg` via un conteneur utilitaire (montage RO) et produisez un tar + hash.
2. Notez la **taille** et la **date** de chaque fichier clé (journaux Postgres, répertoires WAL si présents).

**Atelier B3 — Réseau & processus**
1. Capture réseau ciblée : `tcpdump` sur le port `8000` pendant 60 s (ou durée convenue).
2. `nsenter` dans le netns du conteneur API pour inventorier les sockets (`ss -lntp`).
3. Échantillon court `strace -fp <PID>` (10–15 s) pour constater les syscalls réseau.

**Atelier B4 — Emballage & manifeste d’intégrité**
1. `find evidence -type f -exec sha256sum {} + > evidence/sha256_manifest.txt`.
2. `tar -czf evidence_bundle.tgz evidence/` puis `sha256sum evidence_bundle.tgz`.
3. Documentez la **chaîne de possession** (qui a fait quoi/quand/comment).

### Livrables
- `evidence_bundle.tgz` + `sha256_manifest.txt`.
- Journal d’intervention (UTC), commandes exécutées, observations clés.
- Tableau de mapping : artefact ↔ source ↔ horodatage ↔ hash.

### Critères d’évaluation
- **Exhaustivité** (containers, images, volumes, réseau) et **intégrité** (hashes).
- Procédure **reproductible** ; aucun impact destructif sur l’environnement.
- Alignement **ISO 27001** : registres, horodatage, chaîne de possession.

---

## Annexe C — Politique de nommage, immutabilité & rétention d’images

### Objectifs
- Définir des **conventions homogènes** de nommage/version et les appliquer au mini‑projet.
- Garantir l’**immutabilité** des versions déployées et un **plan de rétention** maîtrisé.

### Prérequis
- Accès à un registry d’entreprise (GHCR/Harbor/ECR…).
- Pipeline CI de base (ou publication manuelle) pour pousser `mini-api`.

### Théorie (synthèse)
- **SemVer** + canaux (`-dev/-rc/-stable`) et **promotion par digest**.
- **Interdit** en prod : `:latest` ; **obligatoire** : labels OCI (source/revision/created), signatures, SBOM.
- **Rétention** adaptée par environnement (dev/rc/prod) + GC planifié.

### Ateliers pas‑à‑pas
**Atelier C1 — Nommage & labels pour mini‑api**
1. Choisissez un chemin canonique : `ghcr.io/<org>/mini/mini-api` (ou `registry.intra/mini/mini-api`).
2. Ajoutez au Dockerfile les labels OCI (source, revision, created). Rebuild.
3. Publiez `v1.0.0` puis **récupérez le digest** : `crane digest ...`.

**Atelier C2 — Immutabilité & déploiement par digest**
1. Remplacez partout l’image par le **digest** (`@sha256:...`) dans Compose/K8s/Helm.
2. Activez l’immutabilité des tags de release (politique Harbor/paramètres GHCR).
3. Testez un **rollback** simple en changeant le digest de référence et en relançant la stack.

**Atelier C3 — Rétention & GC**
1. Définissez une politique : dev (20/30j), rc (10/90j), prod (10/2 ans) et implémentez‑la dans le registry.
2. Planifiez le GC hebdo et un export des **métadonnées** (digests, SBOM, signatures) avant purge.

### Livrables
- Politique écrite (1–2 pages) + capture de configuration du registry.
- Manifests mis à jour pour le **digest pinning**.
- Log de promotion **par digest** (ex. `crane copy`), plus SBOM & signature associées.

### Critères d’évaluation
- Déploiement **immutabilisé** (digest partout) et tags de release **figés**.
- Politique de rétention appliquée et documentée (preuve GC/retention rules). 
- Traçabilité **commit → digest → release** clairement établie.


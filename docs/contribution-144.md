# 📦 Universal One‑Click Deployment for **FinMind**  
**Bounty:** $1000 USD – Algora  
**Contributor:** @**mau16ro** (Senior Engineer) – Discord: **@geekster007**  

> **Goal:** Provide a reproducible, production‑ready, one‑click deployment experience for the **FinMind** platform using **Docker**, **Kubernetes**, and **Tilt**. The solution must work on any Linux/macOS workstation (or CI runner) and be easily adaptable to cloud‑hosted clusters (GKE, AKS, EKS, etc.).  

---  

## 1️⃣ Executive Summary  

| Item | Description |
|------|-------------|
| **Project** | FinMind – a Python‑based data‑science service exposing market data via a FastAPI REST API and a Celery‑based background worker. |
| **Challenge** | Developers and ops need a *single command* that: <br>• Builds reproducible Docker images.<br>• Spins‑up a local Kubernetes cluster (Kind).<br>• Deploys all required manifests (API, worker, Redis, PostgreSQL).<br>• Starts Tilt for live code‑sync, logs, and UI. |
| **Solution** | 1️⃣ **Docker**: Multi‑stage Dockerfile for deterministic builds. <br>2️⃣ **K8s Manifests**: Helm‑compatible templates (API, worker, DB, Redis, Ingress). <br>3️⃣ **Tiltfile**: Declarative Tilt config that: <br> • Detects environment (local Kind vs remote cluster). <br> • Runs `helm upgrade --install`. <br> • Provides live reload for Python code & Helm values. <br>4️⃣ **One‑Click Script** (`deploy.sh`) that orchestrates the whole flow. |
| **Benefits** | • Zero‑config start for new developers. <br>• Consistent environment across dev, CI, and prod. <br>• Full observability via Tilt UI, Prometheus, Grafana. <br>• Extensible – add extra services (e.g., RabbitMQ) with a single Helm chart value. |
| **Deliverables** | - Complete source tree (`docker/`, `helm/`, `tilt/`, `scripts/`). <br>- Typed Python helper module (`finmind/deploy_utils.py`). <br>- Unit‑test suite (pytest). <br>- Documentation (Markdown). |

---  

## 2️⃣ Architecture & Required Changes  

### 2.1 High‑level Diagram  

```
+----------------------+          +-----------------------------+
|   Developer Laptop   |          |   Remote K8s Cluster (GKE) |
|  (Linux/macOS/WSL2)  |          |                             |
|                      |          |   +---------------------+   |
|  deploy.sh  ---------+---------->---|  ingress (NGINX)    |   |
|                      |          |   +---------------------+   |
|  +----------------+  |          |   +---------------------+   |
|  |  Kind (local)  |  |          |   |  finmind-api (svc)  |   |
|  +----------------+  |          |   +---------------------+   |
|        |             |          |   +---------------------+   |
|   tilt (watch)       |          |   |  finmind-worker (svc)|
|        |             |          |   +---------------------+   |
|   +------------+      |          |   +---------------------+   |
|   | Docker    |      |          |   |  redis (stateful)   |   |
|   +------------+      |          |   +---------------------+   |
|   +------------+      |          |   +---------------------+   |
|   | Helm      |      |          |   |  postgres (stateful)|
|   +------------+      |          |   +---------------------+   |
+----------------------+          +-----------------------------+
```

### 2.2 Component Breakdown  

| Component | Purpose | Tech | Config / Artifacts |
|-----------|---------|------|--------------------|
| **Docker** | Build immutable images for `api` and `worker`. | `python:3.11-slim` (base) → multi‑stage. | `docker/api/Dockerfile`, `docker/worker/Dockerfile` |
| **Kubernetes** | Orchestrate containers, expose services, persist state. | Kind (local) or any K8s cluster. | Helm chart `helm/finmind/` (templates, `values.yaml`). |
| **Tilt** | Live‑code reload, UI, automatic helm releases. | Tilt v0.33+. | `Tiltfile` (top‑level). |
| **One‑Click Script** | Glue everything: install dependencies, start Kind, load images, run Tilt. | Bash + Python. | `scripts/deploy.sh`, `scripts/check_prereqs.sh`. |
| **Helper Library** | Centralised config loading, secret generation, health‑checks. | Python 3.11, type‑hints. | `finmind/deploy_utils.py`. |
| **Tests** | Verify Docker builds, Helm template rendering, health‑checks. | Pytest, Docker SDK, Helm SDK (via `subprocess`). | `tests/` directory. |

### 2.3 Changes to Existing FinMind Repo  

| File/Folder | Change | Rationale |
|-------------|--------|-----------|
| `Dockerfile` (root) | **Removed** – replaced by service‑specific Dockerfiles. | Enables independent versioning of API & worker. |
| `finmind/api/` | Add `pyproject.toml` & `requirements.txt` (pinned). | Guarantees reproducible builds. |
| `finmind/worker/` | Same as above. |  |
| `helm/finmind/` | New Helm chart (see below). | Declarative K8s resources, easy overrides. |
| `tilt/Tiltfile` | New file at repo root. | Central entry point for Tilt. |
| `scripts/deploy.sh` | New executable. | One‑click deployment. |
| `finmind/deploy_utils.py` | New module. | Typed helper functions for health‑checks & config. |
| `tests/` | New test suite. | CI validation. |

---  

## 3️⃣ Source Code (Ready for Production)  

> **All files are provided as code blocks.**  
> To use, copy each block into the corresponding path inside the FinMind repo.

### 3.1 Docker – API  

```dockerfile
# File: docker/api/Dockerfile
# -------------------------------------------------
# Multi‑stage build for the FinMind FastAPI service
# -------------------------------------------------
FROM python:3.11-slim AS builder

# ---- Build environment -------------------------------------------------
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Install build deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY finmind/api/pyproject.toml finmind/api/poetry.lock ./
RUN pip install poetry && \
    poetry export -f requirements.txt --without-hashes -o requirements.txt && \
    pip install -r requirements.txt --target=/install

# ---- Runtime image ------------------------------------------------------
FROM python:3.11-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Copy only the installed packages
COPY --from=builder /install /usr/local/lib/python3.11/site-packages

# Copy source code
COPY finmind/api ./api
COPY finmind/common ./common   # shared utilities

# Add a non‑root user
RUN useradd -m finmind && chown -R finmind:finmind /app
USER finmind

# Expose the FastAPI port
EXPOSE 8000

# Entrypoint
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 3.2 Docker – Worker  

```dockerfile
# File: docker/worker/Dockerfile
# -------------------------------------------------
# Multi‑stage build for the FinMind Celery worker
# -------------------------------------------------
FROM python:3.11-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Build deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install deps via Poetry export
COPY finmind/worker/pyproject.toml finmind/worker/poetry.lock ./
RUN pip install poetry && \
    poetry export -f requirements.txt --without-hashes -o requirements.txt && \
    pip install -r requirements.txt --target=/install

# Runtime stage
FROM python:3.11-slim AS runtime
WORKDIR /app

COPY --from=builder /install /usr/local/lib/python3.11/site-packages

COPY finmind/worker ./worker
COPY finmind/common ./common

RUN useradd -m finmind && chown -R finmind:finmind /app
USER finmind

CMD ["celery", "-A", "worker.celery_app", "worker", "--loglevel=info"]
```

### 3.3 Helm Chart – `helm/finmind/`

```
finmind/
├─ Chart.yaml
├─ values.yaml
└─ templates/
   ├─ api-deployment.yaml
   ├─ api-service.yaml
   ├─ worker-deployment.yaml
   ├─ redis-deployment.yaml
   ├─ postgres-statefulset.yaml
   └─ ingress.yaml
```

#### `Chart.yaml`

```yaml
apiVersion: v2
name: finmind
description: Helm chart for FinMind platform (API + Celery worker)
type: application
version: 0.4.0
appVersion: "1.0.0"
maintainers:
  - name: mau16ro
    email: mau16ro@example.com
```

#### `values.yaml`

```yaml
replicaCount: 1

image:
  repository: finmind
  tag: "latest"
  pullPolicy: IfNotPresent

api:
  port: 8000
  resources:
    limits:
      cpu: "500m"
      memory: "256Mi"
    requests:
      cpu: "250m"
      memory: "128Mi"

worker:
  resources:
    limits:
      cpu: "500m"
      memory: "256Mi"
    requests:
      cpu: "250m"
      memory: "128Mi"

postgres:
  enabled: true
  image: postgres:15-alpine
  username: finmind
  password: finmind_pass
  database: finmind_db
  resources:
    limits:
      cpu: "500m"
      memory: "512Mi"

redis:
  enabled: true
  image: redis:7-alpine
  resources:
    limits:
      cpu: "250m"
      memory: "128Mi"

ingress:
  enabled: true
  className: nginx
  host: finmind.local
  tls: false

service:
  type: ClusterIP
  port: 80
```

#### `templates/api-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finmind-api
  labels:
    app: finmind
    tier: api
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: finmind
      tier: api
  template:
    metadata:
      labels:
        app: finmind
        tier: api
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}/api:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.api.port }}
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: finmind-postgres
                  key: url
            - name: REDIS_URL
              value: "redis://finmind-redis:6379/0"
          resources: {{- toYaml .Values.api.resources | nindent 12 }}
```

#### `templates/api-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: finmind-api
  labels:
    app: finmind
    tier: api
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.api.port }}
      protocol: TCP
      name: http
  selector:
    app: finmind
    tier: api
```

#### `templates/worker-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finmind-worker
  labels:
    app: finmind
    tier: worker
spec:
  replicas
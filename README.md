# incident-intel-platform
MLOps platform predicting production incidents from live service telemetry — FastAPI + XGBoost/MLflow, Kafka streaming, Next.js dashboard, deployed via Docker/Kubernetes/Terraform on AWS

# AI Incident Intelligence Platform

An end-to-end MLOps platform that predicts production incidents from live
service telemetry (error rate, latency, CPU/memory, deploy frequency) using
an XGBoost model tracked and served via MLflow, with a real-time streaming
path through Kafka and a Next.js dashboard.

## Architecture

```
                     ┌──────────────┐
                     │   Next.js    │  dashboard: score services,
                     │  Dashboard   │  browse recent incidents
                     └──────┬───────┘
                            │ REST
                     ┌──────▼───────┐        ┌───────────────┐
   telemetry ──────► │   FastAPI    │◄──────►│  MLflow        │
   (services)        │   Backend    │        │  Registry      │
                     └──┬───┬───┬───┘        └───────────────┘
                        │   │   │
              ┌─────────┘   │   └─────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │PostgreSQL│  │  Redis   │  │  Kafka   │──► incident-worker
        │(incidents)│ │ (cache)  │  │ (events) │    (async scoring)
        └──────────┘  └──────────┘  └──────────┘
```

## Tech stack

- **Backend**: Python, FastAPI, SQLAlchemy
- **ML**: XGBoost, MLflow (tracking + model registry)
- **Data**: PostgreSQL, Redis, Kafka
- **Frontend**: Next.js (App Router), SWR
- **DevOps**: Docker, Docker Compose, Kubernetes, Terraform (AWS EKS/RDS/ElastiCache), GitHub Actions CI/CD

## Repo layout

```
backend/            FastAPI service, XGBoost training, Kafka producer/consumer
frontend/           Next.js dashboard
k8s/                Kubernetes manifests (Deployments, Services, HPA, Ingress)
infra/              Terraform for AWS (EKS cluster, RDS, ElastiCache)
monitoring/         Prometheus scrape config
.github/workflows/  CI (test/lint/build) and CD (build, push, deploy) pipelines
docker-compose.yml  Full local stack: postgres, redis, kafka, mlflow, backend, worker, frontend
```

## Run it locally

```bash
cp .env.example .env
make up                 # builds + starts postgres, redis, kafka, mlflow, backend, worker, frontend
make train               # trains + registers the first XGBoost model in MLflow
```

- Dashboard: http://localhost:3000
- API docs: http://localhost:8000/docs
- MLflow UI: http://localhost:5000

Until `make train` has run once, `/predict` falls back to a deterministic
heuristic so the API and dashboard stay usable end-to-end.

## Deploying

- **CI** (`.github/workflows/ci.yml`): lints and tests the backend, builds
  the frontend, and builds both Docker images on every push/PR.
- **CD** (`.github/workflows/cd.yml`): on push to `main`, builds and pushes
  images to GHCR, then applies the manifests in `k8s/` to the cluster
  referenced by the `KUBE_CONFIG_DATA` repo secret.
- **Infra** (`infra/`): `terraform init && terraform apply` provisions an
  EKS cluster, RDS Postgres, and ElastiCache Redis on AWS.

Before your first deploy: copy `k8s/secret.yaml.example` to `k8s/secret.yaml`
with real values (it's gitignored), and set the `KUBE_CONFIG_DATA` secret in
your GitHub repo settings.

## Retraining

`backend/app/ml/train.py` currently trains on a synthetic telemetry dataset
so the pipeline is runnable end-to-end out of the box — swap
`_make_synthetic_dataset()` for a real query against your incident history
once you have labeled data.

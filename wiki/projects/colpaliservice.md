---
type: project
status: active
created: 2026-09-24
updated: 2026-09-24
tags: [python, fastapi, llm, embeddings, kubernetes, aws, data-engineering, colpali]
repo: "git@git.cybersoftbpo.com:data-engineering/colpaliservice.git"
---

# colpaliservice

## Purpose

FastAPI service owned by the [[CyberSoft]] data-engineering group. Provides
an HTTP API over an embeddings + PDF-processing pipeline: ingests PDFs from
S3, generates dense embeddings via `fastembed` (`BAAI/bge-small-en-v1.5`),
and persists task/log records to PostgreSQL. Consumed as the retrieval side
of downstream document workflows.

## Status

Active, production. Deployed to the `colpali` namespace on
[[ccsi-msd-prd EKS cluster]]. Public hostname `colpali.cybersoftbpo.ai`
(ALB + ACM + external-dns). HPA `min 1 / max 10`.

## Architecture (durable knowledge)

**Language / framework:** Python 3.11, FastAPI + Uvicorn (module entry
`colpaliapi:app`). Gunicorn is in `requirements.txt` but not used in the
container CMD.

**Key libraries:**

- `fastembed` + `llama-index-embeddings-fastembed` — embeddings
  (`BAAI/bge-small-en-v1.5`, cached to `localfastembed/`)
- `llama-index`, `llama-index-workflows`, `llama-index-instrumentation`
- `PyMuPDF`, `pdf2image` (needs `poppler-utils` at the OS layer)
- `psycopg2-binary` — Postgres client
- `boto3` — S3 access
- `slowapi` — per-IP rate limiting

**Runtime dependencies:**

- **Postgres RDS:** `colpali_db` on `prdcpluw2pql01…us-west-2.rds.amazonaws.com`
- **S3:** `s3://dnb-ai-bucket-staging/` (us-west-2)
- **ECR image:** `872194582181.dkr.ecr.us-west-2.amazonaws.com/colpali`

**Container:**

- Base: `python:3.11-slim`, non-root `appuser`
- Exposes `8000`; CMD is `uvicorn colpaliapi:app --host 0.0.0.0 --port 8000`
- OS deps: `poppler-utils`, `libpq-dev`, `build-essential`
- `process.py` is intentionally excluded from the image copy

**Security posture:**

- `X-API-Key` header required on all endpoints except `/healthz`, compared
  via `hmac.compare_digest`
- FastAPI initialized with `docs_url=None, redoc_url=None` — Swagger/ReDoc
  disabled at app level
- ALB Ingress *also* returns a fixed 404 for `/docs`, `/redoc`,
  `/openapi.json` — defense-in-depth if the FastAPI defaults ever regress
- Rate limiting via `slowapi` keyed on `X-Forwarded-For` first hop
- Runs as non-root inside the container

**Kubernetes shape:**

- ConfigMap `colpali-configmap` — non-secret env (DB host/port/name, log
  paths, S3 region + bucket)
- Secret `colpali-secret` — DB username/password, S3 access/secret keys,
  `X_API_KEY`; **synced from local gitignored `k8s/colpali.env` at deploy
  time**, never committed
- Deployment `colpali-deployment`, single container `colpali-api`
- Resources: `250m` / `1Gi` requests, `1000m` / `2Gi` limits
- Readiness + liveness probes both hit `/healthz` on `8000`
- Service `colpali-service` (ClusterIP, `80 → 8000`)
- Ingress `colpali-ingress` — ALB, public, `colpali.cybersoftbpo.ai`,
  ACM cert, `/docs|/redoc|/openapi.json` denied, healthcheck `/healthz`

## Deploy sequence

`./deploy.sh [git-sha]` — SHA defaults to `git rev-parse --short HEAD`.
Pipeline:

1. Verify `k8s/colpali.env` exists (gitignored).
2. `aws sts get-caller-identity` (default profile `msd-admin`, region
   `us-west-2`).
3. ECR login.
4. `docker build --platform linux/amd64` tagged with the short SHA **and**
   `:production`; push both.
5. `kubectl apply -f k8s/kube.yaml -f k8s/hpa.yaml`.
6. `kubectl create secret generic colpali-secret --from-env-file=k8s/colpali.env`
   piped through `kubectl apply -f -` (idempotent secret sync).
7. `kubectl set image deployment/colpali-deployment colpali-api=<image>:<sha>`
   — pins to the immutable SHA even though the manifest references
   `:production`.
8. `kubectl rollout status … --timeout=5m`.

**Rollback:** `kubectl -n colpali rollout undo deployment/colpali-deployment`,
or re-run `./deploy.sh <older-sha>`.

## Operational knowledge

- **`:production` tag vs SHA tag.** The manifest hardcodes `:production`
  for bootstrap-from-scratch, but every deploy overrides via
  `kubectl set image` with the SHA. Result: reproducible rollbacks by
  SHA, plus a human-readable "latest prod" reference tag.
- **Secrets never in the manifest.** `k8s/colpali.env` is gitignored; the
  Secret is (re)created from it at deploy time. `k8s/colpali.env.example`
  is the tracked template.
- **Namespace already provisioned.** `colpali` namespace is enumerated on
  the [[ccsi-msd-prd EKS cluster]] page as an active workload.

> [!warning] Historical secret leak
> Prior to 2026-09-24, `k8s/kube.yaml` on `main` embedded hardcoded
> `DATABASE_PASSWORD`, `S3_WEBHUB_ACCESS_KEY`, and `S3_WEBHUB_SECRET_KEY`
> in the ConfigMap. They were removed in the deploy-scaffold work on
> branch `chore/jhonatsz`, but remain reachable through git history.
> **Rotate the AWS key and DB password regardless of the manifest fix.**

## Related

- [[ccsi-msd-prd EKS cluster]] — hosting cluster
- [[CyberSoft]] — owning organization (data-engineering group)

## Sources

- Repo: `git@git.cybersoftbpo.com:data-engineering/colpaliservice.git`
- Captured state: branch `chore/jhonatsz`, HEAD `44455c1` rebased onto
  `origin/main` `1d4b43b` on 2026-09-24

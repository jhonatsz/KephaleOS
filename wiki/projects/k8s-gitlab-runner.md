---
type: project
status: active
created: 2026-09-13
updated: 2026-09-13
aliases:
  - "GitLab Runner on EKS"
  - "Cybersoft GitLab Runner"
  - "k8s-gitlab-runner"
  - "eks-gitlab-runner"
tags: [kubernetes, eks, gitlab-ci, cybersoft, dind, irsa, docker-in-docker]
repo: git@git.cybersoftbpo.com:devops/k8s-gitlab-runner.git
time-sensitive: true
confidence: high
---

# Project: k8s-gitlab-runner

## Purpose

Helm-based deployment of the official GitLab Runner chart to AWS EKS
(`us-west-2`), providing shared CI/CD execution capacity for the
`git.cybersoftbpo.com` GitLab instance. Runner manager registers as
`eks-gitlab-runner` (prod) / `eks-gitlab-runner-production` and spawns
short-lived Kubernetes job pods, each with a privileged Docker-in-Docker
sidecar for image builds.

## Context

- Organization: CyberSoft (DevOps)
- Repository: `git@git.cybersoftbpo.com:devops/k8s-gitlab-runner.git`
- Two environments managed from this repo: **nonprod** and **production**
- Prod deployment target: [[ccsi-msd-prd EKS cluster]] (high confidence — matches namespace, region, and observed workload in the 2026-09-12 incident)
- Nonprod deployment target: separate cluster (name not yet captured in Kephaleos)
- AWS account: `872194582181` (shared for both envs)

## Architecture (durable knowledge)

**Deployment shape.** One `Deployment` running the runner manager (1 replica).
The manager polls GitLab, and for each accepted CI job it creates a fresh
`Pod` on EKS with three containers:

| Container | Image | Role |
|-----------|-------|------|
| `build`   | `docker:27.3.1` | Runs the `.gitlab-ci.yml` script |
| `docker`  | `docker:27.3.1-dind` (privileged) | Docker daemon for image builds |
| `helper`  | GitLab-provided | Clone, cache, artifact upload |

Shared via `emptyDir` volumes: `/var/run` (docker socket), `/var/lib/docker`
(10Gi cache, `overlay2`), `/certs/client` (memory-backed). BuildKit enabled
(`DOCKER_BUILDKIT=1`).

**Concurrency.** 4 jobs in flight per runner.

**Per-job resources** (sized for `t3.large` nodes):
- build: `1500m` CPU / `3Gi` memory
- dind service: `1000m` / `2Gi`
- helper: `200m` / `256Mi`

**Node pinning.** Dedicated node group with label `role=gitlab-runner` and
taint `dedicated=gitlab-runner:NoSchedule`. Both the manager pod and the
runner-spawned job pods carry the corresponding `nodeSelector` and
`tolerations` — but only in `values.yaml` (nonprod). See _Known issues_.

**AWS auth (IRSA).** ServiceAccount `gitlab-runner` in namespace
`gitlab-runner` is annotated with
`eks.amazonaws.com/role-arn: arn:aws:iam::872194582181:role/gitlab-runner-irsa-role`.
The IAM trust policy scopes assumption to
`system:serviceaccount/gitlab-runner/gitlab-runner` on the EKS OIDC provider
`oidc.eks.us-west-2.amazonaws.com/id/EA249663CA256FF8F2153C3448EF8B4F`.
No static AWS keys anywhere. Trust policy lives at
`docs/gitlab-runner-trust.json` in the repo.

**RBAC.** Cluster-wide with full CRUD on `pods`, `services`, `configmaps`,
`secrets`, `deployments`, `replicasets`, `jobs`, `cronjobs`. Any CI job can
touch these anywhere in the cluster. See _Security tradeoffs_.

**Deploy command** (production):

```bash
kubectl apply -f yamls/secret-prd.yaml
helm upgrade --install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner \
  --values values-production.yaml
```

Non-production is symmetric, using `values.yaml` + `yamls/secret.yaml`.

**Repo layout** (post-cleanup 2026-09-13):

```
README.md
values.yaml              ← nonprod
values-production.yaml   ← production
yamls/
  secret.yaml            ← nonprod runner-registration-token
  secret-prd.yaml        ← prod runner-registration-token
docs/
  gitlab-runner-trust.json   ← one-time IRSA bootstrap
```

## Key decisions

- Executor: `kubernetes` (not `docker+machine`, not `shell`) — leverages EKS
  scheduling, no persistent EC2 fleet to manage.
- DinD sidecar with `privileged: true` rather than Kaniko / rootless BuildKit
  — accepted container-escape risk in exchange for docker-native builds and
  ecosystem parity. Revisit if RBAC scope tightens or blast radius becomes
  a compliance issue.
- IRSA (not static AWS keys, not `kube2iam`) — standard EKS pattern.
- One IAM role shared across nonprod and prod (`gitlab-runner-irsa-role`) —
  same AWS account for both envs. Unusual; note for future separation if
  environments diverge.

## Security tradeoffs

- **Privileged DinD.** Container escape risk. Alternatives: Kaniko, rootless
  BuildKit — no current appetite to migrate.
- **Cluster-wide RBAC with `secrets: delete`.** Big blast radius. Any CI
  job can read/write/delete secrets anywhere in the cluster.
- **Tokens historically committed to `yamls/secret*.yaml`.** Real `glrt-`
  tokens have been in git history (at least twice: nonprod and prod values,
  observed 2026-09-13). Deletion cleans the working tree, not git history.
  On any exposure: **rotate in GitLab admin first**, then update secret. Longer-term,
  move to External Secrets or Sealed Secrets. See
  [[Runbook: GitLab Runner token rotation]].

## Known issues

> [!warning] Hypothesis (2026-09-13)
> `values-production.yaml` does **not** set `nodeSelector` / `tolerations`
> for either the manager pod or the runner-spawned job pods.
> `values.yaml` (nonprod) does. If the prod cluster contains node groups
> beyond the dedicated `role=gitlab-runner` one, prod pods may schedule
> off-target. If the prod cluster is entirely composed of dedicated runner
> nodes, this is harmless. Verify with
> `kubectl -n gitlab-runner get pods -o wide` before "fixing."

## Recent maintenance

- **2026-09-13** — Cleanup pass: moved `gitlab-runner-trust.json` from repo
  root into `docs/`; removed unused `images/Dockerfile` (custom builder image
  that was never wired into any build/push flow — runner uses
  `docker:27.3.1` from Docker Hub directly). Rewrote `README.md` from a
  5-line install snippet to a full service doc with Mermaid architecture
  diagram, deploy/ops/security sections. Commits `1074466`, `e39d25d`.

## Lessons learned

- [[HPA memory-requests pin at max]] — this runner was blocked by an
  unrelated HPA misconfiguration in `sbiqai` (2026-09-12); the runner
  itself is fine, but its scheduling depends on the shared cluster's
  reservation state.

## Related

- [[Incident: GitLab Runner CPU reservation]] — runner pod stuck in
  `Pending` on `ccsi-msd-prd` due to cluster-wide CPU-reservation
  saturation
- [[ccsi-msd-prd EKS cluster]] — deployment target for production runner
- [[Runbook: GitLab Runner token rotation]]

## Sources

- Local repo working directory: `/Users/jhonatsz/workspace/cybersoft/k8s-gitlab-runner`
- Git remote: `git@git.cybersoftbpo.com:devops/k8s-gitlab-runner.git`
- Compiled from live inspection of `values.yaml`, `values-production.yaml`,
  `yamls/secret*.yaml`, `docs/gitlab-runner-trust.json`, `README.md`, and
  commits `e77031c`, `1074466`, `e39d25d` on `main` (state as of 2026-09-13).

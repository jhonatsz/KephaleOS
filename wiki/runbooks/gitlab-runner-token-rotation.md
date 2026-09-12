---
type: runbook
status: active
created: 2026-09-13
updated: 2026-09-13
aliases:
  - "Rotate GitLab Runner token"
  - "GitLab Runner token rotation"
tags: [gitlab-ci, kubernetes, eks, secret-rotation, cybersoft]
last-executed: null
---

# Runbook: GitLab Runner token rotation

## When to use

- A `glrt-` runner registration/authentication token has been exposed
  (committed to git, posted in Slack, pasted in a ticket, etc.).
- Scheduled rotation as part of periodic secret hygiene.
- Suspected compromise of a shared CI credential.

Applies to the [[k8s-gitlab-runner]] deployments on
[[ccsi-msd-prd EKS cluster]] and its nonprod counterpart.

## Prerequisites

- GitLab admin access on `git.cybersoftbpo.com`
- `kubectl` context on the target cluster with permission to write
  Secrets in the `gitlab-runner` namespace and restart the runner
  Deployment
- Local clone of `git@git.cybersoftbpo.com:devops/k8s-gitlab-runner.git`
- Ability to edit `yamls/secret.yaml` (nonprod) or `yamls/secret-prd.yaml`
  (prod) **without committing real token values**

## Steps

1. **Reset the token in GitLab.**
   GitLab → **Admin → CI/CD → Runners** → select the affected runner →
   **Reset token**. Copy the new `glrt-…` value.
   *Expected outcome:* GitLab shows a new token; the old one immediately
   stops working.

2. **Update the K8s Secret** (do **not** commit the file with the real
   value):
   ```bash
   kubectl -n gitlab-runner create secret generic gitlab-runner-secret \
     --from-literal=runner-registration-token='<NEW_TOKEN>' \
     --from-literal=runner-token='<NEW_TOKEN>' \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
   *Expected outcome:* `secret/gitlab-runner-secret configured` on stdout.

3. **Restart the runner Deployment** so it picks up the new token:
   ```bash
   kubectl -n gitlab-runner rollout restart deploy/gitlab-runner
   kubectl -n gitlab-runner rollout status deploy/gitlab-runner
   ```
   *Expected outcome:* new runner pod comes up `Running`, old one
   terminates.

4. **If the leaked token lived in git history**, decide with the team:
   accept as burned (already rotated in step 1 — the value is now dead),
   or rewrite history (`git filter-repo`) and force-push. History rewrite
   is disruptive; usually rotation is enough.

## Verification

- Runner pod logs show successful registration:
  `kubectl -n gitlab-runner logs deploy/gitlab-runner | grep -i registered`
- GitLab **Admin → Runners** shows the runner as **online** with a recent
  contacted-at timestamp.
- Kick off a trivial CI job on any project targeted at the affected runner
  (`tags: eks-runner` or `k8s-runner-production`) and confirm the job pod
  is scheduled and completes.

## Rollback / recovery

- If the runner fails to register after restart:
  - Re-check that the token was pasted correctly (no trailing newline).
  - Confirm GitLab didn't lock the runner during reset (`locked: false` in
    values; verify in Admin UI).
  - Revert to the previous secret only if you still have the old token
    somewhere — usually you don't. Reset a fresh token via step 1 and try
    again.

## Gotchas

- `values.yaml` sets `runners.secret: gitlab-runner-secret` — the runner
  reads the token from the Secret, not from Helm values. Do **not** put a
  real token in `runnerRegistrationToken:` in the values file.
- Never `git commit` a real `glrt-` token. Anti-pattern already burned this
  repo (see [[k8s-gitlab-runner]] → Security tradeoffs).
- Both keys in the Secret (`runner-registration-token`, `runner-token`)
  are typically set to the same value in this deployment.
- On uninstall, `unregisterRunners: true` deregisters the runner from
  GitLab — a rotation is preferable to an uninstall/reinstall.

## Related

- [[k8s-gitlab-runner]]
- [[ccsi-msd-prd EKS cluster]]

## Sources

- Compiled from `values.yaml` and `yamls/secret*.yaml` in
  `git@git.cybersoftbpo.com:devops/k8s-gitlab-runner.git` (2026-09-13).

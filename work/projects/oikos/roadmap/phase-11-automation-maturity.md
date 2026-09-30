---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-11, automation, ansible, terraform, gitops, netbox, ci-cd]
confidence: high
---

# Phase 11 — Automation Maturity

## Objective

Bring Oikos to a state where **every non-trivial change is expressed in code, reviewed via a pull request, and applied through CI/CD**. Not "everything automated." Every operational change: reproducible, auditable, recoverable.

## Why now

This phase is a *horizontal thread* — Ansible starts in Phase 0, GitOps arrives in Phase 9, IaC coverage grows every phase. Phase 11 exists to formalize what's been building all along and fill the remaining gaps (secrets, network automation, DCIM, drift detection).

**Invariant 4 still applies**: automate what you've configured manually and understand. Do not automate anything you can't restore by hand.

## Prerequisites

- Phases 0–10 complete
- Homelab repo has been the source of truth for VM templates, k8s manifests, LiteLLM configs
- You've been bitten at least once by manual drift (this is a feature, not a bug)

## Hardware

None new. Automation runs on existing compute.

## Software

- **Ansible** — playbooks + roles for host/service config
- **OpenTofu** (or Terraform) — VM provisioning, DNS records, firewall rules where APIs allow
- **NetBox** — IPAM + DCIM (source of truth for addresses, VLANs, cabling)
- **ArgoCD** — already deployed (Phase 9), extended coverage
- **Renovate** or **Dependabot** — automated dependency PRs
- **pre-commit** — linters + security scans on every commit (`ansible-lint`, `tflint`, `yamllint`, `gitleaks`, `trivy-config`)
- **Gitea Actions** or **GitHub Actions** — CI to test PRs, apply on merge
- **HashiCorp Vault** or **Bitwarden Secrets Manager** — dynamic secrets, cert issuance (or upgrade Sealed-Secrets → External Secrets Operator + Vault)
- **Nornir/NAPALM** or **Netmiko** — for network device configuration
- **ntfy** / **Slack** / **Telegram** — automation alerting

## Network changes

- New VMs on VLAN 20:
  - `netbox01.oikos.home.arpa` — NetBox
  - `vault01.oikos.home.arpa` — Vault (or use k8s for HA Vault, ADR the decision)
  - `ci01.oikos.home.arpa` — Gitea + Gitea Actions runners (or self-hosted GitHub Actions runners)
- Renovate has scheduled outbound to Internet (GitHub API + registries)
- Vault talks to k8s auth backend

## Implementation sequence

1. **NetBox first** — because IPAM should be the source of truth *before* automation writes anything:
   - Deploy NetBox
   - Import current-state.md inventory + address plan into NetBox
   - `current-state.md` becomes a derived view (or is retired in favor of NetBox exports)
2. **Vault** — deploy; unseal strategy documented; k8s auth method configured
   - Migrate at least one secret from Sealed-Secrets to External Secrets Operator + Vault to prove pattern
3. **CI/CD baseline** — Gitea (self-hosted git if not already) or GitHub with runners
   - Every PR triggers: lint, secret-scan, dry-run
   - Merge to main triggers: apply (Terraform `apply`, Ansible playbook, ArgoCD sync)
4. **Ansible playbooks** — one role per service touched so far:
   - `proxmox-base` (from Phase 0's manual install)
   - `opnsense-baseline` (from Phase 1)
   - `switch-baseline` (from Phase 2; vendor-specific)
   - `adguard` (from Phase 3)
   - `obs01-stack` (from Phase 5)
   - `nas-baseline` (from Phase 7)
   - `samba-ad-dc` (from Phase 8)
   - Each role has a `tests/` dir with molecule or a lightweight smoke test
5. **Terraform/OpenTofu** modules:
   - `proxmox-vm` — the atomic VM creator
   - `dns-record` — LiteLLM DNS records, k8s ingress records
   - `firewall-rule` — OPNsense API
   - State: encrypted, stored on NAS (`nas01:/tank/terraform-state`) with locking
6. **Renovate / Dependabot** — enable on the homelab repo; weekly PRs for image tags, helm chart versions, provider versions
7. **Pre-commit hooks** — enforce lint + secret-scan locally + in CI
8. **Drift detection** — nightly job that runs `terraform plan` + `ansible-playbook --check` and posts diffs to ntfy/Slack
9. **Network automation** — Nornir/NAPALM inventory sourced from NetBox; playbook that pushes VLAN config to the switch (start read-only with `--dry-run` for a month before actually pushing)
10. **Runbook automation** — the top 5 runbooks (proxmox-install, opnsense-config-backup, dc-backup, k8s-add-worker, cert-rotation) get an "automated" section referencing the playbook

## CCNA topics reinforced

- Network programmability / APIs (RESTCONF / NETCONF on modern IOS-XE; SSH-driven Netmiko/NAPALM on classic IOS)
- YANG models (advanced; skim only)
- Data-driven network config: NetBox → templates → device
- Version control for network config
- CI/CD applied to network changes (blast-radius awareness)

## Study track lab

- **EVE-NG**: Nornir/Netmiko playbook that reads a YAML topology and configures 3 routers end-to-end
- Practice `git blame` on network configs — if you can't answer "why is this rule here" from git history, the automation isn't done

## Physical Oikos lab exercise

- Make a change to `172.27.60.0/24` (IoT) via NetBox → automation pipeline → OPNsense reflects it. Compare to the manual path.
- Break a Terraform state lock; recover.
- Blow up a Vault seal; unseal from ceremony.
- Renovate opens a PR; merge it; watch ArgoCD roll a new image; observe monitoring.

## Break/fix drill

1. **Drift detection**: manually change an OPNsense firewall rule via UI. Wait for nightly drift-check to alert. Reconcile by updating IaC (not by reverting via UI).
2. **Broken CI**: introduce a syntax error in a role; watch PR fail; fix; watch merge apply.
3. **Vault disaster**: simulate loss of unseal keys — restore from the sealed backup ceremony (must have documented that ceremony).

## Verification commands

```bash
# CI status
gh pr checks <PR>              # or gitea equivalent

# Terraform drift
terraform plan -detailed-exitcode

# Ansible dry run
ansible-playbook site.yml --check --diff

# NetBox exports (source of truth vs. reality)
curl -sH "Authorization: Token $TOKEN" https://netbox01.oikos.home.arpa/api/ipam/ip-addresses/ | jq

# Vault
vault status
vault kv get secret/opnsense/api-key

# Renovate
gh pr list --label renovate

# Nornir
nornir_cli inventory list
nornir_cli run --task napalm_get --getters facts
```

## Security posture at this phase

- **Every secret in Vault or Sealed-Secrets**; grep the whole repo for anything else, remediate
- **Pre-commit gitleaks** — no plaintext credentials ever committed
- **CI runs in dedicated runners** on isolated VLAN 20 with restricted egress
- **Vault**: audit device to Loki; alert on any root-token use
- **Signed commits** on the homelab repo
- **ADR** for every automation that has blast radius (mass config pushes)

## Monitoring coverage

- CI: build success rate, mean build time, flaky-test rate
- Terraform: drift count per resource
- Renovate: open PR count, oldest PR age
- Vault: seal status, unseal events, error rate
- NetBox: data staleness (last change > 7 days on hot resources = investigate)

## Kephaleos docs to create / update

- **Create**: `runbooks/netbox-baseline.md`, `runbooks/vault-baseline.md`, `runbooks/vault-unseal-ceremony.md` (**critical — treat as break-glass**)
- **Create**: `runbooks/ci-cd-baseline.md`
- **Create**: `runbooks/renovate-baseline.md`
- **Create**: `runbooks/drift-detection.md`
- **Create**: `runbooks/network-automation-nornir.md`
- **Update**: existing runbooks — add "Automated version:" section pointing at the playbook
- **New ADRs**: `adr/000N-source-of-truth-netbox.md`, `adr/000N-secrets-vault-vs-sealed-secrets.md`, `adr/000N-ci-cd-platform.md`, `adr/000N-network-automation-approach.md`
- **Append**: `CHANGELOG.md`

## Diagram updates

- Automation topology diagram (new): NetBox → git → CI → target systems
- Service dependency: mark automation-managed resources
- Update master diagram: NetBox, Vault, CI/CD nodes

## Completion checklist

- [ ] NetBox is the source of truth for IPAM/DCIM (imported + maintained)
- [ ] Vault deployed, unseal ceremony documented and rehearsed
- [ ] Every service touched in Phases 0–10 has an Ansible role
- [ ] Terraform/OpenTofu modules for VM, DNS, firewall exist and are used
- [ ] CI/CD pipeline: PR → lint → dry-run → apply on merge
- [ ] Renovate opening PRs against the repo weekly
- [ ] Pre-commit hooks enforced
- [ ] Nightly drift detection reports to ntfy/Slack
- [ ] At least one full-loop change made via automation only (no manual UI touches)
- [ ] Signed commits enforced
- [ ] Docs, ADRs, diagrams updated

**This phase never really "ends"**. Set a completion bar (all checklist items) and then adopt an ongoing hygiene cadence: monthly hardening review, quarterly ADR review.

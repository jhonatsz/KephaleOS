---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-9, kubernetes, k8s, gitops, argocd]
confidence: high
---

# Phase 9 — Kubernetes

## Objective

Deploy a **multi-node Kubernetes cluster** on Proxmox VMs. Persistent storage via NFS from `nas01`. Ingress, load balancing, monitoring, GitOps. Not "toy k8s" — realistic enough that platform engineering practice transfers.

## Why now

You have: virtualization (cluster), shared storage, identity, DNS, monitoring. K8s can now consume all of them properly. Doing it before those existed meant faking half the stack.

## Prerequisites

- Phases 0–8 complete
- 32 GB free RAM across the cluster (control-plane + workers)
- 200 GB free NFS storage for PVCs

## Hardware

| Item | Required? | Buy timing | Notes |
|---|---|---|---|
| — | — | none | All VMs on existing Proxmox cluster |
| Additional RAM for either host | Conditional | BUY as needed | If OOM-pressure appears during workload testing |

## Software

Two credible choices — record the pick in an ADR:

**Option A — k3s (light, fast, real)**
- Pros: single binary, easy install, still upstream-compatible, great for homelab
- Cons: fewer knobs to fiddle with (arguably a pro)
- Recommended for Oikos v1

**Option B — kubeadm + Talos Linux (production-realistic)**
- Pros: exactly what you'd operate at scale
- Cons: more moving parts, more to break

**Ecosystem (either option)**:
- **MetalLB** — load balancer (or Cilium LB if using Cilium CNI)
- **Ingress-NGINX** or **Traefik** — ingress
- **cert-manager** — TLS via Let's Encrypt (public wildcard for `*.oikos.home.arpa` via DNS-01) or internal CA
- **Longhorn** or **NFS CSI driver** — persistent volumes
- **ArgoCD** — GitOps
- **Sealed-Secrets** or **External Secrets Operator + Vault** — secrets
- **kube-prometheus-stack** — reuses `obs01`, or in-cluster scrape

## Network changes

- New VMs on VLAN 30 (k8s), `172.27.30.0/24`:
  - `k8s-cp-01`, `k8s-cp-02`, `k8s-cp-03` — control plane (3 for HA)
  - `k8s-worker-01`, `k8s-worker-02` — initial workers (grows over time)
- MetalLB address pool: `172.27.30.240-.250` (10 IPs for LoadBalancer services)
- Ingress virtual IP: `172.27.30.240`
- Firewall:
  - VLAN 30 → nas01 (NFS): allow
  - home → k8s ingress VIP: allow HTTPS
  - k8s → Internet: allow (registry pulls, etc.)
  - k8s → mgmt/servers/storage: allow narrow ports as needed

## Implementation sequence

1. **Design decision** — k3s vs kubeadm (ADR).
2. **Provision VMs** — Terraform module against Proxmox provider; 3 CP nodes (2 vCPU, 4 GB) + 2 worker nodes (4 vCPU, 8 GB); cloud-init for base config.
3. **Install cluster**:
   - k3s: `curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init" sh -` on cp-01; join others
   - Or Ansible role via `kubernetes/kubespray` for kubeadm route
4. **Verify** — `kubectl get nodes`; all Ready.
5. **CNI check** — if k3s, Flannel default; switch to Cilium if you want BGP/eBPF experiments.
6. **NFS CSI driver** — install; create StorageClass `nas-nfs`; test PVC.
7. **MetalLB** — install; L2 mode; announce pool.
8. **Ingress-NGINX** — install as DaemonSet or Deployment; Service type LoadBalancer.
9. **cert-manager**:
   - Cluster issuer for Let's Encrypt (DNS-01 via a public domain you own — this is a pain-point; alternative: internal CA for now)
   - Internal CA option: sign with a CA on your workstation, distribute trust
10. **Bootstrap ArgoCD** — install; point at a `homelab-gitops` repo (subfolder of the main homelab repo); ArgoCD manages everything else from git thereafter.
11. **First app** — deploy something small and useful (Homer dashboard, Uptime Kuma, or a portfolio site) via git commit to prove the GitOps loop.
12. **Sealed-Secrets** — install controller; encrypt secrets before committing.
13. **Monitoring integration**:
    - Enable ServiceMonitor CRDs
    - `obs01` scrapes cluster via ServiceMonitor
    - Or install kube-prometheus-stack in-cluster and federate to `obs01`
14. **Backup strategy** — `velero` with NFS backend (or `kasten k10` if learning enterprise); store cluster backups on `nas01` + off-site.

## CCNA topics reinforced

- Overlay networking (VXLAN, WireGuard depending on CNI)
- BGP (if Cilium/Calico in BGP mode) — advanced, revisit in Phase 11 automation
- Service discovery (Kubernetes DNS uses SRV-like records)
- Load balancer types (L2 vs L4 vs L7)
- Ingress vs Egress traffic policy

## Study track lab

EVE-NG: build a BGP peering lab between two "top-of-rack" routers before enabling Cilium BGP against your real network.

## Physical Oikos lab exercise

- `kubectl get nodes -o wide` — describe each node's role, CIDR
- `kubectl get pods -A` — identify system pods
- Deploy a Deployment + Service + Ingress; hit it from your workstation via DNS
- Delete a worker VM; watch pods reschedule; verify SLA
- `kubectl exec` into a pod; probe the CoreDNS resolution chain

## Break/fix drill

1. Kill `k8s-cp-01`. If HA is working, cluster survives; `kubectl` still works via any other CP node.
2. Break the NFS mount on one worker. Observe: pods with PVCs on that worker fail; StorageClass reconciles or admin intervenes.
3. Corrupt ArgoCD state: manually change a resource that ArgoCD owns. Watch it self-heal.
4. Certificate expiry: shorten a cert's TTL, watch cert-manager renew.

## Verification commands

```bash
kubectl get nodes
kubectl get pods -A
kubectl top nodes
kubectl top pods -A
kubectl get ingress -A
kubectl get svc -A -o wide | grep LoadBalancer
kubectl get pvc -A
argocd app list
```

## Security posture at this phase

- RBAC: default deny; explicit role bindings
- Pod Security Standards: `restricted` on user namespaces
- NetworkPolicies: default deny + explicit allow (start with kube-system and dns)
- Sealed-Secrets or Vault — never plaintext secrets in git
- Container images: pinned SHA digests where possible; image scanning (Trivy) as pre-commit
- API server audit log → Loki
- Nodes NOT accessible via SSH from anywhere but mgmt VLAN + a jump host

## Monitoring coverage

- kube-state-metrics (workload health)
- cAdvisor (container resource use)
- API server metrics
- etcd metrics (kubeadm route)
- Certificate expiry (cert-manager exports)
- ArgoCD app sync status

## Kephaleos docs to create / update

- **Create**: `runbooks/k8s-provision.md`
- **Create**: `runbooks/k8s-add-worker.md`
- **Create**: `runbooks/k8s-cert-manager-lets-encrypt.md` (DNS-01)
- **Create**: `runbooks/k8s-argocd-bootstrap.md`
- **Create**: `runbooks/k8s-velero-backup-restore.md`
- **Update**: `current-state.md`, `inventory/nodes.md`, `architecture/address-plan.md` (VLAN 30 active)
- **New ADRs**: `adr/000N-k8s-distribution.md` (k3s vs kubeadm), `adr/000N-k8s-cni.md`, `adr/000N-k8s-tls-strategy.md`
- **Append**: `CHANGELOG.md`

## Diagram updates

- Kubernetes diagram (new): CP/worker layout, CNI, ingress path
- Master diagram: k8s VLAN, MetalLB pool
- Service dependency: add ArgoCD, cert-manager, Longhorn/NFS-CSI

## Completion checklist

- [ ] Distribution choice recorded in ADR
- [ ] Cluster running with HA control plane (3 CP nodes) + workers
- [ ] CNI healthy; pod-to-pod ping works across nodes
- [ ] NFS CSI or Longhorn PVCs work
- [ ] MetalLB assigns LB IPs
- [ ] Ingress-NGINX reachable; TLS via cert-manager
- [ ] ArgoCD deploys at least one app from git
- [ ] Sealed-Secrets in use
- [ ] Monitoring integrated
- [ ] Backup tested (velero restore)
- [ ] Docs, ADRs, diagrams updated

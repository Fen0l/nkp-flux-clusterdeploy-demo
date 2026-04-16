# NKP Flux CD — GitOps Workload Cluster Deployment

Deploy NKP workspaces and workload clusters via Flux CD using ClusterClass
topology, with **zero secrets in Git**.

Cluster configuration is split into two on-cluster resources:
- **ConfigMap** — infrastructure config (IPs, names, images)
- **Secret** — credentials only (passwords, SSH key)

Both are named per-cluster for multi-cluster support.

## Architecture

```
                          NKP Management Cluster
                         +---------------------------------------------------------+
  GitHub / GitLab        |                                                         |
 +-------------------+   |  Phase 1                     Phase 2                    |
 | workspace/         |   |  +-----------+              +-----------+              |
 |   workspace.yaml   |--+->| Workspace |--dependsOn-->|  Cluster  |              |
 |                    |   |  |  (Flux)   |              |  (Flux)   |              |
 | clusters/          |   |  +-----+-----+              +-----+-----+              |
 |   secrets.yaml     |   |        |                          | substituteFrom     |
 |   cluster.yaml     |   |        v                    +-----+------+             |
 |   (${VARS})        |   |  +-----------+              |            |             |
 +-------------------+   |  | Namespace  |         ConfigMap    Secret             |
                         |  | flux-demo- |         ...-config   ...-secrets        |
  namespace-sync Job     |  | github-xxx |              ^                          |
  (one-shot, bootstrap)  |  +-----------+              |                          |
  patches ConfigMap -----+-----------------------------|                          |
  with real namespace    |                              |                          |
                         |                         +----+------+                   |
                         |                         | Cluster + |                   |
                         |                         | Companion |                   |
                         |                         | Secrets   |                   |
                         |                         +-----+-----+                   |
                         |                               | CAPI                    |
                         |                               v                         |
                         |                        +-----------+                    |
                         |                        | Workload  |                    |
                         |                        | Cluster   |                    |
                         |                        +-----------+                    |
                         +---------------------------------------------------------+
```

## Prerequisites

- NKP 2.17 management cluster with ClusterClass `nkp-nutanix-v2.17.1`
- Flux CD controllers running (installed by NKP in `kommander-flux` namespace)
- Air-gapped: `bitnami/kubectl:1.34` image available in registry mirror

## Repository Structure

```
├── README.md
├── workspace/                                     # Phase 1: Workspace CR
│   └── workspace.yaml
├── clusters/                                      # Phase 2: Cluster deployment
│   └── nkp-wrkld-demo-flux-github01/
│       ├── kustomization.yaml
│       ├── secrets.yaml                           # PC, CSI, Harbor creds (${PLACEHOLDERS})
│       └── cluster.yaml                           # CAPI Cluster CR (${PLACEHOLDERS})
└── management/                                    # Bootstrap (manual, one-time)
    ├── git-source.yaml                            # Flux GitRepository CR
    ├── cluster-kustomization.yaml                 # Flux Kustomizations (Phase 1 + 2)
    ├── namespace-sync.yaml                        # Job: patches ConfigMap with namespace
    ├── cluster-config.yaml.template               # Template: ConfigMap (non-sensitive)
    └── cluster-secrets.yaml.template              # Template: Secret (credentials)
```

## Deployment (3 commands)

### Step 1: Create ConfigMap + Secret

```bash
# ── ConfigMap (non-sensitive) ────────────────────────────────────────────────
cp management/cluster-config.yaml.template management/cluster-config.yaml
vim management/cluster-config.yaml    # fill infrastructure values
kubectl apply -f management/cluster-config.yaml && rm management/cluster-config.yaml

# ── Secret (credentials) ────────────────────────────────────────────────────
cp management/cluster-secrets.yaml.template management/cluster-secrets.yaml
vim management/cluster-secrets.yaml   # fill passwords + SSH key
kubectl apply -f management/cluster-secrets.yaml && rm management/cluster-secrets.yaml
```

Leave `WORKSPACE_NAMESPACE` as-is — the namespace-sync Job fills it automatically.

### Step 2: Bootstrap Flux + namespace sync

```bash
kubectl apply -f management/git-source.yaml \
              -f management/cluster-kustomization.yaml \
              -f management/namespace-sync.yaml
```

That's it. What happens next:
1. Flux creates the Workspace CR (Phase 1)
2. The namespace-sync Job patches the ConfigMap with the real namespace
3. Flux deploys the cluster into the workspace namespace (Phase 2)

### Step 3: Watch

```bash
# Flux phases
kubectl get kustomization -n kommander

# Namespace sync Job
kubectl logs job/namespace-sync-nkp-wrkld-demo-flux-github01 -n kommander

# Cluster provisioning
WS_NS=$(kubectl get workspace flux-demo-github -o jsonpath='{.status.namespaceRef.name}')
kubectl get cluster -n ${WS_NS}
kubectl get machines -n ${WS_NS} -w
```

## Day 2 — Update via Git

```bash
# Scale workers, change image, etc.
vim clusters/nkp-wrkld-demo-flux-github01/cluster.yaml
git add -A && git commit -m "Scale workers to 4" && git push

# Force immediate reconciliation
kubectl annotate gitrepository flux-clusterdeploy-demo -n kommander \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Variable Reference

### ConfigMap: `<cluster-name>-config` (non-sensitive)

| Variable | Description | Example |
|----------|-------------|---------|
| `WORKSPACE_NAMESPACE` | **Auto-populated** by namespace-sync Job | `flux-demo-github-a1b2c` |
| `PC_ENDPOINT` | Prism Central IP | `10.12.54.8` |
| `PE_CLUSTER` | Prism Element cluster name | `PIKATCHU` |
| `SUBNET` | AHV subnet name | `pika-vlan52-workloads` |
| `STORAGE_CONTAINER` | Nutanix storage container | `nkp` |
| `VM_IMAGE` | Node OS image name | `nkp-rocky-9.7-release-cis-...qcow2` |
| `CONTROL_PLANE_VIP` | Control plane virtual IP | `10.12.52.227` |
| `LB_IP_START` | MetalLB pool start IP | `10.12.52.228` |
| `LB_IP_END` | MetalLB pool end IP | `10.12.52.229` |
| `REGISTRY_MIRROR_URL` | Air-gapped registry URL | `https://10.12.52.11/nkp` |

### Secret: `<cluster-name>-secrets` (credentials)

| Variable | Description |
|----------|-------------|
| `PC_USER` | Prism Central username |
| `PC_PASSWORD` | Prism Central password |
| `SSH_PUBLIC_KEY` | SSH public key for node access |
| `HARBOR_USER` | Registry mirror username |
| `HARBOR_PASSWORD` | Registry mirror password |

## Private Repository Setup

```bash
# GitHub PAT
kubectl create secret generic github-repo-auth \
  --namespace=kommander \
  --from-literal=username=git \
  --from-literal=password=ghp_your_token_here

# GitLab Deploy Token
kubectl create secret generic gitlab-deploy-token \
  --namespace=kommander \
  --from-literal=username=gitlab+deploy-token-12345 \
  --from-literal=password=your-deploy-token-value
```

Then uncomment the matching `secretRef` in `management/git-source.yaml`.

## Adding More Clusters

1. Copy `clusters/nkp-wrkld-demo-flux-github01` → new name
2. Find/replace cluster name in all manifests
3. Create new ConfigMap + Secret from templates
4. Add a new Job block in `management/namespace-sync.yaml`
5. Add a new Phase 2 Kustomization in `management/cluster-kustomization.yaml`
6. Push to Git

## Troubleshooting

```bash
# Flux status
kubectl get kustomization -n kommander

# Namespace sync
kubectl get job namespace-sync-nkp-wrkld-demo-flux-github01 -n kommander
kubectl logs job/namespace-sync-nkp-wrkld-demo-flux-github01 -n kommander

# Verify ConfigMap was patched
kubectl get cm nkp-wrkld-demo-flux-github01-config -n kommander \
  -o jsonpath='{.data.WORKSPACE_NAMESPACE}'

# Flux logs
kubectl logs -n kommander-flux -l app=kustomize-controller --tail=50

# Force reconciliation
kubectl annotate gitrepository flux-clusterdeploy-demo -n kommander \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Security Model

| Concern | How it's handled |
|---------|------------------|
| Secrets in Git | Never — all sensitive values are `${PLACEHOLDERS}` |
| Credential storage | K8s Secret in `kommander` namespace |
| Git authentication | HTTPS + PAT/Deploy Token (if private) |
| Cluster deletion | `prune: false` — Flux never auto-deletes |
| Secret rotation | Update K8s Secret, Flux re-applies on next interval |
| Namespace-sync Job | Minimal RBAC: read workspace, patch configmap |

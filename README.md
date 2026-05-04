# NKP Flux CD — GitOps Workload Cluster Deployment + NDK Protection

End-to-end GitOps demo:
1. **Phase 1–2** (mgmt cluster) — workspace + CAPI workload cluster lifecycle
2. **Phase 3** (workload cluster) — stateful demo app
3. **Phase 4** (workload cluster) — NDK protection (Application + ProtectionPlan)

All managed via Flux CD with **zero secrets in Git**. Two Flux instances reconcile:

| Flux instance | Reconciles | Path in repo |
|---------------|------------|--------------|
| `kommander` ns on mgmt cluster | Workspace + Cluster | `workspace/`, `clusters/<name>/`, `clusters/<name>/cluster/` |
| `kommander-flux` ns on workload cluster | Demo app + NDK CRs | `clusters/<name>/app/`, `clusters/<name>/ndk/` |

Cluster configuration is split into two on-cluster resources:
- **ConfigMap** — infrastructure config (IPs, names, images, retention, intervals)
- **Secret** — credentials only (passwords, SSH key)

Both are named per-cluster and stored in a dedicated NKP Project namespace
to keep the `kommander` namespace clean.

For the on-stage walkthrough see [`DEMO-RUNBOOK.md`](DEMO-RUNBOOK.md).

## Architecture

```
  GitHub / GitLab                NKP Management Cluster                          Workload Cluster
 +-------------------+          +-------------------------------------------+   +-----------------------+
 | workspace/         |          | Project ns (flux-clusters-lh5x6)          |   | kommander-flux ns     |
 |   workspace.yaml   |---+      |  +---------+  +---------+  +---------+    |   |                       |
 | clusters/          |   |      |  | Phase 1 |->| Phase 2a|->| Phase 2b|    |   |  +-----------------+  |
 |   secrets.yaml     |   +----->|  |Workspace|  |Secrets  |  |Cluster  |    |   |  | GitRepository   |  |
 |   cluster/         |          |  +----+----+  +----+----+  +----+----+    |   |  +-------+---------+  |
 |     cluster.yaml   |          |       |  ConfigMap +substituteFrom        |   |          |            |
 |   app/             |---+      |       |  +Secret                          |   |          v            |
 |     stateful-demo  |   |      |       |       |             |             |   |  +---------+  +----+  |
 |   ndk/             |   |      |       v       v             v             |   |  | Phase 3 |  |    |  |
 |     application    |   |      |  +-----------+  +-----------+  +--------+ |   |  | App     |->|Phase|  |
 |     plan, appPP    |   |      |  |Workspace  |  |Companion  |  |Cluster | |   |  +----+----+  | 4   |  |
 +-------------------+   |      |  |Namespace  |<-|Secrets    |  |CR      | |   |       |        |NDK |  |
                          |      |  +-----------+  +-----------+  +---+----+ |   |       v        +-+--+  |
                          |      |                                    | CAPI|   |  +----------+    |     |
                          |      +------------------------------------|----+   |   | demo-app  |<---+     |
                          |                                            v        |   | Deploy+PVC|         |
                          |                                       +---------+   |   +-----------+         |
                          +-------------------------------------->| Workload|   |   | Application,        |
                                                                  | Cluster |---+>  | ProtectionPlan,     |
                                                                  +---------+       | AppProtectionPlan,  |
                                                                                    | ApplicationSnapshot |
                                                                                    +---------------------+
```

### Why 3 phases?

1. **Phase 1** — Flux creates the Workspace CR. Kommander generates a namespace
   with a random suffix (e.g., `flux-demo-github-a1b2c`).
2. **Phase 2a** — Flux creates the companion Secrets (PC credentials, CSI, Harbor)
   in the workspace namespace. These **must exist before** the Cluster CR because
   the CAREN preflight webhook validates Prism Central connectivity during dry-run.
3. **Phase 2b** — Flux creates the Cluster CR. CAPI provisions VMs on Nutanix AHV.

A **namespace-sync Job** (applied once during bootstrap) bridges the gap between
workspace creation and cluster deployment by patching the ConfigMap with the
real workspace namespace.

## Prerequisites

- NKP 2.17 management cluster with ClusterClass `nkp-nutanix-v2.17.1`
- Flux CD controllers running (installed by NKP in `kommander-flux`)
- An NKP Project on the management cluster (dedicated namespace for Flux resources)

## Repository Structure

```
├── workspace/                                     # Phase 1: Workspace CR
│   └── workspace.yaml
├── clusters/                                      # Phase 2a + 2b (mgmt) + Phase 3 + 4 (workload)
│   └── nkp-wrkld-demo-flux-github01/
│       ├── kustomization.yaml                     # Phase 2a: companion secrets
│       ├── secrets.yaml                           # PC, CSI, Harbor creds (${VARS})
│       ├── cluster/
│       │   ├── kustomization.yaml                 # Phase 2b: cluster only
│       │   └── cluster.yaml                       # CAPI Cluster CR (${VARS})
│       ├── app/                                   # Phase 3: stateful demo workload
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml                     # ${APP_NAMESPACE}
│       │   └── stateful-demo.yaml                 # Deployment + PVC + Service
│       └── ndk/                                   # Phase 4: NDK protection
│           ├── kustomization.yaml
│           ├── application.yaml                   # Application CR (label selector)
│           ├── job-scheduler.yaml                 # JobScheduler (cadence)
│           ├── protection-plan.yaml               # ProtectionPlan (LOCAL)
│           ├── app-protection-plan.yaml           # AppProtectionPlan (binds App ↔ Plan)
│           └── extras/                            # Activated mid-demo (NOT in kustomization)
│               ├── manual-snapshot.yaml           # ApplicationSnapshot (pre-disaster)
│               └── restore.yaml                   # ApplicationSnapshotRestore
└── management/                                    # Bootstrap (manual, one-time)
    ├── git-source.yaml                            # Flux GitRepository (mgmt-side)
    ├── cluster-kustomization.yaml                 # Flux Kustomizations (Phases 1, 2a, 2b)
    ├── namespace-sync.yaml                        # SA + RBAC + Job
    ├── cluster-config.yaml.template               # ConfigMap (non-sensitive, mgmt-side)
    ├── cluster-secrets.yaml.template              # Secret (creds, mgmt-side) — NOT tracked
    ├── workload-flux-bootstrap.yaml               # Workload-side Flux: GitRepo + 2 Kustomizations
    ├── workload-cluster-config.yaml.template      # ConfigMap (non-sensitive, workload-side)
    └── storage-cluster.yaml.template              # NDK StorageCluster (one-time, env-specific)
```

## Deployment

### Step 1: Create ConfigMap + Secret in the project namespace

```bash
# ConfigMap (non-sensitive infrastructure config)
# Copy the template, fill in values, apply, delete the copy.
# Leave WORKSPACE_NAMESPACE as-is — the namespace-sync Job fills it.
kubectl apply -f management/cluster-config.yaml.template

# Secret (credentials — template NOT in git, create manually)
kubectl create secret generic nkp-wrkld-demo-flux-github01-secrets \
  -n <PROJECT_NAMESPACE> \
  --from-literal=PC_USER='admin' \
  --from-literal=PC_PASSWORD='<your-pc-password>' \
  --from-literal=SSH_PUBLIC_KEY="$(cat ~/.ssh/id_rsa.pub)" \
  --from-literal=HARBOR_USER='admin' \
  --from-literal=HARBOR_PASSWORD='<your-harbor-password>'
```

### Step 2: Bootstrap Flux + namespace-sync

```bash
kubectl apply -f management/git-source.yaml \
              -f management/cluster-kustomization.yaml \
              -f management/namespace-sync.yaml
```

That's it. What happens automatically:
1. Flux creates the Workspace (Phase 1)
2. The namespace-sync Job patches the ConfigMap with the real namespace
3. Flux creates companion Secrets in the workspace namespace (Phase 2a)
4. Flux creates the Cluster CR (Phase 2b) — CAPI provisions VMs

### Step 3: Watch the cluster come up

```bash
# Flux phases
kubectl get kustomization -n <PROJECT_NAMESPACE>

# Namespace sync Job
kubectl logs job/namespace-sync-nkp-wrkld-demo-flux-github01 -n <PROJECT_NAMESPACE>

# Verify ConfigMap was patched
kubectl get cm nkp-wrkld-demo-flux-github01-config -n <PROJECT_NAMESPACE> \
  -o jsonpath='{.data.WORKSPACE_NAMESPACE}'

# Cluster provisioning
WS_NS=$(kubectl get workspace flux-demo-github -o jsonpath='{.status.namespaceRef.name}')
kubectl get cluster -n $WS_NS
kubectl get machines -n $WS_NS -w
```

### Step 4: Bootstrap workload-side Flux (Phase 3 + 4)

Once the workload cluster is `Ready` and you have its kubeconfig:

```bash
# Get the workload cluster kubeconfig
nkp get kubeconfig -c nkp-wrkld-demo-flux-github01 -n $WS_NS \
  > nkp-wrkld-demo-flux-github01.conf
export KUBECONFIG=nkp-wrkld-demo-flux-github01.conf

# 4a. Install NDK 2.1 on the workload cluster (Helm — one-time, see NDK guide)
#     This is a controller install with CRDs; not GitOps-managed.
helm upgrade --install ndk -n ntnx-system ntnx-charts/ndk \
  --version 2.1.0 \
  --set imageCredentials.credentials.username=<docker-user> \
  --set imageCredentials.credentials.password=<docker-token> \
  --set tls.server.clusterName=nkp-wrkld-demo-flux-github01 \
  --set config.secret.name=nutanix-csi-credentials

# 4b. Register the StorageCluster (env-specific, NOT GitOps)
cp management/storage-cluster.yaml.template /tmp/storage-cluster.yaml
# Fill in <PE_UUID> and <PC_UUID> — see comments in the template
kubectl apply -f /tmp/storage-cluster.yaml
rm /tmp/storage-cluster.yaml

# 4c. Apply the workload-side ConfigMap
cp management/workload-cluster-config.yaml.template /tmp/workload-cluster-config.yaml
# Adjust APP_NAMESPACE / SCHEDULE_INTERVAL_MINUTES / RETENTION_COUNT if needed
kubectl apply -f /tmp/workload-cluster-config.yaml

# 4d. Bootstrap workload-side Flux
kubectl apply -f management/workload-flux-bootstrap.yaml
```

Verify the workload-side Flux picked up the repo:

```bash
flux get sources git -n kommander-flux
flux get kustomizations -n kommander-flux

# Watch the demo app come up
kubectl get pvc,deploy,svc -n demo-app

# Watch NDK CRs reconcile and the first ApplicationSnapshot fire
kubectl get application,jobscheduler,protectionplan,appprotectionplan -n demo-app
kubectl get appsnap -n demo-app -w
```

## Day 2 — Update via Git

```bash
# Scale workers, change image, adjust LB range, etc.
vim clusters/nkp-wrkld-demo-flux-github01/cluster/cluster.yaml
git add -A && git commit -m "Scale workers to 4" && git push

# Force immediate reconciliation
flux reconcile source git flux-clusterdeploy-demo -n <PROJECT_NAMESPACE>
```

## Variable Reference

### ConfigMap: `<cluster-name>-config` (non-sensitive)

| Variable | Description | Example |
|----------|-------------|---------|
| `WORKSPACE_NAMESPACE` | **Auto-populated** by namespace-sync Job | `flux-demo-github-a1b2c` |
| `PC_ENDPOINT` | Prism Central IP | `10.12.54.8` |
| `PE_CLUSTER` | Prism Element cluster name | `PIKACHU` |
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

### ConfigMap: `workload-cluster-config` (workload cluster, `kommander-flux` ns)

| Variable | Default | Notes |
|----------|---------|-------|
| `APP_NAMESPACE` | `demo-app` | Namespace for the stateful demo workload |
| `PVC_SIZE` | `1Gi` | PVC size for `/data` |
| `SCHEDULE_INTERVAL_MINUTES` | `60` | JobScheduler floor is 60 min — controller rejects below |
| `RETENTION_COUNT` | `5` | ProtectionPlan retention; valid range 1–15 |

## Key Design Decisions

| Decision | Reason |
|----------|--------|
| Dedicated project namespace | Keep `kommander` ns clean; RBAC isolation |
| `serviceAccountName` on Kustomizations | Required by NKP Gatekeeper policy |
| Secrets before Cluster CR (2a → 2b) | CAREN preflight webhook validates PC credentials during dry-run |
| namespace-sync Job | Kommander appends random suffix to workspace namespace — Job patches ConfigMap automatically |
| `prune: false` on all Kustomizations | Safety — Flux never auto-deletes workspaces or clusters |
| `wait: false` on Phase 2b | Cluster provisioning takes 15-30 min — don't block Flux |

## Private Repository Setup

```bash
# GitHub PAT
kubectl create secret generic github-repo-auth \
  -n <PROJECT_NAMESPACE> \
  --from-literal=username=git \
  --from-literal=password=ghp_your_token_here

# GitLab Deploy Token
kubectl create secret generic gitlab-deploy-token \
  -n <PROJECT_NAMESPACE> \
  --from-literal=username=gitlab+deploy-token-12345 \
  --from-literal=password=your-deploy-token-value
```

Then uncomment the matching `secretRef` in `management/git-source.yaml`.

## NDK Protection Demo (Phase 4)

Once Phase 3 + 4 are healthy, the demo gestures are all `git push`-driven:

| Step | Gesture | Outcome |
|------|---------|---------|
| **Drift check** | `kubectl delete protectionplan stateful-demo-local -n demo-app` | Flux reconciles within 10 min (or `flux reconcile kustomization stateful-demo-ndk -n kommander-flux` to force) — Plan reappears |
| **Manual snapshot** | Add `extras/manual-snapshot.yaml` to `ndk/kustomization.yaml`, commit, push, `flux reconcile` | New `ApplicationSnapshot/pre-disaster` reaches `readyToUse: true` |
| **Disaster** | `kubectl scale deploy/stateful-demo --replicas=0`, `kubectl delete pvc stateful-demo-data` | Pod and data are gone |
| **Restore** | Add `extras/restore.yaml` to `ndk/kustomization.yaml`, commit, push, `flux reconcile` | `ApplicationSnapshotRestore` runs → Pod and PVC come back; `wc -l /data/log.txt` matches the snapshot moment |

Reset the demo:

```bash
# Remove the extras lines from ndk/kustomization.yaml, push
# Then on the workload cluster:
kubectl delete appsnaprestore pre-disaster-restore -n demo-app --ignore-not-found
kubectl delete appsnap pre-disaster                -n demo-app --ignore-not-found
```

## Adding More Clusters

1. Copy `clusters/nkp-wrkld-demo-flux-github01/` → new cluster name
2. Find/replace cluster name in all manifests
3. Create new ConfigMap + Secret from templates
4. Add a new namespace-sync Job block in `management/namespace-sync.yaml`
5. Add new Phase 2a + 2b Kustomizations in `management/cluster-kustomization.yaml`
6. Push to Git

## Troubleshooting

```bash
# All Flux Kustomizations
kubectl get kustomization -n <PROJECT_NAMESPACE>

# Phase 1: Workspace
kubectl get workspace flux-demo-github

# Phase 2a: Secrets
kubectl get kustomization nkp-wrkld-demo-flux-github01-secrets -n <PROJECT_NAMESPACE> -o yaml

# Phase 2b: Cluster
kubectl get kustomization nkp-wrkld-demo-flux-github01 -n <PROJECT_NAMESPACE> -o yaml

# CAREN preflight failures (check controller logs)
kubectl logs -n ntnx-system -l app.kubernetes.io/name=caren --tail=50

# Namespace sync Job
kubectl logs job/namespace-sync-nkp-wrkld-demo-flux-github01 -n <PROJECT_NAMESPACE>

# Flux controller logs
kubectl logs -n kommander-flux -l app=kustomize-controller --tail=50

# Force reconciliation
flux reconcile source git flux-clusterdeploy-demo -n <PROJECT_NAMESPACE>
```

## Security Model

| Concern | How it's handled |
|---------|------------------|
| Secrets in Git | Never — all sensitive values are `${PLACEHOLDERS}` |
| Secret template | `.gitignore`'d — never tracked |
| Credential storage | K8s Secret in project namespace |
| Git authentication | HTTPS + PAT/Deploy Token (if private) |
| Flux RBAC | Dedicated `flux-deployer` SA with scoped ClusterRole |
| Cluster deletion | `prune: false` — Flux never auto-deletes |
| Secret rotation | Update K8s Secret, Flux re-applies on next interval |
| Namespace isolation | All Flux resources in dedicated project namespace |

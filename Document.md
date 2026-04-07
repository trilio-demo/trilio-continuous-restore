# Trilio Continuous Restore — Red Hat Validated Pattern

## Overview

This Validated Pattern delivers an automated, GitOps-driven Disaster Recovery (DR) solution for stateful applications running on Red Hat OpenShift. It integrates [Trilio for Kubernetes](https://trilio.io) with the [Red Hat Validated Patterns framework](https://validatedpatterns.io) to provide:

- **Automated backup** of stateful workloads on the primary (hub) cluster
- **Continuous Restore** — Trilio's accelerated Recovery Time Objective (RTO) DR path that continuously pre-stages backup data on the DR cluster so that recovery requires only metadata retrieval, not a full data transfer
- **Automated DR testing** — the full backup-to-restore lifecycle runs as a scheduled, self-healing GitOps workflow with no human intervention after initial setup
- **Multi-cluster lifecycle management** via Red Hat Advanced Cluster Management (ACM)

### Use Case

The pattern targets organisations that need a documented, repeatable DR posture for Kubernetes-native workloads — particularly those that must demonstrate RTO/Recovery Point Objective (RPO) targets through regular, automated DR tests rather than annual manual exercises.

A WordPress + MySQL deployment is included as a representative stateful application. It serves as the reference workload for the full backup, restore, and URL-rewrite lifecycle.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Hub Cluster (primary)                                  │
│                                                         │
│  ACM ──► ArgoCD ──► Trilio Operator (OLM)               │
│                       └─► TrilioVaultManager            │
│                       └─► BackupTarget (S3)             │
│                       └─► BackupPlan (wordpress)        │
│                       └─► CR BackupPlan (Continuous)    │
│  Vault + ESO ──► S3 credentials + License Secret        │
│  Imperative CronJob ──► automated DR lifecycle          │
└────────────────────┬────────────────────────────────────┘
                     │ shared S3 BackupTarget
┌────────────────────▼────────────────────────────────────┐
│  DR Cluster / Spoke (secondary)                         │
│                                                         │
│  ACM-managed ──► ArgoCD ──► Trilio Operator (OLM)       │
│                               └─► TrilioVaultManager   │
│                               └─► BackupTarget (S3)    │
│                                   (EventTarget enabled) │
│  EventTarget pod ──► polls S3 ──► pre-stages PVCs       │
│  ConsistentSet ──► Imperative restore ──► wordpress-restore │
└─────────────────────────────────────────────────────────┘
```

> An architecture diagram is available in `docs/architecture.drawio` (to be added).

### Component Roles

| Component | Where | Role |
|-----------|-------|------|
| Trilio Operator | Hub + Spoke | Installed via Operator Lifecycle Manager (OLM) from the `certified-operators` catalog, channel `5.3.x` |
| TrilioVaultManager | Hub + Spoke | Trilio operand Custom Resource (CR); manages the Trilio data plane |
| Red Hat OpenShift | Hub + Spoke | Container orchestration platform; provides OLM, storage, networking, and the GitOps operator substrate |
| Red Hat OpenShift GitOps (ArgoCD) | Hub + Spoke | GitOps sync engine; all configuration is driven from Git |
| Red Hat Advanced Cluster Management (ACM) | Hub | Cluster lifecycle, policy enforcement, and spoke provisioning |
| Validated Patterns Imperative CronJob | Hub + Spoke | Runs the automated DR lifecycle on a 10-minute schedule |
| BackupTarget | Hub + Spoke | Points to the shared S3 bucket; the spoke BackupTarget has the EventTarget flag set |
| BackupPlan | Hub | Defines backup scope (wordpress namespace), quiesce/unquiesce hooks, and retention |
| CR BackupPlan | Hub | Continuous Restore variant of BackupPlan; drives pre-staging on the spoke |
| EventTarget pod | Spoke | Watches the shared S3 bucket for new backups; pre-stages Persistent Volume Claims (PVCs) locally |
| ConsistentSet | Spoke | Cluster-scoped CR representing a fully pre-staged restore point |
| HashiCorp Vault and External Secrets Operator (ESO) | Hub | Secret management; S3 credentials and Trilio license are never stored in Git |

### How Continuous Restore Works

1. The hub creates a backup using the CR BackupPlan and writes it to the shared S3 storage.
2. The EventTarget pod on the spoke detects the new backup and begins copying volume data locally — ahead of any DR event.
3. When the spoke's imperative job detects an Available ConsistentSet, it submits a Restore CR. Because the data is already local, only backup metadata is fetched — resulting in significantly lower RTO than a standard on-demand restore.
4. The post-restore Hook CR rewrites WordPress database URLs to the DR cluster's ingress domain.

---

## Prerequisites

### Clusters

| Cluster | Role | Minimum size |
|---------|------|-------------|
| Hub | Primary; runs ACM, HashiCorp Vault, ArgoCD, Trilio | 3 worker nodes, 8 vCPU / 32 GB each |
| DR Spoke | Disaster Recovery target | 3 worker nodes, 8 vCPU / 32 GB each |

Both clusters must:
- Run Red Hat OpenShift 4.18 or later
- Have network access to the shared S3 bucket
- Be reachable by ACM on the hub

### S3 Storage

A single S3-compatible bucket accessible from both clusters. Required values:
- Bucket name
- Bucket region (must match the bucket's actual region — always set this explicitly)
- Access key and secret key with read/write permissions on the bucket

### Trilio License

A valid Trilio for Kubernetes license key. This pattern supports Trilio for Kubernetes version 5.3.0 and later. Obtain a license from [trilio.io](https://trilio.io) or your Trilio representative.

### Tooling (hub workstation)

- `oc` CLI logged in to the hub cluster with cluster-admin
- `ansible-navigator` (for manual DR operations)
- `make`
- `git`
- `python3`
- `rhvp.cluster_utils` Ansible collection (for `make install`):

  ```bash
  ansible-galaxy collection install community.okd kubernetes.core \
    https://github.com/validatedpatterns/rhvp.cluster_utils/releases/download/v0.0.6/rhvp-cluster_utils-0.0.6.tar.gz
  ```

---

## Deployment

### 1. Clone the repository

```bash
git clone https://github.com/trilio-demo/trilio-continuous-restore
cd trilio-continuous-restore
```

### 2. Configure S3 bucket details

Edit `values-hub.yaml` and `values-secondary.yaml` to set your S3 bucket name and region:

```yaml
# In both values-hub.yaml and values-secondary.yaml, under the trilio-operand app overrides:
overrides:
  - name: backupTarget.bucketName
    value: <your-bucket-name>
  - name: backupTarget.region
    value: <your-bucket-region>   # e.g. us-east-1
```

### 3. Populate secrets

Create `values-secret.yaml` from the template:

```bash
cp values-secret.yaml.template values-secret.yaml
```

Edit `values-secret.yaml` and fill in your credentials:

```yaml
secrets:
  - name: trilio-license
    vaultPrefixes:
    - global
    fields:
    - name: key
      value: <your-trilio-license-key>   # single unbroken line, no escape characters

  - name: trilio-s3
    vaultPrefixes:
    - global
    fields:
    - name: accessKey
      value: <your-s3-access-key>
    - name: secretKey
      value: <your-s3-secret-key>
```

> `values-secret.yaml` is listed in `.gitignore` and must never be committed to Git.

### 4. Install the pattern

```bash
make install
```

This command:
1. Bootstraps HashiCorp Vault and loads secrets from `values-secret.yaml`
2. Installs the Validated Patterns operator on the hub
3. Creates the `ValidatedPattern` CR which triggers ArgoCD to deploy all hub components

Monitor progress in the ArgoCD UI or via:

```bash
oc get application -n openshift-gitops
```

All applications should reach `Synced / Healthy` within 10–15 minutes.

**Alternative: manual secret population via `oc`**

To write or rotate secrets directly in HashiCorp Vault without re-running `make install`:

```bash
# Extract Vault root token
VAULT_TOKEN=$(oc get secret vaultkeys -n imperative \
  -o jsonpath='{.data.vault_data_json}' | \
  base64 -d | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")

# Write Trilio license
oc exec -n vault vault-0 -- env VAULT_TOKEN=$VAULT_TOKEN \
  vault kv put secret/global/trilio-license key="<your-license-key>"

# Write S3 credentials
oc exec -n vault vault-0 -- env VAULT_TOKEN=$VAULT_TOKEN \
  vault kv put secret/global/trilio-s3 accessKey="<key>" secretKey="<secret>"
```

### 5. Verify hub deployment

Check that Trilio is healthy:

```bash
oc get triliovaultmanager -n trilio-system
# STATUS should be Deployed or Updated

oc get target -n trilio-system
# STATUS should be Available
```

Check the end-to-end DR status (updated automatically by the imperative framework):

```bash
make dr-status
```

Initial run: `trilio-enable-cr` and `trilio-backup` will complete within the first two CronJob cycles (~20 minutes). Standard restore follows. All phases `PASS` indicates the hub is fully operational.

---

## Spoke (DR Cluster) Onboarding

### 1. Import the DR cluster into ACM

Import the DR cluster via the ACM console or `oc` CLI. Note the cluster name assigned during import.

### 2. Label and onboard

```bash
make onboard-spoke CLUSTER=<acm-cluster-name>
```

This labels the cluster with `clusterGroup=secondary`, which triggers ACM to deploy the spoke configuration via ArgoCD.

After running `make onboard-spoke`, kick the spoke-side ArgoCD application to sync immediately (run on the spoke cluster context):

```bash
oc patch application.argoproj.io main-trilio-continuous-restore-secondary \
  -n openshift-gitops --type merge \
  -p '{"operation":{"sync":{}}}'
```

### 3. Monitor spoke onboarding

```bash
make spoke-status CLUSTER=<acm-cluster-name>
```

Expected progression:
1. Trilio operator installs (OLM subscription)
2. TrilioVaultManager deploys (ESO delivers S3 + license secrets)
3. BackupTarget becomes Available (EventTarget pod starts)
4. ConsistentSets begin appearing as hub backups are detected (~10–20 minutes after the hub's CR backup completes)
5. Spoke imperative restore runs automatically once the first ConsistentSet is Available

The full spoke onboarding sequence typically takes 15–25 minutes from label application to a running TrilioVaultManager. The imperative restore adds another 30–45 minutes on top of that for the first ConsistentSet to appear and the restore to complete.

### Known: trilio-operand OutOfSync on spoke after onboarding

ArgoCD may show `trilio-operand` as `OutOfSync / Missing` immediately after spoke onboarding. This is a CRD timing issue — ArgoCD attempts to sync the TrilioVaultManager CR before the Trilio operator has finished registering its Custom Resource Definitions (CRDs).

The `SkipDryRunOnMissingResource=true` sync option is set in `values-secondary.yaml` to handle this automatically. If the issue persists after 5–10 minutes, manually refresh the ArgoCD application:

```bash
oc patch application trilio-operand -n main-trilio-continuous-restore-secondary \
  --type merge -p '{"operation":{"sync":{}}}'
```

---

## Operations

### Monitoring DR status

```bash
# Hub — all phases
make dr-status

# Spoke — ConsistentSet and restore status (run on spoke context)
oc get configmap trilio-cr-status -n imperative -o yaml
```

### Automated DR lifecycle

The imperative framework runs continuously on a 10-minute schedule with no manual intervention required. The full lifecycle from a standing start (hub up, spoke just joined) to a completed Continuous Restore typically completes within 30–45 minutes.

**Hub job sequence:**

| Job | What it does | Skips when |
|-----|-------------|------------|
| `trilio-enable-cr` | Creates CR BackupPlan + ContinuousRestore Policy | CR BackupPlan already Available |
| `trilio-cr-backup` | Creates a backup against the CR BackupPlan | Available CR backup exists |
| `trilio-backup` | Creates a standard backup | Available backup exists |
| `trilio-restore-standard` | Restores to `wordpress-restore` on hub | Completed restore exists |
| `trilio-e2e-status` | Writes status ConfigMap; fails until all phases pass | — (always runs) |

**Spoke job sequence (per DR cluster):**

| Job | What it does | Skips when |
|-----|-------------|------------|
| `trilio-cr-status` | Validates ConsistentSet available; writes status ConfigMap | — (always runs; fails until Available) |
| `trilio-cr-restore` | Restores from latest ConsistentSet to `wordpress-restore` | Completed restore exists |

### Manual backup

To trigger a backup outside the automated schedule:

```bash
ansible-navigator run ansible/playbooks/dr-backup.yaml
```

### Manual DR restore

**Standard restore** (from a named backup):

```bash
ansible-navigator run ansible/playbooks/dr-restore.yaml \
  -e restore_method=backup \
  -e restore_namespace=<target-namespace>
```

**Continuous Restore** (from a pre-staged ConsistentSet on the DR cluster — accelerated RTO):

```bash
ansible-navigator run ansible/playbooks/dr-restore.yaml \
  -e restore_method=consistentset \
  -e restore_namespace=<target-namespace>
```

Both commands discover the cluster ingress domain automatically and apply the Route hostname transform.

### Offboarding a spoke

```bash
# Step 1 — on the hub context
make unlabel-spoke CLUSTER=<acm-cluster-name>

# Step 2 — on the spoke context
make offboard-spoke CLUSTER=<acm-cluster-name>
```

### Uninstalling the pattern

```bash
# On the hub context
make offboard-hub
```

> Save your HashiCorp Vault root token and unseal keys before running `offboard-hub`. They are stored in the `imperative` namespace which is removed during offboard.

---

## Ansible Playbook Reference

| Playbook | When to use | Key inputs |
|----------|-------------|------------|
| `dr-backup.yaml` | Trigger a manual backup on the hub | — |
| `dr-restore.yaml` | Manual restore (backup or ConsistentSet method) | `restore_method`, `restore_namespace`, `source_backup` (optional) |
| `validate-trilio.yaml` | Pre/post-change Trilio health validation | — |
| `offboard-spoke.yaml` | Remove spoke-side Trilio resources | `cluster_name` |
| `offboard-hub.yaml` | Full hub pattern teardown | — |

Playbooks are run via `ansible-navigator`:

```bash
ansible-navigator run ansible/playbooks/<playbook>.yaml [-e key=value ...]
```

---

## Troubleshooting

### Trilio operator not installing

```bash
oc get subscription k8s-triliovault -n trilio-system -o yaml
oc get installplan -n trilio-system
```

Check that the `certified-operators` CatalogSource is healthy:

```bash
oc get catalogsource -n openshift-marketplace
```

### TrilioVaultManager not reaching Deployed/Updated

```bash
oc get triliovaultmanager -n trilio-system -o yaml
oc logs -n trilio-system -l app=k8s-triliovault-operator --tail=50
```

Common cause: the license Secret has not been created yet. Check External Secrets Operator (ESO) ExternalSecret status:

```bash
oc get externalsecret -n trilio-system
```

### BackupTarget stuck in Failed

```bash
oc get target -n trilio-system -o yaml
```

Common causes:
- S3 credentials are incorrect or the Secret has not been created by ESO yet
- `backupTarget.region` does not match the bucket's actual region — always set it explicitly

### No ConsistentSets appearing on the spoke

1. Verify the EventTarget pod is running: `oc get pods -n trilio-system | grep event`
2. Verify the spoke BackupTarget is Available: `oc get target -n trilio-system`
3. Verify at least one Available backup exists on the hub using the CR BackupPlan: `oc get backup -n wordpress`
4. Check that hub and spoke are running the same Trilio version: `oc get csv -n trilio-system`

### Imperative jobs stuck in Init:Error

```bash
# View logs from the failing init container
oc logs -n imperative <pod-name> -c <init-container-name>

# List init containers in order
oc get pod <pod-name> -n imperative -o jsonpath='{.spec.initContainers[*].name}'
```

The init container name matches the job name (e.g., `trilio-backup`). Each init container runs one playbook; a failure stops all subsequent jobs.

### Spoke ArgoCD not syncing after values-secondary.yaml changes

The spoke application has no automated sync. Kick it manually on the spoke context:

```bash
oc patch application.argoproj.io main-trilio-continuous-restore-secondary \
  -n openshift-gitops --type merge \
  -p '{"operation":{"sync":{}}}'
```

### BackupTarget or TrilioVaultManager perpetually OutOfSync in ArgoCD

Trilio continuously writes status fields to its own Custom Resources. ArgoCD detects these writes as drift and marks the application `OutOfSync` — even though the configuration is correct. This is expected behaviour and does not indicate a problem.

The Helm chart includes a `ServerSideDiff=true` annotation on Trilio CR templates to suppress this. If you see persistent `OutOfSync` without any configuration changes, verify the annotation is present:

```bash
oc get application trilio-operand -n openshift-gitops -o jsonpath='{.spec.syncPolicy}'
```

### Secrets written to Vault after ArgoCD has already synced

If ESO ExternalSecrets were created before the Vault secrets were populated, they may be in a `SecretSyncedError` state. Force an immediate re-sync:

```bash
oc annotate externalsecret trilio-s3-credentials -n trilio-system \
  force-sync=$(date +%s) --overwrite
oc annotate externalsecret trilio-license -n trilio-system \
  force-sync=$(date +%s) --overwrite
```

Wait 30 seconds and re-check:

```bash
oc get externalsecret -n trilio-system
```

### Vault root token — how to extract

The Vault root token and unseal keys are stored in the `vaultkeys` Secret in the `imperative` namespace. Extract the root token:

```bash
VAULT_TOKEN=$(oc get secret vaultkeys -n imperative \
  -o jsonpath='{.data.vault_data_json}' | \
  base64 -d | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")
echo $VAULT_TOKEN
```

> Save the root token and unseal keys before running `offboard-hub` — the `imperative` namespace is deleted during offboard and the Secret is lost.

---

## Operational Notes

### Secret values must be plain text

Secrets written to HashiCorp Vault must be plain text values, not Base64-encoded. ESO handles Base64 encoding when creating Kubernetes Secrets. If values are pre-encoded, ESO double-encodes them and Trilio receives garbled data, causing the BackupTarget to stay in `Failed` state.

### TrilioVaultManager healthy states

Both `Deployed` and `Updated` are healthy TrilioVaultManager states. `Updated` indicates a recent upgrade completed successfully. Monitoring scripts and health checks should accept either value.

### Imperative job update lag

When a configuration change is pushed to Git, there is a delay before the imperative CronJob picks it up:

1. ArgoCD polls Git every ~3 minutes and updates the ConfigMap
2. The CronJob runs every 10 minutes — the next pod starts at the next scheduled tick
3. The pod must mount the updated ConfigMap before the playbook runs

**Total lag: typically 15–30 minutes from `git push` to effect.** This is normal behaviour, not a failure.

---

## Links

- [Trilio for Kubernetes documentation](https://docs.trilio.io/kubernetes)
- [Red Hat Validated Patterns](https://validatedpatterns.io)
- [Validated Patterns imperative framework](https://validatedpatterns.io/learn/imperative-actions/)
- [Red Hat Advanced Cluster Management (ACM)](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes)
- [External Secrets Operator](https://external-secrets.io)

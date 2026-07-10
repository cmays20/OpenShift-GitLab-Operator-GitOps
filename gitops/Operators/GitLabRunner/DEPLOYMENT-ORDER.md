# GitLab Runner - Deployment Order

This document explains how the GitLab Runner integrates into the overall GitOps deployment.

## ArgoCD Application Sync Waves

The GitLab Runner application is deployed in **Wave 5**, which ensures all dependencies are ready:

```
Wave 0: ArgoCD Permissions
Wave 1: Cert-Manager, ODF-Lite
Wave 2: CloudNativePG
Wave 3: Keycloak, Vault Secrets Operator
Wave 4: OAuth, Vault Init, Vault Seed Secrets, GitLab
Wave 5: GitLab Runner ← YOU ARE HERE
```

## Why Wave 5?

GitLab Runner depends on:
- ✅ **GitLab** (Wave 4) - Must be fully deployed and healthy
- ✅ **Vault** (Wave 4) - For token storage
- ✅ **VSO** (Wave 3) - For secret sync from Vault
- ✅ **ODF** (Wave 1) - For S3 cache storage

## Internal Runner Sync Waves

Within the GitLab Runner application, resources are deployed in order:

```
Wave 4: RBAC
  - ServiceAccount: gitlab-runner-token-creator
  - Roles (gitlab-system, vault namespaces)
  - RoleBindings
  - ConfigMap: CA certificates

Wave 5: Token Creation
  - Job: create-gitlab-runner-token
    → Creates runner via GitLab API
    → Stores token in Vault

Wave 6: Vault Sync
  - VaultStaticSecret: gitlab-runner-registration-token
    → VSO syncs token from Vault
    → Creates K8s secret

Wave 7: Runner Deployment
  - Runner CR: gitlab-runner
    → GitLab Runner Operator processes
    → Runner pods start
    → Registers with GitLab
```

## Complete Bootstrap Order

When deploying from scratch:

1. **Bootstrap ArgoCD** (manual step)
2. **Deploy ArgoCD Applications**:
   ```bash
   oc apply -k gitops/ArgoCD-Applications/overlay
   ```

3. **ArgoCD deploys everything in order**:
   - Wave 0-3: Infrastructure operators
   - Wave 4: GitLab + Vault setup
   - **Wave 5: GitLab Runner** (automatic)

## Verification

After deployment, verify GitLab Runner is healthy:

```bash
# Check ArgoCD Application
argocd app get gitlab-runner

# Expected status: Healthy, Synced

# Check Runner pods
oc get pods -n gitlab-runner-system

# Check Runner registration
oc get runner -n gitlab-runner-system

# Check in GitLab UI
# Admin Area → CI/CD → Runners
# Should show runner "Online" with tags: kubernetes, openshift, demo
```

## Dependencies Graph

```
┌─────────────┐
│ Cert-Manager│
│    ODF      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ PostgreSQL  │
│   (CNPG)    │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│  Keycloak   │     │     VSO     │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └────────┬──────────┘
                │
                ▼
┌─────────────────────────────┐
│         Vault Init          │
│    Vault Seed Secrets       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│          GitLab             │
│   (includes Keycloak SSO)   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       GitLab Runner         │
│  (creates token via API)    │
│  (stores in Vault)          │
│  (syncs via VSO)            │
└─────────────────────────────┘
```

## Troubleshooting Deployment Order

### GitLab Runner Stuck in "OutOfSync"

**Cause**: GitLab not ready yet

**Fix**: Wait for GitLab to be healthy first
```bash
argocd app get gitlab
# Should show: Status: Healthy
```

### Token Creation Job Failed

**Cause**: GitLab not fully started

**Fix**: Job will retry automatically (backoffLimit: 5)
```bash
oc logs -n gitlab-runner-system job/create-gitlab-runner-token
```

### VSO Not Syncing Secret

**Cause**: Vault or VSO not ready

**Fix**: Check dependencies
```bash
# Check Vault
oc exec vault-0 -n vault -- vault status

# Check VSO
oc get pods -n vault-secrets-operator-system

# Check VaultAuth
oc get vaultauth -n vault
```

## Manual Deployment (Without ArgoCD)

If deploying manually, follow this order:

```bash
# 1. Deploy dependencies first
oc apply -k gitops/Operators/GitLab/aggregate
# Wait for GitLab to be fully healthy

# 2. Deploy GitLab Runner operator
oc apply -k gitops/Operators/GitLabRunner/operator

# 3. Approve InstallPlan
INSTALL_PLAN=$(oc get installplan -n gitlab-runner-system -o jsonpath='{.items[0].metadata.name}')
oc patch installplan $INSTALL_PLAN -n gitlab-runner-system \
  --type merge -p '{"spec":{"approved":true}}'

# 4. Wait for operator
oc wait --for=condition=ready pod -l app=gitlab-runner-operator \
  -n gitlab-runner-system --timeout=300s

# 5. Deploy runner instance
oc apply -k gitops/Operators/GitLabRunner/instance/base
```

## Automated Health Checks

ArgoCD monitors these resources for health:

- **Operator Subscription**: InstallPlan approved & CSV succeeded
- **Runner CR**: Status phase is "Running"
- **Job**: create-gitlab-runner-token completed successfully
- **VaultStaticSecret**: Secret synced successfully
- **Pods**: All runner pods ready

If any fail, ArgoCD shows the application as "Degraded".

## Next Steps

After GitLab Runner is deployed and healthy:
1. Test with a simple pipeline (see `../GitLab/TESTING.md`)
2. Verify builds use S3 cache from ODF
3. Configure additional runners for different workloads
4. Set up runner-specific resource quotas

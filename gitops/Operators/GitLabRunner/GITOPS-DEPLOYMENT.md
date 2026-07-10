# GitLab Runner - GitOps Deployment Guide

This guide explains the fully automated, GitOps-based deployment of GitLab Runner using Vault for secret management.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitOps Workflow                          │
└─────────────────────────────────────────────────────────────┘

1. ArgoCD syncs GitLabRunner resources
   │
   ├─► Wave 4: RBAC (ServiceAccount, Roles, RoleBindings)
   │
   ├─► Wave 5: Job: create-gitlab-runner-token
   │   │
   │   ├─► Wait for GitLab to be ready
   │   ├─► Get GitLab root password from K8s secret
   │   ├─► Create runner via GitLab Rails console
   │   ├─► Store token in Vault: secret/gitlab-runner/registration-token
   │   └─► Job completes
   │
   ├─► Wave 6: VaultStaticSecret syncs token
   │   │
   │   ├─► VSO watches Vault path
   │   ├─► Retrieves token from Vault
   │   └─► Creates K8s secret: gitlab-runner-registration-token
   │
   └─► Wave 7: Runner CR deployed
       │
       ├─► References secret: gitlab-runner-registration-token
       ├─► GitLab Runner Operator processes CR
       ├─► Registers runner with GitLab
       └─► Runner pods start and go online

```

## Components

### 1. Token Creation Job (`create-runner-token-job.yaml`)

**Purpose**: Automates runner creation via GitLab API and stores token in Vault

**What it does**:
1. Waits for GitLab to be ready (checks readiness endpoint)
2. Retrieves GitLab root password from `gitlab-gitlab-initial-root-password` secret
3. Uses `gitlab-rails runner` to create a new runner instance:
   - Type: Instance runner
   - Tags: `kubernetes`, `openshift`, `demo`
   - Description: "OpenShift Kubernetes Runner"
4. Stores the runner token in Vault at `secret/gitlab-runner/registration-token`

**RBAC Requirements**:
- Read secrets in `gitlab-system` namespace (for root password)
- Execute commands in Vault pod (to store token)

**Sync Wave**: `5` (runs after GitLab is deployed, before VSO sync)

### 2. Vault Secret Sync (`vault-runner-token.yaml`)

**Purpose**: Syncs runner token from Vault to Kubernetes secret

**What it does**:
- VSO watches Vault path: `secret/gitlab-runner/registration-token`
- Syncs to K8s secret: `gitlab-runner-registration-token` in `gitlab-runner-system` namespace
- Refreshes every 30 seconds to detect changes

**Sync Wave**: `6` (runs after token creation job)

### 3. Runner Custom Resource (`runner.yaml`)

**Purpose**: Defines the GitLab Runner configuration

**What it does**:
- References the Vault-synced secret for authentication
- Configures Kubernetes executor settings
- Sets up S3 cache backend (ODF)
- Defines resource limits and requests

**Sync Wave**: `7` (runs after VSO creates the secret)

## Prerequisites

Ensure these are deployed first:

```bash
# 1. Vault with unsealed status
oc get pods -n vault
# Should show vault-0 running and unsealed

# 2. Vault Secrets Operator (VSO)
oc get pods -n vault-secrets-operator-system

# 3. VaultAuth configured
oc get vaultauth -n vault

# 4. GitLab fully deployed and healthy
oc get gitlab -n gitlab-system
# Status should be "Running"

# 5. GitLab root password secret exists
oc get secret gitlab-gitlab-initial-root-password -n gitlab-system
```

## Deployment

### Option 1: ArgoCD Application (Recommended)

Create an ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab-runner
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: <your-repo-url>
    targetRevision: main
    path: gitops/Operators/GitLabRunner/aggregate
  destination:
    server: https://kubernetes.default.svc
    namespace: gitlab-runner-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Apply:
```bash
oc apply -f gitlab-runner-argocd-app.yaml

# Watch the sync
argocd app get gitlab-runner --refresh
```

### Option 2: Manual Kustomize (Testing)

```bash
# Deploy operator
oc apply -k gitops/Operators/GitLabRunner/operator

# Wait for operator to be installed
sleep 30

# Approve InstallPlan
INSTALL_PLAN=$(oc get installplan -n gitlab-runner-system -o jsonpath='{.items[0].metadata.name}')
oc patch installplan $INSTALL_PLAN -n gitlab-runner-system \
  --type merge -p '{"spec":{"approved":true}}'

# Wait for operator to be ready
oc wait --for=condition=ready pod -l app=gitlab-runner-operator \
  -n gitlab-runner-system --timeout=300s

# Deploy runner instance
oc apply -k gitops/Operators/GitLabRunner/instance/base
```

## Monitoring Deployment

### Watch the Entire Flow

```bash
# Terminal 1: Watch jobs
oc get jobs -n gitlab-runner-system -w

# Terminal 2: Watch secrets
oc get secrets -n gitlab-runner-system -w

# Terminal 3: Watch runner CR
oc get runner -n gitlab-runner-system -w

# Terminal 4: Watch pods
oc get pods -n gitlab-runner-system -w
```

### Step-by-Step Verification

#### Step 1: Verify RBAC Created

```bash
oc get sa gitlab-runner-token-creator -n gitlab-runner-system
oc get role gitlab-runner-token-creator -n gitlab-system
oc get role gitlab-runner-token-creator-vault -n vault
```

Expected: All resources exist

#### Step 2: Watch Token Creation Job

```bash
# Check job status
oc get job create-gitlab-runner-token -n gitlab-runner-system

# Follow logs
oc logs -n gitlab-runner-system job/create-gitlab-runner-token -f
```

Expected output:
```
=== GitLab Runner Token Creation ===
GitLab URL: https://gitlab.apps.mays-demo.sandbox3446.opentlc.com

Waiting for GitLab to be ready...
✓ GitLab is ready
✓ GitLab root password retrieved
Creating runner authentication token via GitLab Rails...
✓ Runner token created: glrt-xxxxx...
Storing runner token in Vault...
✓ Runner token stored in Vault at: secret/gitlab-runner/registration-token

=== GitLab Runner Token Creation Complete ===
```

#### Step 3: Verify Token in Vault

```bash
# Get Vault root token
VAULT_TOKEN=$(oc get secret vault-unseal-keys -n vault -o jsonpath='{.data.root-token}' | base64 -d)

# Check token exists in Vault
oc exec vault-0 -n vault -- sh -c "VAULT_TOKEN=$VAULT_TOKEN vault kv get secret/gitlab-runner/registration-token"
```

Expected output:
```
====== Data ======
Key                         Value
---                         -----
runner-registration-token   glrt-xxxxxxxxxxxx
```

#### Step 4: Verify VSO Synced Secret

```bash
# Check VaultStaticSecret status
oc get vaultstaticsecret gitlab-runner-registration-token -n gitlab-runner-system

# Should show: AGE and STATUS columns

# Check the actual K8s secret
oc get secret gitlab-runner-registration-token -n gitlab-runner-system

# View secret data (verify it matches Vault)
oc get secret gitlab-runner-registration-token -n gitlab-runner-system \
  -o jsonpath='{.data.runner-registration-token}' | base64 -d
```

Expected: Secret exists with valid token data

#### Step 5: Verify Runner Registration

```bash
# Check Runner CR
oc get runner gitlab-runner -n gitlab-runner-system

# Should show: NAME, STATUS, READY columns
# STATUS should be "Running" or "Registered"

# Check runner pods
oc get pods -n gitlab-runner-system -l app=gitlab-runner

# View runner logs
oc logs -n gitlab-runner-system -l app=gitlab-runner

# Should see:
# "Runner registered successfully"
```

#### Step 6: Verify in GitLab UI

1. Login to GitLab: `https://gitlab.apps.mays-demo.sandbox3446.opentlc.com`
2. Go to **Admin Area** → **CI/CD** → **Runners**
3. Look for runner with:
   - ✅ **Status: Online** (green circle)
   - **Tags**: `kubernetes`, `openshift`, `demo`
   - **Description**: "OpenShift Kubernetes Runner"
   - **Last contact**: Within the last minute

## Troubleshooting

### Job Stuck in "ContainerCreating"

**Cause**: RBAC not applied yet or image pull issues

**Fix**:
```bash
oc describe pod -n gitlab-runner-system -l job-name=create-gitlab-runner-token
```

### Job Failed: "GitLab not ready"

**Cause**: GitLab pods not fully started

**Fix**:
```bash
# Check GitLab status
oc get pods -n gitlab-system

# Wait for all GitLab pods to be ready
oc wait --for=condition=ready pod -l app=webservice -n gitlab-system --timeout=600s

# The job will retry automatically (backoffLimit: 5)
```

### Job Failed: "Could not retrieve GitLab root password"

**Cause**: GitLab root password secret doesn't exist

**Fix**:
```bash
# Verify secret exists
oc get secret gitlab-gitlab-initial-root-password -n gitlab-system

# If missing, GitLab deployment failed - check GitLab operator logs
oc logs -n gitlab-system -l app=gitlab-operator
```

### Job Failed: "Could not retrieve Vault root token"

**Cause**: Vault not unsealed or root token secret missing

**Fix**:
```bash
# Check Vault status
oc exec vault-0 -n vault -- vault status

# Check for unseal keys secret
oc get secret vault-unseal-keys -n vault

# If Vault is sealed, unseal it first
```

### VSO Not Creating Secret

**Cause**: VaultAuth not configured or VSO not running

**Fix**:
```bash
# Check VaultAuth
oc get vaultauth -n vault
oc describe vaultauth vault-auth -n vault

# Check VSO pods
oc get pods -n vault-secrets-operator-system

# Check VSO logs
oc logs -n vault-secrets-operator-system deployment/vault-secrets-operator-controller-manager

# Check VaultStaticSecret status
oc describe vaultstaticsecret gitlab-runner-registration-token -n gitlab-runner-system
```

### Runner Shows "Offline" in GitLab

**Cause**: Runner can't connect to GitLab or token invalid

**Fix**:
```bash
# Check runner pod logs
oc logs -n gitlab-runner-system -l app=gitlab-runner

# Common errors:
# - "certificate signed by unknown authority" → CA cert issue
# - "401 Unauthorized" → Token invalid
# - "dial tcp: i/o timeout" → Network/firewall issue

# Test GitLab connectivity from runner namespace
oc run test-curl -n gitlab-runner-system --image=curlimages/curl --rm -it --restart=Never -- \
  curl -v https://gitlab.apps.mays-demo.sandbox3446.opentlc.com/-/readiness
```

### Token Exists but Runner Won't Register

**Cause**: Token format or secret key name mismatch

**Fix**:
```bash
# Check secret structure
oc get secret gitlab-runner-registration-token -n gitlab-runner-system -o yaml

# The data field MUST contain key: runner-registration-token
# If it has a different key name, VSO mapping is wrong

# Check Runner CR references correct secret
oc get runner gitlab-runner -n gitlab-runner-system -o yaml | grep -A5 "token:"
```

## Updating the Runner Token

To rotate the runner token:

```bash
# 1. Delete the job to trigger recreation
oc delete job create-gitlab-runner-token -n gitlab-runner-system

# 2. ArgoCD will recreate the job automatically
# Or manually trigger:
oc apply -k gitops/Operators/GitLabRunner/instance/base

# 3. New token will be created and stored in Vault
# 4. VSO will sync the new token within 30s
# 5. Runner pods will automatically use the new token on next registration
```

## Cleanup

To remove the runner:

```bash
# Delete the runner instance (keeps operator)
oc delete -k gitops/Operators/GitLabRunner/instance/base

# Or delete everything including operator
oc delete -k gitops/Operators/GitLabRunner/aggregate

# Clean up Vault secret
VAULT_TOKEN=$(oc get secret vault-unseal-keys -n vault -o jsonpath='{.data.root-token}' | base64 -d)
oc exec vault-0 -n vault -- sh -c "VAULT_TOKEN=$VAULT_TOKEN vault kv delete secret/gitlab-runner/registration-token"
```

## Best Practices

1. **Always use sync waves** - Ensures proper ordering (RBAC → Job → VSO → Runner)
2. **Monitor job logs** - The token creation job provides detailed output
3. **Verify Vault sync** - Check both Vault and K8s secret before troubleshooting runner
4. **Use ArgoCD health checks** - Configure health checks for Runner CR
5. **Enable auto-sync** - Let ArgoCD handle updates automatically
6. **Set retry policies** - Job has backoffLimit: 5 to handle transient failures

## Security Considerations

- ✅ **No tokens in Git**: Token stored only in Vault, not in manifests
- ✅ **RBAC minimized**: Job only has access to specific secrets/namespaces
- ✅ **Vault-backed**: Centralized secret management
- ✅ **Auto-rotation ready**: Easy to rotate by recreating the job
- ✅ **Audit trail**: Vault logs all secret access
- ⚠️ **Root token used**: Job uses Vault root token (consider using scoped policies in production)

## Next Steps

After successful deployment:
1. Test the runner with a simple pipeline (see `../GitLab/TESTING.md`)
2. Configure additional runners for different workloads
3. Set up runner-specific tags for job routing
4. Configure cache to speed up builds
5. Enable DinD if container image builds are needed

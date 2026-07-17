# GitLab Runner Operator

This directory contains the configuration for deploying GitLab Runner on OpenShift using the GitLab Runner Operator.

## Overview

The GitLab Runner Operator manages GitLab Runner instances that execute CI/CD jobs from your GitLab instance. This setup uses:
- **Kubernetes executor**: Runs jobs in ephemeral pods within the `gitlab-runner-system` namespace
- **S3 cache backend**: Uses OpenShift Data Foundation (ODF) for build cache storage
- **OpenShift-optimized images**: Uses Red Hat UBI-based runner images

## Directory Structure

```
GitLabRunner/
├── operator/                  # Operator installation
│   ├── namespace.yaml
│   ├── operator-group.yaml
│   ├── subscription.yaml
│   └── kustomization.yaml
├── instance/                  # Runner instance configuration
│   └── base/
│       ├── rbac.yaml          # Service account and permissions
│       ├── ca-cert-configmap.yaml
│       ├── runner-token-secret.yaml
│       ├── runner.yaml        # Runner CR
│       └── kustomization.yaml
├── aggregate/                 # Combined deployment
│   └── kustomization.yaml
└── README.md
```

## Prerequisites

1. **GitLab instance running**: `https://gitlab.apps.mays-demo.sandbox1374.opentlc.com`
2. **OpenShift Data Foundation (ODF)**: For S3-compatible cache storage
3. **HashiCorp Vault**: With VSO (Vault Secrets Operator) installed
4. **Vault authentication configured**: For syncing secrets

## How It Works (GitOps Approach)

This setup uses a **fully automated GitOps workflow** - no manual token creation needed:

1. **Job creates runner via GitLab API**: `create-runner-token-job.yaml` uses GitLab Rails to create a runner instance
2. **Token stored in Vault**: The job stores the runner token at `secret/gitlab-runner/registration-token`
3. **VSO syncs to Kubernetes**: `vault-runner-token.yaml` syncs the Vault secret to the `gitlab-runner-system` namespace
4. **Runner uses the token**: The Runner CR references the synced secret

### Deployment Flow (ArgoCD Sync Waves)

```
Wave 4: RBAC (service accounts, roles)
Wave 5: Job creates runner token → stores in Vault
Wave 6: VSO syncs token from Vault → K8s secret
Wave 7: Runner CR deployed → uses synced token
```

**No manual steps required** - the entire workflow is declarative and GitOps-friendly!

## Installation Steps

### Step 1: Deploy via ArgoCD

If using ArgoCD, create an Application:

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
```

### Step 2: Approve the Operator InstallPlan

```bash
# Wait for the InstallPlan to be created
oc get installplan -n gitlab-runner-system

# Approve it
INSTALL_PLAN=$(oc get installplan -n gitlab-runner-system -o jsonpath='{.items[0].metadata.name}')
oc patch installplan $INSTALL_PLAN -n gitlab-runner-system \
  --type merge -p '{"spec":{"approved":true}}'
```

### Step 3: Monitor the Automated Setup

Watch the automated workflow:

```bash
# Step 1: Watch the token creation job
oc get jobs -n gitlab-runner-system -w

# Step 2: View job logs to see token creation
oc logs -n gitlab-runner-system job/create-gitlab-runner-token

# Step 3: Verify token stored in Vault
oc exec vault-0 -n vault -- sh -c \
  "VAULT_TOKEN=$(oc get secret vault-unseal-keys -n vault -o jsonpath='{.data.root-token}' | base64 -d) \
  vault kv get secret/gitlab-runner/registration-token"

# Step 4: Verify VSO synced the secret
oc get secret gitlab-runner-registration-token -n gitlab-runner-system

# Step 5: Check runner registration
oc get runner -n gitlab-runner-system

# Step 6: View runner logs
oc logs -n gitlab-runner-system -l app=gitlab-runner
```

## Verify in GitLab UI

1. Login to GitLab as admin
2. Go to **Admin Area** → **CI/CD** → **Runners**
3. You should see your runner listed with:
   - Status: **Online** (green)
   - Tags: `kubernetes, openshift, demo`
   - IP address from OpenShift cluster

## Testing the Runner

### Create a Test Pipeline

In your GitLab project, create `.gitlab-ci.yml`:

```yaml
stages:
  - test
  - build

hello-world:
  stage: test
  tags:
    - kubernetes
  script:
    - echo "Hello from GitLab Runner on OpenShift!"
    - uname -a
    - cat /etc/os-release

build-example:
  stage: build
  tags:
    - kubernetes
  script:
    - echo "Building application..."
    - mkdir -p build
    - echo "Build artifact" > build/artifact.txt
  artifacts:
    paths:
      - build/
    expire_in: 1 hour
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .cache/
```

Commit and push:
```bash
git add .gitlab-ci.yml
git commit -m "Add CI pipeline"
git push origin main
```

Check the pipeline runs successfully in GitLab UI: **CI/CD** → **Pipelines**

## Configuration Details

### Runner Executor Settings

The runner uses the **Kubernetes executor** with these settings:

- **Default image**: `registry.access.redhat.com/ubi9/ubi-minimal:latest`
- **Namespace**: `gitlab-runner-system`
- **Privileged mode**: Disabled (more secure)
- **Resource requests**:
  - Build pod: 100m CPU, 128Mi memory
  - Helper pod: 100m CPU, 128Mi memory

### S3 Cache Configuration

Build caches are stored in ODF S3:
- **Bucket**: `gitlab-runner-cache`
- **Endpoint**: `s3.openshift-storage.svc:80`
- **Path**: `gitlab-runner`
- **Shared**: Yes (cache shared across jobs)

### Security Context

The runner operates with:
- Dedicated service account: `gitlab-runner`
- Minimal RBAC permissions (pod creation, secret access in its namespace only)
- Non-privileged containers

## Customization

### Change Runner Concurrency

Edit `instance/base/runner.yaml`:

```yaml
spec:
  concurrent: 10  # Number of jobs that can run simultaneously
```

### Add More Tags

Edit `instance/base/runner.yaml`:

```yaml
spec:
  tags: kubernetes, openshift, demo, docker, maven, nodejs
```

### Use Different Base Image

Edit `instance/base/runner.yaml`:

```yaml
spec:
  config: |
    [[runners]]
      [runners.kubernetes]
        image = "registry.access.redhat.com/ubi9/nodejs-18:latest"
```

### Enable Docker-in-Docker (DinD)

For building container images, enable privileged mode:

```yaml
spec:
  config: |
    [[runners]]
      [runners.kubernetes]
        privileged = true  # Required for DinD
        image = "docker:24-dind"
```

⚠️ **Warning**: Privileged containers have security implications. Only enable if necessary.

## Troubleshooting

### Token Creation Job Failed

```bash
# Check job status
oc get jobs -n gitlab-runner-system

# View job logs
oc logs -n gitlab-runner-system job/create-gitlab-runner-token

# Common issues:
# - GitLab not ready yet (job will retry)
# - Cannot access GitLab root password secret
# - Cannot access Vault
```

### VSO Not Syncing Token

```bash
# Check VaultStaticSecret status
oc get vaultstaticsecret -n gitlab-runner-system

# Check VSO logs
oc logs -n vault-secrets-operator-system deployment/vault-secrets-operator-controller-manager

# Verify Vault authentication
oc get vaultauth -n vault

# Check if secret exists in Vault
oc exec vault-0 -n vault -- sh -c \
  "VAULT_TOKEN=$(oc get secret vault-unseal-keys -n vault -o jsonpath='{.data.root-token}' | base64 -d) \
  vault kv get secret/gitlab-runner/registration-token"
```

### Runner Not Registering

```bash
# Check runner pod logs
oc logs -n gitlab-runner-system -l app=gitlab-runner

# Check the synced secret
oc get secret gitlab-runner-registration-token -n gitlab-runner-system -o yaml

# Verify token is valid
TOKEN=$(oc get secret gitlab-runner-registration-token -n gitlab-runner-system \
  -o jsonpath='{.data.runner-registration-token}' | base64 -d)
echo "Token: ${TOKEN:0:10}..."

# Common issues:
# - VSO hasn't synced the token yet (wait 30s)
# - Token secret has wrong key name
# - GitLab URL not reachable from cluster
# - Certificate validation failures
```

### Jobs Stuck in Pending

```bash
# Check if runner is online in GitLab UI
# Check runner logs
oc logs -n gitlab-runner-system -l app=gitlab-runner

# Check for resource constraints
oc describe runner gitlab-runner -n gitlab-runner-system
```

### Certificate Errors

If using custom CA certificates, update `instance/base/ca-cert-configmap.yaml`:

```yaml
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    <your-ca-certificate>
    -----END CERTIFICATE-----
```

For Let's Encrypt (default setup), the CA ConfigMap can remain empty.

### Cache Not Working

```bash
# Verify S3 bucket exists
oc get obc gitlab-runner-cache -n gitlab-system

# Check cache credentials in runner config
oc get runner gitlab-runner -n gitlab-runner-system -o yaml

# View runner cache configuration
oc logs -n gitlab-runner-system -l app=gitlab-runner | grep -i cache
```

### Jobs Failing with "Image Pull" Errors

```bash
# Check if the default image is accessible
oc run test-image --image=registry.access.redhat.com/ubi9/ubi-minimal:latest \
  -n gitlab-runner-system --rm -it --restart=Never -- echo "success"

# If image pull fails, check image pull secrets or use a different registry
```

## Scaling Runners

### Horizontal Scaling (Multiple Runners)

Create additional Runner CRs with different names:

```yaml
apiVersion: apps.gitlab.com/v1beta2
kind: Runner
metadata:
  name: gitlab-runner-2
  namespace: gitlab-runner-system
spec:
  # ... same config as gitlab-runner
  token: gitlab-runner-registration-token-2
```

### Vertical Scaling (More Concurrent Jobs)

Edit runner concurrency:

```yaml
spec:
  concurrent: 20  # Increase from default 10
```

## Cleanup

To remove the GitLab Runner:

```bash
# Delete the runner instance
oc delete runner gitlab-runner -n gitlab-runner-system

# Delete the namespace (removes everything)
oc delete namespace gitlab-runner-system

# Or delete via ArgoCD
argocd app delete gitlab-runner
```

## Additional Resources

- [GitLab Runner Operator Docs](https://docs.gitlab.com/runner/install/operator.html)
- [GitLab Runner Kubernetes Executor](https://docs.gitlab.com/runner/executors/kubernetes.html)
- [GitLab CI/CD Configuration](https://docs.gitlab.com/ee/ci/yaml/)

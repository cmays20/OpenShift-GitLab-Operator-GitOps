# OpenShift GitLab Platform - GitOps Deployment

This repository contains a fully declarative, GitOps-driven deployment of a complete GitLab platform and supporting infrastructure on OpenShift. Every component — from operator subscriptions to secret generation to SSO configuration — is defined in Git and reconciled by ArgoCD, making the entire stack repeatable and self-healing.

Two `oc apply` commands bootstrap the entire platform from a bare OpenShift cluster with OpenShift GitOps installed.

## Architecture

The deployment uses the **App-of-Apps** pattern. A single root ArgoCD Application generates 14 child Applications, each responsible for a component of the stack. ArgoCD sync waves enforce dependency ordering so components deploy in the correct sequence.

```
argocd-setup.yaml          ──>  ArgoCD self-configuration (health checks, sync settings)
demo-setup.yaml            ──>  App-of-Apps root
                                 │
                                 ├── Wave 0: ArgoCD namespace permissions
                                 ├── Wave 1: Cert-Manager, CloudNativePG, ODF, Vault, Redis Operator, Web Terminal
                                 ├── Wave 1: Vault initialization + unsealing
                                 ├── Wave 2: Keycloak, Vault Secrets Operator, Vault secret seeding
                                 ├── Wave 3: OpenShift OAuth (Keycloak OIDC)
                                 ├── Wave 4: GitLab
                                 └── Wave 5: GitLab Runner
```

All Applications use `selfHeal: true` and `ServerSideApply=true`, so ArgoCD continuously reconciles drift.

## Components

### Operators (OLM)

| Operator | Namespace | Purpose |
|----------|-----------|---------|
| OpenShift Cert-Manager | cert-manager-operator | TLS certificates via Let's Encrypt (DNS-01 / Route53) |
| CloudNativePG | openshift-operators | PostgreSQL clusters for GitLab and Keycloak |
| OpenShift Data Foundation | openshift-storage | S3-compatible object storage via NooBaa/MCG |
| Red Hat Build of Keycloak | keycloak | Identity management and SSO (OIDC) |
| Vault Secrets Operator | openshift-operators | Syncs secrets from Vault to Kubernetes |
| GitLab Operator | gitlab-system | GitLab instance lifecycle |
| GitLab Runner Operator | gitlab-runner-system | CI/CD runner pods |
| Web Terminal | openshift-operators | In-console web terminal |

### Helm Charts

| Component | Chart Version | Namespace | Purpose |
|-----------|---------------|-----------|---------|
| HashiCorp Vault | v0.28.1 | vault | Secret management (HA, 3 replicas, Raft consensus) |
| Redis Operator (OT) | v0.25.0 | ot-operators | Redis instance for GitLab |

### Supporting Infrastructure

| Component | Details |
|-----------|---------|
| PostgreSQL (GitLab) | 3-instance CloudNativePG cluster, 50Gi storage |
| PostgreSQL (Keycloak) | 3-instance CloudNativePG cluster, 5Gi storage |
| Redis | Standalone instance with exporter, 5Gi storage |
| Object Storage | 15 NooBaa ObjectBucketClaims (artifacts, uploads, LFS, registry, backups, etc.) |
| Wildcard TLS Certificate | Let's Encrypt via cert-manager, DNS-01 challenge with Route53 |

## Automated Pipelines

Several one-time setup Jobs run during initial deployment to handle tasks that can't be expressed as static manifests:

**Vault Initialization** — Initializes Vault with Shamir key shares, unseals all 3 replicas, joins the Raft cluster, enables Kubernetes auth and KV-v2 secrets engine, and creates RBAC policies. Unseal keys and root token are stored in a Kubernetes Secret.

**Secret Seeding** — Generates initial passwords and OAuth client secrets, stores them in Vault. All operations are idempotent (skipped if secrets already exist).

**Secret Distribution** — The Vault Secrets Operator syncs secrets from Vault to target namespaces via VaultStaticSecret resources, keeping Kubernetes Secrets in sync with Vault.

**GitLab Runner Token** — Creates a runner via the GitLab Rails console, stores the token in Vault, and VSO syncs it to the runner namespace.

**Object Storage Secrets** — Reads OBC-generated credentials and transforms them into GitLab's expected secret format.

**SSO Integration** — Two Keycloak realms are imported: one for OpenShift OAuth (OIDC) and one for GitLab OmniAuth (OIDC with PKCE). OAuth client secrets are managed through Vault.

## Prerequisites

- An OpenShift cluster (tested on ROSA)
- OpenShift GitOps operator installed (provides ArgoCD)
- AWS credentials available for Route53 DNS-01 challenges (cert-manager)
- A fork of this repository (the git repo URL and revision are injected at deployment time)

## Getting Started

1. **Configure ArgoCD:**

   ```bash
   oc apply -f argocd-setup.yaml
   ```

   Wait for the ArgoCD setup Application to sync. This configures custom health checks, sync wave delay, and kustomize-helm support.

2. **Deploy the platform:**

   ```bash
   oc apply -f demo-setup.yaml
   ```

   This creates the App-of-Apps, which generates all child Applications. ArgoCD will deploy the entire stack in dependency order via sync waves. Full deployment takes approximately 30-45 minutes.

## Project Structure

```
├── argocd-setup.yaml                    # Bootstrap: ArgoCD self-configuration
├── demo-setup.yaml                      # Bootstrap: App-of-Apps root
│
└── gitops/
    ├── ArgoCD-Setup/                    # ArgoCD configuration Jobs
    ├── ArgoCD-Applications/             # App-of-Apps child Application manifests
    │   ├── base/                        #   Kustomize-based Applications (12)
    │   ├── helm-charts/                 #   Helm-based Applications (2) + values + fixes
    │   ├── base-overlay/                #   Injects git repo URL/revision into base
    │   ├── helm-charts-overlay/         #   Injects git repo URL/revision into helm apps
    │   └── aggregate/                   #   Combines both overlays
    │
    ├── ArgoCD-permissions/              # Per-namespace RBAC for ArgoCD
    │
    ├── Operators/
    │   ├── CertManager/                 # operator/ + instance/ (certs, DNS)
    │   ├── CloudNativePG/               # operator/ only
    │   ├── ODF/                         # operator/ + instance/ (MCG on AWS)
    │   ├── Keycloak/                    # operator/ + instance/ (realms, SSO)
    │   ├── WebTerminal/                 # operator/ only
    │   ├── VaultSecretsOperator/        # operator/ + instance/ (VaultAuth, VaultConnection)
    │   ├── GitLab/                      # operator/ + instance/ (GitLab CR, storage, OmniAuth)
    │   └── GitLabRunner/                # operator/ + instance/ + cross-namespace RBAC
    │
    ├── Vault-Setup/
    │   ├── vault-init/                  # Init, unseal, Raft join, K8s auth, policies
    │   └── vault-seed-secrets/          # Seed initial passwords and OAuth secrets
    │
    └── OAuth-Setup/                     # OpenShift OAuth via Keycloak OIDC
```

Each operator follows a **base / overlay / aggregate** pattern:
- **base/** contains the core manifests
- **overlay/** patches environment-specific values (domains, hostnames, channels)
- **aggregate/** combines both via Kustomize for the final rendered output

## Customization

The App-of-Apps layer uses Kustomize `replacements` to inject the git repository URL and revision into all child Application manifests. The base manifests use `GITREPO` and `REVISION` placeholders that are replaced at render time via a ConfigMapGenerator.

To point at your own fork:
1. Update the `repoURL` and `targetRevision` in `demo-setup.yaml`
2. Update environment-specific overlays (domains, Route53 hosted zone, email) under each operator's `overlay/` directory

## Known Issues

**Redis Operator ClusterRole** — The redis-operator Helm chart hardcodes an `aggregate-to-admin` label and a `nonResourceURLs` rule on its ClusterRole. These get aggregated into the built-in `admin` ClusterRole, which breaks the OpenShift GitOps operator's reconciliation of namespace-scoped Roles. A fix Job in `redis-operator-fixes/` patches the ClusterRole, and `ignoreDifferences` on the ArgoCD Application prevents self-heal from restoring the broken state.

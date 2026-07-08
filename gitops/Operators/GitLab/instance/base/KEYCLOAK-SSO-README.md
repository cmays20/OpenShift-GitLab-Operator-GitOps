# GitLab + Keycloak SSO Integration

## Overview

This setup integrates GitLab with Keycloak for Single Sign-On (SSO) authentication using OpenID Connect.

## Components Created

### 1. Keycloak Realm Import (`gitlab-realm-import.yaml`)
- Creates a new Keycloak realm named `gitlab`
- Configures an OAuth2/OIDC client for GitLab
- Sets up required scopes: `openid`, `profile`, `email`, `groups`
- Client ID: `gitlab`
- Client secret: Stored in Vault and synced via VSO

### 2. Vault Secret Sync (`vault-gitlab-secrets.yaml`)
- Two VaultStaticSecret resources:
  - One for `gitlab-system` namespace (used by GitLab)
  - One for `keycloak` namespace (used by realm import)
- Both sync from `secret/gitlab/oauth-client-secret` in Vault
- Secret contains the OAuth client secret shared between Keycloak and GitLab

### 3. Vault Seed Job (Updated)
- Extended `vault-seed-secrets-job.yaml` to generate GitLab OAuth secret
- Secret generated on first run: `secret/gitlab/oauth-client-secret`
- Uses strong random secret (base64 encoded 32 bytes)

### 4. OmniAuth Configuration Job (`create-omniauth-secret-job.yaml`)
- Runs after VSO syncs the OAuth secret (sync-wave: 7)
- Retrieves client secret from `gitlab-oauth-client-secret`
- Creates `gitlab-omniauth-provider` secret with full OmniAuth config
- Configures GitLab to connect to Keycloak OIDC endpoint

### 5. GitLab Configuration (Updated `gitlab.yaml`)
Added OmniAuth configuration:
```yaml
global:
  omniauth:
    enabled: true
    allowSingleSignOn: ['openid_connect']
    blockAutoCreatedUsers: false
    autoLinkUser: ['openid_connect']
    providers:
      - secret: gitlab-omniauth-provider
        key: provider
```

## Deployment Order (ArgoCD Sync Waves)

1. **Wave 1**: ObjectBucketClaims, Redis, PostgreSQL
2. **Wave 2**: Service accounts, RBAC
3. **Wave 3**: Object storage secrets job
4. **Wave 5**: VaultStaticSecrets (syncs OAuth secret from Vault)
5. **Wave 6**: OmniAuth job service account/RBAC
6. **Wave 7**: OmniAuth secret creation job
7. **Wave 10**: Keycloak realm import (uses synced secret)

## User Management Flow

### Initial Setup
1. Vault seed job generates OAuth client secret
2. VSO syncs secret to both `gitlab-system` and `keycloak` namespaces
3. Keycloak realm import creates the `gitlab` realm and OAuth client
4. OmniAuth job creates GitLab provider configuration
5. GitLab loads OmniAuth configuration on startup

### User Authentication
1. User visits GitLab login page
2. Clicks "Keycloak SSO" button
3. Redirected to Keycloak (`https://keycloak.apps.mays-demo.sandbox3446.opentlc.com/realms/gitlab`)
4. Authenticates with Keycloak credentials
5. Keycloak redirects back to GitLab with OAuth token
6. GitLab auto-creates user account (JIT provisioning)
7. User is logged into GitLab

### User Attributes Synced from Keycloak
- **Email**: Primary identifier (unique in GitLab)
- **Username**: From `preferred_username` claim
- **Full Name**: From `given_name` and `family_name` claims
- **Groups**: From `groups` claim (for future group mapping)

## Configuration Details

### Keycloak Client Settings
- **Client ID**: `gitlab`
- **Client Type**: Confidential (requires secret)
- **Valid Redirect URIs**: `https://gitlab.apps.mays-demo.sandbox3446.opentlc.com/users/auth/openid_connect/callback`
- **Web Origins**: `https://gitlab.apps.mays-demo.sandbox3446.opentlc.com`
- **PKCE**: Enabled (for enhanced security)

### GitLab OmniAuth Settings
- **Provider**: `openid_connect`
- **Discovery**: Enabled (auto-discovers endpoints from Keycloak)
- **Auto Sign-On**: Enabled for `openid_connect`
- **Block Auto-Created Users**: `false` (allows JIT provisioning)
- **Auto Link**: Enabled (links existing GitLab users with same email)

## Security Features

✅ **PKCE Enabled**: Protects against authorization code interception  
✅ **Client Secret**: Stored securely in Vault  
✅ **Encrypted Transit**: All communication over HTTPS  
✅ **Brute Force Protection**: Enabled in Keycloak realm  
✅ **Email Verification**: Can be enforced in Keycloak  

## User Administration

### Adding Users to Keycloak

**Via Keycloak UI:**
1. Login to Keycloak: `https://keycloak.apps.mays-demo.sandbox3446.opentlc.com`
2. Select `gitlab` realm (top-left dropdown)
3. Navigate to Users → Add user
4. Fill in: Username, Email, First Name, Last Name
5. Save
6. Go to Credentials tab → Set password
7. User can now login to GitLab via SSO

**Via GitLab UI (after first login):**
1. User logs in via Keycloak SSO (auto-created in GitLab)
2. GitLab admin adds user to projects/groups
3. User permissions managed in GitLab

### Group Management

**Current Setup (Basic):**
- Keycloak sends `groups` claim in token
- GitLab receives group memberships but doesn't auto-assign
- Manual group assignment required in GitLab UI

**Future Enhancement (Requires GitLab Premium):**
- Configure SAML group sync
- Map Keycloak groups to GitLab groups
- Auto-assign users to GitLab groups based on Keycloak membership

## Emergency Access

**Always keep root user active!**

If Keycloak is unavailable or SSO breaks:
```bash
# Get root password
oc get secret gitlab-gitlab-initial-root-password -n gitlab-system \
  -o jsonpath='{.data.password}' | base64 -d && echo

# Login directly with username: root
# URL: https://gitlab.apps.mays-demo.sandbox3446.opentlc.com/users/sign_in
```

## Testing the Integration

### 1. Verify Secrets Exist
```bash
# Check OAuth secret in gitlab-system
oc get secret gitlab-oauth-client-secret -n gitlab-system

# Check OAuth secret in keycloak
oc get secret gitlab-oauth-client-secret -n keycloak

# Check OmniAuth provider config
oc get secret gitlab-omniauth-provider -n gitlab-system
```

### 2. Verify Keycloak Realm
```bash
# Check realm import status
oc get keycloakrealmimport gitlab-realm -n keycloak

# Should show status: Ready
```

### 3. Test SSO Login
1. Navigate to `https://gitlab.apps.mays-demo.sandbox3446.opentlc.com`
2. Look for "Keycloak SSO" button on login page
3. Click it - should redirect to Keycloak
4. Login with Keycloak credentials
5. Should redirect back to GitLab and auto-create account

### 4. Verify User Creation
```bash
# In GitLab UI, check Admin Area → Users
# Or via GitLab Rails console:
oc exec -it deployment/gitlab-toolbox -n gitlab-system -- \
  gitlab-rails runner "puts User.all.pluck(:username, :email)"
```

## Troubleshooting

### "Keycloak SSO" Button Not Showing
```bash
# Check OmniAuth configuration loaded
oc exec -it deployment/gitlab-toolbox -n gitlab-system -- \
  gitlab-rails runner "puts Gitlab.config.omniauth.enabled"

# Should return: true

# Check provider configuration
oc get secret gitlab-omniauth-provider -n gitlab-system \
  -o jsonpath='{.data.provider}' | base64 -d
```

### SSO Login Fails
```bash
# Check GitLab logs
oc logs -n gitlab-system deployment/gitlab-webservice-default --tail=100 | grep -i oauth

# Check Keycloak realm is ready
oc get keycloakrealmimport gitlab-realm -n keycloak -o yaml

# Verify client secret matches
oc get secret gitlab-oauth-client-secret -n gitlab-system -o jsonpath='{.data.client-secret}' | base64 -d
oc get secret gitlab-oauth-client-secret -n keycloak -o jsonpath='{.data.client-secret}' | base64 -d
# Both should match
```

### "Redirect URI Mismatch" Error
- Check Keycloak client redirect URIs include:
  `https://gitlab.apps.mays-demo.sandbox3446.opentlc.com/users/auth/openid_connect/callback`
- Verify GitLab URL in OmniAuth config matches

## Configuration Files

- **Realm Import**: `gitlab-realm-import.yaml`
- **Vault Secrets Sync**: `vault-gitlab-secrets.yaml`
- **OmniAuth Job**: `create-omniauth-secret-job.yaml`
- **GitLab Config**: `gitlab.yaml` (omniauth section)
- **Vault Seed**: `../../Vault-Setup/vault-seed-secrets/vault-seed-secrets-job.yaml`

## Maintenance

### Rotating OAuth Client Secret
```bash
# 1. Generate new secret
NEW_SECRET=$(openssl rand -base64 32)

# 2. Update in Vault (requires vault CLI or web UI)
vault kv put secret/gitlab/oauth-client-secret client-secret="$NEW_SECRET"

# 3. VSO will auto-sync to both namespaces within 30s
# 4. Keycloak realm will update automatically
# 5. Restart GitLab pods to pick up new secret
oc rollout restart deployment -n gitlab-system
```

### Disabling SSO (Emergency)
```bash
# Remove OmniAuth configuration from gitlab.yaml
# Set: omniauth.enabled: false
# This will remove the SSO button but keep existing users
```

## Additional Resources

- [GitLab OmniAuth Documentation](https://docs.gitlab.com/ee/integration/omniauth.html)
- [Keycloak OIDC Documentation](https://www.keycloak.org/docs/latest/server_admin/#_oidc)
- [GitLab OpenID Connect Documentation](https://docs.gitlab.com/ee/administration/auth/oidc.html)

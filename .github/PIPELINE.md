# 🚀 CI/CD Pipeline — Documentation

## Architecture: 3 independent workflows

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  pipeline.yml          │  release.yml             │  deploy.yml             │
│  🔨 CI                 │  🚀 Release              │  🔧 Deploy Manual       │
├────────────────────────┼──────────────────────────┼─────────────────────────┤
│  push → main           │  tag v*.*.*              │  workflow_dispatch      │
│  PR → main             │                          │  (manual)               │
└────────────────────────┴──────────────────────────┴─────────────────────────┘
```

---

## 1 · `pipeline.yml` — 🔨 CI

**Trigger:** push to `main` or PR against `main`

```
push / PR
  └─▶ 🔨 Compile & Test
```

**Steps:**
1. Checkout
2. Context info (PR vs direct push)
3. Simulated build
4. Simulated tests (42 unit + 8 integration)
5. Step Summary

---

## 2 · `release.yml` — 🚀 Release

**Trigger:** tag push matching `v[0-9]+.[0-9]+.[0-9]+`

```
git tag v1.2.3
git push origin v1.2.3
       │
       ▼
┌─────────────────┐
│ 🐳 build-push   │  build image + push to ACR
└─────────────────┘
```

**Steps:**
1. Extract tag metadata
2. Simulated production build
3. Simulated ACR login
4. Simulated `docker build`
5. Simulated `docker push` (tag + `latest`)
6. Step Summary with image info

> To connect a real ACR, replace the steps with `azure/docker-login@v2`
> and add the secrets `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`.

### How to create a release

```bash
git tag v1.2.3
git push origin v1.2.3
```

---

## 3 · `deploy.yml` — 🔧 Deploy Manual

**Trigger:** `workflow_dispatch` (Actions → Run workflow)

**Use it for:**
- Deploying a published tag to a specific environment
- Rolling back to a previous version
- Emergency deployments
- Targeted environment testing

**How to use:**
1. Actions → **Deploy Manual** → **Run workflow**
2. **"Use workflow from"** → **Tag** → select the tag (e.g. `v1.2.3`)
3. Select the **target environment**
4. Click **Run workflow**

The tag comes from `github.ref_name` — GitHub populates it automatically
from the "Use workflow from" selection. No text input needed.

**Inputs:**
| Input | Type | Description |
|-------|------|-------------|
| `environment` | **environment** | Dropdown with the repo's configured environments |

**Jobs:**
1. `validate-tag` → verifies the workflow was triggered from a tag (not a branch)
2. `deploy` → deploys to the selected environment, respecting its protection rules

---

## Configure GitHub Environments

For environment gates to work, configure them at:

**Settings → Environments → New environment**

### Recommended environments

| Environment | Protection rules |
|-------------|-----------------|
| `development` | None (automatic deploy) |
| `staging` | Wait timer: 5 min |
| `production` | Required reviewers: 1+ approver |

### How to add protection rules

1. Go to **Settings → Environments**
2. Select the environment
3. Under **Deployment protection rules**:
   - ✅ **Required reviewers** → add users/teams as approvers
   - ✅ **Wait timer** → minutes to wait before executing
   - ✅ **Restrict deployments to protected branches** → only from `main`/tags

---

## Variables to customize

In `release.yml` and `deploy.yml`, update:

```yaml
env:
  ACR_NAME: myregistry    # ← your Azure Container Registry name
  IMAGE_NAME: my-app      # ← your Docker image name
```

## Recommended secrets

Add under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | Service Principal for Azure login |
| `AZURE_CLIENT_SECRET` | Service Principal credentials |
| `AZURE_TENANT_ID` | Azure AD tenant |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription |

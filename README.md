# flow

A proof-of-concept CI/CD pipeline using **GitHub Actions**, demonstrating a clean separation of concerns across three independent workflows: continuous integration, image release, and manual deployment.

---

## Workflows

```mermaid
flowchart LR
    subgraph ci["🔨 pipeline.yml"]
        A([push / PR → main]) --> B[Compile & Test]
    end

    subgraph release["🚀 release.yml"]
        C([workflow_dispatch\ntag input]) --> D[Build] --> E[Push → ACR] --> F[Create tag]
    end

    subgraph deploy["🔧 deploy.yml"]
        G([workflow_dispatch\nUse workflow from: tag]) --> H[Validate tag] --> I[Deploy] --> J[Health check]
    end
```

### 🔨 CI (`pipeline.yml`)
Runs on every push to `main` and on pull requests. Compiles the project and executes the test suite.

### 🚀 Release (`release.yml`)
Triggered manually from the Actions tab. Enter the tag name (e.g. `v1.2.3`), select any branch as the source, and the workflow builds the Docker image, pushes it to ACR, and **only if everything succeeds** creates the git tag. If the build fails, no tag is left behind.

### 🔧 Deploy Manual (`deploy.yml`)
Triggered manually from the Actions tab. Select the tag via **"Use workflow from"** and choose the target environment from the dropdown. Supports environment protection rules (approvals, wait timers).

---

## Flow

```mermaid
flowchart TD
    push(["push to main / PR"]) --> ci["🔨 Compile & Test"]

    dispatch(["Actions → Release\nBranch: main\nTag input: v1.2.3"]) --> build["🐳 Build Docker image"]
    build --> acr["📤 Push to ACR\nmyregistry.azurecr.io/my-app:v1.2.3"]
    acr --> createtag["🏷️ Create git tag v1.2.3"]

    build -- failure --> stop(["❌ Stop — no tag created"])

    createtag -. image ready .-> manual

    manual(["Actions → Deploy Manual\nUse workflow from: Tag v1.2.3"]) --> env["🌍 Select environment"]
    env --> validate["✅ Validate tag"]
    validate --> dep["🚀 Deploy"]
    dep --> health["🏥 Health check"]
```

---

## Getting started

### 1. Create a release

1. Go to **Actions → Release → Run workflow**
2. **"Use workflow from"** → select a branch (e.g. `main`)
3. Enter the **tag name** in the input (e.g. `v1.2.3`)
4. Click **Run workflow**

The `release.yml` workflow builds the image, pushes it to ACR, and only then creates the git tag. If anything fails, no tag is left behind.

### 2. Deploy to an environment

1. Go to **Actions → Deploy Manual → Run workflow**
2. Under **"Use workflow from"**, select **Tag** → pick the tag (e.g. `v1.2.3`)
3. Select the **target environment**
4. Click **Run workflow**

---

## Environments

Configure environments under **Settings → Environments**.

| Environment | Recommended protection rules |
|-------------|------------------------------|
| `development` | None — deploys automatically |
| `staging` | Wait timer: 5 min |
| `production` | Required reviewers: 1+ approver |

---

## Configuration

Update the global variables in `release.yml` and `deploy.yml`:

```yaml
env:
  ACR_NAME: myregistry   # Azure Container Registry name
  IMAGE_NAME: my-app     # Docker image name
```

Add the following secrets under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | Service Principal for Azure login |
| `AZURE_CLIENT_SECRET` | Service Principal credentials |
| `AZURE_TENANT_ID` | Azure AD tenant |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription |

---

## Documentation

See [`.github/PIPELINE.md`](.github/PIPELINE.md) for full pipeline documentation.



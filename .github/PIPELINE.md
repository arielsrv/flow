# 🚀 CI/CD Pipeline — Documentación

## Arquitectura: 3 workflows independientes

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

**Trigger:** push a `main` o PR contra `main`

```
push/PR
  └─▶ 🔨 Compile & Test
```

**Pasos:**
1. Checkout
2. Info del contexto (PR vs push directo)
3. Compilación simulada
4. Tests simulados (42 unit + 8 integration)
5. Step Summary

---

## 2 · `release.yml` — 🚀 Release (estilo GitLab)

**Trigger:** push de tag `v[0-9]+.[0-9]+.[0-9]+`

```
git tag v1.2.3
git push origin v1.2.3
       │
       ▼
┌─────────────────┐
│ 🐳 build-push   │  build + push imagen al ACR
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 🟢 development  │  deploy automático (sin gates)
└────────┬────────┘
         │  (gate: protection rules de GitHub Environment)
         ▼
┌─────────────────┐
│ 🟡 staging      │  deploy con gate opcional (wait timer / approval)
└────────┬────────┘
         │  (gate: protection rules de GitHub Environment)
         ▼
┌─────────────────┐
│ 🔴 production   │  deploy con gate (required reviewers)
└─────────────────┘
```

**Sin intervención manual** — el tag dispara todo el pipeline.
Los gates entre entornos se configuran en **Settings → Environments**.

### Cómo crear un release

```bash
git tag v1.2.3
git push origin v1.2.3
```

---

## 3 · `deploy.yml` — 🔧 Deploy Manual

**Trigger:** `workflow_dispatch` (Actions → Run workflow)

**Úsalo para:**
- Re-deploy de un tag ya publicado
- Rollback a una versión anterior
- Deploy de emergencia a un entorno específico
- Pruebas puntuales

**Inputs:**
| Input | Tipo | Descripción |
|-------|------|-------------|
| `tag` | string | Tag a desplegar — los últimos 10 se listan en el primer job |
| `environment` | **environment** | Selector con los environments del repo |

**Jobs:**
1. `discover-tags` → lista últimos 10 tags, valida el ingresado
2. `discover-environments` → lista environments del repo
3. `deploy` → valida entorno, despliega, health check

---

## Configurar GitHub Environments

Para que los gates del `release.yml` funcionen, configurá los environments en:

**Settings → Environments → New environment**

### Environments recomendados

| Environment | Protection rules |
|-------------|-----------------|
| `development` | Ninguna (deploy automático) |
| `staging` | Wait timer: 5 min |
| `production` | Required reviewers: 1+ aprobador |

### Cómo agregar protection rules

1. Ir a **Settings → Environments**
2. Seleccionar el environment
3. En **Deployment protection rules**:
   - ✅ **Required reviewers** → agregar usuarios/teams aprobadores
   - ✅ **Wait timer** → minutos de espera antes de ejecutar
   - ✅ **Restrict deployments to protected branches** → solo desde `main`/tags

---

## Variables a personalizar

En `release.yml` y `deploy.yml`, ajustar:

```yaml
env:
  ACR_NAME: myregistry    # ← nombre de tu Azure Container Registry
  IMAGE_NAME: my-app      # ← nombre de la imagen Docker
```

## Secrets recomendados

Agregar en **Settings → Secrets and variables → Actions**:

| Secret | Descripción |
|--------|-------------|
| `AZURE_CLIENT_ID` | Service Principal para login en Azure |
| `AZURE_CLIENT_SECRET` | Credencial del Service Principal |
| `AZURE_TENANT_ID` | Tenant de Azure AD |
| `AZURE_SUBSCRIPTION_ID` | Suscripción de Azure |

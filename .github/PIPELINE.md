# 🚀 CI/CD Pipeline — Documentación

## Diagrama de flujo

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TRIGGERS                                      │
├──────────────────┬──────────────────┬───────────────────────────────┤
│  push → main     │  PR → main       │  tag v*.*.*                   │
│  pull_request    │                  │  workflow_dispatch (manual)    │
└────────┬─────────┴────────┬─────────┴──────────────┬────────────────┘
         │                  │                          │
         ▼                  ▼                          ▼
  ┌─────────────┐    ┌─────────────┐          ┌──────────────────────┐
  │ 🔨 Compile  │    │ 🔨 Compile  │          │ 🐳 Build & Push ACR  │
  │ 🧪 Test     │    │ 🧪 Test     │          │   (simulado)         │
  └─────────────┘    └─────────────┘          └──────────┬───────────┘
                                                          │
                                              ┌───────────▼───────────┐
                                              │  📌 Imagen disponible  │
                                              │  en ACR con tag v*.*.*  │
                                              └───────────────────────┘

  ─────────────────── DEPLOY MANUAL (workflow_dispatch) ──────────────

  Inputs requeridos:
    • tag         → ej: v1.2.3  (podés ver los disponibles en el primer job)
    • environment → elegido del selector (auto-descubre los del repo)

  ┌──────────────────────┐     ┌──────────────────────┐     ┌──────────────────────────────────────┐
  │ 🏷️ Discover Tags     │     │ 🔍 Discover Envs     │────▶│ 🚀 Deploy → [environment]            │
  │ (últimos 10, GitHub  │     │ (GitHub API)         │     │  ├─ 🔐 Autenticación Azure           │
  │  API, valida input)  │     └──────────────────────┘     │  ├─ 📥 Pull imagen ACR               │
  └──────────────────────┘                                   │  ├─ 🚀 Deploy Container App         │
                                                             │  └─ 🏥 Health check                 │
                                                             └──────────────────────────────────────┘
```

---

## Jobs

### 1 · 🔨 `ci` — Compile & Test

**Se ejecuta cuando:**
- Push directo a `main`
- Pull Request cuyo destino es `main`

**Pasos:**
1. Checkout del código
2. Info del contexto (PR vs push directo)
3. Compilación simulada
4. Ejecución de tests simulada (42 unit + 8 integration)
5. Resumen en GitHub Step Summary

---

### 2 · 🐳 `build-and-push` — Build & Push a ACR

**Se ejecuta cuando:**
- Se crea un tag con el patrón `v[0-9]+.[0-9]+.[0-9]+` (ej: `v1.2.3`)

**Cómo crear un tag:**
```bash
git tag v1.2.3
git push origin v1.2.3
```

**Pasos:**
1. Extraer metadata del tag
2. Build de producción simulado
3. Login en ACR simulado
4. `docker build` simulado
5. `docker push` simulado (tag + `latest`)
6. Resumen con info de la imagen publicada

> Para conectar el ACR real, reemplazar los pasos con `azure/docker-login@v2`
> y agregar los secrets `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`.

---

### 3 · 🚀 `deploy` — Deploy Manual

**Se ejecuta cuando:**
- Se dispara manualmente desde **Actions → Run workflow**

**Inputs:**
| Input | Tipo | Descripción |
|-------|------|-------------|
| `tag` | string | Tag de la imagen a desplegar (ej: `v1.2.3`) — los últimos 10 se listan en el primer job |
| `environment` | **environment** | Selector que muestra los environments del repo |

> ℹ️ El input `type: environment` hace que GitHub muestre automáticamente
> un dropdown con todos los environments configurados en **Settings → Environments**.

> ℹ️ El input `tag` es texto libre. Como GitHub Actions **no soporta dropdowns dinámicos**
> en `workflow_dispatch`, el job `discover-tags` consulta la API al inicio y muestra los
> últimos 10 tags en el **Step Summary**, resaltando el que ingresaste. También valida
> que exista y avisa si hay un posible error tipográfico.

**Pasos:**
1. `discover-tags` → consulta la GitHub API, lista los últimos 10 tags, valida que el tag ingresado exista y lo resalta en el Step Summary
2. `discover-environments` → consulta la GitHub API y lista los entornos disponibles
3. `deploy` → valida el entorno, despliega y hace health check
   - Si el environment tiene **protection rules** (required reviewers, wait timer), GitHub
     pausará el job hasta que alguien apruebe antes de continuar.

---

## Configurar GitHub Environments

Para que el deploy manual funcione correctamente, configurá los environments en:

**Settings → Environments → New environment**

### Environments sugeridos

| Environment | Protection rules recomendadas |
|-------------|-------------------------------|
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

En el archivo `pipeline.yml`, ajustar las variables globales:

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


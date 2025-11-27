# CI/CD Flow Documentation

Este documento explica el flujo completo del sistema de CI/CD centralizado.

## Flujo Completo

### 1. Feature Branch → PR a `develop`

**Trigger**: Pull Request abierto/actualizado contra `develop`

**Workflow**: `pull-request-events.yml` → `shared-pull-request-events.yml`

**Validaciones**:
- ✅ Conventional Commit Check (título del PR)
- ✅ Linter (según lenguaje)
- ✅ Security Scan (CodeQL, Dependabot, Trivy)
- ✅ Unit Tests
- ✅ Coverage Validation (threshold: 80%)

**Status Check**: `pull-request-events / workflow-summary`

**Resultado**: Si todas las validaciones pasan, el PR puede ser mergeado a `develop`

---

### 2. `develop` (Push después del merge)

**Trigger**: Push a la rama `develop` (después del merge del PR)

**Workflow**: `develop-push-events.yml` → `shared-develop-push-events.yml`

**Proceso**:
1. Security Scan (validación adicional)
2. Build & Tag Docker image:
   - Tag: `{commit-sha-short}` (ej: `a1b2c3d`)
   - Tag: `develop-latest`
3. Deploy to Dev (simulado)
4. Deploy Verification
5. Health Check (espera 3 min + verificación)

**Resultado**: Código desplegado en ambiente de desarrollo

---

### 3. `develop` → PR a `main`

**Trigger**: Pull Request abierto/actualizado contra `main` (desde `develop`)

**Workflow**: `pull-request-events.yml` → `shared-pull-request-events.yml`

**Validaciones** (mismas que para PRs a develop):
- ✅ Conventional Commit Check
- ✅ Linter
- ✅ Security Scan
- ✅ Unit Tests
- ✅ Coverage Validation

**Status Check**: `pull-request-events / workflow-summary`

**Resultado**: Si todas las validaciones pasan, el PR puede ser mergeado a `main`

**Nota**: Este es el paso crítico que valida que el código en `develop` está listo para producción.

---

### 4. `main` (Push después del merge)

**Trigger**: Push a la rama `main` (después del merge del PR desde `develop`)

**Workflow**: `main-push-events.yml` → `shared-main-push-events.yml`

**Proceso**:

#### 4.1. Release & Tag
- Ejecuta `release-please-action` que:
  - Analiza los commits desde el último release
  - Genera un changelog automáticamente
  - Crea un nuevo tag semántico (ej: `v1.2.3`) basado en conventional commits
  - Crea un GitHub Release con el tag y changelog

#### 4.2. Build & Push
- Build Docker image con tags:
  - `{version}` (ej: `v1.2.3`)
  - `{version}-stg` (ej: `v1.2.3-stg`)
  - `{version}-prd` (ej: `v1.2.3-prd`)
- Push a ECR

#### 4.3. Deploy to Staging
- Deploy a staging environment usando el tag `{version}-stg`
- Deploy Verification
- Health Check

#### 4.4. Staging Validation
- Valida métricas en staging (error rate, latency, APM)
- Consulta Datadog para verificar salud del servicio
- Si falla, se documenta el rollback

#### 4.5. Manual Approval (PCI Compliance)
- **Requiere aprobación manual** para deploy a producción
- Se configura en GitHub Environment `prd`
- Solo después de la aprobación se procede con producción

#### 4.6. Deploy to Production (Opcional)
- Solo se ejecuta si se aprueba manualmente
- Deploy a producción usando el tag `{version}-prd`
- Deploy Verification
- Health Check
- Rollback mechanism disponible si falla

**Resultado**: 
- Tag creado: `v1.2.3`
- Imagen en ECR con tags: `v1.2.3`, `v1.2.3-stg`, `v1.2.3-prd`
- Desplegado en staging
- Listo para promover a producción (requiere aprobación)

---

## Promoción a Producción

Una vez que el código está en `main` y desplegado en staging:

1. **Validación en Staging**: El equipo valida que todo funciona correctamente
2. **Aprobación Manual**: Se aprueba el deploy a producción desde GitHub
3. **Deploy a Prod**: Se ejecuta automáticamente usando el tag `{version}-prd`
4. **Verificación**: Health checks y validaciones post-deploy

**Nota**: El mismo build/tag que se usó en staging se promueve a producción. No se hace un nuevo build.

---

## Diagrama de Flujo

```
Feature Branch
    │
    ├─→ PR a develop ──→ [Validaciones] ──→ ✅ Merge
    │                                           │
    │                                           ▼
    │                                    develop (push)
    │                                           │
    │                                           ├─→ Security Scan
    │                                           ├─→ Build & Tag (SHA, develop-latest)
    │                                           ├─→ Deploy Dev
    │                                           └─→ Health Check
    │
    └─→ PR a main (desde develop) ──→ [Validaciones] ──→ ✅ Merge
                                                              │
                                                              ▼
                                                         main (push)
                                                              │
                                                              ├─→ Create Tag (v1.2.3)
                                                              ├─→ Build & Tag (v1.2.3, v1.2.3-stg, v1.2.3-prd)
                                                              ├─→ Deploy Staging (v1.2.3-stg)
                                                              ├─→ Staging Validation
                                                              ├─→ ⚠️ Manual Approval
                                                              └─→ Deploy Production (v1.2.3-prd)
```

---

## Puntos Clave

1. **develop es la rama de integración**: Aquí se integran todas las features
2. **main es la rama de producción**: Solo código validado y probado en dev
3. **Tags semánticos**: Se crean automáticamente en cada merge a main
4. **Mismo build para staging y prod**: El tag `{version}-prd` es el mismo código que `{version}-stg`
5. **Aprobación manual**: Requerida para producción (PCI compliance)
6. **Validaciones en cada paso**: No se puede avanzar sin pasar todas las validaciones

---

## Branch Protection

- **develop**: Requiere 1 reviewer + `workflow-summary` status check
- **main**: Requiere 2 reviewers + `workflow-summary` status check

Ambas ramas usan el mismo status check (`workflow-summary`) que valida todas las pruebas del PR.


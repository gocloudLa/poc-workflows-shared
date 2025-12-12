# Sistema CI/CD Centralizado

Sistema de workflows de CI/CD centralizados y reutilizables para estandarizar pipelines en múltiples servicios y lenguajes de programación (PHP, Python, JavaScript/Node.js, Go) en un entorno enterprise con compliance PCI e ISO27001.

## Características Principales

- ✅ **Pipelines estandarizados** para todos los servicios
- ✅ **Soporte multi-lenguaje**: PHP, Python, JavaScript, Go
- ✅ **Compliance PCI/ISO27001**: Security scans, aprobaciones manuales, auditoría
- ✅ **Mecanismos de bypass**: Permite a equipos saltar o extender workflows
- ✅ **Versionado automático**: Release-please con semantic versioning
- ✅ **Rollback mechanisms**: Documentados y simulados

## Arquitectura

### Estructura de Workflows

```
.github/workflows/
  # Workflows Orquestadores
  shared-pull-request-php.yml        # Validaciones en PRs para PHP
  shared-pull-request-python.yml     # Validaciones en PRs para Python
  shared-develop-push-events-php.yml    # Deploy a dev para PHP
  shared-develop-push-events-python.yml # Deploy a dev para Python
  shared-main-push-events-php.yml       # Release y deploy a staging/prod para PHP
  shared-main-push-events-python.yml    # Release y deploy a staging/prod para Python
  
  # Workflows de Jobs Individuales
  conventional-commit-check.yml       # Validación de conventional commits
  linter-php.yml
  linter-python.yml
  security-scan-php.yml               # CodeQL PHP, Dependabot, Trivy
  security-scan-python.yml            # CodeQL Python, Dependabot, Trivy
  unit-tests-php.yml                  # Tests unitarios PHP
  unit-tests-python.yml               # Tests unitarios Python
  coverage-validation-php.yml         # Validación de cobertura PHP
  coverage-validation-python.yml      # Validación de cobertura Python
  build-and-push.yml                  # Build Docker y push a ECR
  deploy-dev.yml
  deploy-staging.yml
  deploy-production.yml
  release-and-changelog.yml           # Release y changelog con release-please
```

### Flujo Completo

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
                                                              ├─→ Release & Tag (v1.2.3)
                                                              ├─→ Build & Tag (v1.2.3, v1.2.3-stg, v1.2.3-prd)
                                                              ├─→ Deploy Staging (v1.2.3-stg)
                                                              ├─→ Staging Validation
                                                              ├─→ ⚠️ Manual Approval
                                                              └─→ Deploy Production (v1.2.3-prd)
```

## Configuración Inicial

### 1. GitHub Secrets

En cada repositorio de servicio, configurar:

- `ECR_REPOSITORY_URL`: URL del repositorio ECR
- `AWS_ROLE_ARN_SHARED`: ARN del rol IAM para acceso a ECR
- `AWS_ROLE_ARN_DEV`: ARN del rol IAM para deploy a dev
- `AWS_ROLE_ARN_STAGING`: ARN del rol IAM para deploy a staging
- `AWS_ROLE_ARN_PROD`: ARN del rol IAM para deploy a producción
- `AWS_REGION`: Región de AWS (ej: `us-east-2`)

### 2. GitHub Environments

**dev Environment**: Sin protection rules (deploy automático)

**staging Environment**: Sin protection rules (deploy automático)

**prd Environment**:
- **Required reviewers**: Agregar equipo de seguridad/desarrollo
- **Deployment branches**: Solo `main`

### 3. Branch Protection Rules

**develop Branch**:
- Require pull request reviews: 1 reviewer
- Require status checks: `pull-request-events / workflow-summary`
- Require branches to be up to date: Yes

**main Branch**:
- Require pull request reviews: 2 reviewers
- Require status checks: `pull-request-events / workflow-summary`
- Require branches to be up to date: Yes
- Require linear history: Yes

### 4. AWS IAM Trust Policy

Cada rol IAM debe tener un trust policy que permita a GitHub Actions asumirlo:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:*"
      }
    }
  }]
}
```

## Uso

### Crear Workflows en Repositorio de Servicio

Crear tres archivos en `.github/workflows/`:

#### `pull-request-events.yml`

```yaml
name: "Pull Request Events Workflow"

on:
  pull_request:
    branches: [develop, main]
    types: [opened, edited, reopened, synchronize, ready_for_review]

permissions:
  pull-requests: write
  id-token: write
  contents: read

jobs:
  pull-request-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-pull-request-php.yml@main
    # Para Python usar: shared-pull-request-python.yml
    with:
      language: php  # o python, javascript, go
    secrets: inherit
```

#### `develop-push-events.yml`

```yaml
name: "Develop Push Events Workflow"

on:
  push:
    branches: [develop]

permissions:
  pull-requests: write
  id-token: write
  contents: read

jobs:
  develop-push-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-develop-push-events-php.yml@main
    # Para Python usar: shared-develop-push-events-python.yml
    with:
      ecr_repository_url: ${{ secrets.ECR_REPOSITORY_URL }}
      aws_role_arn_shared: ${{ secrets.AWS_ROLE_ARN_SHARED }}
      aws_role_arn_dev: ${{ secrets.AWS_ROLE_ARN_DEV }}
      aws_region: ${{ secrets.AWS_REGION || 'us-east-1' }}
      service_name: your-service-name
    secrets: inherit
```

#### `main-push-events.yml`

```yaml
name: "Main Push Events Workflow"

on:
  push:
    branches: [main]

permissions:
  pull-requests: write
  id-token: write
  contents: write

jobs:
  main-push-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-main-push-events-php.yml@main
    # Para Python usar: shared-main-push-events-python.yml
    with:
      ecr_repository_url: ${{ secrets.ECR_REPOSITORY_URL }}
      aws_role_arn_shared: ${{ secrets.AWS_ROLE_ARN_SHARED }}
      aws_role_arn_staging: ${{ secrets.AWS_ROLE_ARN_STAGING }}
      aws_role_arn_prod: ${{ secrets.AWS_ROLE_ARN_PROD }}
      aws_region: ${{ secrets.AWS_REGION || 'us-east-1' }}
      service_name: your-service-name
    secrets: inherit
```

## Flujos de Trabajo

### Feature Branch → PR a `develop`

**Validaciones**:
1. Conventional Commit Check
2. Linter (específico del lenguaje)
3. Security Scan (CodeQL, Dependabot, Trivy)
4. Unit Tests
5. Coverage Validation (>= 80%)

**Status Check**: `pull-request-events / workflow-summary`

### `develop` (Push después del merge)

**Proceso**:
1. Security Scan
2. Build & Tag: `{commit-sha-short}`, `develop-latest`
3. Deploy to Dev
4. Health Check

### `develop` → PR a `main`

Mismas validaciones que PRs a develop.

### `main` (Push después del merge)

**Proceso**:
1. **Release & Tag**: Release-please crea tag semántico (v1.2.3) y changelog
2. **Build & Tag**: `{version}`, `{version}-stg`, `{version}-prd`
3. **Deploy to Staging**: Usando tag `{version}-stg`
4. **Staging Validation**: Valida métricas (error rate, latency, APM)
5. **Manual Approval**: Requerido para producción (PCI compliance)
6. **Deploy to Production**: Usando tag `{version}-prd` (solo si se aprueba)

## Bypassing Jobs

Todos los workflows soportan flags `skip_*` para saltar jobs específicos:

```yaml
with:
  language: php
  skip_linter: true
  skip_coverage_validation: true
```

**Flags disponibles**:
- `skip_conventional_commit`
- `skip_linter`
- `skip_security_scan`
- `skip_unit_tests`
- `skip_coverage_validation`
- `skip_build`
- `skip_deploy`
- `skip_release`

**⚠️ Advertencia**: Los bypasses deben ser justificados y documentados para auditoría.

## Procesos Operativos

### Revert en develop

Si necesitas deshacer un PR mergeado en `develop`:

```bash
git revert <commit-sha>
git push origin feature/revert-commit
```

Crear PR con el revert. Pasará por las mismas validaciones y al mergear se deployará la versión revertida a dev.

### Rollback desde Staging/Production

1. Identificar la versión anterior estable
2. Usar el mecanismo de rollback del workflow
3. O crear hotfix desde `main`

### Hotfix

Para fixes críticos que necesitan ir directo a producción:

1. Crear branch desde `main`:
   ```bash
   git checkout main
   git checkout -b hotfix/fix-critico
   ```

2. Hacer el fix y commit con conventional commit:
   ```bash
   git commit -m "fix: corregir bug crítico en producción"
   ```

3. Crear PR contra `main`
4. Merge a `main` → Release automático → Deploy a staging → Aprobación → Deploy a prod
5. Merge el hotfix también a `develop` para mantener sincronización

## Troubleshooting

### Workflow Falla en Language Detection

**Solución**: Especificar el lenguaje explícitamente:

```yaml
with:
  language: php
```

### Build Falla por Dockerfile No Encontrado

**Solución**: Asegurar que el Dockerfile esté en la raíz del repositorio.

### Security Scan Encuentra Vulnerabilidades

**Solución**: 
1. Revisar el GitHub Security tab
2. Actualizar dependencias vulnerables
3. Si es un falso positivo, documentar y considerar bypass temporal

### Coverage Validation Falla

**Solución**:
1. Aumentar cobertura de tests
2. O ajustar threshold temporalmente (no recomendado):
   ```yaml
   with:
     coverage_threshold: 70
   ```

### Release-Please No Crea Release

**Solución**:
- Verificar que los commits usen conventional commits
- Verificar que release-please tenga permisos de `contents: write`
- Revisar logs del workflow para errores

## Best Practices

### Conventional Commits

Siempre usar conventional commits:
- `feat:` Nueva funcionalidad
- `fix:` Corrección de bug
- `docs:` Cambios en documentación
- `chore:` Cambios en build, dependencias, etc.

### Versionado

- **Patch** (1.2.3 → 1.2.4): Fixes, patches
- **Minor** (1.2.3 → 1.3.0): Nuevas features (backward compatible)
- **Major** (1.2.3 → 2.0.0): Breaking changes

Release-please determina automáticamente el tipo de versión basado en conventional commits.

### Seguridad

- ✅ Nunca commitear secrets
- ✅ Usar GitHub Secrets/Environments
- ✅ Rotar secrets regularmente
- ✅ Revisar security scans regularmente
- ✅ Mantener dependencias actualizadas

### Testing

- ✅ Escribir tests para todo código nuevo
- ✅ Mantener cobertura >= 80%
- ✅ Tests deben ser rápidos y determinísticos

### Deployment

- ✅ Validar en staging antes de producción
- ✅ Monitorear métricas post-deploy
- ✅ Tener plan de rollback listo
- ✅ Documentar deployments importantes

## Referencias

### Documentación Oficial

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [AWS OIDC](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [Release-Please](https://github.com/google-github-actions/release-please-action)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)

### Herramientas

- [CodeQL](https://codeql.github.com/docs/)
- [Dependabot](https://docs.github.com/en/code-security/dependabot)
- [Trivy](https://aquasecurity.github.io/trivy/)

---

**Última actualización**: 2024

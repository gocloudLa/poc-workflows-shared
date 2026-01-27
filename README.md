# Sistema CI/CD Centralizado (POC)

Repositorio de workflows reutilizables para estandarizar CI/CD en servicios Python y PHP. Centraliza validaciones, build/deploy y aprobaciones, y aplica GitOps sobre el repo de valores de ArgoCD.

## Alcance

- Orquestacion de PRs, deploy a dev, release a staging y paso controlado a prod.
- Reutilizacion por repo de servicio (Python/PHP) con minima configuracion local.
- Integracion con GitHub App para bypass controlado de reglas en automatizaciones.

## Estructura principal

```
.github/workflows/
  shared-pull-request-php.yml
  shared-pull-request-python.yml
  shared-develop-push-events-php.yml
  shared-develop-push-events-python.yml
  shared-main-push-events-php.yml
  shared-main-push-events-python.yml
  build-and-push.yml
  deploy.yml
  deploy-verification.yml
  health-check.yml
  release-and-changelog.yml
  create-release.yml
  rollback-mechanism.yml
  security-scan-php.yml
  security-scan-python.yml
  linter-php.yml
  linter-python.yml
  unit-tests-php.yml
  unit-tests-python.yml
  coverage-validation-php.yml
  coverage-validation-python.yml
  staging-validation.yml
```

## Flujo funcional (resumen)

1. PR a `develop`: conventional commit -> linter -> security scan -> unit tests -> coverage.
2. Merge a `develop`: build/push -> deploy dev -> verification -> health check.
3. Merge a `main`: semantic-release -> build/push -> deploy staging -> validaciones -> rollback si falla.
4. Release manual: `create-release.yml` (aprobacion) -> actualiza `values.prd.yml`.
5. Rollback manual: `rollback-mechanism.yml` -> valida tag/release -> actualiza `values.prd.yml`.

## Decisiones (best practices)

- Workflows reutilizables para reducir drift entre servicios y lenguajes.
- Outputs explicitos (tags/version) para evitar `image.tag` vacio.
- Defaults en inputs criticos (`values_dir`, `environment`) para evitar `values..yml`.
- Environments protegidos (`release`, `prd`) para aprobaciones.
- Merge commit forzado en PR de changelog para evitar squash cuando el actor tiene bypass.
- GitOps para trazabilidad y auditoria en prod.

## Configuracion esperada en repos de servicio

Secrets:
- `ECR_REPOSITORY_URL`
- `AWS_ROLE_ARN_SHARED`
- `AWS_ROLE_ARN_DEV`
- `AWS_ROLE_ARN_STAGING`
- `AWS_ROLE_ARN_PROD`
- `AWS_REGION` (opcional, default `us-east-1`)

Environments:
- `dev` sin aprobacion.
- `staging` sin aprobacion.
- `prd` con aprobacion.
- `release` con aprobacion para crear releases.

Reglas GitHub (rulesets):
- `develop`: PR requerido, checks requeridos, merge strategy tipo squash.
- `main`: PR requerido, checks requeridos, merge commit.
- GitHub App en bypass controlado para automatizaciones.

## Soporte de lenguajes

- PHP (default 8.2, configurable en inputs).
- Python (default 3.11, configurable en inputs).

Cobertura:
- Minimo 80% (configurable via `coverage_threshold`).
- Reporte esperado en `coverage.xml`.

## Ejemplos de workflows caller (repo de servicio)

`pull-request-events.yml`
```yaml
name: "Pull Request Events Workflow"
on:
  pull_request:
    branches: [develop, main]
    types: [opened, edited, reopened, synchronize, ready_for_review]
  merge_group:
permissions:
  pull-requests: write
  id-token: write
  contents: read
  actions: read
  security-events: write
jobs:
  pull-request-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-pull-request-python.yml@main
    secrets: inherit
```

`develop-push-events.yml`
```yaml
name: "Develop Push Events Workflow"
on:
  push:
    branches: [develop]
permissions:
  pull-requests: write
  id-token: write
  contents: write
  actions: read
  security-events: write
jobs:
  develop-push-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-develop-push-events-python.yml@main
    with:
      service_name: python-service
    secrets: inherit
```

`main-push-events.yml`
```yaml
name: "Main Push Events Workflow"
on:
  push:
    branches: [main]
permissions:
  pull-requests: write
  id-token: write
  contents: write
  actions: read
  security-events: write
jobs:
  main-push-events:
    uses: gocloudLa/poc-workflows-shared/.github/workflows/shared-main-push-events-python.yml@main
    with:
      service_name: python-service
    secrets: inherit
```

## AWS IAM trust policy (referencia)

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

## Workflows manuales

- `create-release.yml`:
  - Input: `tag_name`, `release_notes` (opcional), `values_dir`.
  - Usa `environment: release` para aprobacion.
  - Actualiza `values.prd.yml` en el repo GitOps.

- `rollback-mechanism.yml`:
  - Inputs: `rollback_tag`, `values_dir`.
  - Valida que el tag exista y tenga release.
  - Actualiza `values.prd.yml` para volver a un tag previo.

## Pruebas realizadas (paso a paso)

1. PR a `develop` con validaciones obligatorias.
2. Merge a `develop` con deploy a dev y health check.
3. Merge a `main` con semantic-release, deploy a staging y validaciones.
4. `Create Release` con aprobacion -> update a prod en GitOps.
5. `Rollback` con tag valido -> rollback en prod.

## Soluciones aplicadas (prevencion)

- Outputs en workflows reutilizables para evitar tags vacios.
- Defaults en inputs para evitar errores de ruta y parametros vacios.
- Environments protegidos para evitar deploys no autorizados.
- Bypass controlado para que la automatizacion no rompa reglas.

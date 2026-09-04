# ci-templates

Workflows **reutilizables** de GitHub Actions (`workflow_call`) para los proyectos del
banco. Cada repo consumidor invoca uno de estos workflows y obtiene CI (build + tests +
SonarCloud + Quality Gate) y CD hacia el destino correspondiente.

> Versionado por tags. Referencia siempre una versión fija, p. ej.
> `uses: my-banking-app/ci-templates/.github/workflows/aws-java-ecs-ci-cd.yml@v1.3.4`.

## Workflows disponibles

| Workflow | Destino de despliegue | Stack |
|----------|-----------------------|-------|
| `java-ms-ci-cd.yml` | VPS (SSH + DockerHub) | Java / Spring Boot |
| `node-api-ci-cd.yml` | VPS (SSH + DockerHub) | Node |
| `node-ms-ci-cd.yml` | VPS (SSH + DockerHub) | Node / NestJS |
| `nest-db-ci-cd.yml` | VPS (SSH + DockerHub) | NestJS + Prisma |
| `vite-front-ci-cd.yml` | VPS (SSH + DockerHub) | React / Vite |
| `python-mocks.yml` | VPS (SSH + DockerHub) | Python |
| **`aws-java-ecs-ci-cd.yml`** | **AWS ECS Fargate + ECR** | **Java / Spring Boot** |
| **`aws-angular-s3-ci-cd.yml`** | **AWS S3 + CloudFront** | **Angular** |

Los dos últimos (AWS) se documentan en detalle abajo.

---

## `aws-java-ecs-ci-cd.yml` — Java → Amazon ECS Fargate

**En PR a `main`:** `mvn clean verify` (tests + JaCoCo) → SonarCloud → Quality Gate.
**En push/merge a `main`:** build de imagen → **Amazon ECR** → despliegue en **ECS Fargate**.

### Inputs

| Input | Req. | Default | Descripción |
|-------|------|---------|-------------|
| `service-name` | ✅ | — | Nombre legible |
| `ecr-repository` | ✅ | — | Repo en Amazon ECR |
| `ecs-cluster` | ✅ | — | Cluster de ECS |
| `ecs-service` | ✅ | — | Servicio de ECS |
| `ecs-task-definition` | | `task-definition.json` | Ruta del task def en el repo |
| `container-name` | ✅ | — | Nombre del contenedor en el task def |
| `aws-region` | | `us-east-1` | Región |
| `java-version` | | `21` | Versión de Java |
| `sonar-project-key` | ✅ | — | Project key de SonarCloud |
| `maven-command` | | `mvn -B clean verify` | Comando de build |

### Secrets

| Secret | Descripción |
|--------|-------------|
| `SONAR_TOKEN` | Token de SonarCloud |
| `AWS_ROLE_ARN` | ARN del rol IAM que GitHub asume vía OIDC |

## `aws-angular-s3-ci-cd.yml` — Angular → Amazon S3 + CloudFront

**En PR a `main`:** `npm ci` → tests con cobertura → build → SonarCloud → Quality Gate.
**En push/merge a `main`:** build de producción (inyectando `api-url`) → `aws s3 sync` →
invalidación de CloudFront.

### Inputs

| Input | Req. | Default | Descripción |
|-------|------|---------|-------------|
| `service-name` | ✅ | — | Nombre legible |
| `sonar-project-key` | ✅ | — | Project key de SonarCloud |
| `node-version` | | `22` | Versión de Node |
| `api-url` | ✅ | — | URL pública de la API (build-time) |
| `aws-region` | | `us-east-1` | Región |
| `s3-bucket` | ✅ | — | Bucket destino |
| `cloudfront-distribution-id` | | `''` | Distribución a invalidar |
| `dist-path` | | `dist/code-insight-ai-web/browser` | Carpeta a subir |
| `env-prod-file` | | `src/environments/environment.prod.ts` | Archivo donde se inyecta `api-url` |

### Secrets

| Secret | Descripción |
|--------|-------------|
| `SONAR_TOKEN` | Token de SonarCloud |
| `AWS_ROLE_ARN` | ARN del rol IAM que GitHub asume vía OIDC |

### Ejemplo de uso (repo consumidor)

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD
on:
  pull_request: { branches: [main] }
  push: { branches: [main] }
permissions:
  id-token: write   # OBLIGATORIO para OIDC
  contents: read
jobs:
  ci-cd:
    uses: my-banking-app/ci-templates/.github/workflows/aws-java-ecs-ci-cd.yml@v1.3.4
    with:
      service-name: code-insight-ai-api
      ecr-repository: code-insight-ai-api
      ecs-cluster: code-insight-ai
      ecs-service: code-insight-ai-api
      container-name: code-insight-ai-api
      sonar-project-key: camilobr89_code-insight-ai-api
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

> ⚠️ El repo consumidor **debe** declarar `permissions: id-token: write` a nivel de
> workflow; sin eso, la autenticación OIDC con AWS falla.
> Además, `ci-templates` debe ser **público** para poder invocarse desde repos de otra
> cuenta/organización.

---

## Configuración manual (AWS + GitHub)

Guía paso a paso en [`docs/aws-setup.md`](docs/aws-setup.md): creación del proveedor OIDC
y el rol IAM, ECR, RDS PostgreSQL, ECS Fargate, S3 + CloudFront, SonarCloud y los
secrets/variables de GitHub.

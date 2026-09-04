# Configuración AWS + GitHub — Code Insight AI

Guía paso a paso para dejar operativos los pipelines
`aws-java-ecs-ci-cd.yml` (API → ECS Fargate) y `aws-angular-s3-ci-cd.yml`
(Angular → S3 + CloudFront).

Sustituye estos valores en todos los comandos:

| Placeholder | Ejemplo | Qué es |
|-------------|---------|--------|
| `<ACCOUNT_ID>` | `123456789012` | ID de tu cuenta AWS (`aws sts get-caller-identity`) |
| `<REGION>` | `us-east-1` | Región donde crearás todo |
| `<GH_OWNER>` | `camilobr89` | Dueño de los repos de la app |

> Convención de nombres usada por los pipelines (puedes cambiarla en los `with:` del caller):
> cluster `code-insight-ai`, servicio/ECR/contenedor `code-insight-ai-api`,
> parámetro de contraseña `/code-insight-ai/db-password`.

---

## 0. Requisitos

- AWS CLI v2 configurado (`aws configure`) con el usuario IAM `code-insight-ai`.
- Los pasos de **IAM (§2)** requieren permisos para crear *OIDC provider* y *roles*.
  Si tu usuario `code-insight-ai` sólo tiene permisos de "crear recursos" pero no IAM,
  pídele a un administrador que ejecute la §2 (una sola vez).

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGION=us-east-1
export GH_OWNER=camilobr89
```

---

## 1. SonarCloud

1. Entra a <https://sonarcloud.io> con tu cuenta de GitHub.
2. Crea (o usa) una organización, p. ej. `camilobr89`.
3. Crea **dos proyectos** (Analyze new project → manualmente) con estas keys:
   - `camilobr89_code-insight-ai-api`
   - `camilobr89_code-insight-ai-web`
   - En cada uno, **New Code → Previous version** y desactiva "Automatic Analysis"
     (usamos el scanner del pipeline).
4. Genera un token: **My Account → Security → Generate Token**. Guárdalo para el
   secret `SONAR_TOKEN`.
5. Verifica que `sonar-project.properties` de cada repo tenga `sonar.organization` y
   `sonar.projectKey` correctos.

---

## 2. Autenticación OIDC (rol IAM que GitHub asume) — recomendado

Con OIDC, GitHub Actions obtiene credenciales **temporales**; no guardas llaves en GitHub.

### 2.1 Crear el proveedor OIDC (una vez por cuenta)

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

> Si ya existe, AWS responderá `EntityAlreadyExists` — puedes ignorarlo.

### 2.2 Trust policy del rol

Crea `trust-policy.json` (permite asumir el rol sólo desde tus dos repos):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:<GH_OWNER>/code-insight-ai-api:*",
            "repo:<GH_OWNER>/code-insight-ai-web:*"
          ]
        }
      }
    }
  ]
}
```

### 2.3 Crear el rol y su política de despliegue

```bash
aws iam create-role \
  --role-name code-insight-ai-deploy \
  --assume-role-policy-document file://trust-policy.json
```

Crea `deploy-policy.json` (permisos mínimos de despliegue):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "EcrAuth", "Effect": "Allow", "Action": "ecr:GetAuthorizationToken", "Resource": "*" },
    { "Sid": "EcrPushPull", "Effect": "Allow",
      "Action": ["ecr:BatchCheckLayerAvailability","ecr:CompleteLayerUpload","ecr:InitiateLayerUpload","ecr:PutImage","ecr:UploadLayerPart","ecr:BatchGetImage","ecr:GetDownloadUrlForLayer"],
      "Resource": "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/code-insight-ai-api" },
    { "Sid": "EcsDeploy", "Effect": "Allow",
      "Action": ["ecs:RegisterTaskDefinition","ecs:DeregisterTaskDefinition","ecs:DescribeTaskDefinition","ecs:UpdateService","ecs:DescribeServices","ecs:DescribeTasks","ecs:ListTasks"],
      "Resource": "*" },
    { "Sid": "PassExecRole", "Effect": "Allow", "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole" },
    { "Sid": "S3Deploy", "Effect": "Allow",
      "Action": ["s3:ListBucket","s3:PutObject","s3:DeleteObject","s3:GetObject"],
      "Resource": ["arn:aws:s3:::code-insight-ai-web-<ACCOUNT_ID>","arn:aws:s3:::code-insight-ai-web-<ACCOUNT_ID>/*"] },
    { "Sid": "CloudFrontInvalidate", "Effect": "Allow", "Action": "cloudfront:CreateInvalidation", "Resource": "*" }
  ]
}
```

```bash
aws iam put-role-policy \
  --role-name code-insight-ai-deploy \
  --policy-name deploy \
  --policy-document file://deploy-policy.json

# Guarda el ARN → será el secret AWS_ROLE_ARN
aws iam get-role --role-name code-insight-ai-deploy --query 'Role.Arn' --output text
```

> **Alternativa (menos segura):** si no puedes crear el rol, usa las *access keys* del
> usuario `code-insight-ai` como secrets `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
> En ese caso avísame y ajusto el paso `configure-aws-credentials` de los reusables
> (hoy están configurados sólo para OIDC).

---

## 3. Amazon ECR (registro de imágenes de la API)

```bash
aws ecr create-repository --repository-name code-insight-ai-api --region $REGION
```

---

## 4. Amazon RDS PostgreSQL (base de datos)

```bash
# Grupo de seguridad para la BD
aws ec2 create-security-group \
  --group-name code-insight-ai-db-sg \
  --description "Code Insight AI RDS" \
  --output text --query 'GroupId'
# -> guarda el valor como DB_SG

# Instancia (free-tier: db.t3.micro). Cambia la contraseña.
aws rds create-db-instance \
  --db-instance-identifier code-insight-ai-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --allocated-storage 20 \
  --db-name codeinsight \
  --master-username codeinsight \
  --master-user-password 'CAMBIA_ESTA_CLAVE' \
  --vpc-security-group-ids <DB_SG> \
  --no-publicly-accessible \
  --backup-retention-period 1
```

Cuando esté `available`, obtén el endpoint:

```bash
aws rds describe-db-instances --db-instance-identifier code-insight-ai-db \
  --query 'DBInstances[0].Endpoint.Address' --output text
# -> <RDS_ENDPOINT> (va en task-definition.json)
```

> La regla de entrada del `code-insight-ai-db-sg` (puerto 5432) se abre para el SG de la
> tarea ECS en el paso §6.

### 4.1 Contraseña de la BD en SSM Parameter Store (SecureString)

```bash
aws ssm put-parameter \
  --name /code-insight-ai/db-password \
  --type SecureString \
  --value 'CAMBIA_ESTA_CLAVE' \
  --region $REGION
```

---

## 5. Rol de ejecución de ECS (`ecsTaskExecutionRole`)

Permite a Fargate hacer *pull* de ECR, escribir logs y leer el secret de SSM.

```bash
cat > ecs-trust.json <<'JSON'
{ "Version": "2012-10-17", "Statement": [
  { "Effect": "Allow", "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole" } ] }
JSON

aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-trust.json

aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Permiso para leer el parámetro de la contraseña
cat > ecs-ssm.json <<JSON
{ "Version": "2012-10-17", "Statement": [
  { "Effect": "Allow", "Action": ["ssm:GetParameters"],
    "Resource": "arn:aws:ssm:$REGION:$ACCOUNT_ID:parameter/code-insight-ai/db-password" } ] }
JSON

aws iam put-role-policy --role-name ecsTaskExecutionRole \
  --policy-name read-db-password --policy-document file://ecs-ssm.json
```

---

## 6. Amazon ECS Fargate (cómputo de la API)

1. **Edita `task-definition.json`** en el repo `code-insight-ai-api` y reemplaza
   `<AWS_ACCOUNT_ID>`, `<REGION>` y `<RDS_ENDPOINT>`.

2. **Cluster + log group:**

```bash
aws ecs create-cluster --cluster-name code-insight-ai
aws logs create-log-group --log-group-name /ecs/code-insight-ai-api --region $REGION
```

3. **Registra la primera revisión** del task definition (el pipeline registrará las
   siguientes automáticamente):

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

4. **Security group de la tarea** y regla hacia la BD:

```bash
# SG de la tarea ECS
aws ec2 create-security-group --group-name code-insight-ai-api-sg \
  --description "Code Insight AI API task" --output text --query 'GroupId'
# -> TASK_SG

# Permite que la tarea llegue a Postgres
aws ec2 authorize-security-group-ingress --group-id <DB_SG> \
  --protocol tcp --port 5432 --source-group <TASK_SG>
```

5. **Crea el servicio** (usa subredes públicas de tu VPC default y IP pública para que
   pueda hacer pull de ECR):

```bash
# Subredes de la VPC default
aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" \
  --query 'Subnets[].SubnetId' --output text   # -> subnet-a,subnet-b

aws ecs create-service \
  --cluster code-insight-ai \
  --service-name code-insight-ai-api \
  --task-definition code-insight-ai-api \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-a,subnet-b],securityGroups=[<TASK_SG>],assignPublicIp=ENABLED}"
```

6. La URL pública de la API es la IP pública de la tarea (o, si añades un
   Application Load Balancer, su DNS). Obtén la IP:

```bash
TASK=$(aws ecs list-tasks --cluster code-insight-ai --service-name code-insight-ai-api --query 'taskArns[0]' --output text)
ENI=$(aws ecs describe-tasks --cluster code-insight-ai --tasks $TASK \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
aws ec2 describe-network-interfaces --network-interface-ids $ENI \
  --query 'NetworkInterfaces[0].Association.PublicIp' --output text
# -> esta IP:8080 es tu API_URL (para la variable API_URL del front)
```

> Para un demo estable conviene un ALB delante del servicio (DNS fijo + HTTPS). No es
> obligatorio para la kata.

---

## 7. Amazon S3 + CloudFront (frontend Angular)

```bash
export S3_BUCKET=code-insight-ai-web-$ACCOUNT_ID
aws s3api create-bucket --bucket $S3_BUCKET --region $REGION

# Habilita hosting estático (SPA: index.html también como documento de error)
aws s3 website s3://$S3_BUCKET/ --index-document index.html --error-document index.html
```

Crea la distribución de CloudFront (consola → *Create distribution*):
- **Origin**: el bucket S3 (usa *Origin access control* — recomendado).
- **Default root object**: `index.html`.
- **Custom error responses**: 403 y 404 → `/index.html` (200), para el routing SPA.
- Guarda el **Distribution ID** y el **dominio** (`dxxxx.cloudfront.net`).

> Con Origin Access Control, CloudFront añadirá automáticamente la policy al bucket
> (acéptala en la consola). El `s3 sync` del pipeline usa el rol OIDC del §2.

---

## 8. Secrets y Variables en GitHub

En **cada** repo de la app (`Settings → Secrets and variables → Actions`):

### `code-insight-ai-api`

| Tipo | Nombre | Valor |
|------|--------|-------|
| Secret | `SONAR_TOKEN` | token de SonarCloud |
| Secret | `AWS_ROLE_ARN` | ARN del rol `code-insight-ai-deploy` |

### `code-insight-ai-web`

| Tipo | Nombre | Valor |
|------|--------|-------|
| Secret | `SONAR_TOKEN` | token de SonarCloud |
| Secret | `AWS_ROLE_ARN` | ARN del rol `code-insight-ai-deploy` |
| Variable | `API_URL` | `http://<IP_o_ALB_de_la_API>:8080` |
| Variable | `S3_BUCKET` | `code-insight-ai-web-<ACCOUNT_ID>` |
| Variable | `CLOUDFRONT_DISTRIBUTION_ID` | ID de la distribución |

> Los *Secrets* se ocultan; las *Variables* son visibles (úsalas sólo para datos no
> sensibles como nombres de bucket o URLs públicas).

---

## 9. Orden de la primera ejecución

1. §1–§7 completos (infra creada, `task-definition.json` con valores reales, servicio ECS
   corriendo la primera revisión).
2. Configura secrets/variables (§8).
3. Haz push del código de la app a `main` (o abre un PR):
   - **PR** → corre `analyze` (tests + SonarCloud + Quality Gate).
   - **push/merge a `main`** → corre `deploy` (ECR+ECS / S3+CloudFront).
4. Verifica: la API responde en `/actuator/health` y el sitio carga desde CloudFront.

## Checklist rápido

- [ ] Proveedor OIDC + rol `code-insight-ai-deploy` creados
- [ ] ECR `code-insight-ai-api`
- [ ] RDS `code-insight-ai-db` + parámetro SSM `/code-insight-ai/db-password`
- [ ] `ecsTaskExecutionRole` con SSM
- [ ] Cluster + log group + task def registrado + servicio ECS
- [ ] Bucket S3 + CloudFront
- [ ] Secrets/Variables en ambos repos
- [ ] `ci-templates` es **público** y tag `v1.4.0` publicado

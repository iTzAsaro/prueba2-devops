# Informe de validación y cumplimiento (AWS) — Aplicación web distribuida

## Alcance y limitaciones

Este repositorio no incluye IaC (Terraform/CloudFormation/CDK) ni export de la configuración real de AWS. Por lo tanto:

- La validación “completa” de VPC, tablas de enrutamiento, Security Groups y EC2 requiere acceso a la cuenta AWS (consola o CLI).
- Lo que sí se valida aquí es la parte reproducible desde el repo: pipeline de GitHub Actions, construcción de imágenes para ECR y configuración de comunicación Frontend↔Backend a nivel de contenedores y reverse-proxy.

## Hallazgos principales (riesgos y fallos probables)

### Comunicación Frontend ↔ Backend

- El frontend realiza llamadas a `/api/v1/despachos` y `/api/v1/ventas` (misma origin), lo que requiere reverse-proxy en el servidor del frontend.
- El Nginx del frontend tenía `proxy_pass` con una IP privada fija (`10.0.4.149:8081`), lo que rompe despliegues al cambiar IP/instancia y limita escalabilidad.
- El pipeline solo construía un backend (Despachos), pero el frontend consume también el backend de Ventas (`/api/v1/ventas`). Esto puede producir 404/502 en producción.

### CI/CD hacia ECR (GitHub Actions)

- Se dependía de Access Keys (secretos de larga vida) para autenticar a AWS. Esto es una debilidad frente a OIDC con rol asumible (mejor práctica actual).
- No existía “ensure repo exists”, lo que puede provocar fallos si el repositorio ECR no está creado previamente.

### CORS en Backends

- Se permitía `allowedOrigins("*")` y además anotaciones `@CrossOrigin(origins="*")` en controladores. Esto amplía innecesariamente superficie de ataque si el backend se expone públicamente.

## Cambios implementados en el repositorio

### 1) Reverse-proxy Nginx parametrizable (sin IP hardcodeada)

- Se reemplazó el `proxy_pass` fijo por upstreams configurables por variables de entorno:
  - `DESPACHOS_UPSTREAM` (default: `backend-despachos:8081`)
  - `VENTAS_UPSTREAM` (default: `backend-ventas:8080`)
- Se agregaron rutas específicas para:
  - `/api/v1/despachos` → Despachos
  - `/api/v1/ventas` → Ventas
- Archivo afectado: [Dockerfile](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/front_despacho/Dockerfile)

### 2) Se incorpora backend de Ventas al stack (contenedorizado)

- Se agregó Dockerfile para el servicio de Ventas.
- Se añadió el servicio `backend-ventas` al `docker-compose.yml` y se actualizó el frontend para depender de ambos backends.
- Archivos afectados:
  - [docker-compose.yml](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/docker-compose.yml)
  - [Dockerfile (Ventas)](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/back-Ventas_SpringBoot/Springboot-API-REST/Dockerfile)

### 3) Pipeline de ECR endurecido y ampliado (Frontend + 2 Backends)

- Se agregó soporte para OIDC con `role-to-assume` (si se define `AWS_ROLE_TO_ASSUME`).
- Se mantiene compatibilidad con Access Keys si no se usa OIDC.
- Se agregaron:
  - Buildx (para build consistente)
  - Caché GHA para builds
  - Creación automática de repositorios ECR si no existen
- Se incluyen tres imágenes:
  - Frontend
  - Backend Despachos
  - Backend Ventas
- Archivo afectado: [push-to-ecr.yml](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/.github/workflows/push-to-ecr.yml)

### 4) CORS parametrizable (reduce exposición por defecto en producción)

- Se eliminaron anotaciones `@CrossOrigin` de los controladores y se centralizó en configuración.
- Se implementó lectura por propiedad:
  - `app.cors.allowed-origins=${CORS_ALLOWED_ORIGINS:*}`
- Archivos afectados:
  - [CorsConfig (Despachos)](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/src/main/java/com/citt/config/CorsConfig.java)
  - [application.properties (Despachos)](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/src/main/resources/application.properties)
  - [CorsConfig (Ventas)](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/back-Ventas_SpringBoot/Springboot-API-REST/src/main/java/com/citt/config/CorsConfig.java)
  - [application.properties (Ventas)](file:///c:/Users/alexs/Documents/Developer/ISY1101_EP2_Proyecto%20Semestral/proyecto%20semestral/back-Ventas_SpringBoot/Springboot-API-REST/src/main/resources/application.properties)

## Validación en AWS (checklist ejecutable)

### VPC / Subnets / Routing

- Confirmar que:
  - Frontend EC2 está en subnet pública (ruta `0.0.0.0/0` al IGW).
  - Backend EC2 está en subnet privada (sin IP pública) o, si está en pública por simplicidad, que no expone puertos de API a Internet.
  - Si backend está en subnet privada y necesita salir a ECR, existe NAT Gateway o VPC Endpoints para ECR/S3.
- CLI sugerida:
  - `aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"`
  - `aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>"`

### Security Groups (mínimo necesario)

- Frontend SG:
  - Inbound: 80/443 desde `0.0.0.0/0`
  - Inbound admin: 22 solo desde IPs de administración (o preferir SSM y cerrar 22)
  - Outbound: permitir hacia backend (puerto(s) necesarios) y hacia ECR/NTP/DNS si aplica
- Backend SG:
  - Inbound: 8081/8080 solo desde el SG del frontend (source: frontend SG)
  - Inbound admin: 22 cerrado o restringido; preferir SSM
  - Outbound: hacia DB (si existe), ECR (si pull), DNS/NTP
- Validaciones:
  - `aws ec2 describe-security-groups --group-ids <sg-id>`

### ECR (acceso seguro)

- Recomendación:
  - EC2 con instance profile (rol IAM) para `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`
  - GitHub Actions con OIDC + rol asumible con permisos mínimos para `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:DescribeRepositories`, `ecr:CreateRepository` (si se usa autocración)
- Validación:
  - `aws ecr describe-repositories --repository-names <repos...>`

### Conectividad Frontend ↔ Backend

- Si el frontend sirve el SPA por Nginx con proxy:
  - Asegurar que la URL pública del frontend responde y que:
    - `GET https://<frontend-domain>/api/v1/despachos` retorna 200
    - `GET https://<frontend-domain>/api/v1/ventas` retorna 200
- Si no se usa proxy (no recomendado):
  - Asegurar CORS restringido y que el frontend use baseURL explícito.

## Cumplimiento de buenas prácticas (resumen)

- Identidad y acceso
  - Preferir OIDC en GitHub Actions y roles en EC2 (sin claves estáticas).
  - Principio de mínimo privilegio en políticas IAM (separar push vs pull).
- Red
  - Restringir API backend a origen SG del frontend (no 0.0.0.0/0).
  - Subnet privada + NAT o Endpoints para ECR si backend no tiene salida directa.
- Observabilidad
  - Logs centralizados (CloudWatch Logs) y métricas/alarms (CPU, memoria, 5xx, latency).
- Resiliencia y escalabilidad
  - Evitar IP fija en proxy (usar DNS/ALB/target groups o variables de entorno).
  - Considerar ALB delante del backend y Auto Scaling Group para frontend/backend si el tráfico crece.

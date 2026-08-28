# 🏦 NEOBANK

Plataforma bancaria con arquitectura hexagonal (Spring Boot 3 / Java 21) y un núcleo
transaccional serverless (8 Lambdas en AWS), frontend en Next.js 15 e infraestructura
completa en Terraform.

📄 Ver también [SERVICES.md](SERVICES.md) (detalle de cada servicio) y [PORTFOLIO.md](PORTFOLIO.md).

---

## 📦 Contenido

| Carpeta | Qué es | Cómo se ejecuta en local |
|---|---|---|
| `neobank-backend/` | API REST Spring Boot 3.2 / Java 21 (9 controladores, hexagonal) | ✅ Docker Compose o Maven |
| `neobank-frontend/neobank-frontend/` | SPA Next.js 15 / React 19 / Tailwind v4 (18 rutas) | ✅ `npm run dev` |
| `neobank-lambdas/` | 8 funciones Lambda (7 Java 17 + 1 Python 3.11) | ⚠️ Solo tests unitarios en local; se ejecutan en AWS |
| `neobank-terraform/` | VPC, RDS, EC2, S3, DynamoDB, SNS, SQS, Cognito, API Gateway | ⚠️ Solo despliegue real en AWS |
| `jmeter/` | 2 planes de carga (API REST y Lambdas vía API Gateway) | ✅ Requiere un entorno ya levantado |
| `neobank.postman_collection.json` | Colección Postman de ambas superficies de API | ✅ |

---

## ✅ Requisitos previos

| Herramienta | Versión requerida | Necesaria para |
|---|---|---|
| **Docker Desktop** | 24+ (con Compose v2) | Ruta recomendada: backend + PostgreSQL + Zipkin. También lo usan los tests de integración (Testcontainers) |
| **Node.js** | 20+ | Frontend |
| **JDK 21** | Temurin 21 (**exactamente 21**) | Solo si ejecutas el backend de forma nativa, sin Docker |
| **Maven** | No hace falta instalarlo para el backend: usa el wrapper `./mvnw` | Backend (los Lambdas sí requieren `mvn` instalado) |
| **Python 3.11+** | Opcional | Tests del Lambda `fraud-checker` |
| **AWS CLI + Terraform 1.6+** | Opcional | Solo para desplegar en AWS |

> ⚠️ **Nota sobre la versión de Java.** El proyecto compila con Lombok 1.18.30, que **no es
> compatible con JDK 25**. Si tu `java -version` reporta 25, la compilación nativa con
> `./mvnw` fallará. Tienes dos opciones: instalar **Temurin JDK 21** y apuntar `JAVA_HOME` a
> él, o —más simple— usar la **Ruta A (Docker)**, donde la compilación ocurre dentro de un
> contenedor `eclipse-temurin:21` y la versión de Java de tu máquina es irrelevante.

---

## ⚠️ Paso 0 — Prerrequisito de AWS (léelo antes de empezar)

La autenticación **no es local**: `signup`, `login`, `verify-email`, `refresh-token` y
`reset-password` llaman directamente a **AWS Cognito** (`CognitoAdapter`), y el resto de
endpoints exige un JWT emitido por Cognito y validado contra su JWKS. **No existe un modo
mock ni un bypass local.**

Esto significa que:

- **Sin credenciales de AWS + un User Pool de Cognito** puedes: levantar la base de datos,
  arrancar el backend, ver Swagger UI, consultar `/actuator/health`, ejecutar todos los tests
  automatizados y levantar el frontend — pero **no podrás iniciar sesión** ni probar ningún
  endpoint autenticado.
- **Con un User Pool de Cognito** puedes probar el flujo completo del backend contra una base
  de datos PostgreSQL local.

### Crear el User Pool mínimo

Puedes crearlo a mano en la consola de AWS o aplicar solo el módulo de Terraform. La
configuración que el código espera es:

| Ajuste | Valor requerido | Por qué |
|---|---|---|
| Auth flows del App Client | `ALLOW_USER_PASSWORD_AUTH` y `ALLOW_REFRESH_TOKEN_AUTH` | `CognitoAdapter` usa `USER_PASSWORD_AUTH` e `InitiateAuth` con refresh token |
| Client secret | **Sin secreto** (`generate_secret = false`) | El código no calcula `SECRET_HASH`; con secreto, el login falla |
| Atributos autoverificados | `email` | Necesario para `verify-email` / `forgot-password` |
| Política de contraseña | mín. 8, mayúscula + minúscula + número + símbolo | Es la que define el módulo Terraform; tus contraseñas de prueba deben cumplirla |

Con Terraform (solo el módulo de Cognito):

```bash
cd neobank-terraform
terraform init
terraform apply -target=module.cognito
```

Los servicios AWS **opcionales** son S3 (subida de documentos KYC), SES (correos) y DynamoDB
(historial de transacciones y analytics). Si los dejas vacíos, el backend arranca igual: solo
fallarán los endpoints de KYC, analytics y el historial de transacciones.

---

## 🔐 Paso 1 — Archivo `.env`

Crea `neobank-backend/.env` 

```bash
# --- Base de datos (obligatorio) ---
DB_PASSWORD=postgres

# --- AWS: obligatorio para poder autenticarte ---
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
AWS_COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx

# --- AWS: opcional (KYC / correos) ---
AWS_S3_BUCKET_NAME=neobank-kyc-dev
AWS_SES_FROM_EMAIL=tu-correo-verificado@ejemplo.com

# --- Secreto de firma local (mín. 256 bits) ---
JWT_SECRET=DevSecretKeyForLocalDevelopmentOnlyMinimum256BitsLong!!
```

> `docker-compose.yml` falla de forma explícita si `DB_PASSWORD` no está definida
> (`${DB_PASSWORD:?set DB_PASSWORD in .env}`). Las demás variables pueden quedar vacías.

---

## 🐳 Ruta A — Backend con Docker Compose (recomendada)

Levanta PostgreSQL 15, Zipkin y el backend. Flyway aplica las 5 migraciones automáticamente
al arrancar.

```bash
cd neobank-backend

# 1. Asegúrate de que Docker Desktop esté corriendo
docker info

# 2. Construir y levantar (la primera vez tarda: descarga dependencias Maven)
docker compose up --build -d

# 3. Seguir los logs del backend
docker compose logs -f neobank-backend
```

Servicios expuestos:

| Servicio | URL |
|---|---|
| API backend | http://localhost:8080 |
| Swagger UI | http://localhost:8080/swagger-ui/index.html |
| OpenAPI JSON | http://localhost:8080/v3/api-docs |
| Health check | http://localhost:8080/actuator/health |
| Zipkin (trazas) | http://localhost:9411 |
| PostgreSQL | `localhost:5432` · db `neobank_db` · user `postgres` |

Espera a que el healthcheck pase (tiene un `start_period` de 45 s) y verifica:

```bash
curl http://localhost:8080/actuator/health
# -> {"status":"UP", ...}
```

Comandos útiles:

```bash
docker compose ps                 # estado de los 3 contenedores
docker compose restart neobank-backend
docker compose down               # parar (conserva los datos)
docker compose down -v            # parar y BORRAR la base de datos
```

---

## ☕ Ruta B — Backend nativo con Maven (requiere JDK 21)

Útil si quieres depurar desde el IDE con hot reload.

```bash
cd neobank-backend

# 1. Levantar solo PostgreSQL y Zipkin con Docker
docker compose up -d postgres zipkin

# 2. Comprobar que estás en JDK 21 (NO 25 — ver nota de Lombok arriba)
java -version

# 3. Arrancar con el perfil dev
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

Antes del paso 3 necesitas exportar las variables del `.env` a tu sesión:

```powershell
# PowerShell
Get-Content .env | Where-Object { $_ -match '=' -and $_ -notmatch '^#' } | ForEach-Object {
    $k, $v = $_ -split '=', 2
    [Environment]::SetEnvironmentVariable($k.Trim(), $v.Trim())
}
```

```bash
# Git Bash
set -a && source .env && set +a
```

El perfil `dev` (`application-dev.yml`) apunta a `localhost:5432`, activa logs SQL, escribe en
`logs/neobank-dev.log` y usa un JWT con expiración de 24 h para que no caduque mientras
pruebas.

Empaquetar el JAR:

```bash
./mvnw clean package -DskipTests
java -jar target/neobank-backend-1.0.0.jar
```

---

## 🌐 Frontend (Next.js)

> Ojo con la ruta: el proyecto está **anidado** en `neobank-frontend/neobank-frontend/`.

```bash
cd neobank-frontend/neobank-frontend
npm install
```

Crea `.env.local` en esa misma carpeta:

```bash
# Backend Spring Boot
NEXT_PUBLIC_API_URL=http://localhost:8080

# API Gateway de las Lambdas de transacciones (déjala vacía si no desplegaste en AWS)
NEXT_PUBLIC_LAMBDA_URL=
```

```bash
npm run dev     # http://localhost:3000
```

Si `NEXT_PUBLIC_LAMBDA_URL` queda vacía, las pantallas de **transferencias** e **historial de
transacciones** fallarán (dependen de las Lambdas); el resto de la app funciona contra el
backend Spring.

> El backend ya permite CORS desde `http://localhost:3000` en el perfil `dev`.

---

## 🧪 Flujo de prueba manual (end-to-end)

Con el backend arriba y Cognito configurado:

```bash
# 1. Registro (crea el usuario en Cognito + Postgres y abre una cuenta con saldo 0)
curl -X POST http://localhost:8080/api/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"prueba@ejemplo.com","password":"Prueba123!","fullName":"Usuario Prueba","phone":"5512345678","dateOfBirth":"1990-01-01","curp":"XXXX900101HDFXXX01"}'

# 2. Cognito envía un código al correo. Confírmalo:
curl -X POST http://localhost:8080/api/auth/verify-email \
  -H "Content-Type: application/json" \
  -d '{"email":"prueba@ejemplo.com","code":"123456"}'

# 3. Login -> devuelve accessToken
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"prueba@ejemplo.com","password":"Prueba123!"}'

# 4. Usar el token en cualquier endpoint protegido
curl http://localhost:8080/api/accounts -H "Authorization: Bearer <accessToken>"
```

### Sembrar saldo para probar transferencias

El registro crea la cuenta con **saldo 0**. Para tener fondos con los que probar:

```bash
docker exec -it neobank-postgres psql -U postgres -d neobank_db -c "UPDATE accounts SET balance = 50000.00, available_balance = 50000.00 WHERE user_id = (SELECT id FROM users WHERE email = 'prueba@ejemplo.com');"
```

---

## 📮 Postman

1. Importa `neobank.postman_collection.json`.
2. Ajusta las variables de la colección:
   - `backend_url` → `http://localhost:8080`
   - `transaction_api_url` → tu endpoint de API Gateway (solo si desplegaste en AWS)
3. Ejecuta **Signup** → **Login**. El token se captura automáticamente en `{{accessToken}}` y
   autoriza el resto de peticiones de ambas superficies de API.

---

## 🧪 Pruebas automatizadas

### Backend

```bash
cd neobank-backend

./mvnw test        # solo unitarios (JUnit 5 + Mockito)
./mvnw verify      # unitarios + integración (Testcontainers -> requiere Docker corriendo)
```

Reportes en `target/surefire-reports/` y `target/failsafe-reports/`.

### Frontend

```bash
cd neobank-frontend/neobank-frontend
npm test           # Vitest
npm run lint
npm run build
```

### Lambdas

Cada Lambda de Java es un proyecto Maven independiente y **no** trae wrapper `mvnw`, así que
aquí sí necesitas Maven instalado:

```bash
cd neobank-lambdas/transaction-service && mvn test
# igual para: transaction-query, ledger-writer, notification-service,
#             analytics-processor, kyc-validator, lex-fulfillment
```

La Lambda de Python:

```bash
cd neobank-lambdas/fraud-checker
python -m venv .venv
.venv/Scripts/activate          # Linux/Mac: source .venv/bin/activate
pip install -r requirements-dev.txt
pytest
```

### JMeter (pruebas de carga)

Requiere un entorno ya levantado y usuarios reales verificados en Cognito.

1. Edita `jmeter/test-users.csv` con credenciales reales (el archivo trae placeholders).
2. Abre `jmeter/neobank-backend-api.jmx` (API REST) o
   `jmeter/neobank-transactions-api-gateway.jmx` (Lambdas) en JMeter y ajusta host/puerto.

---

## ☁️ Despliegue en AWS (opcional)

### Infraestructura

```bash
cd neobank-terraform
terraform init
terraform plan
terraform apply
```

Variables sin valor por defecto que **debes** definir en `terraform.tfvars`: `key_name`,
`db_password`, `aws_access_key_id`, `aws_secret_access_key`, `s3_bucket_name`,
`ses_from_email`, `jwt_secret` y `ssh_allowed_cidr` (nunca `0.0.0.0/0`).

> El backend de estado de Terraform está **comentado** en `main.tf`: por defecto el estado se
> guarda localmente y sin bloqueo. Descoméntalo y ejecuta `terraform init -migrate-state`
> antes de trabajar en equipo.

### Lambdas

```bash
cd neobank-lambdas
./deploy-all.sh     # compila las 7 Java, empaqueta la de Python y actualiza el código en AWS
```

Requiere Maven, AWS CLI autenticada, `pip` y `zip`.

---

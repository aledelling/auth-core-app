# 🔐 AuthCore - API REST Multitenant de Autenticación y Autorización

[![Java](https://img.shields.io/badge/Java-17+-blue.svg)](https://openjdk.java.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2+-green.svg)](https://spring.io/projects/spring-boot)
[![Security](https://img.shields.io/badge/OWASP-TOP%2010-red.svg)](https://owasp.org/www-project-top-ten/)
[![Architecture](https://img.shields.io/badge/Architecture-Hexagonal-orange.svg)]()

Módulo central reutilizable para gestión de usuarios, JWT, roles y seguridad basada en políticas con arquitectura hexagonal y cumplimiento OWASP TOP 10.

---

## 📋 Índice

1. [Arquitectura](#-arquitectura)
2. [Endpoints](#-endpoints)
3. [Seguridad](#-seguridad)
4. [Configuración](#-configuración)
5. [Despliegue](#-despliegue)

---

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────────────────┐
│              ADAPTERS (Infrastructure)               │
│   REST Controller | Security Filter | JPA Repo      │
├─────────────────────────────────────────────────────┤
│                    PORTS                             │
│   Input: Authenticate, CreateUser, ValidateToken    │
│   Output: UserRepository, RoleRepository            │
├─────────────────────────────────────────────────────┤
│                 DOMAIN (Core)                        │
│   Entities: User, Role, Permission, Tenant          │
│   Value Objects: Email, Password, JwtToken          │
└─────────────────────────────────────────────────────┘
```

**Capas:**
- **Domain**: Entidades puras sin dependencias externas
- **Application**: Casos de uso y servicios
- **Infrastructure**: Implementaciones concretas

---

## 🌐 Endpoints

**Base URL:** `http://localhost:8080/api/v1`

### Autenticación

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/auth/login` | Iniciar sesión |
| POST | `/auth/refresh` | Renovar token |
| POST | `/auth/logout` | Cerrar sesión |

**Ejemplo Login:**
```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: tenant1" \
  -d '{"email":"user@empresa.com","password":"P@ssw0rd123!"}'
```

**Response:**
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

### Usuarios
```bash
GET    /users              # Listar
POST   /users              # Crear
GET    /users/{id}         # Obtener
PUT    /users/{id}         # Actualizar
DELETE /users/{id}         # Eliminar
```

### Roles y Permisos
```bash
GET    /roles                         # Listar roles
POST   /roles                         # Crear rol
POST   /users/{id}/roles              # Asignar rol
GET    /permissions                   # Listar permisos
POST   /permissions/check             # Verificar permiso
```

### Tenants
```bash
GET    /tenants            # Listar tenants
POST   /tenants            # Crear tenant
```

---

## 🛡️ Seguridad OWASP TOP 10

| Vulnerabilidad | Mitigación |
|----------------|------------|
| **A01: Broken Access Control** | @PreAuthorize, principio menor privilegio |
| **A02: Cryptographic Failures** | TLS 1.3, bcrypt/Argon2 para passwords |
| **A03: Injection** | Prepared Statements, validación de inputs |
| **A04: Insecure Design** | Patrones seguros, threat modeling |
| **A05: Security Misconfiguration** | Hardening, scans automáticos |
| **A06: Vulnerable Components** | OWASP Dependency Check, Dependabot |
| **A07: Auth Failures** | MFA, rate limiting, account lockout |
| **A08: Data Integrity** | Firmas digitales en JWT |
| **A09: Logging Failures** | Logs estructurados, auditoría completa |
| **A10: SSRF** | Validación de URLs, allowlisting |

---

## ⚙️ Configuración

### Variables de Entorno

```bash
SPRING_PROFILES_ACTIVE=dev
DB_HOST=localhost
DB_PORT=5432
DB_NAME=authcore
DB_USERNAME=authcore_user
DB_PASSWORD=ChangeMe123!
JWT_SECRET=change-to-very-long-random-secret-minimum-256-bits
JWT_EXPIRATION_MS=900000
MULTITENANT_STRATEGY=SCHEMA_PER_TENANT
```

### Estrategias Multitenant

| Estrategia | Descripción | Uso recomendado |
|------------|-------------|-----------------|
| DATABASE_PER_TENANT | BD aislada por tenant | Máximo aislamiento |
| SCHEMA_PER_TENANT | Esquema separado | Balance costo/aislamiento |
| DISCRIMINATOR_COLUMN | Columna tenant_id | Bajo costo, menos aislamiento |

### Resolución de Tenant

- **Header HTTP:** `X-Tenant-ID: tenant1`
- **Subdominio:** `tenant1.api.authcore.com`
- **JWT Claim:** `tenant_id`

---

## 🚀 Despliegue

### Prerrequisitos
- Java JDK 17+
- PostgreSQL 14+
- Maven 3.8+
- Docker 20+ (opcional)

### Local

```bash
git clone https://github.com/tu-organizacion/auth-core.git
cd auth-core
mvn clean install -DskipTests
mvn flyway:migrate -Dspring.profiles.active=dev
mvn spring-boot:run -Dspring.profiles.active=dev
```

Verificar: `curl http://localhost:8080/actuator/health`

### Docker Compose

```yaml
version: '3.8'
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: authcore
      POSTGRES_PASSWORD: ChangeMe123!
    ports:
      - "5432:5432"
  
  app:
    build: .
    environment:
      DB_HOST: db
    ports:
      - "8080:8080"
    depends_on:
      - db
```

```bash
docker-compose up -d
```

### Kubernetes (Deploy mínimo)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authcore
spec:
  replicas: 3
  selector:
    matchLabels:
      app: authcore
  template:
    spec:
      containers:
        - name: authcore
          image: registry.example.com/authcore:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
```

---

## 🔌 Integración Rápida

### Java (WebClient)

```java
@Component
public class AuthCoreClient {
    private final WebClient webClient;
    
    public AuthCoreClient(String baseUrl) {
        this.webClient = WebClient.builder().baseUrl(baseUrl).build();
    }
    
    public AuthResponse login(String email, String password) {
        return webClient.post()
            .uri("/auth/login")
            .bodyValue(new LoginRequest(email, password))
            .retrieve()
            .bodyToMono(AuthResponse.class)
            .block();
    }
}
```

### Node.js (Axios)

```javascript
const client = axios.create({
  baseURL: 'https://api.authcore.com/v1',
  headers: { 'X-Tenant-ID': 'tenant1' }
});

const response = await client.post('/auth/login', {
  email: 'user@empresa.com',
  password: 'P@ssw0rd123!'
});
```

---

## 📊 Monitoreo

**Actuator Endpoints:**
- `/actuator/health` - Salud general
- `/actuator/health/liveness` - Liveness probe (K8s)
- `/actuator/health/readiness` - Readiness probe (K8s)
- `/actuator/metrics` - Métricas Prometheus

**Métricas clave:** `auth_login_total`, `auth_login_failure_total`, `auth_token_validation_total`

---

## 🧪 Testing

```bash
mvn test                    # Tests unitarios
mvn verify                  # Tests integración
mvn jacoco:report           # Cobertura (mínimo 80%)
```

---

## 📄 Licencia

MIT License

---

<div align="center">

**AuthCore** - Módulo de seguridad reutilizable para cualquier proyecto

</div>

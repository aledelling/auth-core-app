# 🔐 AuthCore - Módulo Central de Autenticación y Autorización Multitenant

[![Java](https://img.shields.io/badge/Java-17+-blue.svg)](https://openjdk.java.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2+-green.svg)](https://spring.io/projects/spring-boot)
[![Security](https://img.shields.io/badge/OWASP-TOP%2010-red.svg)](https://owasp.org/www-project-top-ten/)
[![Architecture](https://img.shields.io/badge/Architecture-Hexagonal-orange.svg)]()

---

## 📋 Tabla de Contenidos

1. [Descripción General](#-descripción-general)
2. [Arquitectura Hexagonal](#-arquitectura-hexagonal)
3. [Características Principales](#-características-principales)
4. [Seguridad OWASP TOP 10](#-seguridad-owasp-top-10)
5. [API REST - Endpoints](#-api-rest---endpoints)
6. [Modelo de Datos](#-modelo-de-datos)
7. [Configuración Multitenant](#-configuración-multitenant)
8. [Instalación y Despliegue](#-instalación-y-despliegue)
9. [Integración](#-integración)

---

## 🎯 Descripción General

**AuthCore** es un módulo central de autenticación y autorización diseñado con **Arquitectura Hexagonal** para Java Spring Boot, proporcionando una solución reutilizable, escalable y segura para cualquier proyecto futuro.

### Propósito

- ✅ Gestión centralizada de usuarios y credenciales
- ✅ Autenticación basada en JWT (JSON Web Tokens)
- ✅ Sistema de roles y permisos (RBAC/ABAC/PBAC)
- ✅ Soporte Multitenant (multi-inquilino)
- ✅ Cumplimiento OWASP TOP 10
- ✅ Alta disponibilidad y escalabilidad

### Casos de Uso

| Caso | Descripción |
|------|-------------|
| SSO Centralizado | Punto único de autenticación para múltiples aplicaciones |
| Microservicios | Servicio de seguridad compartido |
| Multi-tenant SaaS | Aislamiento de datos por cliente |
| API Gateway Security | Validación centralizada de tokens |

---

## 🏗️ Arquitectura Hexagonal

```
┌─────────────────────────────────────────────────────────────┐
│                    ADAPTERS (Infrastructure)                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │   REST   │  │ Security │  │   JPA    │  │  Email   │    │
│  │ Controller│  │  Filter  │  │Repository│  │ Service  │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
├───────┴─────────────┴─────────────┴─────────────┴───────────┤
│                         PORTS                                │
│  ┌─────────────────┐              ┌─────────────────────┐   │
│  │  Input Ports    │              │   Output Ports      │   │
│  │  (Use Cases)    │◄──────►│   (Interfaces)        │   │
│  │ - Authenticate  │              │ - UserRepository    │   │
│  │ - CreateUser    │              │ - RoleRepository    │   │
│  │ - ValidateToken │              │ - TokenRepository   │   │
│  └────────┬────────┘              └─────────────────────┘   │
├─────────┴───────────────────────────────────────────────────┤
│                      DOMAIN (Core Business)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Entities   │  │Value Objects│  │Domain Svcs  │         │
│  │ - User      │  │ - Email     │  │ - AuthSvc   │         │
│  │ - Role      │  │ - Password  │  │ - PolicySvc │         │
│  │ - Permission│  │ - JwtToken  │  │             │         │
│  │ - Tenant    │  │ - TenantId  │  │             │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Capas

1. **Domain Layer**: Entidades de negocio puras, sin dependencias externas
2. **Application Layer**: Casos de uso y servicios de aplicación
3. **Infrastructure Layer**: Implementaciones concretas (adapters)

---

## ✨ Características Principales

### Autenticación

| Feature | Descripción |
|---------|-------------|
| JWT | Tokens con firma RS256/HS512 |
| Refresh Tokens | Rotación segura de tokens |
| MFA | TOTP, SMS, Email |
| OAuth2/OIDC | Proveedores externos |
| Session Management | Control de sesiones concurrentes |

### Autorización

| Feature | Descripción |
|---------|-------------|
| RBAC | Role-Based Access Control |
| ABAC | Attribute-Based Access Control |
| PBAC | Policy-Based Access Control |
| Permisos Granulares | Control por recurso y acción |

### Multitenancy

| Estrategia | Descripción |
|------------|-------------|
| Database per Tenant | BD aislada por tenant |
| Schema per Tenant | Esquema separado |
| Discriminator Column | Columna tenant_id compartida |

---

## 🛡️ Seguridad OWASP TOP 10

| Vulnerabilidad | Mitigación |
|----------------|------------|
| **A01: Broken Access Control** | @PreAuthorize, PBAC, menor privilegio |
| **A02: Cryptographic Failures** | TLS 1.3, AES-256, bcrypt/Argon2 |
| **A03: Injection** | Prepared Statements, validación inputs |
| **A04: Insecure Design** | Threat modeling, patrones seguros |
| **A05: Security Misconfiguration** | Hardening, scans automáticos |
| **A06: Vulnerable Components** | Dependabot, OWASP Dependency Check |
| **A07: Auth Failures** | MFA, rate limiting, account lockout |
| **A08: Data Integrity** | Firmas digitales, checksums |
| **A09: Logging Failures** | Logs estructurados, auditoría |
| **A10: SSRF** | Validación URLs, allowlisting |

---

## 🌐 API REST - Endpoints

### Base URL
```
Production: https://api.authcore.com/v1
Development: http://localhost:8080/api/v1
```

### Autenticación

#### Login
```http
POST /auth/login
Content-Type: application/json
X-Tenant-ID: tenant1

{
  "email": "usuario@empresa.com",
  "password": "P@ssw0rd123!",
  "mfaCode": "123456"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
    "tokenType": "Bearer",
    "expiresIn": 900,
    "user": {
      "id": "usr_123456",
      "email": "usuario@empresa.com",
      "roles": ["USER", "ADMIN"]
    }
  }
}
```

#### Refresh Token
```http
POST /auth/refresh
Content-Type: application/json

{"refreshToken": "..."}
```

#### Logout
```http
POST /auth/logout
Authorization: Bearer {accessToken}
```

### Usuarios

```http
GET  /users?page=0&size=20           # Listar
POST /users                           # Crear
GET  /users/{userId}                  # Obtener
PUT  /users/{userId}                  # Actualizar
DELETE /users/{userId}                # Eliminar (soft delete)
```

### Roles

```http
GET  /roles                           # Listar roles
POST /roles                           # Crear rol
POST /users/{userId}/roles            # Asignar rol
DELETE /users/{userId}/roles/{role}   # Remover rol
```

### Permisos

```http
GET  /permissions                     # Listar permisos
POST /permissions/check               # Verificar permiso
```

### Tenants

```http
GET  /tenants                         # Listar tenants
POST /tenants                         # Crear tenant
```

---

## 💾 Modelo de Datos

```sql
-- Tenants
CREATE TABLE tenants (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255) UNIQUE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    config JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Usuarios
CREATE TABLE users (
    id VARCHAR(50) PRIMARY KEY,
    tenant_id VARCHAR(50) REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    enabled BOOLEAN DEFAULT true,
    mfa_enabled BOOLEAN DEFAULT false,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(tenant_id, email)
);

-- Roles
CREATE TABLE roles (
    id VARCHAR(50) PRIMARY KEY,
    tenant_id VARCHAR(50) REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    UNIQUE(tenant_id, name)
);

-- Permisos
CREATE TABLE permissions (
    id VARCHAR(50) PRIMARY KEY,
    tenant_id VARCHAR(50) REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    resource VARCHAR(255) NOT NULL,
    action VARCHAR(50) NOT NULL,
    UNIQUE(tenant_id, name)
);

-- Relaciones
CREATE TABLE user_roles (
    user_id VARCHAR(50) REFERENCES users(id),
    role_id VARCHAR(50) REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE role_permissions (
    role_id VARCHAR(50) REFERENCES roles(id),
    permission_id VARCHAR(50) REFERENCES permissions(id),
    PRIMARY KEY (role_id, permission_id)
);

-- Auditoría
CREATE TABLE audit_logs (
    id VARCHAR(50) PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,
    user_id VARCHAR(50),
    action VARCHAR(100),
    status VARCHAR(20),
    details JSONB,
    ip_address INET,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 🏢 Configuración Multitenant

### Estrategias

```yaml
# Database per Tenant
multitenant:
  strategy: DATABASE_PER_TENANT
  datasource:
    master: jdbc:postgresql://localhost:5432/auth_master

# Schema per Tenant
multitenant:
  strategy: SCHEMA_PER_TENANT
  default-schema: public

# Discriminator Column
multitenant:
  strategy: DISCRIMINATOR_COLUMN
  column-name: tenant_id
```

### Resolución de Tenant

| Método | Ejemplo |
|--------|---------|
| Subdominio | tenant1.api.authcore.com |
| Header HTTP | X-Tenant-ID: tenant1 |
| JWT Claim | claim: tenant_id |
| Path Parameter | /api/v1/tenants/{tenantId}/users |

---

## 🚀 Instalación y Despliegue

### Prerrequisitos

- Java JDK 17+
- Maven 3.8+
- PostgreSQL 14+
- Redis 7.0+ (opcional)
- Docker 20+

### Variables de Entorno

```bash
SPRING_PROFILES_ACTIVE=dev
DB_HOST=localhost
DB_PORT=5432
DB_NAME=authcore
DB_USERNAME=authcore_user
DB_PASSWORD=ChangeMe123!
REDIS_HOST=localhost
JWT_SECRET=change-this-to-very-long-random-secret-minimum-256-bits
JWT_EXPIRATION_MS=900000
MULTITENANT_STRATEGY=SCHEMA_PER_TENANT
```

### Instalación Local

```bash
git clone https://github.com/tu-organizacion/auth-core.git
cd auth-core

mvn clean install -DskipTests
mvn flyway:migrate -Dspring.profiles.active=dev
mvn spring-boot:run -Dspring.profiles.active=dev

# Verificar
curl http://localhost:8080/actuator/health
```

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

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authcore
  namespace: auth-system
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
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
```

### CI/CD Pipeline

El pipeline incluye:
- Build y tests unitarios
- SonarQube analysis
- OWASP Dependency Check
- Trivy security scan
- Build y push Docker image
- Deploy a Kubernetes

---

## 🔌 Integración

### Cliente Java (WebClient)

```java
@Component
public class AuthCoreClient {
    
    private final WebClient webClient;
    
    public AuthCoreClient(@Value("${authcore.base-url}") String baseUrl) {
        this.webClient = WebClient.builder()
            .baseUrl(baseUrl)
            .build();
    }
    
    public AuthResponse authenticate(String email, String password) {
        return webClient.post()
            .uri("/auth/login")
            .bodyValue(new LoginRequest(email, password))
            .retrieve()
            .bodyToMono(AuthResponse.class)
            .block();
    }
}
```

### Cliente Node.js (Axios)

```javascript
const axios = require('axios');

class AuthCoreClient {
  constructor(baseUrl, tenantId) {
    this.client = axios.create({
      baseURL: baseUrl,
      headers: { 'X-Tenant-ID': tenantId }
    });
  }
  
  async login(email, password) {
    const response = await this.client.post('/auth/login', {
      email, password
    });
    this.accessToken = response.data.data.accessToken;
    return response.data;
  }
}
```

---

## 📊 Monitoreo

### Actuator Endpoints

| Endpoint | Descripción |
|----------|-------------|
| /actuator/health | Salud de la aplicación |
| /actuator/health/liveness | Liveness probe (K8s) |
| /actuator/health/readiness | Readiness probe (K8s) |
| /actuator/metrics | Métricas |
| /actuator/prometheus | Formato Prometheus |

### Métricas Principales

- `auth_login_total`: Total logins
- `auth_login_failure_total`: Logins fallidos
- `auth_token_validation_total`: Validaciones de token
- `http_requests_total`: Peticiones HTTP

---

## 🧪 Testing

```bash
# Tests unitarios
mvn test

# Tests de integración
mvn verify -DskipUnitTests

# Cobertura
mvn jacoco:report
```

Cobertura mínima requerida: 80%

---

## 🤝 Contribución

1. Fork el repositorio
2. Crea rama feature (`git checkout -b feature/AmazingFeature`)
3. Commit cambios (`git commit -m 'feat: add AmazingFeature'`)
4. Push (`git push origin feature/AmazingFeature`)
5. Abre Pull Request

### Convenciones

- Java: Google Java Style Guide
- Commits: Conventional Commits
- Tests: Cobertura mínima 80%

---

## 📄 Licencia

MIT License - Ver archivo LICENSE

---

## 📞 Soporte

- **Docs**: https://docs.authcore.com
- **API**: https://api.authcore.com/swagger-ui.html
- **Email**: support@authcore.com

---

<div align="center">

**AuthCore** - Autenticación y Autorización Seguro y Escalable

Hecho con ❤️ para la comunidad de desarrollo

</div>

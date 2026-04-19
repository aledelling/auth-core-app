# 🔐 AuthCore - API REST Multitenant de Autenticación y Autorización

[![Java](https://img.shields.io/badge/Java-17+-blue.svg)](https://openjdk.java.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2+-green.svg)](https://spring.io/projects/spring-boot)
[![Security](https://img.shields.io/badge/OWASP-TOP%2010-red.svg)](https://owasp.org/www-project-top-ten/)
[![Architecture](https://img.shields.io/badge/Architecture-Hexagonal-orange.svg)]()

Módulo central reutilizable para gestión de usuarios, JWT, roles y seguridad basada en políticas con arquitectura hexagonal y cumplimiento OWASP TOP 10.

---

## 📋 Índice

1. [¿Qué es AuthCore?](#-qué-es-authcore)
2. [Arquitectura](#-arquitectura)
3. [Endpoints](#-endpoints)
4. [Seguridad](#-seguridad)
5. [Configuración](#-configuración)
6. [Guía Ultra Detallada de Despliegue](#-guía-ultra-detallada-de-despliegue)
   - [Prerrequisitos Paso a Paso](#prerrequisitos-paso-a-paso)
   - [Despliegue en Local (Paso a Paso)](#despliegue-en-local-paso-a-paso)
   - [Despliegue con Docker (Paso a Paso)](#despliegue-con-docker-paso-a-paso)
   - [Despliegue en Kubernetes (Paso a Paso)](#despliegue-en-kubernetes-paso-a-paso)
   - [Troubleshooting Común](#troubleshooting-común)
7. [Integración](#-integración)
8. [Monitoreo](#-monitoreo)
9. [Testing](#-testing)

---

## 🎯 ¿Qué es AuthCore?

AuthCore es una **API REST de autenticación y autorización** diseñada para ser integrada en cualquier aplicación que necesite:

- ✅ **Gestión de usuarios**: Crear, listar, actualizar y eliminar usuarios
- ✅ **Autenticación segura**: Login con JWT, refresh tokens y logout
- ✅ **Autorización basada en roles (RBAC)**: Roles y permisos granulares
- ✅ **Multitenancy**: Aislamiento de datos por cliente/empresa
- ✅ **Seguridad OWASP TOP 10**: Protección contra vulnerabilidades comunes

**¿Para quién es este proyecto?**
- Desarrolladores Junior que quieren aprender a desplegar una API REST profesional
- Equipos que necesitan un módulo de autenticación reutilizable
- Arquitectos que buscan implementar arquitectura hexagonal en Spring Boot

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

## 🚀 Guía Ultra Detallada de Despliegue

Esta sección está diseñada para **desarrolladores Junior** o personas que están aprendiendo. Cada paso está explicado en detalle para que puedas desplegar la API REST sin problemas.

### Prerrequisitos Paso a Paso

Antes de comenzar, necesitas instalar las siguientes herramientas en tu computadora:

#### 1. Java JDK 17+

**¿Qué es Java?** El lenguaje de programación en el que está construida esta API.

**Verificar si ya lo tienes instalado:**
```bash
java -version
```

**Si NO lo tienes instalado:**

- **Windows:**
  1. Descarga desde [Oracle JDK](https://www.oracle.com/java/technologies/downloads/#jdk17-windows) o [OpenJDK](https://adoptium.net/)
  2. Ejecuta el instalador y sigue los pasos
  3. Agrega Java al PATH (el instalador usualmente lo hace automático)
  4. Abre una nueva terminal y verifica: `java -version`

- **macOS:**
  ```bash
  # Si tienes Homebrew instalado
  brew install openjdk@17
  
  # Agrega al PATH (agrega esto a tu ~/.zshrc o ~/.bash_profile)
  echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
  source ~/.zshrc
  ```

- **Linux (Ubuntu/Debian):**
  ```bash
  sudo apt update
  sudo apt install openjdk-17-jdk
  java -version
  ```

**✅ Verificación exitosa:** Deberías ver algo como `openjdk version "17.0.x"`

#### 2. Maven 3.8+

**¿Qué es Maven?** Herramienta que gestiona las dependencias (librerías) y compila el proyecto.

**Verificar si ya lo tienes:**
```bash
mvn -version
```

**Si NO lo tienes instalado:**

- **Windows:**
  1. Descarga desde [Apache Maven](https://maven.apache.org/download.cgi)
  2. Extrae el ZIP en `C:\Program Files\Maven`
  3. Agrega al PATH:
     - Click derecho en "Este equipo" → Propiedades → Configuración avanzada
     - Variables de entorno → Editar PATH → Nueva → `C:\Program Files\Maven\bin`
  4. Abre nueva terminal: `mvn -version`

- **macOS:**
  ```bash
  brew install maven
  mvn -version
  ```

- **Linux:**
  ```bash
  sudo apt install maven
  mvn -version
  ```

**✅ Verificación exitosa:** Deberías ver `Apache Maven 3.8.x`

#### 3. PostgreSQL 14+

**¿Qué es PostgreSQL?** Base de datos donde se guardará la información de usuarios, roles, etc.

**Opción A: Instalar PostgreSQL localmente**

- **Windows:**
  1. Descarga desde [PostgreSQL](https://www.postgresql.org/download/windows/)
  2. Instala con opciones por defecto
  3. Recuerda la contraseña que estableciste para el usuario `postgres`

- **macOS:**
  ```bash
  brew install postgresql@14
  brew services start postgresql@14
  ```

- **Linux:**
  ```bash
  sudo apt install postgresql-14 postgresql-contrib
  sudo systemctl start postgresql
  ```

**Opción B: Usar Docker (Recomendado para principiantes)**
```bash
docker run -d \
  --name authcore-db \
  -e POSTGRES_DB=authcore \
  -e POSTGRES_USER=authcore_user \
  -e POSTGRES_PASSWORD=ChangeMe123! \
  -p 5432:5432 \
  postgres:15-alpine
```

**Crear la base de datos manualmente (si instalaste PostgreSQL localmente):**
```bash
# Conéctate a PostgreSQL
psql -U postgres

# Dentro de psql, ejecuta:
CREATE DATABASE authcore;
CREATE USER authcore_user WITH PASSWORD 'ChangeMe123!';
GRANT ALL PRIVILEGES ON DATABASE authcore TO authcore_user;
\q
```

**✅ Verificación:** 
```bash
psql -h localhost -U authcore_user -d authcore
# Ingresa la contraseña y deberías conectarte exitosamente
```

#### 4. Git

**¿Qué es Git?** Sistema para descargar y gestionar el código fuente.

**Verificar:**
```bash
git --version
```

**Instalar:**
- Windows: [Descargar](https://git-scm.com/download/win)
- macOS: `xcode-select --install`
- Linux: `sudo apt install git`

#### 5. Docker (Opcional pero recomendado)

**¿Qué es Docker?** Permite ejecutar la aplicación en contenedores aislados.

**Verificar:**
```bash
docker --version
docker-compose --version
```

**Instalar:**
- Windows/macOS: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Linux: `sudo apt install docker.io docker-compose`

---

### Despliegue en Local (Paso a Paso)

Sigue estos pasos en orden para ejecutar la API en tu computadora:

#### Paso 1: Clonar el repositorio

```bash
# Navega a la carpeta donde guardarás el proyecto
cd ~/proyectos  # o cd C:\proyectos en Windows

# Clona el repositorio
git clone https://github.com/tu-organizacion/auth-core.git

# Entra al directorio del proyecto
cd auth-core
```

**¿Qué acabas de hacer?** Descargaste todo el código fuente del proyecto a tu computadora.

#### Paso 2: Verificar la estructura del proyecto

```bash
# Lista los archivos (Linux/Mac)
ls -la

# O en Windows
dir
```

Deberías ver:
```
README.md
pom.xml              ← Archivo de configuración de Maven
src/                 ← Código fuente
  main/
  test/
```

#### Paso 3: Configurar las variables de entorno

**Opción A: Crear archivo `.env` (Recomendado)**

Crea un archivo llamado `.env` en la raíz del proyecto:

```bash
# .env
SPRING_PROFILES_ACTIVE=dev
DB_HOST=localhost
DB_PORT=5432
DB_NAME=authcore
DB_USERNAME=authcore_user
DB_PASSWORD=ChangeMe123!
JWT_SECRET=cambia-esto-por-un-secreto-muy-largo-y-aleatorio-minimo-256-bits
JWT_EXPIRATION_MS=900000
MULTITENANT_STRATEGY=SCHEMA_PER_TENANT
```

**⚠️ IMPORTANTE:** Cambia `JWT_SECRET` por un valor único y seguro. Puedes generar uno con:
```bash
# En Linux/Mac
openssl rand -base64 32

# O usa este generador online: https://generate-secret.vercel.app/32
```

**Opción B: Exportar variables en la terminal**

```bash
export SPRING_PROFILES_ACTIVE=dev
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=authcore
export DB_USERNAME=authcore_user
export DB_PASSWORD=ChangeMe123!
export JWT_SECRET="tu-secreto-largo-y-aleatorio"
export JWT_EXPIRATION_MS=900000
export MULTITENANT_STRATEGY=SCHEMA_PER_TENANT
```

#### Paso 4: Compilar el proyecto

```bash
# Limpia compilaciones anteriores y descarga dependencias
mvn clean install -DskipTests
```

**¿Qué está pasando?**
- `clean`: Elimina archivos de compilaciones anteriores
- `install`: Descarga todas las librerías necesarias (Spring Boot, JWT, etc.) y compila el código
- `-DskipTests`: Omite las pruebas para agilizar el proceso

**Tiempo estimado:** 2-5 minutos (dependiendo de tu conexión a internet)

**✅ Éxito:** Verás `BUILD SUCCESS` al final

**❌ Error común:** Si ves `Could not resolve dependencies`, verifica tu conexión a internet.

#### Paso 5: Ejecutar migraciones de base de datos

Las migraciones crean las tablas necesarias en PostgreSQL:

```bash
mvn flyway:migrate -Dspring.profiles.active=dev
```

**¿Qué está pasando?** Flyway ejecuta scripts SQL que crean:
- Tabla `users` (usuarios)
- Tabla `roles` (roles)
- Tabla `permissions` (permisos)
- Tabla `tenants` (clientes/empresas)
- Tablas intermedias para relaciones

**✅ Éxito:** Verás `Successfully applied X migrations to database`

**❌ Error común:** Si ves `Connection refused`, verifica que PostgreSQL esté corriendo:
```bash
# Linux/Mac
sudo systemctl status postgresql

# Windows
# Abre el servicio "PostgreSQL" en Services.msc
```

#### Paso 6: Iniciar la aplicación

```bash
mvn spring-boot:run -Dspring.profiles.active=dev
```

**¿Qué está pasando?** Spring Boot inicia un servidor web embebido (Tomcat) en el puerto 8080.

**✅ Éxito:** Después de 30-60 segundos, verás algo como:
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.x)

AuthCoreApplication started in X.XXX seconds
```

**⚠️ No cierres la terminal** mientras quieras mantener la API corriendo.

#### Paso 7: Verificar que la API está funcionando

Abre **otra terminal** (no cierres la anterior) y ejecuta:

```bash
# Verificar salud de la aplicación
curl http://localhost:8080/actuator/health
```

**Respuesta esperada:**
```json
{"status":"UP"}
```

**¡Felicidades!** 🎉 La API está corriendo exitosamente.

#### Paso 8: Probar el endpoint de login

Primero, necesitas crear un usuario. Usa este comando:

```bash
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: tenant1" \
  -d '{
    "email": "admin@empresa.com",
    "password": "Admin123!",
    "username": "admin",
    "tenantId": "tenant1"
  }'
```

Ahora haz login:

```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: tenant1" \
  -d '{
    "email": "admin@empresa.com",
    "password": "Admin123!"
  }'
```

**Respuesta esperada:**
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

---

### Despliegue con Docker (Paso a Paso)

Docker simplifica el despliegue empaquetando todo lo necesario en contenedores.

#### Paso 1: Verificar que Docker está instalado

```bash
docker --version
docker-compose --version
```

**✅ Éxito:** Deberías ver versiones de ambos comandos.

#### Paso 2: Crear archivo `docker-compose.yml`

En la raíz del proyecto, crea un archivo llamado `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # Servicio de Base de Datos
  db:
    image: postgres:15-alpine
    container_name: authcore-db
    environment:
      POSTGRES_DB: authcore
      POSTGRES_USER: authcore_user
      POSTGRES_PASSWORD: ChangeMe123!
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - authcore-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U authcore_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Servicio de la Aplicación
  app:
    build: .
    container_name: authcore-api
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: authcore
      DB_USERNAME: authcore_user
      DB_PASSWORD: ChangeMe123!
      JWT_SECRET: ${JWT_SECRET:-cambia-esto-por-un-secreto-muy-largo}
      JWT_EXPIRATION_MS: 900000
      MULTITENANT_STRATEGY: SCHEMA_PER_TENANT
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - authcore-network
    restart: unless-stopped

networks:
  authcore-network:
    driver: bridge

volumes:
  postgres_data:
```

#### Paso 3: Crear archivo `Dockerfile`

En la raíz del proyecto, crea un archivo llamado `Dockerfile` (sin extensión):

```dockerfile
# Etapa 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copiar archivos de configuración
COPY pom.xml .
COPY src ./src

# Compilar el proyecto
RUN mvn clean package -DskipTests

# Etapa 2: Runtime
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copiar el JAR compilado desde la etapa de build
COPY --from=builder /app/target/*.jar app.jar

# Exponer puerto
EXPOSE 8080

# Comando para ejecutar la aplicación
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Paso 4: Construir y levantar los contenedores

```bash
# Construye las imágenes y levanta los contenedores
docker-compose up -d --build
```

**¿Qué está pasando?**
- `up`: Crea e inicia los contenedores
- `-d`: Modo detach (en segundo plano)
- `--build`: Fuerza la construcción de la imagen Docker

**Monitorear el progreso:**
```bash
# Ver logs en tiempo real
docker-compose logs -f app
```

**✅ Éxito:** Después de 2-3 minutos, verás:
```
app  | AuthCoreApplication started in X.XXX seconds
```

#### Paso 5: Verificar que todo está corriendo

```bash
# Listar contenedores
docker-compose ps

# Verificar salud de la API
curl http://localhost:8080/actuator/health
```

**Respuesta esperada:**
```json
{"status":"UP"}
```

#### Comandos útiles de Docker

```bash
# Detener todos los contenedores
docker-compose down

# Detener y eliminar volúmenes (¡CUIDADO! Borra la base de datos)
docker-compose down -v

# Ver logs
docker-compose logs app

# Reiniciar la aplicación
docker-compose restart app

# Escalar a múltiples instancias
docker-compose up -d --scale app=3
```

---

### Despliegue en Kubernetes (Paso a Paso)

Kubernetes es ideal para entornos de producción.

#### Paso 1: Prerrequisitos

- Cluster de Kubernetes (Minikube, kind, o un cluster cloud)
- kubectl configurado
- Registry de imágenes (Docker Hub, ECR, GCR, etc.)

#### Paso 2: Crear Namespace

```bash
kubectl create namespace authcore
```

#### Paso 3: Crear Secrets

```bash
# Crear secret con las credenciales de la base de datos
kubectl create secret generic authcore-db-secret \
  --from-literal=DB_USERNAME=authcore_user \
  --from-literal=DB_PASSWORD=ChangeMe123! \
  --from-literal=JWT_SECRET=$(openssl rand -base64 32) \
  -n authcore
```

#### Paso 4: Deploy de PostgreSQL

Crea un archivo `postgres-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authcore-db
  namespace: authcore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: authcore-db
  template:
    metadata:
      labels:
        app: authcore-db
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: authcore
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: authcore-db-secret
                  key: DB_USERNAME
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: authcore-db-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - authcore_user
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - authcore_user
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: authcore-db-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: authcore-db-pvc
  namespace: authcore
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: authcore-db
  namespace: authcore
spec:
  selector:
    app: authcore-db
  ports:
    - port: 5432
      targetPort: 5432
  clusterIP: None
```

#### Paso 5: Deploy de la Aplicación

Crea un archivo `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authcore
  namespace: authcore
spec:
  replicas: 3
  selector:
    matchLabels:
      app: authcore
  template:
    metadata:
      labels:
        app: authcore
    spec:
      containers:
        - name: authcore
          image: registry.example.com/authcore:v1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: DB_HOST
              value: authcore-db
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: authcore
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: authcore-db-secret
                  key: DB_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: authcore-db-secret
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: authcore-db-secret
                  key: JWT_SECRET
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
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: authcore-service
  namespace: authcore
spec:
  selector:
    app: authcore
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

#### Paso 6: Aplicar los deployments

```bash
# Aplicar PostgreSQL
kubectl apply -f postgres-deployment.yaml

# Esperar a que PostgreSQL esté listo
kubectl wait --for=condition=available deployment/authcore-db -n authcore --timeout=300s

# Aplicar la aplicación
kubectl apply -f app-deployment.yaml

# Verificar estado
kubectl get pods -n authcore
kubectl get svc -n authcore
```

#### Paso 7: Acceder a la API

```bash
# Obtener la IP externa
kubectl get svc authcore-service -n authcore

# La API estará disponible en http://<EXTERNAL-IP>/api/v1
```

---

### Troubleshooting Común

#### Error: "Connection refused to localhost:5432"

**Causa:** PostgreSQL no está corriendo.

**Solución:**
```bash
# Verificar estado del servicio
# Linux
sudo systemctl status postgresql

# Mac
brew services list

# Si está detenido, iniciarlo
sudo systemctl start postgresql  # Linux
brew services start postgresql   # Mac
```

#### Error: "Could not find or load main class"

**Causa:** Problema con la compilación de Maven.

**Solución:**
```bash
# Limpiar y recompilar
mvn clean
mvn install -DskipTests

# Verificar que el JAR se creó
ls -la target/*.jar
```

#### Error: "Port 8080 already in use"

**Causa:** Otro proceso está usando el puerto 8080.

**Solución:**
```bash
# Encontrar el proceso
# Linux/Mac
lsof -i :8080
sudo kill -9 <PID>

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F

# O cambiar el puerto en application.yml
server:
  port: 8081
```

#### Error: "JWT_SECRET is too short"

**Causa:** El secreto JWT debe tener al menos 256 bits (32 caracteres).

**Solución:**
```bash
# Generar un secreto seguro
openssl rand -base64 32

# Actualizar la variable de entorno
export JWT_SECRET="el-valor-generado-arriba"
```

#### Error: "Flyway migration failed"

**Causa:** La base de datos ya tiene migraciones aplicadas o hay un conflicto.

**Solución:**
```bash
# Opción 1: Reparar Flyway
mvn flyway:repair -Dspring.profiles.active=dev

# Opción 2: Resetear la base de datos (SOLO EN DESARROLLO)
psql -U authcore_user -d authcore -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
mvn flyway:migrate -Dspring.profiles.active=dev
```

#### La aplicación tarda mucho en iniciar

**Causa normal:** Spring Boot carga muchas librerías y configura componentes.

**Tiempo esperado:** 30-90 segundos dependiendo de tu hardware.

**Optimización:** Agrega esto a tu `pom.xml` para build más rápido:
```xml
<properties>
    <maven.compiler.parameters>true</maven.compiler.parameters>
</properties>
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
            .header("X-Tenant-ID", "tenant1")
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

// Usar el token en peticiones posteriores
client.defaults.headers.common['Authorization'] = `Bearer ${response.data.accessToken}`;
```

### Python (Requests)

```python
import requests

# Login
response = requests.post(
    'http://localhost:8080/api/v1/auth/login',
    headers={'X-Tenant-ID': 'tenant1'},
    json={'email': 'user@empresa.com', 'password': 'P@ssw0rd123!'}
)

token = response.json()['accessToken']

# Usar el token en peticiones posteriores
headers = {
    'Authorization': f'Bearer {token}',
    'X-Tenant-ID': 'tenant1'
}
users = requests.get('http://localhost:8080/api/v1/users', headers=headers)
```

### cURL (Para testing rápido)

```bash
# Login y guardar el token en una variable
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: tenant1" \
  -d '{"email":"admin@empresa.com","password":"Admin123!"}' \
  | jq -r '.accessToken')

# Usar el token para listar usuarios
curl -X GET http://localhost:8080/api/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant-ID: tenant1"
```

---

## 📊 Monitoreo

**Actuator Endpoints:**
- `/actuator/health` - Salud general
- `/actuator/health/liveness` - Liveness probe (K8s)
- `/actuator/health/readiness` - Readiness probe (K8s)
- `/actuator/metrics` - Métricas Prometheus
- `/actuator/loggers` - Ver/modificar niveles de log

**Métricas clave:**
- `auth_login_total` - Total de logins
- `auth_login_failure_total` - Logins fallidos
- `auth_token_validation_total` - Validaciones de token
- `http_server_requests_seconds` - Tiempo de respuesta HTTP

**Configurar Prometheus (docker-compose.yml):**

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

---

## 🧪 Testing

### Ejecutar tests

```bash
# Tests unitarios (rápidos, sin base de datos)
mvn test

# Tests de integración (requieren PostgreSQL corriendo)
mvn verify

# Test específico
mvn test -Dtest=AuthenticateUserUseCaseTest

# Generar reporte de cobertura
mvn jacoco:report

# Abrir reporte en navegador (Linux/Mac)
xdg-open target/site/jacoco/index.html  # Linux
open target/site/jacoco/index.html       # Mac
```

### Cobertura de código

El proyecto requiere mínimo **80% de cobertura**:

```bash
# Verificar cobertura
mvn checkstyle:check

# Reporte detallado
mvn jacoco:report
```

**Estructura de tests:**
- `src/test/java/com/authcore/domain/` - Tests de entidades y objetos de valor
- `src/test/java/com/authcore/application/` - Tests de casos de uso
- `src/test/java/com/authcore/infrastructure/` - Tests de integración
- `src/test/resources/` - Datos de prueba y configuraciones

---

## 📄 Licencia

MIT License - Libre uso para proyectos personales y comerciales.

---

## 🤝 Contribuciones

1. Fork el repositorio
2. Crea una rama (`git checkout -b feature/nueva-funcionalidad`)
3. Commit tus cambios (`git commit -m 'Agrega nueva funcionalidad'`)
4. Push a la rama (`git push origin feature/nueva-funcionalidad`)
5. Abre un Pull Request

**Convenciones de commits:**
- `feat:` Nueva funcionalidad
- `fix:` Corrección de bug
- `docs:` Cambios en documentación
- `test:` Agregar/modificar tests
- `refactor:` Refactorización de código

---

## 📚 Recursos Adicionales

- [Documentación de Arquitectura](./authcore/README.md) - Guía completa de arquitectura hexagonal
- [Estructura del Proyecto](./authcore/STRUCTURE_SUMMARY.md) - Resumen de carpetas y archivos
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [OWASP TOP 10](https://owasp.org/www-project-top-ten/)
- [JWT.io](https://jwt.io/) - Herramienta para debuggear tokens JWT

---

<div align="center">

**AuthCore** - Módulo de seguridad reutilizable para cualquier proyecto

[⬆️ Volver al inicio](#-authcore---api-rest-multitenant-de-autenticación-y-autorización)

</div>

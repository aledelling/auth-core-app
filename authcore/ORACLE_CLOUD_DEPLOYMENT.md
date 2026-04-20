# 📦 Guía Completa: Despliegue de AuthCore en Oracle Cloud Free Tier

## 🎯 Objetivo de esta Guía

Esta documentación te llevará paso a paso desde **cero absoluto** hasta tener tu API REST AuthCore funcionando en un servidor gratuito de Oracle Cloud, accesible desde internet con HTTPS, base de datos PostgreSQL, y todas las mejores prácticas de producción.

### ¿Qué lograrás al finalizar?

✅ Una VM (máquina virtual) gratuita en Oracle Cloud  
✅ Java 17+ instalado y configurado  
✅ PostgreSQL instalado y asegurado  
✅ Tu aplicación AuthCore desplegándose automáticamente  
✅ Dominio personalizado con HTTPS gratuito  
✅ Monitoreo básico del servidor  
✅ Backups automáticos de la base de datos  

---

## 📋 Índice General

1. [Prerrequisitos](#-prerrequisitos)
2. [Creación de Cuenta en Oracle Cloud](#-creación-de-cuenta-en-oracle-cloud)
3. [Configuración del Servidor (VM)](#-configuración-del-servidor-vm)
4. [Instalación de Software Base](#-instalación-de-software-base)
5. [Configuración de PostgreSQL](#-configuración-de-postgresql)
6. [Despliegue de la Aplicación](#-despliegue-de-la-aplicación)
7. [Configuración de Dominio y HTTPS](#-configuración-de-dominio-y-https)
8. [Automatización con Systemd](#-automatización-con-systemd)
9. [Monitoreo y Logs](#-monitoreo-y-logs)
10. [Backups Automáticos](#-backups-automáticos)
11. [Troubleshooting](#-troubleshooting)
12. [Consideraciones de Seguridad](#-consideraciones-de-seguridad)

---

## 🛠️ Prerrequisitos

### Lo que necesitas antes de empezar:

| Requisito | Descripción | ¿Es obligatorio? |
|-----------|-------------|------------------|
| Tarjeta de crédito/débito | Para verificar identidad (no se cobra nada) | ✅ Sí |
| Correo electrónico válido | Para recibir notificaciones de Oracle | ✅ Sí |
| Número de teléfono | Para verificación por SMS | ✅ Sí |
| Conocimientos básicos de terminal Linux | Saber ejecutar comandos simples | ✅ Sí |
| Tiempo estimado | 2-3 horas para completar todo | ✅ Sí |
| Proyecto AuthCore | Tu código ya desarrollado | ✅ Sí |

### ¿Por qué piden tarjeta si es gratis?

Oracle requiere verificar que eres una persona real. **No te cobrarán nada** mientras te mantengas dentro de los límites del tier gratuito:
- 2 VMs con hasta 4 CPUs ARM y 24 GB RAM (o 1 VM AMD con 1 CPU y 1 GB RAM)
- 200 GB de almacenamiento
- 10 TB de transferencia de datos al mes

---

## 🌟 Creación de Cuenta en Oracle Cloud

### Paso 1: Registrarse en Oracle Cloud Free Tier

1. **Ve a la página oficial**: https://www.oracle.com/cloud/free/
2. **Haz clic en "Start for free"**
3. **Selecciona tu país/región** (ej: México, Colombia, España, Argentina)
4. **Completa el formulario**:
   - Nombre completo (como aparece en tu identificación)
   - Correo electrónico
   - Número de teléfono
   - Contraseña segura (mínimo 8 caracteres, mayúscula, número y símbolo)

### Paso 2: Verificación de Identidad

1. **Verifica tu email**: Oracle te enviará un código de 6 dígitos
2. **Verifica tu teléfono**: Recibirás un SMS con otro código
3. **Ingresa tus datos de pago**:
   - Número de tarjeta (crédito o débito)
   - Fecha de expiración
   - CVV (los 3 dígitos atrás)
   - Dirección de facturación

> ⚠️ **IMPORTANTE**: Oracle hará un cargo temporal de $1 USD que se reembolsa inmediatamente. Es solo para verificar que la tarjeta es válida.

### Paso 3: Esperar Aprobación

- **Tiempo típico**: 15 minutos a 24 horas
- **Notificación**: Recibirás un email cuando tu cuenta esté activa
- **Si es rechazado**: Contacta a soporte de Oracle Cloud

### Paso 4: Primer Inicio de Sesión

1. Ve a https://cloud.oracle.com/
2. Ingresa tu **Cloud Account Name** (te lo dieron por email)
3. Ingresa tu **username** (usualmente `identity.domainname`)
4. Ingresa tu contraseña

---

## 🖥️ Configuración del Servidor (VM)

### Paso 1: Acceder al Dashboard de Compute

1. Una vez dentro de Oracle Cloud, haz clic en el menú hamburguesa (☰)
2. Navega a **Compute** → **Instances**
3. Haz clic en **"Create Instance"**

### Paso 2: Seleccionar Configuración de la VM

#### 2.1 Información Básica

```
Name: authcore-production
Compartment in: root compartment
Availability Domain: Cualquiera (AD-1, AD-2, o AD-3)
Image: Oracle Linux 8.x o Ubuntu 22.04 LTS (recomendado Ubuntu)
Shape: 
  - Cambiar shape → Always Free ARM Ampere
  - OCPUs: 4 (máximo gratuito)
  - Memory: 24 GB (máximo gratuito)
```

> 💡 **¿Por qué ARM?** Las instancias ARM Ampere ofrecen mejor rendimiento por watt y son completamente gratuitas. Tu aplicación Java funcionará perfectamente.

#### 2.2 Configuración de Red

```
Virtual cloud network: 
  - Click en "Edit"
  - Subnet: Selecciona una subnet pública
  
SSH keys:
  - Generar un par de claves SSH
  - Descarga la clave privada (authcore-key.pem)
  - GUÁRDALA EN UN LUGAR SEGURO (la necesitarás siempre)
```

**Comando para proteger tu clave SSH** (en tu computadora local):
```bash
chmod 400 ~/Downloads/authcore-key.pem
```

#### 2.3 Almacenamiento

```
Boot Volume Size: 50 GB (suficiente para OS + app + DB)
Enable automatic backup: ✅ Sí (recomendado)
```

#### 2.4 Script de Inicio (Opcional pero recomendado)

En la sección **"Add SSH keys"**, pega este script en **"Paste your public key here"** o configúralo después:

```bash
#!/bin/bash
# Este script se ejecuta automáticamente al crear la VM

# Actualizar sistema
dnf update -y || apt update && apt upgrade -y

# Instalar herramientas básicas
dnf install -y vim wget curl git net-tools || apt install -y vim wget curl git net-tools

echo "✅ Configuración inicial completada"
```

### Paso 3: Crear la Instancia

1. Revisa toda la configuración
2. Haz clic en **"Create"**
3. Espera 2-5 minutos mientras se crea la VM
4. Cuando el estado sea **"Running"**, anota:
   - **Public IP Address** (ej: 129.146.85.234)
   - **Instance Name** (authcore-production)

### Paso 4: Configurar Reglas de Firewall en Oracle

1. En el dashboard de tu instancia, haz clic en **"Subnet"**
2. Luego en **"Security Lists"**
3. Edita las reglas de ingreso (**Ingress Rules**) y agrega:

| Tipo | Puerto | Descripción |
|------|--------|-------------|
| TCP | 22 | SSH (administración remota) |
| TCP | 8080 | HTTP (aplicación Spring Boot) |
| TCP | 443 | HTTPS (si usas reverse proxy) |
| TCP | 5432 | PostgreSQL (solo si necesitas acceso remoto) |

**Ejemplo de regla**:
```
Source Type: CIDR
Source CIDR: 0.0.0.0/0
Destination Port Range: 8080
Description: Spring Boot Application
```

> ⚠️ **Seguridad**: Para PostgreSQL (puerto 5432), NUNCA abras al público (0.0.0.0/0). Usa solo IPs específicas o deja cerrado y accede vía SSH tunnel.

---

## 🔧 Instalación de Software Base

### Conexión Inicial al Servidor

Desde tu computadora local:

```bash
# Para Oracle Linux
ssh -i ~/Downloads/authcore-key.pem opc@TU_IP_PUBLICA

# Para Ubuntu
ssh -i ~/Downloads/authcore-key.pem ubuntu@TU_IP_PUBLICA
```

**Ejemplo real**:
```bash
ssh -i ~/Downloads/authcore-key.pem opc@129.146.85.234
```

> 💡 Si es la primera vez conectándote, escribe `yes` cuando pregunte sobre la autenticidad del host.

### Actualización del Sistema

```bash
# Para Oracle Linux
sudo dnf update -y

# Para Ubuntu
sudo apt update && sudo apt upgrade -y
```

### Instalación de Java 17 (OpenJDK)

AuthCore requiere Java 17+. Vamos a instalar OpenJDK 17:

```bash
# Para Oracle Linux
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Para Ubuntu
sudo apt install -y openjdk-17-jdk openjdk-17-jre
```

**Verificar instalación**:
```bash
java -version
```

**Salida esperada**:
```
openjdk version "17.0.x" 202x-xx-xx
OpenJDK Runtime Environment (build 17.0.x+xx)
OpenJDK 64-Bit Server VM (build 17.0.x+xx, mixed mode, sharing)
```

**Configurar JAVA_HOME**:

```bash
# Encontrar ruta de Java
JAVA_PATH=$(readlink -f $(which java) | sed "s:/bin/java::")

# Agregar al perfil del usuario
echo "export JAVA_HOME=$JAVA_PATH" | sudo tee -a /etc/profile.d/java.sh
echo "export PATH=\$PATH:\$JAVA_HOME/bin" | sudo tee -a /etc/profile.d/java.sh

# Hacer efectivo el cambio
source /etc/profile.d/java.sh

# Verificar
echo $JAVA_HOME
```

### Instalación de Maven (para compilar en el servidor)

```bash
# Para Oracle Linux
sudo dnf install -y maven

# Para Ubuntu
sudo apt install -y maven
```

**Verificar instalación**:
```bash
mvn -version
```

**Salida esperada**:
```
Apache Maven 3.8.x (...)
Maven home: /usr/share/maven
Java version: 17.0.x, vendor: ...
```

> 💡 **Alternativa**: Puedes compilar tu aplicación en tu máquina local y subir solo el JAR. Esto ahorra recursos en el servidor.

### Instalación de Git

```bash
# Para Oracle Linux
sudo dnf install -y git

# Para Ubuntu
sudo apt install -y git
```

**Verificar**:
```bash
git --version
```

### Instalación de Herramientas Útiles

```bash
# Para Oracle Linux
sudo dnf install -y vim wget curl net-tools telnet unzip htop iotop

# Para Ubuntu
sudo apt install -y vim wget curl net-tools telnet unzip htop iotop
```

---

## 🐘 Configuración de PostgreSQL

### Instalación de PostgreSQL 15

```bash
# Para Oracle Linux
sudo dnf install -y postgresql-server postgresql-contrib

# Inicializar la base de datos
sudo postgresql-setup --initdb

# Para Ubuntu
sudo apt install -y postgresql postgresql-contrib
```

### Iniciar y Habilitar PostgreSQL

```bash
# Iniciar servicio
sudo systemctl start postgresql

# Habilitar inicio automático
sudo systemctl enable postgresql

# Verificar estado
sudo systemctl status postgresql
```

**Salida esperada**:
```
● postgresql.service - PostgreSQL RDBMS
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled)
   Active: active (running) since ...
```

### Configuración de Seguridad de PostgreSQL

#### 1. Cambiar contraseña del usuario postgres

```bash
sudo -u postgres psql
```

Dentro de psql:
```sql
\password postgres
-- Ingresa una contraseña SEGURA (mínimo 16 caracteres)
\q
```

#### 2. Crear usuario y base de datos para AuthCore

```bash
sudo -u postgres psql
```

```sql
-- Crear base de datos
CREATE DATABASE authcore_db 
    WITH 
    ENCODING = 'UTF8' 
    LC_COLLATE = 'en_US.UTF-8' 
    LC_CTYPE = 'en_US.UTF-8' 
    TEMPLATE = template0;

-- Crear usuario dedicado (NUNCA uses postgres para la app)
CREATE USER authcore_user WITH 
    PASSWORD 'TuContraseñaMuySegura123!' 
    CREATEDB 
    NOSUPERUSER;

-- Otorgar permisos
GRANT ALL PRIVILEGES ON DATABASE authcore_db TO authcore_user;

-- Conectar a la base de datos
\c authcore_db

-- Otorgar permisos en el schema público
GRANT ALL ON SCHEMA public TO authcore_user;
GRANT ALL ON ALL TABLES IN SCHEMA public TO authcore_user;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO authcore_user;

-- Salir
\q
```

#### 3. Configurar autenticación (pg_hba.conf)

```bash
# Editar archivo de configuración
sudo vim /var/lib/pgsql/data/pg_hba.conf
# O en Ubuntu: sudo vim /etc/postgresql/15/main/pg_hba.conf
```

Cambiar las líneas de autenticación:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Conexiones locales (desde el mismo servidor)
local   all             all                                     md5

# Conexiones IPv4 locales
host    all             all             127.0.0.1/32            md5

# Conexiones IPv6 locales
host    all             all             ::1/128                 md5

# ❌ NO permitir conexiones remotas sin autenticación fuerte
# host    all             all             0.0.0.0/0               md5
```

> ⚠️ **IMPORTANTE**: No habilites conexiones remotas (0.0.0.0/0) a menos que sea absolutamente necesario. Si lo haces, usa firewall restrictivo.

#### 4. Configurar puerto y listening (postgresql.conf)

```bash
sudo vim /var/lib/pgsql/data/postgresql.conf
# O en Ubuntu: sudo vim /etc/postgresql/15/main/postgresql.conf
```

Buscar y modificar:

```conf
# Escuchar solo en localhost (más seguro)
listen_addresses = 'localhost'

# Puerto por defecto (puedes cambiarlo si quieres)
port = 5432

# Máximo de conexiones (ajustar según RAM disponible)
max_connections = 100

# Memoria compartida (25% de RAM total está bien)
shared_buffers = 6GB  # Para 24GB RAM

# Logs
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'ddl'
log_min_duration_statement = 1000  # Log queries > 1 segundo
```

#### 5. Reiniciar PostgreSQL

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

#### 6. Probar conexión local

```bash
psql -h localhost -U authcore_user -d authcore_db
```

Te pedirá la contraseña que configuraste. Si conecta exitosamente, verás:

```
psql (15.x)
Type "help" for help.

authcore_db=> 
```

Salir con `\q`

### Crear Tablas Iniciales para AuthCore

Crea un script SQL inicial:

```bash
vim ~/authcore-schema.sql
```

Contenido mínimo requerido:

```sql
-- Tabla de tenants (empresas)
CREATE TABLE IF NOT EXISTS tenants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    subdomain VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    active BOOLEAN DEFAULT TRUE
);

-- Tabla de usuarios
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, username),
    UNIQUE(tenant_id, email)
);

-- Tabla de roles
CREATE TABLE IF NOT EXISTS roles (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, name)
);

-- Tabla intermedia user_roles
CREATE TABLE IF NOT EXISTS user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

-- Tabla de permisos
CREATE TABLE IF NOT EXISTS permissions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    resource VARCHAR(100) NOT NULL,
    action VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, name)
);

-- Tabla intermedia role_permissions
CREATE TABLE IF NOT EXISTS role_permissions (
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Tabla de refresh tokens
CREATE TABLE IF NOT EXISTS refresh_tokens (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(500) NOT NULL UNIQUE,
    expires_at TIMESTAMP NOT NULL,
    revoked BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para mejorar rendimiento
CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_roles_tenant ON roles(tenant_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
CREATE INDEX idx_refresh_tokens_expires ON refresh_tokens(expires_at);

-- Insertar tenant de prueba
INSERT INTO tenants (name, subdomain) VALUES ('Demo Tenant', 'demo');

-- Insertar usuario admin de prueba (password: Admin123!)
-- El hash es bcrypt de "Admin123!"
INSERT INTO users (tenant_id, username, email, password_hash, first_name, last_name) 
VALUES (1, 'admin', 'admin@demo.com', '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', 'Admin', 'User');

-- Insertar roles básicos
INSERT INTO roles (tenant_id, name, description) VALUES 
(1, 'ROLE_ADMIN', 'Administrador con acceso completo'),
(1, 'ROLE_USER', 'Usuario estándar');

-- Asignar rol admin al usuario admin
INSERT INTO user_roles (user_id, role_id) VALUES (1, 1);

-- Insertar permisos básicos
INSERT INTO permissions (tenant_id, name, resource, action) VALUES
(1, 'users.read', 'users', 'read'),
(1, 'users.write', 'users', 'write'),
(1, 'users.delete', 'users', 'delete'),
(1, 'roles.read', 'roles', 'read'),
(1, 'roles.write', 'roles', 'write');

-- Asignar todos los permisos al rol admin
INSERT INTO role_permissions (role_id, permission_id) 
SELECT 1, id FROM permissions WHERE tenant_id = 1;
```

Ejecutar el script:

```bash
sudo -u postgres psql -d authcore_db -f ~/authcore-schema.sql
```

---

## 🚀 Despliegue de la Aplicación

### Opción A: Compilar y Desplegar desde el Servidor

#### 1. Clonar el Repositorio

```bash
cd /opt
sudo git clone https://github.com/tu-usuario/authcore.git
sudo chown -R $USER:$USER authcore
cd authcore
```

> 💡 Si tu repositorio es privado, genera un token en GitHub/GitLab y úsalo:
> ```bash
> git clone https://TOKEN@github.com/tu-usuario/authcore.git
> ```

#### 2. Compilar con Maven

```bash
mvn clean package -DskipTests
```

**¿Qué está pasando?**
- `clean`: Elimina compilaciones anteriores
- `package`: Crea el archivo JAR ejecutable
- `-DskipTests`: Omite tests para acelerar (ya deberías haberlos pasado en local)

El JAR se creará en: `target/authcore-1.0.0.jar`

#### 3. Configurar Variables de Entorno

Crea un archivo de configuración:

```bash
sudo mkdir -p /etc/authcore
sudo vim /etc/authcore/application.properties
```

Contenido:

```properties
# ===========================================
# CONFIGURACIÓN DE AUTHCORE PARA PRODUCCIÓN
# ===========================================

# Puerto de la aplicación
server.port=8080

# ===========================================
# BASE DE DATOS POSTGRESQL
# ===========================================
spring.datasource.url=jdbc:postgresql://localhost:5432/authcore_db
spring.datasource.username=authcore_user
spring.datasource.password=TuContraseñaMuySegura123!
spring.datasource.driver-class-name=org.postgresql.Driver

# Pool de conexiones (HikariCP)
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000

# ===========================================
# JPA / HIBERNATE
# ===========================================
# VALIDAR que las tablas existen (no crear/modificar automáticamente)
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# ===========================================
# JWT CONFIGURATION
# ===========================================
# Genera un secreto seguro con: openssl rand -base64 64
authcore.jwt.secret=tu-secreto-muy-seguro-minimo-64-caracteres-aleatorios-generados-con-openssl
authcore.jwt.expiration-ms=3600000
authcore.jwt.refresh-expiration-ms=604800000
authcore.jwt.issuer=authcore-production
authcore.jwt.audience=authcore-clients

# ===========================================
# MULTITENANCY
# ===========================================
# Header HTTP para identificar tenant
authcore.tenant.header-name=X-Tenant-ID
# O usar subdominio: demo.authcore.com -> tenant: demo
authcore.tenant.resolution-method=header

# ===========================================
# LOGGING
# ===========================================
logging.level.root=INFO
logging.level.com.authcore=INFO
logging.level.org.springframework.security=WARN
logging.file.name=/var/log/authcore/application.log
logging.file.max-size=10MB
logging.file.max-history=10
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n

# ===========================================
# ACTUATOR (MONITOREO)
# ===========================================
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when_authorized
management.endpoint.health.probes.enabled=true
management.metrics.tags.application=authcore

# ===========================================
# SEGURIDAD SPRING
# ===========================================
# URLs públicas (sin autenticación)
authcore.security.public-urls=/api/auth/login,/api/auth/refresh,/api/auth/register,/actuator/health,/actuator/info

# ===========================================
# RATE LIMITING (opcional)
# ===========================================
authcore.rate-limiting.enabled=true
authcore.rate-limiting.requests-per-minute=60
authcore.rate-limiting.burst-size=10
```

#### 4. Crear Usuario y Directorios para la Aplicación

```bash
# Crear usuario dedicado para la app (mejor práctica de seguridad)
sudo useradd -r -s /bin/false authcore

# Crear directorios necesarios
sudo mkdir -p /var/log/authcore
sudo mkdir -p /opt/authcore

# Copiar JAR
sudo cp target/authcore-1.0.0.jar /opt/authcore/

# Establecer permisos
sudo chown -R authcore:authcore /opt/authcore
sudo chown -R authcore:authcore /var/log/authcore
sudo chmod 755 /opt/authcore/authcore-1.0.0.jar
```

#### 5. Crear Script de Inicio

```bash
sudo vim /opt/authcore/start.sh
```

Contenido:

```bash
#!/bin/bash

# ===========================================
# SCRIPT DE INICIO PARA AUTHCORE
# ===========================================

APP_NAME="authcore"
APP_DIR="/opt/authcore"
JAR_FILE="$APP_DIR/authcore-1.0.0.jar"
CONFIG_FILE="/etc/authcore/application.properties"
LOG_DIR="/var/log/authcore"
LOG_FILE="$LOG_DIR/application.log"

# Variables de entorno
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export SPRING_CONFIG_LOCATION="file:$CONFIG_FILE"

# Verificar que el JAR existe
if [ ! -f "$JAR_FILE" ]; then
    echo "❌ Error: El archivo JAR no existe en $JAR_FILE"
    exit 1
fi

# Verificar que Java está disponible
if ! command -v java &> /dev/null; then
    echo "❌ Error: Java no está instalado"
    exit 1
fi

# Crear directorio de logs si no existe
mkdir -p "$LOG_DIR"

echo "🚀 Iniciando $APP_NAME..."
echo "📍 Directorio: $APP_DIR"
echo "📦 JAR: $JAR_FILE"
echo "⚙️  Config: $CONFIG_FILE"
echo "📝 Logs: $LOG_FILE"

# Ejecutar aplicación
cd "$APP_DIR"
java -Xms512m -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/authcore/heapdump.hprof \
     -jar "$JAR_FILE" \
     --spring.config.location="$CONFIG_FILE" \
     >> "$LOG_FILE" 2>&1
```

Hacer ejecutable:

```bash
sudo chmod +x /opt/authcore/start.sh
sudo chown authcore:authcore /opt/authcore/start.sh
```

---

### Opción B: Despliegue con Docker (Recomendado)

#### 1. Instalar Docker

```bash
# Para Oracle Linux
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker
sudo systemctl start docker

# Para Ubuntu
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER
newgrp docker
```

Verificar:
```bash
docker --version
```

#### 2. Crear Dockerfile

En la raíz de tu proyecto AuthCore:

```dockerfile
# ===========================================
# DOCKERFILE PARA AUTHCORE
# ===========================================

# Etapa 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copiar pom.xml y descargar dependencias (cache layer)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copiar código fuente y compilar
COPY src ./src
RUN mvn clean package -DskipTests -B

# Etapa 2: Runtime
FROM eclipse-temurin:17-jre-alpine

# Instalar herramientas útiles para debugging
RUN apk add --no-cache curl bind-tools

WORKDIR /app

# Crear usuario no-root (seguridad)
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Copiar JAR desde etapa de build
COPY --from=builder --chown=appuser:appgroup /app/target/*.jar app.jar

# Exponer puerto
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# Ejecutar como usuario no-root
USER appuser

# Entrypoint
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### 3. Crear docker-compose.yml

```yaml
version: '3.8'

services:
  # ===========================================
  # SERVICIO: AUTHCORE API
  # ===========================================
  authcore:
    build: .
    container_name: authcore-api
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/authcore_db
      - SPRING_DATASOURCE_USERNAME=authcore_user
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - AUTHCORE_JWT_SECRET=${JWT_SECRET}
      - AUTHCORE_JWT_EXPIRATION_MS=3600000
      - AUTHCORE_TENANT_RESOLUTION_METHOD=header
    volumes:
      - authcore-logs:/var/log/authcore
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - authcore-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

  # ===========================================
  # SERVICIO: POSTGRESQL
  # ===========================================
  postgres:
    image: postgres:15-alpine
    container_name: authcore-db
    environment:
      - POSTGRES_DB=authcore_db
      - POSTGRES_USER=authcore_user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"  # Solo para desarrollo, en producción comentar
    networks:
      - authcore-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U authcore_user -d authcore_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G

  # ===========================================
  # SERVICIO: NGINX (Reverse Proxy)
  # ===========================================
  nginx:
    image: nginx:alpine
    container_name: authcore-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
    depends_on:
      - authcore
    networks:
      - authcore-network
    restart: unless-stopped

networks:
  authcore-network:
    driver: bridge

volumes:
  postgres-data:
  authcore-logs:
  nginx-logs:
```

#### 4. Crear archivo .env

```bash
vim .env
```

Contenido:

```env
# ===========================================
# VARIABLES DE ENTORNO PARA DOCKER-COMPOSE
# ===========================================

# Base de datos
DB_PASSWORD=TuContraseñaMuySegura123!

# JWT Secret (generar con: openssl rand -base64 64)
JWT_SECRET=tu-secreto-muy-seguro-minimo-64-caracteres-aleatorios

# Perfil de Spring
SPRING_PROFILES_ACTIVE=production
```

#### 5. Construir y Levantar Contenedores

```bash
# Construir imágenes
docker-compose build

# Iniciar servicios
docker-compose up -d

# Ver logs en tiempo real
docker-compose logs -f

# Ver estado
docker-compose ps
```

---

## 🔒 Automatización con Systemd

Para que AuthCore inicie automáticamente al reiniciar el servidor:

### Crear Servicio Systemd

```bash
sudo vim /etc/systemd/system/authcore.service
```

Contenido:

```ini
[Unit]
Description=AuthCore REST API Service
Documentation=https://github.com/tu-usuario/authcore
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=authcore
Group=authcore
WorkingDirectory=/opt/authcore

# Variables de entorno
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
Environment="SPRING_CONFIG_LOCATION=file:/etc/authcore/application.properties"

# Comando de ejecución
ExecStart=/usr/bin/java \
    -Xms512m \
    -Xmx2g \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/var/log/authcore/heapdump.hprof \
    -jar /opt/authcore/authcore-1.0.0.jar \
    --spring.config.location=/etc/authcore/application.properties

# Reinicio automático
Restart=always
RestartSec=10

# Logging
StandardOutput=append:/var/log/authcore/application.log
StandardError=append:/var/log/authcore/error.log

# Timeouts
TimeoutStartSec=60
TimeoutStopSec=30

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/log/authcore

[Install]
WantedBy=multi-user.target
```

### Habilitar e Iniciar Servicio

```bash
# Recargar systemd
sudo systemctl daemon-reload

# Habilitar inicio automático
sudo systemctl enable authcore

# Iniciar servicio
sudo systemctl start authcore

# Verificar estado
sudo systemctl status authcore
```

**Salida esperada**:
```
● authcore.service - AuthCore REST API Service
   Loaded: loaded (/etc/systemd/system/authcore.service; enabled)
   Active: active (running) since ...
   Main PID: 12345 (java)
   Tasks: 45 (limit: ...)
   Memory: 512.0M
   CGroup: /system.slice/authcore.service
           └─12345 java -Xms512m -Xmx2g ...
```

### Comandos Útiles de Systemd

```bash
# Ver logs en tiempo real
sudo journalctl -u authcore -f

# Ver logs de hoy
sudo journalctl -u authcore --since today

# Reiniciar servicio
sudo systemctl restart authcore

# Detener servicio
sudo systemctl stop authcore

# Ver uso de recursos
systemctl show authcore --property=MemoryCurrent,CPUUsageNSec
```

---

## 🌐 Configuración de Dominio y HTTPS

### Opción A: Usar Certbot con Let's Encrypt (Gratis)

#### 1. Comprar o Configurar Dominio

Puedes usar:
- **Dominio propio**: Comprado en Namecheap, GoDaddy, etc.
- **Subdominio gratuito**: duckdns.org, no-ip.com
- **Oracle Cloud DNS**: Gratis para primeros dominios

**Ejemplo con DuckDNS** (gratis):

1. Ve a https://www.duckdns.org/
2. Inicia sesión con GitHub/Google
3. Crea un subdominio: `authcore-tu-nombre.duckdns.org`
4. Instala el cliente de actualización en tu servidor:

```bash
# Crear script de actualización
vim ~/duckdns-update.sh
```

```bash
#!/bin/bash
TOKEN="tu-token-duckdns"
DOMAIN="authcore-tu-nombre.duckdns.org"
IP=$(curl -s https://api.ipify.org)

curl -s "https://www.duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip=$IP"
```

```bash
# Hacer ejecutable y agregar a cron
chmod +x ~/duckdns-update.sh
(crontab -l 2>/dev/null; echo "*/5 * * * * ~/duckdns-update.sh") | crontab -
```

#### 2. Configurar DNS Records

En tu proveedor de dominio, agrega:

| Tipo | Nombre | Valor | TTL |
|------|--------|-------|-----|
| A | @ | TU_IP_PUBLICA_ORACLE | 300 |
| A | www | TU_IP_PUBLICA_ORACLE | 300 |

#### 3. Instalar Certbot

```bash
# Para Oracle Linux
sudo dnf install -y certbot python3-certbot-nginx

# Para Ubuntu
sudo apt install -y certbot python3-certbot-nginx
```

#### 4. Obtener Certificado SSL

```bash
sudo certbot --nginx -d tudominio.com -d www.tudominio.com
```

**Proceso interactivo**:
1. Ingresa tu email para notificaciones
2. Acepta términos de servicio
3. Decide si compartir email con EFF
4. Certbot configurará Nginx automáticamente

#### 5. Configurar Auto-Renovación

Certbot instala un timer automático, pero verifica:

```bash
# Verificar timer
sudo systemctl list-timers | grep certbot

# Probar renovación (dry-run)
sudo certbot renew --dry-run
```

### Opción B: Nginx como Reverse Proxy

Crear configuración de Nginx:

```bash
sudo vim /etc/nginx/conf.d/authcore.conf
```

Contenido:

```nginx
# ===========================================
# CONFIGURACIÓN NGINX PARA AUTHCORE
# ===========================================

# Redireccionar HTTP a HTTPS
server {
    listen 80;
    server_name tudominio.com www.tudominio.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# Servidor HTTPS
server {
    listen 443 ssl http2;
    server_name tudominio.com www.tudominio.com;
    
    # Certificados SSL
    ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;
    
    # Configuración SSL segura
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Headers de seguridad
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Logs
    access_log /var/log/nginx/authcore-access.log;
    error_log /var/log/nginx/authcore-error.log;
    
    # Proxy a la aplicación Spring Boot
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        
        # Headers importantes
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # Timeout para WebSockets (si los usas)
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffering off;
        proxy_request_buffering off;
    }
    
    # Endpoint de health check (público)
    location /actuator/health {
        proxy_pass http://localhost:8080/actuator/health;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # No cachear health checks
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }
    
    # Bloquear acceso a otros actuator endpoints
    location /actuator/ {
        deny all;
        return 403;
    }
}
```

Reiniciar Nginx:

```bash
sudo nginx -t  # Verificar configuración
sudo systemctl restart nginx
```

---

## 📊 Monitoreo y Logs

### Monitoreo con Spring Boot Actuator

AuthCore ya incluye Actuator. Accede a:

```bash
# Health check
curl https://tudominio.com/actuator/health

# Información de la app
curl https://tudominio.com/actuator/info

# Métricas (requiere autenticación en producción)
curl https://tudominio.com/actuator/metrics
```

### Configurar Prometheus + Grafana (Opcional pero recomendado)

#### 1. Instalar Prometheus

```bash
# Crear usuario
sudo useradd -rs /bin/false prometheus

# Descargar Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
cd prometheus-*

# Mover archivos
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/

# Establecer permisos
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /usr/local/bin/prometheus
```

#### 2. Configurar Prometheus

```bash
sudo vim /etc/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'authcore'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
```

#### 3. Crear Servicio Systemd para Prometheus

```bash
sudo vim /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090

Restart=always

[Install]
WantedBy=multi-user.target
```

#### 4. Instalar Grafana

```bash
# Para Oracle Linux
sudo dnf install -y https://dl.grafana.com/oss/release/grafana-10.1.0-1.x86_64.rpm

# Para Ubuntu
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y grafana
```

Habilitar e iniciar:

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Accede a Grafana: `http://TU_IP:3000` (admin/admin)

#### 5. Importar Dashboard de Spring Boot

1. En Grafana, ve a **Dashboards** → **Import**
2. Usa el ID: `12900` (Spring Boot Statistics)
3. Selecciona tu datasource de Prometheus
4. ¡Listo! Tienes métricas visuales

### Visualización de Logs

#### 1. Usar Journalctl

```bash
# Ver logs de la aplicación
sudo journalctl -u authcore -f

# Ver logs de las últimas 2 horas
sudo journalctl -u authcore --since "2 hours ago"

# Filtrar por nivel de log
sudo journalctl -u authcore | grep ERROR
```

#### 2. Configurar Rotación de Logs

```bash
sudo vim /etc/logrotate.d/authcore
```

```conf
/var/log/authcore/*.log {
    daily
    rotate 10
    compress
    delaycompress
    notifempty
    create 0640 authcore authcore
    sharedscripts
    postrotate
        systemctl reload authcore > /dev/null 2>&1 || true
    endscript
}
```

#### 3. Enviar Logs a la Nube (Opcional)

Puedes configurar envío a:
- **AWS CloudWatch**
- **Google Cloud Logging**
- **Datadog**
- **ELK Stack (Elasticsearch, Logstash, Kibana)**

Ejemplo con Filebeat para ELK:

```bash
# Instalar Filebeat
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.9.0-x86_64.rpm
sudo rpm -vi filebeat-8.9.0-x86_64.rpm

# Configurar
sudo vim /etc/filebeat/filebeat.yml
```

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/authcore/*.log

output.elasticsearch:
  hosts: ["https://tu-elasticsearch:9200"]
  username: "elastic"
  password: "tu-password"
```

---

## 💾 Backups Automáticos

### Backup de PostgreSQL

#### 1. Script de Backup

```bash
sudo vim /opt/scripts/backup-authcore-db.sh
```

```bash
#!/bin/bash

# ===========================================
# SCRIPT DE BACKUP AUTOMÁTICO PARA AUTHCORE
# ===========================================

# Configuración
BACKUP_DIR="/var/backups/authcore"
DB_NAME="authcore_db"
DB_USER="authcore_user"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

# Crear directorio si no existe
mkdir -p "$BACKUP_DIR"

# Crear backup comprimido
echo "📦 Iniciando backup de $DB_NAME..."
PGPASSWORD="TuContraseñaMuySegura123!" pg_dump -U "$DB_USER" -h localhost "$DB_NAME" | gzip > "$BACKUP_FILE"

# Verificar éxito
if [ $? -eq 0 ]; then
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    echo "✅ Backup completado: $BACKUP_FILE ($BACKUP_SIZE)"
else
    echo "❌ Error en el backup"
    exit 1
fi

# Eliminar backups antiguos
echo "🗑️  Eliminando backups antiguos (> $RETENTION_DAYS días)..."
find "$BACKUP_DIR" -name "*.sql.gz" -type f -mtime +$RETENTION_DAYS -delete

# Listar backups actuales
echo "📋 Backups disponibles:"
ls -lh "$BACKUP_DIR"/*.sql.gz
```

Hacer ejecutable:

```bash
sudo chmod +x /opt/scripts/backup-authcore-db.sh
sudo chown -R postgres:postgres /var/backups/authcore
```

#### 2. Programar Backup con Cron

```bash
# Editar crontab
sudo crontab -e
```

Agregar línea para backup diario a las 3 AM:

```cron
0 3 * * * /opt/scripts/backup-authcore-db.sh >> /var/log/authcore-backup.log 2>&1
```

#### 3. Backup Remoto (Recomendado)

Subir backups a S3-compatible storage (Oracle Object Storage es gratis hasta 10 GB):

```bash
# Instalar AWS CLI (compatible con Oracle Object Storage)
sudo dnf install -y awscli

# Configurar credenciales
aws configure set default.region us-phoenix-1
aws configure set default.aws_access_key_id TU_ACCESS_KEY
aws configure set default.aws_secret_access_key TU_SECRET_KEY
aws configure set default.endpoint_url https://objectstorage.us-phoenix-1.oraclecloud.com
```

Modificar script de backup para subir a la nube:

```bash
# Agregar al final del script
aws s3 cp "$BACKUP_FILE" s3://tu-bucket-name/backups/
```

### Backup de la Aplicación

```bash
# Crear script de backup del JAR y configuración
sudo vim /opt/scripts/backup-authcore-app.sh
```

```bash
#!/bin/bash

APP_BACKUP_DIR="/var/backups/authcore-app"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$APP_BACKUP_DIR"

# Backup de configuración
tar -czf "$APP_BACKUP_DIR/config_$DATE.tar.gz" /etc/authcore/

# Backup del JAR actual
cp /opt/authcore/authcore-*.jar "$APP_BACKUP_DIR/"

# Mantener solo últimos 5 backups
ls -t "$APP_BACKUP_DIR"/*.tar.gz | tail -n +6 | xargs -r rm

echo "✅ Backup de aplicación completado"
```

---

## 🐛 Troubleshooting

### Problema 1: La aplicación no inicia

**Síntomas**:
- `systemctl status authcore` muestra "failed"
- Logs muestran errores

**Solución**:

```bash
# 1. Ver logs detallados
sudo journalctl -u authcore -n 100 --no-pager

# 2. Verificar que PostgreSQL está corriendo
sudo systemctl status postgresql

# 3. Probar conexión a la base de datos
psql -h localhost -U authcore_user -d authcore_db

# 4. Verificar que el JAR existe
ls -la /opt/authcore/*.jar

# 5. Verificar variables de entorno
cat /etc/authcore/application.properties

# 6. Intentar iniciar manualmente
sudo -u authcore java -jar /opt/authcore/authcore-1.0.0.jar
```

### Problema 2: Error de conexión a PostgreSQL

**Mensaje común**:
```
Connection refused to host: localhost; nested exception is java.net.ConnectException
```

**Solución**:

```bash
# 1. Verificar que PostgreSQL está activo
sudo systemctl status postgresql

# 2. Verificar puerto listening
sudo netstat -tlnp | grep 5432

# 3. Verificar pg_hba.conf
sudo cat /var/lib/pgsql/data/pg_hba.conf | grep -v "^#"

# 4. Verificar postgresql.conf
sudo cat /var/lib/pgsql/data/postgresql.conf | grep listen_addresses

# 5. Reiniciar PostgreSQL
sudo systemctl restart postgresql

# 6. Ver logs de PostgreSQL
sudo tail -f /var/log/pgsql/postgresql-*.log
```

### Problema 3: Memoria insuficiente (OutOfMemoryError)

**Síntomas**:
- La aplicación se cierra repentinamente
- Logs muestran `java.lang.OutOfMemoryError: Java heap space`

**Solución**:

```bash
# 1. Verificar uso de memoria
free -h
htop

# 2. Ajustar JVM options en el servicio systemd
sudo vim /etc/systemd/system/authcore.service
```

Modificar:
```ini
# Reducir memoria máxima si tienes poca RAM
-Xms256m
-Xmx1g

# O aumentar si tienes RAM disponible
-Xms1g
-Xmx4g
```

```bash
# 3. Recargar y reiniciar
sudo systemctl daemon-reload
sudo systemctl restart authcore

# 4. Monitorear uso de memoria
watch -n 5 'ps aux | grep java | grep -v grep'
```

### Problema 4: Certificados SSL expirados

**Síntomas**:
- Navegador muestra advertencia de seguridad
- curl falla con error SSL

**Solución**:

```bash
# 1. Verificar fecha de expiración
sudo certbot certificates

# 2. Renovar certificados
sudo certbot renew --force-renewal

# 3. Reiniciar Nginx
sudo systemctl restart nginx

# 4. Verificar
curl -I https://tudominio.com
```

### Problema 5: Altos tiempos de respuesta

**Síntomas**:
- Requests toman más de 2 segundos
- Usuarios reportan lentitud

**Solución**:

```bash
# 1. Verificar uso de CPU y memoria
htop

# 2. Verificar queries lentas en PostgreSQL
sudo -u postgres psql -d authcore_db -c "SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;"

# 3. Habilitar slow query log en PostgreSQL
sudo vim /var/lib/pgsql/data/postgresql.conf
```

```conf
log_min_duration_statement = 1000  # Log queries > 1 segundo
```

```bash
# 4. Verificar logs de la aplicación
sudo tail -f /var/log/authcore/application.log | grep "slow\|timeout"

# 5. Ajustar pool de conexiones Hikari
sudo vim /etc/authcore/application.properties
```

```properties
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.minimum-idle=10
```

### Problema 6: Firewall bloqueando conexiones

**Síntomas**:
- No puedes acceder desde internet
- Conexiones locales funcionan

**Solución**:

```bash
# 1. Verificar firewall de Oracle Cloud
# Ir a Oracle Cloud Console → Compute → Instances → Tu instancia → Subnet → Security Lists

# 2. Verificar firewall local (firewalld o ufw)

# Para firewalld (Oracle Linux)
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Para ufw (Ubuntu)
sudo ufw status
sudo ufw allow 8080/tcp
sudo ufw reload

# 3. Verificar SELinux (Oracle Linux)
getenforce
# Si es Enforcing, puede estar bloqueando
sudo setenforce 0  # Temporalmente para testing
```

---

## 🔐 Consideraciones de Seguridad

### 1. Hardening del Servidor

#### Deshabilitar login por contraseña SSH

```bash
sudo vim /etc/ssh/sshd_config
```

```conf
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

Reiniciar SSH:
```bash
sudo systemctl restart sshd
```

#### Configurar Fail2Ban

```bash
# Instalar
sudo dnf install -y fail2ban  # Oracle Linux
sudo apt install -y fail2ban  # Ubuntu

# Configurar
sudo vim /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/secure  # Oracle Linux
# logpath = /var/log/auth.log  # Ubuntu

[authcore]
enabled = true
port = 8080
filter = authcore
logpath = /var/log/authcore/application.log
maxretry = 10
bantime = 7200
```

Crear filtro para AuthCore:

```bash
sudo vim /etc/fail2ban/filter.d/authcore.conf
```

```ini
[Definition]
failregex = ^.*Failed authentication.*$
            ^.*Invalid token.*$
            ^.*Access denied.*$
ignoreregex =
```

Habilitar:
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo systemctl status fail2ban
```

### 2. Seguridad de la Base de Datos

```bash
# 1. Nunca exponer PostgreSQL a internet
# Editar postgresql.conf
listen_addresses = 'localhost'

# 2. Usar contraseñas fuertes
# Mínimo 20 caracteres, mezcla de mayúsculas, minúsculas, números y símbolos

# 3. Limitar privilegios de usuario
# El usuario de la app NO debe ser superusuario

# 4. Habilitar logging de auditoría
sudo vim /var/lib/pgsql/data/postgresql.conf
```

```conf
log_connections = on
log_disconnections = on
log_statement = 'ddl'
log_duration = on
```

### 3. Seguridad de la Aplicación

En `application.properties`:

```properties
# Tokens JWT con expiración corta
authcore.jwt.expiration-ms=3600000  # 1 hora
authcore.jwt.refresh-expiration-ms=604800000  # 7 días

# Habilitar HTTPS only
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=${SSL_KEYSTORE_PASSWORD}
server.ssl.keyStoreType=PKCS12
server.ssl.keyAlias=tudominio

# Headers de seguridad
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.same-site=strict
```

### 4. Actualizaciones de Seguridad

#### Actualizar sistema regularmente

```bash
# Crear script de actualización automática
sudo vim /opt/scripts/security-updates.sh
```

```bash
#!/bin/bash

# Actualizar paquetes de seguridad
dnf update --security -y  # Oracle Linux
# apt upgrade --only-upgrade -y  # Ubuntu

# Reiniciar servicios si hay actualizaciones de kernel
if [ $(rpm -q --last kernel | head -1) != $(rpm -q kernel) ]; then
    echo "🔄 Kernel actualizado. Se requiere reinicio."
    # Enviar notificación (email, Slack, etc.)
fi

echo "✅ Actualizaciones de seguridad completadas"
```

Programar semanalmente:

```bash
sudo crontab -e
```

```cron
0 4 * * 0 /opt/scripts/security-updates.sh >> /var/log/security-updates.log 2>&1
```

#### Monitorear vulnerabilidades en dependencias

```bash
# En tu máquina local, antes de desplegar
cd /ruta/a/authcore
mvn org.owasp:dependency-check-maven:check

# O usar Snyk
npm install -g snyk
snyk test
```

### 5. Backup de Llaves y Secretos

```bash
# Crear backup encriptado de configuraciones sensibles
BACKUP_DATE=$(date +%Y%m%d)

# Encriptar con GPG
tar czf - /etc/authcore/ | gpg -c -o /var/backups/authcore-config-$BACKUP_DATE.gpg

# Guardar passphrase en lugar seguro (NO en el servidor)
# Considerar usar HashiCorp Vault o AWS Secrets Manager para producción
```

---

## 📈 Escalamiento y Optimización

### 1. Ajustes de JVM para Producción

Editar servicio systemd:

```bash
sudo vim /etc/systemd/system/authcore.service
```

```ini
[Service]
# Optimizaciones de JVM
Environment="JAVA_OPTS=-Xms2g -Xmx4g \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:G1HeapRegionSize=16m \
    -XX:InitiatingHeapOccupancyPercent=45 \
    -XX:+ParallelRefProcEnabled \
    -XX:MaxTenuringThreshold=10 \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/var/log/authcore/heapdump.hprof \
    -XX:ErrorFile=/var/log/authcore/hs_err_pid%p.log \
    -XX:+UnlockDiagnosticVMOptions \
    -XX:+LogVMOutput \
    -XX:LogFile=/var/log/authcore/jvm.log"
```

### 2. Optimización de PostgreSQL

```bash
sudo vim /var/lib/pgsql/data/postgresql.conf
```

```conf
# Memoria
shared_buffers = 6GB              # 25% de RAM total
effective_cache_size = 18GB       # 75% de RAM total
work_mem = 64MB                   # Para operaciones complejas
maintenance_work_mem = 512MB      # Para VACUUM, CREATE INDEX

# WAL (Write-Ahead Logging)
wal_buffers = 64MB
checkpoint_completion_target = 0.9
wal_compression = on

# Conexiones
max_connections = 200
superuser_reserved_connections = 3

# Query planning
random_page_cost = 1.1            # Más bajo para SSD
effective_io_concurrency = 200    # Para SSD

# Autovacuum
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 60s
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
```

### 3. Implementar Caché con Redis

Instalar Redis:

```bash
sudo dnf install -y redis  # Oracle Linux
sudo apt install -y redis-server  # Ubuntu

sudo systemctl enable redis
sudo systemctl start redis
```

Configurar en `application.properties`:

```properties
# Redis cache
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379
spring.cache.redis.time-to-live=3600000  # 1 hora
spring.cache.redis.cache-null-values=false
```

---

## ✅ Checklist Final de Despliegue

Antes de considerar tu despliegue como "producción lista":

### Infraestructura
- [ ] VM creada con specs adecuadas (mínimo 2 CPU, 4GB RAM)
- [ ] Firewall configurado (solo puertos necesarios abiertos)
- [ ] SSH con llaves (contraseñas deshabilitadas)
- [ ] Usuario dedicado para la aplicación (no root)

### Base de Datos
- [ ] PostgreSQL instalado y asegurado
- [ ] Usuario dedicado con privilegios mínimos
- [ ] Backups automáticos configurados y probados
- [ ] Monitoreo de espacio en disco habilitado

### Aplicación
- [ ] Java 17+ instalado
- [ ] Variables de entorno configuradas correctamente
- [ ] JWT secret generado aleatoriamente
- [ ] Service systemd creado y habilitado
- [ ] Logs rotados y almacenados adecuadamente

### Seguridad
- [ ] HTTPS configurado con certificado válido
- [ ] Headers de seguridad implementados
- [ ] Rate limiting habilitado
- [ ] Fail2Ban configurado
- [ ] Dependencias escaneadas por vulnerabilidades

### Monitoreo
- [ ] Health checks accesibles
- [ ] Métricas expuestas (Prometheus)
- [ ] Alertas configuradas (email, Slack, etc.)
- [ ] Dashboard de Grafana importado

### Documentación
- [ ] URLs de acceso documentadas
- [ ] Credenciales guardadas en gestor de passwords
- [ ] Procedimiento de rollback documentado
- [ ] Contactos de emergencia identificados

### Pruebas
- [ ] Test de carga realizado (mínimo 100 req/s)
- [ ] Recovery de backup probado
- [ ] Failover testing (reiniciar servicios)
- [ ] Penetration testing básico realizado

---

## 📞 Soporte y Recursos Adicionales

### Documentación Oficial

- [Oracle Cloud Free Tier](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [Spring Boot Deployment](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html)
- [PostgreSQL Production Checklist](https://wiki.postgresql.org/wiki/Production_Checklist)
- [OWASP Security Guidelines](https://cheatsheetseries.owasp.org/)

### Comunidades

- [Stack Overflow - Spring Boot](https://stackoverflow.com/questions/tagged/spring-boot)
- [Reddit - r/devops](https://www.reddit.com/r/devops/)
- [Oracle Cloud Community](https://community.oracle.com/tech/apps-infra/categories/oracle_cloud_infrastructure)

### Herramientas Recomendadas

| Herramienta | Propósito | URL |
|-------------|-----------|-----|
| Uptime Kuma | Monitoreo de uptime | https://uptime.kuma.pet/ |
| Logstash | Agregación de logs | https://www.elastic.co/logstash |
| PgAdmin | GUI para PostgreSQL | https://www.pgadmin.org/ |
| JMeter | Testing de carga | https://jmeter.apache.org/ |
| Certbot | Certificados SSL gratuitos | https://certbot.eff.org/ |

---

## 🎉 ¡Felicidades!

Has completado el despliegue de AuthCore en Oracle Cloud Free Tier. Ahora tienes:

✅ Una API REST profesional funcionando 24/7  
✅ Base de datos PostgreSQL asegurada  
✅ HTTPS con certificado válido  
✅ Monitoreo y alertas configurados  
✅ Backups automáticos  
✅ Seguridad hardened  

### Próximos Pasos Sugeridos

1. **Configurar CI/CD** con GitHub Actions o GitLab CI para despliegues automáticos
2. **Implementar blue-green deployment** para cero downtime en actualizaciones
3. **Agregar más métricas** de negocio a tu dashboard de Grafana
4. **Configurar notificaciones** a Slack/Teams para eventos críticos
5. **Documentar procedimientos** de incidentes y escalación

---

**¿Problemas o preguntas?** Revisa la sección de [Troubleshooting](#-troubleshooting) o consulta los logs en `/var/log/authcore/`.

**Versión de esta guía**: 1.0  
**Última actualización**: 2024  
**Autor**: Equipo AuthCore

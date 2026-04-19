# AuthCore - Guía Completa de Arquitectura para Desarrolladores Junior

## 🎯 ¿Qué es AuthCore?

AuthCore es un **módulo de seguridad reutilizable** que gestiona autenticación (login) y autorización (permisos) para aplicaciones empresariales. Piensa en él como un "guardián" que:

1. **Verifica quién eres** (autenticación con username/password)
2. **Decide qué puedes hacer** (autorización con roles y permisos)
3. **Aísla datos por empresa** (multitenant: cada cliente ve solo su información)

## 🏗️ ¿Por Qué Arquitectura Hexagonal?

Imagina tu aplicación como una **cebolla con capas**:

```
        ┌─────────────────────────────────────┐
        │   INFRASTRUCTURE (capa externa)     │
        │   - Controladores HTTP              │
        │   - Base de datos (JPA)             │
        │   - Redis, Kafka, Email             │
        └─────────────────────────────────────┘
                        ↓
        ┌─────────────────────────────────────┐
        │   APPLICATION (capa intermedia)     │
        │   - Casos de uso                    │
        │   - Orquestación del flujo          │
        └─────────────────────────────────────┘
                        ↓
        ┌─────────────────────────────────────┐
        │   DOMAIN (núcleo, lo más importante)│
        │   - Reglas de negocio               │
        │   - Entidades (User, Role, etc.)    │
        │   - ¡NO depende de NADA externo!    │
        └─────────────────────────────────────┘
```

### Analogía del Restaurante 🍕

- **Domain** = Las recetas secretas de la pizza (lo más valioso, no cambia)
- **Application** = El proceso de pedir: tomar orden → preparar → entregar
- **Infrastructure** = Los meseros, horno, sistema de pago (pueden cambiar)

**Ventaja clave**: Puedes cambiar el horno (infraestructura) sin modificar la receta (dominio).

## 📂 Estructura de Carpetas Explicada

```
authcore/
│
├── src/main/java/com/authcore/
│   │
│   ├── domain/                    ← EL CORAZÓN DEL SISTEMA
│   │   ├── model/                 ← Entidades: User.java, Role.java
│   │   ├── repository/            ← Interfaces: UserRepository.java
│   │   ├── service/               ← Lógica compleja: PasswordService.java
│   │   └── exception/             ← Excepciones: UserNotFoundException.java
│   │
│   ├── application/               ← ORQUESTACIÓN
│   │   ├── port/in/               ← Contratos: AuthenticateUserPort.java
│   │   ├── port/out/              ← Necesidades: JwtTokenProviderPort.java
│   │   ├── usecase/               ← Implementación: AuthenticateUserUseCase.java
│   │   └── dto/                   ← Datos: LoginRequest.java, UserDto.java
│   │
│   └── infrastructure/            ← CONEXIÓN CON EL MUNDO EXTERIOR
│       ├── adapter/
│       │   ├── in/rest/           ← Controladores HTTP
│       │   ├── in/security/       ← Filtros JWT, Spring Security
│       │   └── out/persistence/   ← JPA, entidades SQL
│       └── config/                ← Configuraciones Spring
│
└── src/test/java/com/authcore/    ← TESTS AUTOMATIZADOS
    ├── domain/                    ← Tests de entidades
    ├── application/               ← Tests de casos de uso
    └── infrastructure/            ← Tests de integración
```

## 🔄 Flujo de una Petición HTTP (Paso a Paso)

Cuando un usuario hace login (`POST /api/auth/login`):

```
1. HTTP Request llega
   ↓
2. JwtAuthenticationFilter (infrastructure)
   - Verifica si el token existe (si es requerido)
   ↓
3. AuthController (infrastructure)
   - Recibe JSON: {"username": "juan", "password": "123", "tenantId": "empresa1"}
   - Lo convierte a LoginRequest DTO
   ↓
4. AuthenticateUserUseCase (application)
   - Busca usuario vía UserRepositoryPort
   - Valida password vía PasswordHasherPort
   - Genera token vía JwtTokenProviderPort
   ↓
5. User entity.method() (domain)
   - Ejecuta lógica: user.isValid(), user.hasRole()
   ↓
6. JpaUserRepository (infrastructure)
   - Ejecuta SQL: SELECT * FROM users WHERE username = 'juan'
   ↓
7. Response JSON retorna
   {
     "accessToken": "eyJhbGc...",
     "refreshToken": "dGhpcyBp...",
     "user": {"id": 1, "email": "juan@email.com"}
   }
```

## 🚫 Regla de Oro: Dependencias

```
✅ CORRECTO: Infraestructura → Application → Domain
❌ INCORRECTO: Domain → Application o Domain → Infrastructure
```

**¿Por qué?** El dominio (reglas de negocio) debe ser independiente. Si mañana cambias de base de datos o framework web, el dominio no debería modificarse.

## 📋 Convenciones de Nombres (Para No Perderse)

| Tipo | Patrón | Ejemplo |
|------|--------|---------|
| Entidad | Sustantivo puro | `User`, `Role`, `Permission` |
| Objeto de Valor | Sustantivo inmutable | `Email`, `Password`, `Username` |
| Repositorio | Sustantivo + Repository | `UserRepository` |
| Caso de Uso | Verbo + Sustantivo + UseCase | `AuthenticateUserUseCase` |
| Puerto | Verbo/Sustantivo + Port | `AuthenticateUserPort` |
| DTO | Sustantivo + Dto / Request / Response | `UserDto`, `LoginRequest` |
| Controlador | Sustantivo + Controller | `AuthController` |
| Excepción | Causa + Exception | `UserNotFoundException` |

## 🛠️ Tecnologías Usadas

| Tecnología | Propósito | ¿Dónde se usa? |
|------------|-----------|----------------|
| Java 17+ | Lenguaje base | Todo el proyecto |
| Spring Boot 3.x | Framework principal | Infrastructure |
| Spring Security | Autenticación/autorización | Infrastructure |
| Spring Data JPA | Acceso a base de datos | Infrastructure |
| jjwt (Java JWT) | Generar/validar tokens | Infrastructure |
| PostgreSQL | Base de datos relacional | Infrastructure |
| Redis | Caché de tokens revocados | Infrastructure |
| Lombok | Reducir boilerplate | Todo el proyecto |
| MapStruct | Mapeo entre objetos | Infrastructure |

## 🎓 Conceptos Clave Explicados

### 1. ¿Qué es un Puerto (Port)?

Un **puerto es una interfaz** que define un contrato:

```java
// Puerto de entrada (lo que ofrecemos)
public interface AuthenticateUserPort {
    AuthResponse authenticate(LoginRequest request);
}

// Puerto de salida (lo que necesitamos)
public interface UserRepositoryPort {
    Optional<User> findByUsername(String username);
    User save(User user);
}
```

**Analogía**: Un puerto USB define cómo conectar dispositivos sin saber qué dispositivo es.

### 2. ¿Qué es un Adapter?

Un **adapter es la implementación concreta** de un puerto:

```java
// Adapter que implementa el puerto
@Component
public class AuthenticateUserUseCase implements AuthenticateUserPort {
    
    private final UserRepositoryPort userRepository;
    private final PasswordHasherPort passwordHasher;
    
    @Override
    public AuthResponse authenticate(LoginRequest request) {
        // Implementación real
    }
}
```

**Analogía**: El cable USB-C a USB-A es un adapter que convierte un estándar en otro.

### 3. ¿Qué es un DTO (Data Transfer Object)?

Un **DTO es un objeto que solo lleva datos**, sin lógica:

```java
// Record de Java 17+ (inmutable automáticamente)
public record LoginRequest(String username, String password, String tenantId) {}

public record UserDto(Long id, String email, String username, List<String> roles) {}
```

**¿Por qué no usar entidades directamente?** 
- Las entidades tienen lógica de negocio
- Los DTOs controlan qué datos expones (ej: nunca enviar password hash)
- Permiten versionar la API sin romper el dominio

### 4. ¿Qué es Multitenancy?

**Multitenancy** = Una sola aplicación sirve a múltiples clientes (tenants), aislando sus datos.

**Ejemplo del mundo real**: 
- Salesforce tiene una sola aplicación
- Pero Coca-Cola solo ve sus datos, Pepsi solo ve los suyos
- Ambos usan el mismo código, pero databases separadas

**En AuthCore**:
```java
// Cada query incluye el tenant
userRepository.findByUsername(username, tenantId);

// El tenant se identifica por:
// 1. Header HTTP: X-Tenant-ID: empresa1
// 2. Subdominio: empresa1.authcore.com
// 3. Campo en la DB: tenant_id en cada tabla
```

## 📊 Matriz de Responsabilidades

| Pregunta | ¿Dónde va? | Ejemplo |
|----------|-----------|---------|
| ¿Validar formato de email? | Domain (objeto de valor) | `Email.java` valida regex |
| ¿Verificar si email existe? | Application (use case) | `CreateUserUseCase` llama al repo |
| ¿Hacer SELECT en MySQL? | Infrastructure (repository) | `JpaUserRepository` con JPA |
| ¿Retornar JSON en HTTP? | Infrastructure (controller) | `AuthController` con @RestController |
| ¿Hashear password? | Infrastructure (adapter) | `BcryptPasswordHasher` |
| ¿Decidir longitud mínima de password? | Domain (servicio) | `PasswordDomainService` |

## 🔐 Seguridad OWASP TOP 10 Integrada

Este módulo protege contra:

1. **Inyección SQL** → Usamos JPA con parámetros bound (no concatenación)
2. **Autenticación rota** → JWT con expiración, refresh tokens, bcrypt
3. **Exposición de datos** → DTOs filtran campos sensibles
4. **XXE/XML** → Solo aceptamos JSON
5. **Control de acceso** → RBAC con roles y permisos granulares
6. **Configuración insegura** → Secrets en variables de entorno
7. **XSS** → Sanitización de inputs, headers de seguridad
8. **Deserialización insegura** → Validación estricta de DTOs
9. **Componentes vulnerables** → OWASP Dependency Check en CI/CD
10. **Logging insuficiente** → AuditLogger para todos los eventos críticos

## 🚀 Próximos Pasos (Roadmap Mental)

1. **Primero**: Entiende el dominio (entidades y reglas)
2. **Luego**: Comprende los puertos (qué ofrece/necesita la app)
3. **Después**: Revisa los casos de uso (cómo fluyen los datos)
4. **Finalmente**: Explora la infraestructura (cómo se conecta al mundo)

## 📚 Recursos Adicionales

- [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html) - Martin Fowler
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) - Alistair Cockburn
- [OWASP TOP 10](https://owasp.org/www-project-top-ten/) - Estándar de seguridad

---

**Nota**: Cada subcarpeta tiene su propio README.md con explicaciones detalladas. ¡Úsalos como guía de referencia!

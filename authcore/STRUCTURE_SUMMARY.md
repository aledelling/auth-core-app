# AuthCore - Resumen de Estructura Hexagonal

## 📁 Árbol Completo del Proyecto

```
authcore/
│
├── README.md                              # Documentación principal del proyecto
│
├── src/main/java/com/authcore/
│   │
│   ├── domain/                            # CAPA DE DOMINIO (Enterprise Business Rules)
│   │   ├── README.md                      # Por qué y cómo funciona esta capa
│   │   ├── model/                         # Entidades y objetos de valor
│   │   │   └── README.md                  # User, Role, Permission, Tenant, etc.
│   │   ├── repository/                    # Interfaces de repositorio (puertos)
│   │   │   └── README.md                  # Contratos de persistencia
│   │   ├── service/                       # Servicios de dominio
│   │   │   └── README.md                  # Lógica compleja multi-entidad
│   │   └── exception/                     # Excepciones del dominio
│   │       └── README.md                  # UserNotFoundException, InvalidCredentialsException, etc.
│   │
│   ├── application/                       # CAPA DE APLICACIÓN (Application Business Rules)
│   │   ├── README.md                      # Por qué y cómo funciona esta capa
│   │   ├── port/
│   │   │   ├── in/                        # Puertos de entrada (lo que ofrece application)
│   │   │   │   └── README.md              # AuthenticateUserPort, CreateUserPort, etc.
│   │   │   └── out/                       # Puertos de salida (lo que necesita application)
│   │   │       └── README.md              # UserRepositoryPort, JwtTokenProviderPort, etc.
│   │   ├── usecase/                       # Casos de uso concretos
│   │   │   └── README.md                  # AuthenticateUserUseCase, CreateUserUseCase, etc.
│   │   └── dto/                           # Data Transfer Objects
│   │       └── README.md                  # LoginRequest, UserDto, AuthResponse, etc.
│   │
│   └── infrastructure/                    # CAPA DE INFRAESTRUCTURA (Frameworks & Drivers)
│       ├── README.md                      # Por qué y cómo funciona esta capa
│       ├── adapter/
│       │   ├── in/                        # Adapters de entrada (driven by external actors)
│       │   │   ├── rest/                  # Controllers REST, Exception Handlers
│       │   │   └── security/              # Filtros JWT, Security Config
│       │   └── out/                       # Adapters de salida (driving external systems)
│       │       ├── persistence/           # Repositorios JPA, Entidades SQL
│       │       ├── cache/                 # Redis, Token Blacklist
│       │       └── message/               # Kafka, Email Sender, Audit Logger
│       └── config/                        # Configuraciones Spring
│                                          # SecurityConfig, DataSourceConfig, MultitenantConfig
│
├── src/main/resources/
│   ├── application.yml                    # Configuración principal
│   ├── application-dev.yml                # Perfil desarrollo
│   ├── application-prod.yml               # Perfil producción
│   ├── db/migration/                      # Scripts Flyway/Liquibase
│   └── logback-spring.xml                 # Configuración de logging
│
└── src/test/java/com/authcore/
    ├── README.md                          # Estrategia y estructura de testing
    ├── domain/                            # Tests unitarios de dominio
    ├── application/                       # Tests unitarios de aplicación
    ├── infrastructure/                    # Tests de integración
    └── e2e/                               # Tests end-to-end
```

## 📊 Distribución de READMEs por Capa

| Capa | READMEs | Propósito |
|------|---------|-----------|
| **Root** | 1 | Visión general de arquitectura hexagonal |
| **Domain** | 5 | Model, Repository, Service, Exception + overview |
| **Application** | 5 | Port In/Out, UseCase, DTO + overview |
| **Infrastructure** | 1 | Overview de adapters y configuración |
| **Test** | 1 | Estrategia de testing por capa |
| **TOTAL** | **13 READMEs** | Documentación completa de la estructura |

## 🔄 Flujo de Dependencias

```
┌─────────────────────────────────────────────────────────────┐
│                    INFRAESTRUCTURA                          │
│  (Controllers, JPA, Security, Redis, Kafka, Config)         │
│           ↓ depende de                                      │
├─────────────────────────────────────────────────────────────┤
│                    APLICACIÓN                                │
│  (Use Cases, DTOs, Puertos In/Out)                          │
│           ↓ depende de                                      │
├─────────────────────────────────────────────────────────────┤
│                    DOMINIO                                   │
│  (Entidades, Reglas de Negocio, Interfaces Repository)      │
│           ↓ NO depende de nada externo                      │
└─────────────────────────────────────────────────────────────┘
```

## 🎯 Regla de Oro

**Las dependencias siempre apuntan hacia adentro:**
- Infrastructure → Application → Domain
- Domain ← NO importa nada externo

## 📦 Archivos Clave a Crear (Próximos Pasos)

### Build & Configuración
- [ ] `pom.xml` o `build.gradle` - Dependencias del proyecto
- [ ] `src/main/resources/application.yml` - Configuración base
- [ ] `src/main/resources/application-dev.yml` - Perfil desarrollo
- [ ] `src/main/resources/application-prod.yml` - Perfil producción

### Dominio (model/)
- [ ] `User.java`, `Role.java`, `Permission.java`, `Tenant.java`
- [ ] `Email.java`, `Password.java`, `Username.java` (Value Objects)

### Dominio (repository/)
- [ ] `UserRepository.java`, `RoleRepository.java`, `PermissionRepository.java`

### Dominio (exception/)
- [ ] `UserNotFoundException.java`, `InvalidCredentialsException.java`, etc.

### Application (port/in/)
- [ ] `AuthenticateUserPort.java`, `CreateUserPort.java`, etc.

### Application (port/out/)
- [ ] `UserRepositoryPort.java`, `JwtTokenProviderPort.java`, etc.

### Application (usecase/)
- [ ] `AuthenticateUserUseCase.java`, `CreateUserUseCase.java`, etc.

### Application (dto/)
- [ ] `LoginRequest.java`, `AuthResponse.java`, `UserDto.java`, etc.

### Infrastructure (adapter/in/rest/)
- [ ] `AuthController.java`, `UserController.java`, `GlobalExceptionHandler.java`

### Infrastructure (adapter/in/security/)
- [ ] `JwtAuthenticationFilter.java`, `SecurityConfig.java`

### Infrastructure (adapter/out/persistence/)
- [ ] `JpaUserRepository.java`, `UserEntity.java`, `UserMapper.java`

### Infrastructure (config/)
- [ ] `ApplicationConfig.java`, `MultitenantConfig.java`

---

**Nota**: Esta estructura está diseñada para ser reutilizable en cualquier proyecto futuro. Solo cambia el package name y adapta las entidades según necesidades.

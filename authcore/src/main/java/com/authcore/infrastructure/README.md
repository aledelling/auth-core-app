# Infrastructure Layer - Capa de Infraestructura

## Propósito

La capa **Infrastructure** contiene todas las implementaciones concretas que conectan la aplicación con el mundo exterior: bases de datos, APIs HTTP, mensajería, seguridad, etc.

## Características Clave

✅ **Depende de todo**: Conoce Domain, Application y frameworks externos  
✅ **Implementa puertos**: Concreta las interfaces definidas en Application  
✅ **Configurable**: Todo lo externalizable vive aquí (DB, Redis, JWT secret)  
✅ **Reemplazable**: Puedes cambiar JPA por JDBC, Spring Security por otro, sin tocar el core  

## Estructura Interna

```
infrastructure/
├── adapter/
│   ├── in/                  # Adapters de entrada (driven by external actors)
│   │   ├── rest/            # Controllers REST, mappers, handlers
│   │   └── security/        # Filtros de seguridad, autenticación
│   └── out/                 # Adapters de salida (driving external systems)
│       ├── persistence/     # Repositorios JPA, entidades SQL
│       ├── cache/           # Redis, memorias caché
│       └── message/         # Kafka, RabbitMQ, eventos
└── config/                  # Configuraciones Spring, beans, security
```

### 📁 `adapter/in/rest/` - Adaptadores REST de Entrada

**Qué contiene:**
- `AuthController.java` - Endpoints `/api/auth/login`, `/api/auth/refresh`
- `UserController.java` - CRUD de usuarios
- `RoleController.java` - Gestión de roles y permisos
- `TenantController.java` - Administración de tenants
- `GlobalExceptionHandler.java` - Manejo centralizado de errores
- `Mapper.java` - Conversión DTO ↔ Request/Response de API

**Por qué así:**
- Los controllers SON adapters: traducen HTTP → Use Cases
- Delgados: solo validan input básico y llaman al use case
- No tienen lógica de negocio
- Usan anotaciones Spring MVC (@RestController, @PostMapping)

**Ejemplo:**
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    private final AuthenticateUserPort authenticateUser;
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        AuthResponse response = authenticateUser.authenticate(request);
        return ResponseEntity.ok(response);
    }
}
```

### 📁 `adapter/in/security/` - Seguridad y Autenticación

**Qué contiene:**
- `JwtAuthenticationFilter.java` - Filtro que valida JWT en cada request
- `SecurityConfig.java` - Configuración de Spring Security
- `CustomUserDetailsService.java` - Adapta User del dominio a UserDetails de Spring
- `AccessDeniedHandlerImpl.java` - Manejo de errores 403
- `AuthenticationEntryPointImpl.java` - Manejo de errores 401

**Por qué así:**
- El filtro es un adapter que convierte HTTP → ValidateTokenPort
- Spring Security se configura aquí, no en application
- Los handlers traducen excepciones de dominio a HTTP status

### 📁 `adapter/out/persistence/` - Persistencia de Datos

**Qué contiene:**
- `JpaUserRepository.java` - Implementa UserRepositoryPort usando Spring Data JPA
- `JpaRoleRepository.java` - Implementa RoleRepositoryPort
- `UserEntity.java` - Entidad JPA (@Entity) que mapea a tabla users
- `RoleEntity.java` - Entidad JPA para roles
- `TenantEntity.java` - Entidad JPA para tenants
- `UserMapper.java` - Convierte Entity ↔ Domain Model

**Por qué así:**
- Las entidades JPA SON diferentes a las del dominio (pueden tener @OneToMany, etc.)
- Los repositorios implementan los puertos de salida de application
- El mapper evita que las entidades JPA contaminen el dominio

**Ejemplo:**
```java
@Repository
public class JpaUserRepository implements UserRepositoryPort {
    private final SpringDataUserRepository jpaRepository;
    private final UserMapper mapper;
    
    @Override
    public User findByUsername(String username) {
        return jpaRepository.findByUsername(username)
            .map(mapper::toDomain)
            .orElseThrow(() -> new UserNotFoundException(username));
    }
}
```

### 📁 `adapter/out/cache/` - Caché

**Qué contiene:**
- `RedisTokenRepository.java` - Almacena tokens revocados en Redis
- `CacheConfig.java` - Configuración de Redis/Spring Cache
- `TokenBlacklistAdapter.java` - Implementa TokenBlacklistPort

**Por qué así:**
- Los tokens revocados van a cache (rápido), no a DB
- Permite invalidar sesiones globalmente
- configurable: puedes cambiar Redis por Memcached o memoria local

### 📁 `adapter/out/message/` - Mensajería y Eventos

**Qué contiene:**
- `KafkaEventPublisher.java` - Publica eventos de auditoría a Kafka
- `AuditEventAdapter.java` - Implementa AuditLoggerPort
- `EmailSenderAdapter.java` - Envía emails vía SMTP o servicio externo

**Cuándo usar:**
- Para operaciones asíncronas (enviar email post-registro)
- Auditoría y compliance (loggear todos los logins)
- Integración con otros sistemas (webhooks)

### 📁 `config/` - Configuraciones Spring

**Qué contiene:**
- `ApplicationConfig.java` - Beans globales, ObjectMapper, etc.
- `SecurityConfig.java` - Chain de seguridad de Spring Security
- `CorsConfig.java` - Configuración CORS para frontend
- `SwaggerConfig.java` - Documentación OpenAPI
- `MultitenantConfig.java` - Resolución de tenant por header/subdomain
- `DataSourceConfig.java` - Configuración multi-tenant de DataSource

**Por qué así:**
- Toda la configuración externa vive aquí
- Permite profiles (dev, test, prod)
- Los beans se definen aquí y se inyectan en adapters

## Reglas de Oro

1. **PUEDES** importar de Domain y Application
2. **SIEMPRE** implementa los puertos definidos en Application
3. **NUNCA** pongas lógica de negocio en adapters
4. **SIEMPRE** usa mappers para convertir Entity ↔ Domain
5. **MANTÉN** los controllers delgados (solo traducción HTTP → UseCase)

## Flujo Completo Ejemplo

```
HTTP Request → JwtAuthenticationFilter (Infra)
                    ↓
             AuthController (Infra)
                    ↓
        AuthenticateUserUseCase (Application)
                    ↓
              User Entity.method() (Domain)
                    ↓
         JpaUserRepository (Infra) → PostgreSQL
                    ↓
        JwtTokenProvider (Infra) → Genera token
                    ↓
             AuthResponse (Application DTO)
                    ↓
          HTTP Response JSON (Infra)
```

## Testing

- **Adapters**: Pruebas de integración con @SpringBootTest
- **Config**: Pruebas de contexto con @ContextConfiguration
- **Mocks externos**: WireMock para APIs externas, Testcontainers para DB/Redis
- Cobertura mínima: 75% (infra es más volátil y fácil de cambiar)

## Reusabilidad

Para llevar esta estructura a otro proyecto:

1. Copia toda la carpeta `infrastructure/`
2. Cambia configuraciones en `config/` según el nuevo proyecto
3. Reemplaza adapters de persistencia si cambias de DB (JPA → JDBC)
4. Mantén los mismos puertos de entrada/salida para compatibilidad

---

**Capas relacionadas:**
- [Application Layer](../application/README.md) - Lo que infrastructure implementa
- [Domain Layer](../domain/README.md) - Lo que infrastructure persiste/protege

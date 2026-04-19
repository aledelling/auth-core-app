# Application Port Out - Puertos de Salida

## Propósito

Define las **interfaces que application necesita** para comunicarse con sistemas externos (DB, cache, APIs, messaging).

## Características Clave

✅ Son interfaces que application **requiere**  
✅ Infrastructure las **implementa** como adapters  
✅ Abstractan tecnología externa (JPA, Redis, Kafka, etc.)  
✅ Permiten cambiar implementaciones sin tocar el dominio  

## Contenido Esperado

| Archivo | Descripción | Implementación Típica |
|---------|-------------|----------------------|
| `UserRepositoryPort.java` | Persistencia de usuarios | JPA, JDBC, MongoDB |
| `RoleRepositoryPort.java` | Persistencia de roles | JPA, JDBC |
| `JwtTokenProviderPort.java` | Generación/validación de JWT | jjwt, Nimbus JWT |
| `PasswordHasherPort.java` | Hashing de contraseñas | BCrypt, Argon2 |
| `CachePort.java` | Caché distribuido | Redis, Memcached |
| `EmailSenderPort.java` | Envío de emails | SMTP, SendGrid, SES |
| `AuditLoggerPort.java` | Logging de auditoría | Kafka, ELK, DB |
| `ObjectStoragePort.java` | Almacenamiento de archivos | S3, Azure Blob |

## Ejemplo

```java
// En application/port/out/UserRepositoryPort.java
package com.authcore.application.port.out;

import com.authcore.domain.model.User;
import com.authcore.domain.model.Username;
import java.util.Optional;
import java.util.List;

public interface UserRepositoryPort {
    
    Optional<User> findByUsername(Username username);
    
    Optional<User> findByEmail(String email);
    
    User save(User user);
    
    void deleteById(Long id);
    
    boolean existsByEmail(String email);
    
    List<User> findAllByTenantId(Long tenantId);
}
```

## Relación con Domain Repository

### ¿Por qué dos tipos de repositorios?

```
┌─────────────────────────────────────┐
│  domain/repository/                 │
│  interface UserRepository           │  ← Lo define el dominio
└─────────────────────────────────────┘
              ↑ extiende/usar
┌─────────────────────────────────────┐
│  application/port/out/              │
│  interface UserRepositoryPort       │  ← Lo especializa application
└─────────────────────────────────────┘
              ↑ implementa
┌─────────────────────────────────────┐
│  infrastructure/adapter/out/        │
│  class JpaUserRepository            │  ← Lo implementa infra
└─────────────────────────────────────┘
```

### Diferencias Clave

| Domain Repository | Application Port Out |
|-------------------|---------------------|
| Lo define el dominio | Lo usa application |
| Métodos genéricos | Métodos específicos del caso de uso |
| Sin DTOs | Puede usar DTOs si conviene |
| Estable | Puede evolucionar con nuevos casos de uso |

**En la práctica**: Puedes usar el mismo interfaz o crear uno específico en Port Out que extienda del domain.

## Por Qué Esta Ubicación

### 1. Dependency Rule

Application puede depender de abstracciones, no de implementaciones:

```java
// UseCase depende del puerto (abstracción)
public class CreateUserUseCase implements CreateUserPort {
    
    private final UserRepositoryPort userRepository;  // ✅ Bien
    
    // private final JpaUserRepository jpaRepository;  // ❌ Mal: depende de implementación
}
```

### 2. Testabilidad

Puedes mockear puertos de salida en tests unitarios:

```java
@Mock
private UserRepositoryPort userRepository;

@Mock
private PasswordHasherPort passwordHasher;

@InjectMocks
private CreateUserUseCase useCase;

@Test
void should_create_user_when_data_valid() {
    when(userRepository.existsByEmail(any())).thenReturn(false);
    when(passwordHasher.hash(any())).thenReturn("hashed");
    when(userRepository.save(any())).thenAnswer(i -> i.getArguments()[0]);
    
    UserDto result = useCase.createUser(request);
    
    assertThat(result.id()).isNotNull();
}
```

### 3. Cambios de Tecnología

Cambiar de JPA a JDBC solo afecta `infrastructure/`, application no cambia:

```java
// Hoy: JPA
@Repository
public class JpaUserRepository implements UserRepositoryPort {
    @Autowired private SpringDataUserRepository jpaRepository;
    // implementación con JPA
}

// Mañana: JDBC
@Repository
public class JdbcUserRepository implements UserRepositoryPort {
    @Autowired private JdbcTemplate jdbcTemplate;
    // implementación con JDBC, misma interfaz
}
```

## Ejemplo Completo de Flujo

```java
// 1. UseCase usa puertos de salida
public class AuthenticateUserUseCase implements AuthenticateUserPort {
    
    private final UserRepositoryPort userRepository;
    private final PasswordHasherPort passwordHasher;
    private final JwtTokenProviderPort tokenProvider;
    
    @Override
    public AuthResponse authenticate(LoginRequest request) {
        // Puerto de salida 1: buscar usuario
        User user = userRepository.findByUsername(request.username())
            .orElseThrow(() -> new UserNotFoundException(request.username()));
        
        // Puerto de salida 2: validar password
        if (!passwordHasher.matches(request.password(), user.password())) {
            throw new InvalidCredentialsException();
        }
        
        // Puerto de salida 3: generar token
        String token = tokenProvider.generate(user);
        
        return new AuthResponse(token, user.toDto());
    }
}

// 2. Infrastructure implementa los puertos
@Component
public class BcryptPasswordHasher implements PasswordHasherPort {
    @Override
    public String hash(String rawPassword) {
        return BCrypt.hashpw(rawPassword, BCrypt.gensalt());
    }
    
    @Override
    public boolean matches(String rawPassword, String hashedPassword) {
        return BCrypt.checkpw(rawPassword, hashedPassword);
    }
}
```

## Convenciones de Nombres

| Patrón | Ejemplo |
|--------|---------|
| Sustantivo + `RepositoryPort` | `UserRepositoryPort` |
| Sustantivo + `ProviderPort` | `JwtTokenProviderPort` |
| Sustantivo + `Port` | `EmailSenderPort`, `CachePort` |
| Verbo + Sustantivo + `Port` | `AuditLoggerPort` |

## Testing de Implementaciones

```java
// Test de integración para verificar que la implementación funciona
@SpringBootTest
class JpaUserRepositoryIntegrationTest {
    
    @Autowired
    private UserRepositoryPort userRepository;  // Spring inyecta JpaUserRepository
    
    @Testcontainers
    @Container
    static PostgreSQLContainer<?> postgres = 
        PostgreSQLContainer.forVersion("15-alpine");
    
    @Test
    void should_save_and_find_user() {
        // Given
        User user = User.create("test@email.com", "password123");
        
        // When
        User saved = userRepository.save(user);
        User found = userRepository.findByUsername(saved.username()).orElseThrow();
        
        // Then
        assertThat(found.email()).isEqualTo(user.email());
    }
}
```

---

**Volver a**: [Application Layer](../README.md)  
**Siguiente**: [Infrastructure Adapters Out](../../infrastructure/adapter/out/README.md)

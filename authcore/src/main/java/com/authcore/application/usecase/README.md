# Application UseCase - Casos de Uso

## Propósito

Contiene la **implementación concreta** de los casos de uso que orquestan el flujo de la aplicación.

## Características Clave

✅ Implementan puertos de entrada (`port/in/`)  
✅ Usan puertos de salida (`port/out/`)  
✅ Tienen `@Transactional` (gestión de transacciones)  
✅ Orquestan, NO contienen lógica de negocio  

## Contenido Esperado

| Archivo | Implementa | Descripción |
|---------|-----------|-------------|
| `AuthenticateUserUseCase.java` | `AuthenticateUserPort` | Autenticar usuario y generar tokens |
| `CreateUserUseCase.java` | `CreateUserPort` | Crear nuevo usuario |
| `UpdateUserUseCase.java` | `UpdateUserPort` | Actualizar datos de usuario |
| `DeleteUserUseCase.java` | `DeleteUserPort` | Eliminar usuario (soft delete) |
| `AssignRoleToUserUseCase.java` | `AssignRoleToUserPort` | Asignar rol a usuario |
| `RefreshTokenUseCase.java` | `RefreshTokenPort` | Generar nuevo access token |
| `RevokeTokenUseCase.java` | `RevokeTokenPort` | Revocar token activo |
| `CreateTenantUseCase.java` | `CreateTenantPort` | Crear tenant con datos iniciales |

## Estructura Típica

```java
// En application/usecase/AuthenticateUserUseCase.java
package com.authcore.application.usecase;

import com.authcore.application.port.in.AuthenticateUserPort;
import com.authcore.application.port.out.UserRepositoryPort;
import com.authcore.application.port.out.PasswordHasherPort;
import com.authcore.application.port.out.JwtTokenProviderPort;
import com.authcore.application.dto.AuthResponse;
import com.authcore.application.dto.LoginRequest;
import com.authcore.domain.exception.InvalidCredentialsException;
import com.authcore.domain.exception.UserNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
@RequiredArgsConstructor
public class AuthenticateUserUseCase implements AuthenticateUserPort {
    
    private final UserRepositoryPort userRepository;
    private final PasswordHasherPort passwordHasher;
    private final JwtTokenProviderPort tokenProvider;
    
    @Override
    @Transactional(readOnly = true)
    public AuthResponse authenticate(LoginRequest request) {
        // 1. Buscar usuario (puerto de salida)
        User user = userRepository.findByUsername(request.username())
            .orElseThrow(() -> new UserNotFoundException(request.username()));
        
        // 2. Validar credenciales (servicio externo vía puerto)
        if (!passwordHasher.matches(request.password(), user.password())) {
            throw new InvalidCredentialsException();
        }
        
        // 3. Generar tokens (puerto de salida)
        String accessToken = tokenProvider.generateAccessToken(user);
        String refreshToken = tokenProvider.generateRefreshToken(user);
        
        // 4. Retornar respuesta (DTO)
        return new AuthResponse(accessToken, refreshToken, user.toDto());
    }
}
```

## Reglas de Diseño

### 1. Un Use Case por Operación

```java
// ✅ BIEN: Cada caso de uso hace UNA cosa
public class CreateUserUseCase implements CreateUserPort { ... }
public class UpdateUserUseCase implements UpdateUserPort { ... }
public class DeleteUserUseCase implements DeleteUserPort { ... }

// ❌ MAL: Use case genérico que hace todo
public class UserUseCase implements CreateUserPort, UpdateUserPort, DeleteUserPort { ... }
```

### 2. Inyección por Constructor

```java
// ✅ BIEN: Constructor final, inmutabilidad
@Component
@RequiredArgsConstructor
public class CreateUserUseCase implements CreateUserPort {
    private final UserRepositoryPort userRepository;
    private final PasswordHasherPort passwordHasher;
    private final EmailSenderPort emailSender;
}

// ❌ MAL: Field injection
@Component
public class CreateUserUseCase implements CreateUserPort {
    @Autowired
    private UserRepositoryPort userRepository;  // Evitar
}
```

### 3. Transaccionalidad en Use Cases

```java
// ✅ BIEN: @Transactional en el use case
@Override
@Transactional
public UserDto createUser(CreateUserRequest request) {
    // Múltiples operaciones atómicas
    User savedUser = userRepository.save(user);
    auditLogger.log("User created", savedUser.id());
    emailSender.sendWelcomeEmail(savedUser.email());
    return savedUser.toDto();
}

// ❌ MAL: @Transactional en repository o controller
```

### 4. Manejo de Excepciones

```java
@Override
public AuthResponse authenticate(LoginRequest request) {
    // El use case lanza excepciones de dominio
    // NO las catchea, las propaga para que infrastructure las maneje
    User user = userRepository.findByUsername(request.username())
        .orElseThrow(() -> new UserNotFoundException(request.username()));
    
    if (!passwordHasher.matches(request.password(), user.password())) {
        throw new InvalidCredentialsException();  // ✅ Propagar
    }
    
    // ... resto del código
}
```

## Flujo Completo

```
┌─────────────────────────────────────────┐
│  Controller (Infrastructure)            │
│  POST /api/auth/login                   │
└─────────────────────────────────────────┘
                 ↓ llama a
┌─────────────────────────────────────────┐
│  UseCase (Application)                  │
│  AuthenticateUserUseCase                │
│  ─────────────────────────────────────  │
│  1. userRepository.findByUsername()     │
│  2. passwordHasher.matches()            │
│  3. tokenProvider.generate()            │
│  4. Retorna AuthResponse                │
└─────────────────────────────────────────┘
         ↓ usa puertos de salida ↓
┌─────────────────────────────────────────┐
│  Adapters (Infrastructure)              │
│  JpaUserRepository                      │
│  BcryptPasswordHasher                   │
│  JwtTokenProvider                       │
└─────────────────────────────────────────┘
```

## Diferencia con Domain Service

| Use Case | Domain Service |
|----------|----------------|
| Orquestación | Lógica de negocio |
| Con @Transactional | Sin @Transactional |
| Usa puertos de salida | Usa repositorios del dominio |
| Retorna DTOs | Retorna entidades/VOs |
| Delgado | Puede ser complejo |

**Ejemplo de colaboración:**

```java
// UseCase (orquesta)
public class CreateUserUseCase implements CreateUserPort {
    
    private final PasswordDomainService passwordService;  // Domain Service
    private final UserRepositoryPort userRepository;       // Port Out
    
    @Override
    @Transactional
    public UserDto createUser(CreateUserRequest request) {
        // 1. Domain Service valida reglas complejas
        passwordService.validatePassword(request.password(), request.tenantId());
        
        // 2. UseCase orquesta el flujo
        User user = User.create(request.email(), request.password());
        User saved = userRepository.save(user);
        
        return saved.toDto();
    }
}
```

## Testing

### Unit Test con Mocks

```java
@ExtendWith(MockitoExtension.class)
class CreateUserUseCaseTest {
    
    @Mock
    private UserRepositoryPort userRepository;
    
    @Mock
    private PasswordHasherPort passwordHasher;
    
    @InjectMocks
    private CreateUserUseCase useCase;
    
    @Test
    void should_create_user_when_data_valid() {
        // Given
        CreateUs

```java
        // Given
        CreateUserRequest request = new CreateUserRequest("test@email.com", "password123", 1L);
        User user = User.create(request.email(), request.password());
        
        when(userRepository.existsByEmail(request.email())).thenReturn(false);
        when(passwordHasher.hash(request.password())).thenReturn("hashed");
        when(userRepository.save(any(User.class))).thenAnswer(i -> i.getArguments()[0]);
        
        // When
        UserDto result = useCase.createUser(request);
        
        // Then
        assertThat(result.email()).isEqualTo(request.email());
        verify(userRepository).save(any(User.class));
        verify(emailSender).sendWelcomeEmail(request.email());
    }
    
    @Test
    void should_throw_when_email_exists() {
        // Given
        when(userRepository.existsByEmail(any())).thenReturn(true);
        
        // When & Then
        assertThatThrownBy(() -> useCase.createUser(request))
            .isInstanceOf(UserAlreadyExistsException.class);
    }
}
```

### Integration Test

```java
@SpringBootTest
class CreateUserUseCaseIntegrationTest {
    
    @Autowired
    private CreateUserPort createUser;  // Spring inyecta la implementación
    
    @Autowired
    private UserRepositoryPort userRepository;
    
    @BeforeEach
    void setUp() {
        // Limpiar DB antes de cada test
        userRepository.deleteAll();
    }
    
    @Test
    void should_create_user_in_database() {
        // Given
        CreateUserRequest request = new CreateUserRequest(
            "integration@test.com", 
            "SecurePass123!", 
            1L
        );
        
        // When
        UserDto result = createUser.createUser(request);
        
        // Then
        assertThat(result.id()).isNotNull();
        assertThat(result.email()).isEqualTo(request.email());
        
        // Verificar que realmente se guardó en DB
        Optional<User> saved = userRepository.findByEmail(request.email());
        assertThat(saved).isPresent();
    }
}
```

## Patrones Comunes

### Use Case con Validación

```java
@Override
@Transactional
public UserDto createUser(CreateUserRequest request) {
    // Validaciones básicas
    if (request.email() == null || request.email().isBlank()) {
        throw new InvalidEmailException();
    }
    
    // Verificar duplicados
    if (userRepository.existsByEmail(request.email())) {
        throw new UserAlreadyExistsException(request.email());
    }
    
    // Crear y guardar
    User user = User.create(request.email(), request.password());
    User saved = userRepository.save(user);
    
    // Eventos post-creación (asíncronos idealmente)
    emailSender.sendWelcomeEmail(saved.email());
    auditLogger.log("User created", saved.id());
    
    return saved.toDto();
}
```

### Use Case con Múltiplas Operaciones Atómicas

```java
@Override
@Transactional
public void assignRoleToUser(AssignRoleRequest request) {
    // Todas estas operaciones son atómicas
    User user = userRepository.findById(request.userId())
        .orElseThrow(() -> new UserNotFoundException(request.userId()));
    
    Role role = roleRepository.findById(request.roleId())
        .orElseThrow(() -> new RoleNotFoundException(request.roleId()));
    
    // Validar que el tenant permita esta combinación
    tenantValidator.validateRoleAssignment(user.tenantId(), role);
    
    // Asignar rol
    user.addRole(role);
    
    // Guardar cambios
    userRepository.save(user);
    
    // Actualizar cache de permisos
    permissionCache.invalidate(user.id());
    
    // Log de auditoría
    auditLogger.log("Role assigned", Map.of(
        "userId", user.id(),
        "roleId", role.id(),
        "adminId", SecurityContext.getCurrentUserId()
    ));
}
```

---

**Volver a**: [Application Layer](../README.md)  
**Relacionado**: [Port In](../port/in/README.md), [Port Out](../port/out/README.md), [Domain Service](../../domain/service/README.md)

# Application DTO - Data Transfer Objects

## Propósito

Contiene los **objetos de transferencia de datos** que se usan para comunicar información entre capas y con el exterior.

## Características Clave

✅ Inmutables (usando `record` de Java 16+)  
✅ Sin lógica de negocio, solo datos  
✅ Específicos por caso de uso  
✅ Validan datos en el constructor  

## Contenido Esperado

### DTOs de Request (Entrada)

| Archivo | Descripción |
|---------|-------------|
| `LoginRequest.java` | Credenciales para autenticación |
| `CreateUserRequest.java` | Datos para crear usuario |
| `UpdateUserRequest.java` | Datos para actualizar usuario |
| `AssignRoleRequest.java` | User ID + Role ID para asignar rol |
| `RefreshTokenRequest.java` | Refresh token para obtener nuevo access token |
| `CreateTenantRequest.java` | Datos para crear nuevo tenant |

### DTOs de Response (Salida)

| Archivo | Descripción |
|---------|-------------|
| `AuthResponse.java` | Tokens JWT + información de usuario |
| `UserDto.java` | Representación plana de User |
| `RoleDto.java` | Representación plana de Role |
| `PermissionDto.java` | Representación plana de Permission |
| `TenantDto.java` | Representación plana de Tenant |
| `ErrorResponse.java` | Estructura estándar de errores de API |

### DTOs Internos

| Archivo | Descripción |
|---------|-------------|
| `TokenPair.java` | Access token + refresh token juntos |
| `AuthenticatedUser.java` | Usuario ya validado con contexto de seguridad |

## Ejemplos

### Request con Record (Java 16+)

```java
// En application/dto/LoginRequest.java
package com.authcore.application.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record LoginRequest(
    @NotBlank(message = "Username is required")
    String username,
    
    @NotBlank(message = "Password is required")
    String password,
    
    @NotBlank(message = "Tenant ID is required")
    String tenantId
) {}
```

### Response con Record

```java
// En application/dto/AuthResponse.java
package com.authcore.application.dto;

import java.time.Instant;

public record AuthResponse(
    String accessToken,
    String refreshToken,
    String tokenType,
    Instant expiresAt,
    UserDto user
) {
    public AuthResponse(String accessToken, String refreshToken, UserDto user) {
        this(accessToken, refreshToken, "Bearer", 
             Instant.now().plusSeconds(3600), user);
    }
}
```

### DTO Complejo con Builder

```java
// En application/dto/UserDto.java
package com.authcore.application.dto;

import java.util.List;
import java.time.LocalDateTime;

public record UserDto(
    Long id,
    String email,
    String username,
    boolean active,
    List<RoleDto> roles,
    Long tenantId,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {
    // Método factory desde entidad de dominio
    public static UserDto fromDomain(com.authcore.domain.model.User user) {
        return new UserDto(
            user.id().value(),
            user.email().value(),
            user.username().value(),
            user.isActive(),
            user.roles().stream().map(RoleDto::fromDomain).toList(),
            user.tenantId().value(),
            user.createdAt(),
            user.updatedAt()
        );
    }
}
```

### Error Response Estándar

```java
// En application/dto/ErrorResponse.java
package com.authcore.application.dto;

import java.time.Instant;
import java.util.Map;
import java.util.List;

public record ErrorResponse(
    int status,
    String code,
    String message,
    Instant timestamp,
    String path,
    Map<String, String> validationErrors,
    List<String> details
) {
    public static ErrorResponse of(int status, String code, String message) {
        return new ErrorResponse(
            status, code, message, 
            Instant.now(), null, null, null
        );
    }
    
    public static ErrorResponse validationError(
            int status, 
            Map<String, String> errors) {
        return new ErrorResponse(
            status, "VALIDATION_ERROR", "Validation failed",
            Instant.now(), null, errors, null
        );
    }
}
```

## Por Qué Usar Records

### vs Clases Tradicionales

```java
// ❌ MAL: Boilerplate excesivo
public class LoginRequest {
    private final String username;
    private final String password;
    
    public LoginRequest(String username, String password) {
        this.username = username;
        this.password = password;
    }
    
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    
    // equals(), hashCode(), toString()... 50+ líneas
}

// ✅ BIEN: Record, inmutable y conciso
public record LoginRequest(String username, String password) {}
```

### Ventajas de Records

1. **Inmutabilidad automática**: Todos los campos son `final`
2. **Métodos generados**: `equals()`, `hashCode()`, `toString()`
3. **Pattern matching**: Se pueden desestructurar en switch expressions
4. **Menos código**: 1 línea vs 50+ líneas
5. **Serialización JSON**: Jackson los soporta nativamente

## Validación en DTOs

### Con Bean Validation (JSR-380)

```java
public record CreateUserRequest(
    @Email(message = "Invalid email format")
    @NotBlank
    String email,
    
    @Size(min = 8, max = 100, message = "Password must be between 8 and 100 characters")
    @Pattern(regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z]).*$", 
             message = "Password must contain uppercase, lowercase and number")
    String password,
    
    @NotBlank
    String username,
    
    @NotNull
    Long tenantId
) {}
```

### Validación Manual en Constructor

```java
public record CreateUserRequest(
    String email,
    String password,
    String username,
    Long tenantId
) {
    public CreateUserRequest {
        // Validaciones custom
        if (email == null || !email.matches(".+@.+\\..+")) {
            throw new InvalidEmailException();
        }
        
        if (password.length() < 8) {
            throw new PasswordTooShortException(8);
        }
        
        if (username.length() < 3) {
            throw new UsernameTooShortException(3);
        }
    }
}
```

## Mapeo entre Capas

```
┌─────────────────────────────────────┐
│  HTTP Request JSON                  │
│  { "email": "...", "password": "..." } │
└─────────────────────────────────────┘
              ↓ Jackson deserializa
┌─────────────────────────────────────┐
│  DTO (application/dto/)             │
│  CreateUserRequest                  │
└─────────────────────────────────────┘
              ↓ UseCase usa
┌─────────────────────────────────────┐
│  Domain Model (domain/model/)       │
│  User entity                        │
└─────────────────────────────────────┘
              ↓ UseCase retorna
┌─────────────────────────────────────┐
│  DTO (application/dto/)             │
│  UserDto                            │
└─────────────────────────────────────┘
              ↓ Jackson serializa
┌─────────────────────────────────────┐
│  HTTP Response JSON                 │
│  { "id": 1, "email": "...", ... }   │
└─────────────────────────────────────┘
```

## Reglas de Diseño

1. **Un DTO por operación**: No reuses DTOs para diferentes casos de uso
2. **Específico, no genérico**: `CreateUserRequest`, no `UserRequest`
3. **Inmutable**: Usa `record`, evita setters
4. **Sin lógica**: Solo datos, validaciones básicas permitidas
5. **Documentado**: JavaDoc explica qué significa cada campo

## Testing

```java
class CreateUserRequestTest {
    
    @Test
    void should_create_valid_request() {
        var request = new CreateUserRequest(
            "test@email.com",
            "Password123",
            "testuser",
            1L
        );
        
        assertThat(request.email()).isEqualTo("test@email.com");
        assertThat(request.password()).isEqualTo("Password123");
    }
    
    @Test
    void should_throw_when_email_invalid() {
        assertThatThrownBy(() -> 
            new CreateUserRequest("invalid-email", "Password123", "user", 1L)
        ).isInstanceOf(InvalidEmailException.class);
    }
}
```

---

**Volver a**: [Application Layer](../README.md)  
**Relacionado**: [Use Case](../usecase/README.md), [Port In](../port/in/README.md)

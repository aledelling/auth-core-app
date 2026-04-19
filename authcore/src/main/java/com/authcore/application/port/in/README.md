# Application Port In - Puertos de Entrada

## Propósito

Define las **interfaces que los controllers (infrastructure) llamarán** para ejecutar casos de uso.

## Características Clave

✅ Son interfaces que application **provee**  
✅ Infrastructure las **consume** desde controllers  
✅ Cada puerto representa una operación de negocio  
✅ Permiten múltiples implementaciones del mismo caso de uso  

## Contenido Esperado

| Archivo | Descripción |
|---------|-------------|
| `AuthenticateUserPort.java` | Autenticar usuario con credenciales |
| `CreateUserPort.java` | Crear nuevo usuario en el sistema |
| `UpdateUserPort.java` | Actualizar datos de usuario existente |
| `DeleteUserPort.java` | Eliminar usuario (soft delete) |
| `AssignRoleToUserPort.java` | Asignar rol a usuario |
| `RevokeRoleFromUserPort.java` | Remover rol de usuario |
| `ValidateTokenPort.java` | Validar token JWT |
| `RefreshTokenPort.java` | Generar nuevo token desde refresh token |
| `CreateTenantPort.java` | Crear nuevo tenant/organización |

## Ejemplo

```java
// En application/port/in/AuthenticateUserPort.java
package com.authcore.application.port.in;

import com.authcore.application.dto.AuthResponse;
import com.authcore.application.dto.LoginRequest;

public interface AuthenticateUserPort {
    
    /**
     * Autentica un usuario con sus credenciales y retorna tokens JWT.
     * 
     * @param request Credenciales del usuario
     * @return Response con access token y refresh token
     * @throws InvalidCredentialsException si las credenciales son inválidas
     * @throws UserNotFoundException si el usuario no existe
     */
    AuthResponse authenticate(LoginRequest request);
}
```

## Relación con Use Cases

```
┌─────────────────────────────────────┐
│  port/in/                           │
│  interface AuthenticateUserPort     │  ← Puerto (contrato)
└─────────────────────────────────────┘
              ↑ implementa
┌─────────────────────────────────────┐
│  usecase/                           │
│  class AuthenticateUserUseCase      │  ← Implementación concreta
│    implements AuthenticateUserPort  │
└─────────────────────────────────────┘
              ↑ usa desde
┌─────────────────────────────────────┐
│  infrastructure/adapter/in/rest/    │
│  class AuthController               │  ← Controller inyecta el puerto
└─────────────────────────────────────┘
```

## Por Qué Esta Ubicación

### 1. Inversión de Dependencias

El controller (infraestructura) depende de la abstracción (puerto), no de la implementación (use case):

```java
// Controller depende del puerto, NO del use case directamente
@RestController
public class AuthController {
    
    private final AuthenticateUserPort authenticateUser;  // ✅ Bien
    
    // private final AuthenticateUserUseCase useCase;     // ❌ Mal: depende de implementación
}
```

### 2. Permite Múltiples Implementaciones

Puedes tener diferentes implementaciones del mismo puerto:

```java
// Implementación normal
public class AuthenticateUserUseCase implements AuthenticateUserPort { ... }

// Implementación con caching
public class CachedAuthenticateUserUseCase implements AuthenticateUserPort { ... }

// Implementación para testing
public class FakeAuthenticateUserPort implements AuthenticateUserPort { ... }
```

### 3. Facilita Testing

El controller puede testearse mockeando el puerto:

```java
@Mock
private AuthenticateUserPort authenticateUser;

@InjectMocks
private AuthController controller;

@Test
void should_return_token_when_login_success() {
    when(authenticateUser.authenticate(any()))
        .thenReturn(new AuthResponse("token", userDto));
    
    // Test del controller
}
```

## Convenciones de Nombres

| Patrón | Ejemplo |
|--------|---------|
| Verbo + Sustantivo + `Port` | `AuthenticateUserPort` |
| Sustantivo + `ManagementPort` | `UserManagementPort` |
| Operación específica + `Port` | `ValidateTokenPort` |

## DTOs Relacionados

Los puertos usan DTOs definidos en `application/dto/`:

```java
public interface CreateUserPort {
    UserDto createUser(CreateUserRequest request);  // DTO de entrada → DTO de salida
}

public interface ValidateTokenPort {
    TokenValidationResult validate(String token);   // String → DTO complejo
}
```

## Diferencia con Port Out

| Port In (esta carpeta) | Port Out (`port/out/`) |
|------------------------|------------------------|
| Lo que application **ofrece** | Lo que application **necesita** |
| Infrastructure **llama** | Infrastructure **implementa** |
| Casos de uso lo **implementan** | Use cases lo **usan** |
| Ej: `AuthenticateUserPort` | Ej: `UserRepositoryPort` |

## Testing

```java
// Test de integración del puerto
@SpringBootTest
class AuthenticateUserPortIntegrationTest {
    
    @Autowired
    private AuthenticateUserPort authenticateUser;  // Spring inyecta la implementación
    
    @Test
    void should_authenticate_with_valid_credentials() {
        LoginRequest request = new LoginRequest("admin", "password123", "tenant1");
        
        AuthResponse response = authenticateUser.authenticate(request);
        
        assertThat(response.accessToken()).isNotBlank();
        assertThat(response.user().username()).isEqualTo("admin");
    }
}
```

---

**Volver a**: [Application Layer](../README.md)  
**Relacionado**: [Use Case](../usecase/README.md), [DTO](../dto/README.md)

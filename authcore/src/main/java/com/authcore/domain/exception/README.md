# Domain Exception - Excepciones del Dominio

## Propósito

Define las **excepciones específicas del negocio** que comunican errores de reglas de dominio de forma clara y explícita.

## Características Clave

✅ Extienden de `RuntimeException` (unchecked)  
✅ No dependen de frameworks ni HTTP  
✅ Mensajes claros para debugging  
✅ Pueden llevar metadata del error (IDs, valores inválidos)  

## Por Qué Excepciones Propias

### vs Exception Genéricas

```java
// ❌ MAL: Pierdes información del contexto
throw new IllegalArgumentException("Invalid data");

// ✅ BIEN: Sabes exactamente qué pasó
throw new UserNotFoundException(userId);
throw new InvalidCredentialsException(username);
throw new RoleAlreadyExistsException(roleName);
```

### Ventajas

1. **Claridad**: El nombre dice exactamente qué falló
2. **Manejo específico**: Puedes catchear diferentes excepciones de forma distinta
3. **Traducción a HTTP**: Infrastructure mapea cada excepción a un status code apropiado
4. **Documentación viva**: Las excepciones documentan los casos de error posibles

## Contenido Esperado

| Archivo | Cuándo Se Lanza | HTTP Status Típico |
|---------|-----------------|-------------------|
| `UserNotFoundException.java` | Usuario no existe en DB | 404 Not Found |
| `InvalidCredentialsException.java` | Password o username incorrecto | 401 Unauthorized |
| `UserAlreadyExistsException.java` | Email o username duplicado | 409 Conflict |
| `RoleAlreadyExistsException.java` | Rol duplicado | 409 Conflict |
| `InsufficientPermissionsException.java` | Usuario no tiene permiso para acción | 403 Forbidden |
| `TokenExpiredException.java` | JWT expirado | 401 Unauthorized |
| `InvalidTokenException.java` | JWT malformado o inválido | 401 Unauthorized |
| `InvalidTenantException.java` | Tenant no existe o inactivo | 400 Bad Request |
| `PasswordPolicyViolationException.java` | Password no cumple políticas | 400 Bad Request |
| `EmailAlreadyConfirmedException.java` | Intentar confirmar email ya confirmado | 409 Conflict |

## Estructura de una Excepción

```java
// En domain/exception/UserNotFoundException.java
package com.authcore.domain.exception;

public class UserNotFoundException extends RuntimeException {
    
    private final String username;
    
    public UserNotFoundException(String username) {
        super(String.format("User not found with username: %s", username));
        this.username = username;
    }
    
    public String getUsername() {
        return username;
    }
}
```

## Con Metadata Adicional

```java
// En domain/exception/PasswordPolicyViolationException.java
public class PasswordPolicyViolationException extends RuntimeException {
    
    private final List<String> violatedRules;
    
    public PasswordPolicyViolationException(List<String> violatedRules) {
        super("Password does not meet policy requirements");
        this.violatedRules = violatedRules;
    }
    
    public List<String> getViolatedRules() {
        return violatedRules;
    }
}
```

## Jerarquía de Excepciones

```
RuntimeException
    └── AuthCoreDomainException (abstracta, base común)
            ├── UserNotFoundException
            ├── InvalidCredentialsException
            ├── InsufficientPermissionsException
            └── ...
```

**Opcional**: Crear una clase base para todas las excepciones del dominio:

```java
public abstract class AuthCoreDomainException extends RuntimeException {
    protected AuthCoreDomainException(String message) {
        super(message);
    }
    
    // Métodos comunes, logging, errorCode, etc.
}
```

## Cómo Se Usan en el Flujo

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Domain lanza excepción                                   │
│    throw new UserNotFoundException(username);                │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Application la propaga (no la maneja)                    │
│    public UserDto getUser(String username) { ... }           │
│    // Si falla, la excepción sube                            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Infrastructure traduce a HTTP                            │
│    @ExceptionHandler(UserNotFoundException.class)            │
│    public ResponseEntity<ErrorResponse> handle(...) {        │
│        return ResponseEntity.status(404).body(...);          │
│    }                                                         │
└─────────────────────────────────────────────────────────────┘
```

## Mapeo a HTTP en Infrastructure

```java
// En infrastructure/adapter/in/rest/GlobalExceptionHandler.java

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex) {
        
        ErrorResponse error = ErrorResponse.builder()
            .code("USER_NOT_FOUND")
            .message(ex.getMessage())
            .status(404)
            .build();
        
        return ResponseEntity.status(404).body(error);
    }
    
    @ExceptionHandler(InvalidCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleInvalidCredentials(
            InvalidCredentialsException ex) {
        
        return ResponseEntity.status(401)
            .body(ErrorResponse.authError("INVALID_CREDENTIALS"));
    }
    
    // ... más handlers
}
```

## Testing de Excepciones

```java
@Test
void should_throw_user_not_found_when_username_does_not_exist() {
    // Given
    when(userRepository.findByUsername("nonexistent"))
        .thenThrow(new UserNotFoundException("nonexistent"));
    
    // When & Then
    assertThatThrownBy(() -> 
        useCase.getUser("nonexistent")
    ).isInstanceOf(UserNotFoundException.class)
     .hasMessageContaining("nonexistent");
}
```

## Reglas de Diseño

1. **Nombres descriptivos**: `UserNotFoundException`, no `UserException`
2. **Unchecked exceptions**: Extiende de `RuntimeException`
3. **Constructor con mensaje**: Siempre incluye mensaje claro
4. **Metadata útil**: Guarda IDs o valores que causaron el error
5. **Sin dependencias**: Solo Java estándar y otras clases del dominio

---

**Volver a**: [Domain Layer](../README.md)

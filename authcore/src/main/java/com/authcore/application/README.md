# Application Layer - Capa de Aplicación

## Propósito

La capa **Application** orquesta el flujo de la aplicación. Contiene los casos de uso que coordinan las entidades del dominio para ejecutar tareas específicas.

## Características Clave

✅ **Independiente de frameworks**: Sin anotaciones Spring (@RestController, @Service)  
✅ **Orquestación**: Coordina entidades, repositorios y servicios externos  
✅ **Agnóstica al transporte**: No sabe si viene de HTTP, messaging o CLI  
✅ **Delgada**: Solo coordina, la lógica real está en el dominio  

## Estructura Interna

```
application/
├── port/
│   ├── in/          # Puertos de entrada (interfaces que usa el controller)
│   └── out/         # Puertos de salida (interfaces que implementa infraestructura)
├── usecase/         # Casos de uso concretos
└── dto/             # Objetos de transferencia de datos
```

### 📁 `port/in/` - Puertos de Entrada

**Qué contiene:**
- `AuthenticateUserPort.java` - Interface para autenticación
- `CreateUserPort.java` - Interface para creación de usuarios
- `ManageRolesPort.java` - Interface para gestión de roles
- `ValidateTokenPort.java` - Interface para validación de JWT

**Por qué así:**
- Define el contrato que los controllers deben llamar
- Permite tener múltiples implementaciones del mismo caso de uso
- Facilita testing con mocks

**Ejemplo:**
```java
public interface AuthenticateUserPort {
    AuthResponse authenticate(LoginRequest request);
}
```

### 📁 `port/out/` - Puertos de Salida

**Qué contiene:**
- `UserRepositoryPort.java` - Extiende la interfaz del dominio con métodos específicos
- `JwtTokenProviderPort.java` - Generación y validación de tokens
- `PasswordHasherPort.java` - Hashing de contraseñas
- `EmailSenderPort.java` - Envío de correos (recuperación de password)
- `AuditLoggerPort.java` - Logging de auditoría

**Por qué así:**
- El dominio define interfaces genéricas, application las especializa
- Infrastructure implementa estos puertos
- Permite cambiar implementaciones sin tocar el dominio

### 📁 `usecase/` - Casos de Uso

**Qué contiene:**
- `AuthenticateUserUseCase.java` - Implementa AuthenticateUserPort
- `CreateUserUseCase.java` - Implementa CreateUserPort
- `AssignRoleToUserUseCase.java` - Asigna roles a usuarios
- `RefreshTokenUseCase.java` - Refresca tokens JWT
- `RevokeTokenUseCase.java` - Revoca tokens activos
- `CreateTenantUseCase.java` - Crea nuevos tenants

**Patrones:**
- **1 UseCase = 1 operación de negocio**
- Cada use case implementa un puerto de entrada
- Inyecta puertos de salida vía constructor (dependency injection)
- Maneja transacciones (@Transactional va aquí, NO en el dominio)

**Ejemplo de flujo:**
```java
public class AuthenticateUserUseCase implements AuthenticateUserPort {
    private final UserRepositoryPort userRepository;
    private final PasswordHasherPort passwordHasher;
    private final JwtTokenProviderPort tokenProvider;
    
    @Override
    public AuthResponse authenticate(LoginRequest request) {
        // 1. Buscar usuario (puerto de salida)
        User user = userRepository.findByUsername(request.username());
        
        // 2. Validar password (servicio externo)
        if (!passwordHasher.matches(request.password(), user.password())) {
            throw new InvalidCredentialsException();
        }
        
        // 3. Generar token (puerto de salida)
        String token = tokenProvider.generate(user);
        
        // 4. Retornar respuesta
        return new AuthResponse(token, user.toDto());
    }
}
```

### 📁 `dto/` - Data Transfer Objects

**Qué contiene:**
- `LoginRequest.java` - DTO de entrada para login
- `AuthResponse.java` - DTO de salida con token y user info
- `UserDto.java` - Representación plana de User
- `RoleDto.java` - Representación plana de Role
- `CreateUserRequest.java` - DTO para crear usuario
- `ErrorResponse.java` - DTO para errores de API

**Por qué así:**
- Evita exponer entidades del dominio directamente
- Controla qué datos se envían/reciben
- Permite versionado de API sin romper el dominio
- Usa records de Java 16+ para inmutabilidad

**Ejemplo:**
```java
public record LoginRequest(String username, String password, String tenantId) {}
public record AuthResponse(String accessToken, String refreshToken, UserDto user) {}
```

## Reglas de Oro

1. **NUNCA** importes clases de `infrastructure`
2. **PUEDES** importar clases de `domain`
3. **SIEMPRE** usa DTOs para comunicación entre capas
4. **SIEMPRE** maneja excepciones de dominio y tradúcelas a respuestas apropiadas
5. **NUNCA** pongas lógica de negocio aquí, solo orquestación

## Dependencias

```
Domain ← Application → Infrastructure (vía puertos)
```

- Apunta hacia atrás: conoce el dominio
- Es conocido por infraestructura: los adapters llaman a los use cases

## Testing

- Pruebas unitarias con mocks de puertos de salida
- Pruebas de integración con SpringBootTest (solo esta capa + infra)
- Mocks: Mockito para puertos de salida
- Cobertura mínima: 85%

---

**Capas relacionadas:**
- [Domain Layer](../domain/README.md) - Lo que application orquesta
- [Infrastructure Layer](../infrastructure/README.md) - Quién llama a application

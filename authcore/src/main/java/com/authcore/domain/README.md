# Domain Layer - Capa de Dominio

## Propósito

El **Domain** es el corazón del sistema. Contiene la lógica de negocio esencial y las reglas empresariales que definen qué hace el sistema AuthCore.

## Características Clave

✅ **Cero dependencias externas**: No importa Spring, JPA, ni ningún framework  
✅ **POJOs puros**: Objetos Java simples sin anotaciones  
✅ **Autocontenido**: Toda la lógica crítica vive aquí  
✅ **Estable**: Cambia poco, solo cuando cambian las reglas de negocio  

## Estructura Interna

```
domain/
├── model/           # Entidades y objetos de valor
├── repository/      # Interfaces de repositorio (puertos)
├── service/         # Servicios de dominio (lógica compleja)
└── exception/       # Excepciones específicas del dominio
```

### 📁 `model/` - Entidades y Objetos de Valor

**Qué contiene:**
- `User.java` - Entidad principal con lógica de autenticación
- `Role.java` - Roles del sistema
- `Permission.java` - Permisos granulares
- `Tenant.java` - Información del multitenant
- `JwtToken.java` - Objeto de valor para tokens
- `Password.java` - Objeto de valor con validación de contraseñas

**Por qué así:**
- Las entidades tienen métodos con lógica (no solo getters/setters)
- Los objetos de valor son inmutables y validan su estado interno
- Ejemplo: `user.hasRole("ADMIN")` en lugar de `user.getRoles().contains(...)`

### 📁 `repository/` - Interfaces de Repositorio

**Qué contiene:**
- `UserRepository.java`
- `RoleRepository.java`
- `PermissionRepository.java`
- `TenantRepository.java`
- `JwtTokenRepository.java`

**Por qué así:**
- Son **interfaces**, NO implementaciones
- Definen el contrato de persistencia sin saber CÓMO se guarda
- La implementación real está en `infrastructure/adapter/out/persistence/`
- Ejemplo: `interface UserRepository { User findByUsername(String username); }`

### 📁 `service/` - Servicios de Dominio

**Qué contiene:**
- `UserDomainService.java` - Lógica compleja que involucra múltiples entidades
- `PasswordDomainService.java` - Reglas de validación y hashing
- `PermissionEvaluator.java` - Evaluación de permisos basados en políticas

**Cuándo usar:**
- Cuando una operación requiere coordinar varias entidades
- Para lógica que no pertenece a una sola entidad
- Operaciones que necesitan transaccionalidad a nivel de dominio

### 📁 `exception/` - Excepciones del Dominio

**Qué contiene:**
- `UserNotFoundException.java`
- `InvalidCredentialsException.java`
- `RoleAlreadyExistsException.java`
- `InsufficientPermissionsException.java`
- `TokenExpiredException.java`
- `InvalidTenantException.java`

**Por qué así:**
- Excepciones específicas comunican mejor el error
- El dominio lanza, la infraestructura traduce a HTTP status codes
- Todas extienden de RuntimeException unchecked

## Reglas de Oro

1. **NUNCA** importes clases de `application` o `infrastructure`
2. **NUNCA** uses anotaciones de frameworks (@Entity, @Component, etc.)
3. **SIEMPRE** valida invariantes dentro de las entidades
4. **SIEMPRE** usa objetos de valor para conceptos primitivos (email, password)

## Ejemplo de Flujo

```
Controller (Infra) → UseCase (App) → Entity.method() (Domain)
                                              ↓
                                      Repository Interface (Domain)
                                              ↓
                                   Repository Implementation (Infra)
```

## Testing

- Pruebas unitarias puras (JUnit + AssertJ)
- Sin mocks pesados, las entidades son POJOs testables
- Cobertura mínima requerida: 90%

---

**Siguiente capa**: [Application Layer](../application/README.md)

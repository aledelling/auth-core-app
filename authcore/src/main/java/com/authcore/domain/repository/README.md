# Domain Repository - Interfaces de Repositorio

## Propósito

Define los **contratos de persistencia** que el dominio necesita, sin saber CÓMO se implementarán.

## Características Clave

✅ Son **interfaces**, NO implementaciones  
✅ No tienen anotaciones Spring (@Repository va en infraestructura)  
✅ Usan tipos del dominio (User, Role), no entidades JPA  
✅ El dominio las define, Infrastructure las implementa  

## Contenido Esperado

| Archivo | Métodos Típicos |
|---------|-----------------|
| `UserRepository.java` | `findByUsername()`, `save()`, `deleteById()`, `existsByEmail()` |
| `RoleRepository.java` | `findByName()`, `save()`, `findAll()` |
| `PermissionRepository.java` | `findByCode()`, `findAllByRole()` |
| `TenantRepository.java` | `findBySubdomain()`, `save()`, `findAllActive()` |
| `JwtTokenRepository.java` | `save()`, `findByToken()`, `deleteExpired()` |

## Ejemplo

```java
// En domain/repository/UserRepository.java
public interface UserRepository {
    Optional<User> findByUsername(Username username);
    Optional<User> findByEmail(Email email);
    User save(User user);
    void deleteById(UserId id);
    boolean existsByEmail(Email email);
    List<User> findAllByTenant(TenantId tenantId);
}
```

## Por Qué Así

### 1. Inversión de Dependencias

```
Domain (define interfaz) ← Application (usa interfaz) ← Infrastructure (implementa)
```

El dominio NO depende de JPA/Hibernate. Es al revés.

### 2. Testabilidad

Puedes mockear el repositorio en tests sin necesitar base de datos:

```java
@Mock
private UserRepository userRepository;

@Test
void testSomething() {
    when(userRepository.findByUsername(any())).thenReturn(Optional.of(user));
}
```

### 3. Cambios de Tecnología Sin Dolor

Hoy usas JPA, mañana puedes usar:
- JDBC directo
- MongoDB
- DynamoDB
- Un servicio REST externo

Solo cambias la implementación en `infrastructure/adapter/out/persistence/`, el dominio no se entera.

## Relación con Infrastructure

```
┌─────────────────────────────────────┐
│  domain/repository/                 │
│  interface UserRepository { ... }   │  ← Puerto (contrato)
└─────────────────────────────────────┘
              ↑ implementa
┌─────────────────────────────────────┐
│  infrastructure/adapter/out/        │
│  class JpaUserRepository            │  ← Adapter (implementación)
│    implements UserRepository { ... }│
└─────────────────────────────────────┘
```

## Reglas de Diseño

1. **Métodos expresivos**: Nombres que digan qué hacen (`findActiveByUsername`)
2. **Retorna Optionals**: Para búsquedas que pueden no encontrar nada
3. **Sin excepciones checked**: Usa RuntimeExceptions del dominio
4. **Tipos fuertes**: Usa objetos de valor (`Email`, `Username`), no Strings

---

**Volver a**: [Domain Layer](../README.md)  
**Siguiente**: [Infrastructure Persistence](../../infrastructure/adapter/out/persistence/README.md)

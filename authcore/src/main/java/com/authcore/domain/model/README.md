# Domain Model - Modelo de Dominio

## Propósito

Contiene las **entidades** y **objetos de valor** que representan los conceptos fundamentales del negocio de autenticación y autorización.

## Contenido Esperado

### Entidades Principales

| Archivo | Descripción |
|---------|-------------|
| `User.java` | Usuario del sistema con credenciales, roles y estado |
| `Role.java` | Rol con nombre, descripción y permisos asociados |
| `Permission.java` | Permiso granular (ej: `user:create`, `role:delete`) |
| `Tenant.java` | Tenant/organización para aislamiento multitenant |
| `JwtToken.java` | Representación de token JWT con metadata |

### Objetos de Valor

| Archivo | Descripción |
|---------|-------------|
| `Email.java` | Email validado con regex, inmutable |
| `Password.java` | Password con validación de fortaleza, inmutable |
| `Username.java` | Username único con reglas de formato |

## Reglas de Diseño

1. **Entidades tienen identidad**: Se comparan por ID, no por atributos
2. **Objetos de valor son inmutables**: Una vez creados, no cambian
3. **Validan invariantes en el constructor**: Estado inválido = excepción inmediata
4. **Métodos con lógica**: No solo getters/setters, tienen comportamiento

## Ejemplo de Entidad

```java
// NO HACER: Entity anémica (solo getters/setters)
public class User {
    private Long id;
    private String email;
    // ... getters y setters
}

// HACER: Entidad rica con lógica
public class User {
    private final UserId id;
    private final Email email;
    private boolean active;
    
    public void activate() {
        if (this.active) {
            throw new UserAlreadyActiveException(id);
        }
        this.active = true;
    }
    
    public boolean hasRole(RoleName roleName) {
        return roles.stream()
            .anyMatch(role -> role.getName().equals(roleName));
    }
}
```

## Por Qué Esta Ubicación

- **Aislamiento**: Al estar en `domain/model`, estas clases NO pueden importar nada de fuera
- **Estabilidad**: Los conceptos de negocio cambian menos que la tecnología
- **Reusabilidad**: Puedes copiar estas entidades a otro proyecto y funcionarán igual

---

**Volver a**: [Domain Layer](../README.md)

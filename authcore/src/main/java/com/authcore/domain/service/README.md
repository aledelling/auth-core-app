# Domain Service - Servicios de Dominio

## Propósito

Contiene lógica de negocio que **involucra múltiples entidades** o que no pertenece naturalmente a una sola entidad.

## Cuándo Usar un Domain Service

### ✅ Casos Apropiados

1. **Operaciones que cruzan entidades**:
   ```java
   // Transferir permisos entre roles requiere múltiples entidades
   permissionTransferService.transfer(sourceRole, targetRole, permissions);
   ```

2. **Lógica compleja que no es de una entidad específica**:
   ```java
   // Validar que una combinación de roles sea válida
   roleValidationService.validateRoleCombination(user, newRoles);
   ```

3. **Reglas de negocio que requieren servicios externos** (vía puertos):
   ```java
   // Verificar password contra políticas externas
   passwordPolicyService.checkCompliance(password, tenantPolicies);
   ```

### ❌ Casos NO Apropiados

1. **CRUD simple de una entidad** → Va en la entidad misma
2. **Orquestación de casos de uso** → Va en Application UseCase
3. **Acceso a infraestructura directa** → Va en Infrastructure Adapter

## Contenido Esperado

| Archivo | Responsabilidad |
|---------|-----------------|
| `UserDomainService.java` | Lógica compleja de usuarios que involucra roles, tenants y permisos |
| `PasswordDomainService.java` | Validación de políticas de contraseñas, hashing |
| `PermissionEvaluator.java` | Evaluación de permisos basados en políticas RBAC/ABAC |
| `TenantProvisioningService.java` | Creación inicial de tenant con datos por defecto |

## Ejemplo

```java
// En domain/service/PasswordDomainService.java
public class PasswordDomainService {
    
    private final PasswordPolicyRepository policyRepository;
    
    public PasswordDomainService(PasswordPolicyRepository policyRepository) {
        this.policyRepository = policyRepository;
    }
    
    public void validatePassword(Password password, TenantId tenantId) {
        PasswordPolicy policy = policyRepository.findByTenant(tenantId);
        
        if (password.length() < policy.getMinLength()) {
            throw new PasswordTooShortException(policy.getMinLength());
        }
        
        if (!policy.isSatisfiedBy(password)) {
            throw new PasswordDoesNotMeetPolicyException(policy.getRequirements());
        }
    }
    
    public Password hash(Password password) {
        // Lógica de hashing que podría usar diferentes algoritmos
        return Password.hashWithBCrypt(password);
    }
}
```

## Diferencia con Application UseCase

| Domain Service | Application UseCase |
|---------------|---------------------|
| Lógica de negocio pura | Orquestación de flujo |
| Sin @Transactional | Con @Transactional |
| Usa repositorios del dominio | Usa puertos de salida |
| Retorna entidades/VOs | Retorna DTOs |

**Ejemplo de flujo:**

```java
// Application UseCase (orquesta)
public class CreateUserUseCase implements CreateUserPort {
    public UserDto createUser(CreateUserRequest request) {
        // 1. Llamar al domain service para validación
        passwordDomainService.validatePassword(request.password(), request.tenantId());
        
        // 2. Crear entidad
        User user = User.create(request.email(), request.password());
        
        // 3. Guardar vía repositorio
        userRepository.save(user);
        
        // 4. Retornar DTO
        return user.toDto();
    }
}
```

## Inyección de Dependencias

Los Domain Services pueden recibir:
- ✅ Otros servicios de dominio
- ✅ Repositorios del dominio (interfaces)
- ❌ Puertos de aplicación (Application Port)
- ❌ Implementaciones de infraestructura

## Testing

```java
class PasswordDomainServiceTest {
    
    @Mock
    private PasswordPolicyRepository policyRepository;
    
    @InjectMocks
    private PasswordDomainService service;
    
    @Test
    void should_throw_when_password_too_short() {
        // Given
        when(policyRepository.findByTenant(any()))
            .thenReturn(new PasswordPolicy(8, true, true));
        
        // When & Then
        assertThatThrownBy(() -> 
            service.validatePassword(Password.create("123"), TENANT_ID)
        ).isInstanceOf(PasswordTooShortException.class);
    }
}
```

---

**Volver a**: [Domain Layer](../README.md)

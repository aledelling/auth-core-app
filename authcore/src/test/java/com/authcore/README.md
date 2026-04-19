# Test Layer - Capa de Pruebas

## Propósito

La carpeta **test** contiene todas las pruebas automatizadas que garantizan la calidad, funcionalidad y seguridad del módulo AuthCore.

## Estrategia de Testing

Seguimos la **Pirámide de Testing**:

```
        /\
       /  \      E2E / Integración (pocas, lentas)
      /----\     
     /      \    Integration / Componente (algunas)
    /--------\   
   /          \  Unitarias (muchas, rápidas)
  /------------\ 
```

## Estructura Interna

```
test/
├── java/com/authcore/
│   ├── domain/            # Tests unitarios del dominio
│   ├── application/       # Tests unitarios de casos de uso
│   ├── infrastructure/    # Tests de integración de adapters
│   └── e2e/               # Tests end-to-end de la API completa
└── resources/
    ├── sql/               # Scripts SQL para fixtures de test
    ├── wiremock/          # Mappings para mocks de APIs externas
    └── application-test.yml # Configuración específica para tests
```

### 📁 `domain/` - Tests Unitarios de Dominio

**Qué probar:**
- Entidades: métodos con lógica de negocio
- Objetos de valor: validaciones internas
- Servicios de dominio: reglas complejas
- Excepciones: se lanzan en los casos correctos

**Características:**
- ✅ Sin Spring (JUnit puro)
- ✅ Sin mocks (las entidades son POJOs)
- ✅ Rápidos (< 10ms por test)
- ✅ 90%+ cobertura requerida

**Ejemplo:**
```java
class UserTest {
    @Test
    void should_activate_user_when_confirm_email() {
        User user = User.create("test@email.com", "password123");
        assertThat(user.isActive()).isFalse();
        
        user.confirmEmail();
        assertThat(user.isActive()).isTrue();
    }
    
    @Test
    void should_throw_when_password_too_short() {
        assertThatThrownBy(() -> User.create("test@email.com", "123"))
            .isInstanceOf(InvalidPasswordException.class);
    }
}
```

### 📁 `application/` - Tests Unitarios de Aplicación

**Qué probar:**
- Casos de uso: orquestación correcta
- DTOs: serialización/deserialización
- Puertos: contratos bien definidos

**Características:**
- ✅ Con mocks de puertos de salida (Mockito)
- ✅ Sin levantar contexto Spring completo
- ✅ Rápidos (< 50ms por test)
- ✅ 85%+ cobertura requerida

**Ejemplo:**
```java
class AuthenticateUserUseCaseTest {
    @Mock
    private UserRepositoryPort userRepository;
    
    @Mock
    private PasswordHasherPort passwordHasher;
    
    @InjectMocks
    private AuthenticateUserUseCase useCase;
    
    @Test
    void should_authenticate_when_credentials_valid() {
        // Given
        User user = mock(User.class);
        when(userRepository.findByUsername("test")).thenReturn(user);
        when(passwordHasher.matches("pass123", user.password())).thenReturn(true);
        
        // When
        AuthResponse response = useCase.authenticate(new LoginRequest("test", "pass123", "tenant1"));
        
        // Then
        assertThat(response.accessToken()).isNotBlank();
    }
}
```

### 📁 `infrastructure/` - Tests de Integración

**Qué probar:**
- Controllers: endpoints HTTP responden correctamente
- Repositorios JPA: consultas a base de datos
- Filtros de seguridad: JWT se valida correctamente
- Configuraciones: beans se cargan como esperado

**Características:**
- ✅ Con @SpringBootTest (contexto completo o slice)
- ✅ Con Testcontainers para DB/Redis reales
- ✅ WireMock para APIs externas
- ✅ Más lentos (100-500ms por test)
- ✅ 75%+ cobertura requerida

**Ejemplo:**
```java
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class AuthControllerIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = 
        PostgreSQLContainer.forVersion("15-alpine");
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void should_return_token_when_login_success() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"admin\",\"password\":\"secret\",\"tenantId\":\"t1\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").exists());
    }
}
```

### 📁 `e2e/` - Tests End-to-End

**Qué probar:**
- Flujos completos de usuario
- Escenarios reales de producción
- Integración entre todos los componentes

**Características:**
- ✅ Ambiente casi idéntico a producción
- ✅ Muy lentos (segundos por test)
- ✅ Pocos tests (solo flujos críticos)
- ✅ Se ejecutan en CI/CD antes de deploy

**Ejemplos de flujos:**
1. Registro → Confirmación email → Login → Acceso a recurso protegido
2. Login → Crear rol → Asignar rol → Verificar permisos
3. Login → Revocar token → Intentar usar token revocado (debe fallar)

### 📁 `resources/sql/` - Fixtures de Datos

**Qué contiene:**
- `insert-users.sql` - Usuarios de prueba
- `insert-roles.sql` - Roles y permisos predefinidos
- `insert-tenants.sql` - Tenants para testing multitenant
- `cleanup.sql` - Limpieza post-test

**Uso:**
```java
@Sql(scripts = "/sql/insert-users.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/sql/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
class UserRepositoryTest { ... }
```

### 📁 `resources/wiremock/` - Mocks de APIs Externas

**Qué contiene:**
- `mappings/email-service.json` - Mock del servicio de emails
- `mappings/oauth-provider.json` - Mock de proveedor OAuth externo

**Ventajas:**
- No depende de servicios externos reales
- Control total sobre respuestas (éxito, error, timeout)
- Tests deterministas y reproducibles

### 📄 `application-test.yml` - Configuración de Tests

**Configuración clave:**
```yaml
spring:
  datasource:
    url: jdbc:tc:postgresql:15-alpine:///authcore_test
  jpa:
    hibernate:
      ddl-auto: create-drop
  security:
    jwt:
      secret: test-secret-key-for-testing-only
      expiration: 3600000

logging:
  level:
    com.authcore: DEBUG
```

## Herramientas Utilizadas

| Herramienta | Propósito |
|-------------|-----------|
| **JUnit 5** | Framework de testing |
| **AssertJ** | Assertions fluents y legibles |
| **Mockito** | Mocking de dependencias |
| **Testcontainers** | DB/Redis reales en containers Docker |
| **WireMock** | Mock de APIs HTTP externas |
| **RestAssured** | Testing de APIs REST (opcional) |
| **ArchUnit** | Validar arquitectura (dependencias entre capas) |

## ArchUnit - Validación Arquitectónica

**Qué prueba:**
- Domain no importa Application ni Infrastructure
- Application no importa Infrastructure
- Controllers solo llaman Use Cases, noRepositories directamente

**Ejemplo:**
```java
@AnalyzeClasses(packages = "com.authcore")
class ArchitectureTest {
    @ArchTest
    static final ArchRule domain_should_not_depend_on_infra =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");
}
```

## Ejecución de Tests

```bash
# Todos los tests
./mvnw test

# Solo unitarios rápidos
./mvnw test -Dtest="**/*Test.java" -Dgroups=unit

# Solo integración (lentos)
./mvnw test -Dgroups=integration

# Con cobertura
./mvnw test jacoco:report

# Solo tests de una capa
./mvnw test -Dtest="com.authcore.domain.**"
```

## Cobertura Mínima Requerida

| Capa | Cobertura Mínima |
|------|------------------|
| Domain | 90% |
| Application | 85% |
| Infrastructure | 75% |
| **Total** | **80%** |

## Continuous Testing en CI/CD

```yaml
# GitHub Actions example
- name: Run Tests
  run: ./mvnw clean verify
  
- name: Check Coverage
  run: |
    coverage=$(cat target/site/jacoco/jacoco.xml | grep coverage)
    if [ $coverage -lt 80 ]; then exit 1; fi
```

---

**Consejo**: Ejecuta tests unitarios frecuentemente (`mvn test`), deja integración para pre-commit y E2E para CI/CD.

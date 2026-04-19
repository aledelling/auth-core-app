# AuthCore - Estructura del Proyecto

## Arquitectura Hexagonal (Ports & Adapters)

Este proyecto sigue la **Arquitectura Hexagonal** para garantizar:
- **Independencia tecnológica**: El dominio no depende de frameworks externos
- **Testabilidad**: Cada capa puede probarse de forma aislada
- **Mantenibilidad**: Cambios en infraestructura no afectan la lógica de negocio
- **Reusabilidad**: Módulo portable a cualquier proyecto Spring Boot

```
authcore/
├── domain/                    # Capa de Dominio (Enterprise Business Rules)
├── application/               # Capa de Aplicación (Application Business Rules)
├── infrastructure/            # Capa de Infraestructura (Frameworks & Drivers)
├── resources/                 # Configuraciones y recursos estáticos
└── test/                      # Pruebas unitarias e integración
```

## Flujo de Dependencias

```
Infrastructure → Application → Domain
     ↓                ↓           ↓
  (Adapters)    (Use Cases)   (Models)
```

**Regla de oro**: Las dependencias apuntan hacia adentro. El dominio NO conoce nada del exterior.

## Estructura de Carpetas

| Carpeta | Propósito | Ver README |
|---------|-----------|------------|
| `domain/` | Entidades, reglas de negocio, repositorios (interfaces) | [domain/README.md](src/main/java/com/authcore/domain/README.md) |
| `application/` | Casos de uso, DTOs, puertos de entrada/salida | [application/README.md](src/main/java/com/authcore/application/README.md) |
| `infrastructure/` | Implementaciones concretas, controllers, configuraciones | [infrastructure/README.md](src/main/java/com/authcore/infrastructure/README.md) |
| `resources/` | application.yml, scripts SQL, logback | - |
| `test/` | Pruebas por capa | [test/README.md](src/test/java/com/authcore/README.md) |

## Convenciones de Nomenclatura

- **Domain Models**: Sustantivos puros (`User`, `Role`, `Permission`)
- **Use Cases**: Verbos + Sustantivo (`AuthenticateUser`, `CreateRole`)
- **DTOs**: Sufijo `Dto` (`UserDto`, `AuthResponseDto`)
- **Adapters**: Prefijo descriptivo (`RestUserAdapter`, `JpaUserRepository`)
- **Exceptions**: Sufijo `Exception` (`InvalidTokenException`)

## Módulos Reutilizables

Esta estructura está diseñada para copiarse/pegarse en futuros proyectos:
1. Cambiar `com.authcore` por el nuevo package name
2. Ajustar entidades del dominio según necesidades
3. Mantener la separación de capas intacta

---

**Nota**: Cada subcarpeta contiene su propio README.md explicando su responsabilidad específica.

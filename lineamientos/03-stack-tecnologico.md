# Lenguajes y Frameworks

## Propósito

Este lineamiento establece los lenguajes de programación, frameworks y tecnologías preferidos para nuevos desarrollos, así como las estrategias de modernización para el stack heredado. El objetivo es estandarizar las tecnologías utilizadas, facilitar el mantenimiento y promover la adopción de mejores prácticas de la industria.

> **Ver también**: [18 - Modernización y Stack Heredado](18-modernizacion-y-stack-heredado.md) para estrategias detalladas de migración

## Stack Tecnológico

### Stack Objetivo (Nuevos Desarrollos)

#### Backend
- **C# .NET 8+ (LTS)**: Lenguaje principal para APIs y servicios
- **ASP.NET Core**: Framework web para APIs REST
- **Entity Framework Core**: ORM para acceso a datos
- **MediatR**: Patrón mediator para CQRS
- **FluentValidation**: Validación de datos
- **Mapster**: Mapeo de objetos

> **Ver también**: [04 - APIs y Contratos](04-apis-y-contratos.md) para mejores prácticas de APIs

#### Frontend
- **React 18+**: Framework principal para aplicaciones web
- **TypeScript**: Tipado estático para JavaScript
- **Angular 17+**: Alternativa para aplicaciones empresariales complejas
- **Next.js**: Framework React para SSR/SSG
- **Tailwind CSS**: Framework CSS utility-first

> **Ver también**: [15 - Arquitectura Frontend](15-arquitectura-frontend.md) para patrones y mejores prácticas

#### Base de Datos
- **PostgreSQL 15+**: Base de datos relacional principal
- **DynamoDB**: Base de datos NoSQL para casos específicos
- **Redis**: Cache y almacenamiento en memoria

> **Ver también**: [07 - Datos y Persistencia](07-datos-y-persistencia.md) para estrategias de datos

#### Cloud y Infraestructura
- **AWS**: Plataforma cloud principal
- **AWS Lambda**: Serverless para funciones específicas
- **Docker**: Containerización
- **Kubernetes**: Orquestación de contenedores (cuando sea necesario)

> **Ver también**: [17 - Arquitectura Serverless](17-arquitectura-serverless.md), [10 - DevOps y Entornos](10-devops-y-entornos.md)

### Stack Heredado (Mantenimiento)

#### Tecnologías en Mantenimiento
- **Oracle 19c**: Base de datos heredada (migración gradual a PostgreSQL)
- **C# .NET Framework**: Aplicaciones legacy (migración a .NET 8)
- **C# .NET Core**: Versiones anteriores (migración a .NET 8)
- **Scripts de automatización**: Migrar a PowerShell o .NET Console Apps

## Principios de Selección Tecnológica

### Criterios de Evaluación
- **Madurez**: Tecnología estable y probada en producción
- **Comunidad**: Soporte activo y documentación extensa
- **Performance**: Rendimiento adecuado para los casos de uso
- **Seguridad**: Actualizaciones regulares y buenas prácticas de seguridad
- **Escalabilidad**: Capacidad de crecer con el negocio
- **Mantenibilidad**: Facilidad de mantenimiento y evolución

### Decisiones Arquitectónicas
- **Preferir tecnologías open source**: Mayor transparencia y control
- **Evitar vendor lock-in**: Usar estándares abiertos cuando sea posible
- **Considerar curva de aprendizaje**: Balance entre innovación y productividad
- **Evaluar costos totales**: Licencias, mantenimiento, capacitación

## Estrategia de Modernización

### Migración de Base de Datos
```sql
-- Ejemplo: Estrategia de migración Oracle a PostgreSQL
-- 1. Identificar diferencias de sintaxis
-- Oracle
SELECT SYSDATE FROM DUAL;

-- PostgreSQL equivalente
SELECT NOW();

-- 2. Migrar tipos de datos
-- Oracle
NUMBER(10,2) -> NUMERIC(10,2)
VARCHAR2(255) -> VARCHAR(255)
CLOB -> TEXT

-- 3. Adaptar funciones específicas
-- Oracle
NVL(column, 'default') -> COALESCE(column, 'default')
```

#### Plan de Migración
1. **Inventario**: Catalogar todas las aplicaciones y dependencias
2. **Priorización**: Identificar aplicaciones críticas vs no críticas
3. **Piloto**: Migrar aplicación de menor riesgo como prueba
4. **Gradual**: Migrar aplicación por aplicación
5. **Validación**: Testing exhaustivo en cada fase

### Migración de .NET Framework a .NET 8
```csharp
// Ejemplo: Migración de .NET Framework a .NET 8
// .NET Framework (legacy)
public class UserController : ApiController
{
    public IHttpActionResult GetUser(int id)
    {
        var user = _userService.GetById(id);
        if (user == null)
            return NotFound();
        return Ok(user);
    }
}

// .NET 8 (moderno)
[ApiController]
[Route("api/[controller]")]
public class UserController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        if (user == null)
            return NotFound();
        return Ok(user);
    }
}
```

## Estándares de Desarrollo

### C# .NET 8
```csharp
// Ejemplo: Estándares de código C# .NET 8
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly ILogger<UserService> _logger;

    public UserService(
        IUserRepository userRepository,
        ILogger<UserService> logger)
    {
        _userRepository = userRepository ?? throw new ArgumentNullException(nameof(userRepository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task<Result<UserDto>> CreateUserAsync(CreateUserRequest request)
    {
        try
        {
            var user = request.Adapt<User>();
            var createdUser = await _userRepository.CreateAsync(user);
            var userDto = createdUser.Adapt<UserDto>();

            _logger.LogInformation("User created successfully: {UserId}", createdUser.Id);
            return Result<UserDto>.Success(userDto);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating user");
            return Result<UserDto>.Failure("Error creating user");
        }
    }
}

// Configuración de Mapster
public class MapsterConfig : IRegister
{
    public void Register(TypeAdapterConfig config)
    {
        config.NewConfig<CreateUserRequest, User>()
            .Map(dest => dest.CreatedAt, src => DateTime.UtcNow)
            .Map(dest => dest.IsActive, src => true);

        config.NewConfig<User, UserDto>()
            .Map(dest => dest.FullName, src => $"{src.FirstName} {src.LastName}");
    }
}
```

### React/TypeScript
```typescript
// Ejemplo: Estándares de código React/TypeScript
interface User {
  id: number;
  name: string;
  email: string;
  status: 'active' | 'inactive';
}

interface UserListProps {
  users: User[];
  onUserSelect: (user: User) => void;
  isLoading?: boolean;
}

export const UserList: React.FC<UserListProps> = ({
  users,
  onUserSelect,
  isLoading = false
}) => {
  if (isLoading) {
    return <LoadingSpinner />;
  }

  return (
    <div className="user-list">
      {users.map((user) => (
        <UserCard
          key={user.id}
          user={user}
          onClick={() => onUserSelect(user)}
        />
      ))}
    </div>
  );
};
```

## Gestión de Dependencias

### .NET Dependencies
```xml
<!-- Ejemplo: PackageReference en .NET 8 -->
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.0" />
  <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
  <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
  <PackageReference Include="Mapster" Version="8.0.0" />
  <PackageReference Include="Mapster.DependencyInjection" Version="2.0.0" />
  <PackageReference Include="FluentValidation.AspNetCore" Version="11.0.0" />
</ItemGroup>
```

### Frontend Dependencies
```json
// Ejemplo: package.json para React/TypeScript (solo frontend)
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.0.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0"
  }
}
```

## Herramientas y Entorno de Desarrollo

### IDEs y Editores
- **Visual Studio 2022**: IDE principal para .NET
- **Visual Studio Code**: Editor para frontend y scripts
- **Rider**: Alternativa para desarrollo .NET (JetBrains)

### Herramientas de Build y CI/CD
- **MSBuild**: Build system para .NET
- **npm/yarn**: Package manager para frontend (React/TypeScript)
- **Docker**: Containerización
- **GitHub Actions**: CI/CD pipeline

### Herramientas de Testing
- **xUnit**: Testing framework para .NET
- **Jest**: Testing framework para JavaScript/TypeScript (frontend)
- **Playwright**: E2E testing para aplicaciones web
- **Postman**: Testing de APIs

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] Stack tecnológico seleccionado según lineamientos
- [ ] Versiones LTS de frameworks utilizadas
- [ ] Estándares de código aplicados
- [ ] Herramientas de desarrollo configuradas
- [ ] Dependencias documentadas y versionadas
- [ ] Testing framework configurado
- [ ] CI/CD pipeline implementado

### Para Migraciones
- [ ] Plan de migración documentado
- [ ] Análisis de impacto completado
- [ ] Piloto ejecutado exitosamente
- [ ] Testing exhaustivo realizado
- [ ] Rollback plan preparado
- [ ] Equipo capacitado en nuevas tecnologías

## Excepciones y Justificaciones

### Cuándo Usar Tecnologías Diferentes
- **Integración con sistemas legacy**: Cuando sea necesario mantener compatibilidad
- **Requisitos específicos**: Cuando el stack objetivo no cumple los requisitos
- **Proyectos experimentales**: Para validar nuevas tecnologías
- **Restricciones de licencias**: Cuando los costos sean prohibitivos

### Proceso de Excepción
1. **Análisis técnico**: Evaluar alternativas y justificación
2. **Evaluación de impacto**: Considerar costos y beneficios
3. **Aprobación**: Obtener aprobación del equipo de arquitectura
4. **Documentación**: Documentar decisión y alternativas
5. **Revisión**: Revisar decisión periódicamente

## Referencias y Recursos

### Documentación Oficial
- [.NET 8 Documentation]
- [ASP.NET Core Documentation]
- [React Documentation]
- [TypeScript Documentation]
- [PostgreSQL Documentation]

### Mejores Prácticas
- [C# Coding Conventions]
- [React Best Practices]
- [TypeScript Best Practices]
- [PostgreSQL Best Practices]

### Herramientas de Migración
- [Oracle to PostgreSQL Migration Tools]
- [.NET Framework to .NET Migration Guide]
- [TypeScript Migration Guide]

### Configuración de Mapster
```csharp
// Registro en Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // Registrar Mapster
        builder.Services.AddMapster();
        builder.Services.AddMapster(new MapsterConfig());

        // ... resto de la configuración
    }
}
```
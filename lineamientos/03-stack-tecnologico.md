# Lenguajes y Frameworks

## Propósito

Este lineamiento establece los lenguajes de programación, frameworks y tecnologías preferidos para nuevos desarrollos, así como las estrategias de modernización para el stack heredado. El objetivo es estandarizar las tecnologías utilizadas, facilitar el mantenimiento y promover la adopción de mejores prácticas de la industria.

## Stack Tecnológico

### Stack Objetivo (Nuevos Desarrollos)

#### Backend
- **C# .NET 8+ (LTS)**: Lenguaje principal para APIs y servicios
- **ASP.NET Core**: Framework web para APIs REST
- **Entity Framework Core**: ORM para acceso a datos
- **MediatR**: Patrón mediator para CQRS
- **FluentValidation**: Validación de datos
- **AutoMapper**: Mapeo de objetos

#### Frontend
- **React 18+**: Framework principal para aplicaciones web
- **TypeScript**: Tipado estático para JavaScript
- **Angular 17+**: Alternativa para aplicaciones empresariales complejas
- **Next.js**: Framework React para SSR/SSG
- **Tailwind CSS**: Framework CSS utility-first

#### Base de Datos
- **PostgreSQL 15+**: Base de datos relacional principal
- **DynamoDB**: Base de datos NoSQL para casos específicos
- **Redis**: Cache y almacenamiento en memoria

#### Cloud y Infraestructura
- **AWS**: Plataforma cloud principal
- **AWS Lambda**: Serverless para funciones específicas
- **Docker**: Containerización
- **Kubernetes**: Orquestación de contenedores (cuando sea necesario)

### Stack Heredado (Mantenimiento)

#### Tecnologías en Mantenimiento
- **Oracle 19c**: Base de datos heredada (migración gradual a PostgreSQL)
- **C# .NET Framework**: Aplicaciones legacy (migración a .NET 8)
- **Python**: Scripts y herramientas de automatización
- **.NET Core**: Versiones anteriores (migración a .NET 8)

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
    private readonly IMapper _mapper;

    public UserService(
        IUserRepository userRepository,
        ILogger<UserService> logger,
        IMapper mapper)
    {
        _userRepository = userRepository ?? throw new ArgumentNullException(nameof(userRepository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
    }

    public async Task<Result<UserDto>> CreateUserAsync(CreateUserRequest request)
    {
        try
        {
            var user = _mapper.Map<User>(request);
            var createdUser = await _userRepository.CreateAsync(user);
            var userDto = _mapper.Map<UserDto>(createdUser);

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
  <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
  <PackageReference Include="FluentValidation.AspNetCore" Version="11.0.0" />
</ItemGroup>
```

### Node.js Dependencies
```json
// Ejemplo: package.json para React/TypeScript
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
- **npm/yarn**: Package manager para Node.js
- **Docker**: Containerización
- **GitHub Actions**: CI/CD pipeline

### Herramientas de Testing
- **xUnit**: Testing framework para .NET
- **Jest**: Testing framework para JavaScript/TypeScript
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
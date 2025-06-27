# Gestión de Configuración

## Propósito

Este lineamiento establece los principios y mejores prácticas para la gestión de configuración en aplicaciones y sistemas, incluyendo feature flags, configuración centralizada, gestión de secretos y variables de entorno. El objetivo es proporcionar flexibilidad, seguridad y control sobre el comportamiento de los sistemas.

## Principios Fundamentales

### Configuración Externa
- **Separación de código y configuración**: La configuración debe estar separada del código
- **Configuración por entorno**: Diferentes configuraciones para dev, test, staging, prod
- **Configuración centralizada**: Gestión unificada de configuraciones
- **Versionado de configuración**: Control de versiones para configuraciones

### Seguridad por Diseño
- **Secretos separados**: Gestión segura de credenciales y secretos
- **Cifrado en reposo**: Configuraciones sensibles cifradas
- **Control de acceso**: Acceso basado en roles y permisos
- **Auditoría**: Logs de cambios en configuraciones

### Flexibilidad y Control
- **Feature flags**: Control granular de funcionalidades
- **Configuración dinámica**: Cambios sin redeploy
- **Rollback rápido**: Capacidad de revertir cambios
- **Validación**: Validación de configuraciones antes de aplicar

## Estrategias de Configuración

### 1. Feature Flags (Feature Toggles)

#### Propósito
Permitir activar/desactivar funcionalidades sin cambios de código, facilitando despliegues graduales y experimentación.

#### Tipos de Feature Flags
```csharp
// Ejemplo: Implementación de feature flags
public class UserService
{
    private readonly IFeatureFlagService _featureFlagService;

    public UserService(IFeatureFlagService featureFlagService)
    {
        _featureFlagService = featureFlagService;
    }

    public async Task<UserResponse> CreateUserAsync(UserRequest request)
    {
        // Feature flag para nueva validación
        if (await _featureFlagService.IsEnabledAsync("new-user-validation"))
        {
            return await CreateUserWithNewValidationAsync(request);
        }
        else
        {
            return await CreateUserWithLegacyValidationAsync(request);
        }
    }

    public async Task<List<User>> GetUsersAsync()
    {
        // Feature flag para paginación
        if (await _featureFlagService.IsEnabledAsync("user-pagination"))
        {
            return await GetUsersWithPaginationAsync();
        }
        else
        {
            return await GetAllUsersAsync();
        }
    }
}
```

#### Tipos de Flags
- **Release Flags**: Controlar lanzamiento de funcionalidades
- **Experiment Flags**: A/B testing y experimentación
- **Permission Flags**: Control de acceso por usuario
- **Kill Switches**: Desactivación de emergencia
- **Operational Flags**: Configuración operacional

#### Herramientas Recomendadas
- **LaunchDarkly**: Plataforma completa de feature flags
- **Unleash**: Open source feature flag platform
- **ConfigCat**: Feature flags para múltiples plataformas
- **Split.io**: Feature flags y experimentation

### 2. Configuración Centralizada

#### Propósito
Gestionar configuraciones de múltiples servicios desde un punto central, facilitando consistencia y control.

#### Arquitectura Recomendada
```json
// Ejemplo: Configuración con Azure App Configuration
{
  "AzureAppConfiguration": {
    "ConnectionString": "your-connection-string",
    "CacheExpiration": "00:05:00",
    "RefreshInterval": "00:01:00"
  },
  "Configuration": {
    "FailFast": true,
    "Retry": {
      "InitialInterval": "00:00:01",
      "MaxInterval": "00:00:02",
      "MaxAttempts": 6
    }
  }
}
```

#### Servicios de Configuración
- **AWS Systems Manager Parameter Store**: Configuración centralizada en AWS
- **Azure App Configuration**: Servicio de configuración de Azure
- **HashiCorp Consul**: Service mesh y configuración distribuida
- **.NET Configuration**: Sistema nativo de configuración de .NET

### 3. Gestión de Secretos

#### Propósito
Gestionar de forma segura credenciales, tokens y información sensible.

#### Mejores Prácticas
```csharp
// Ejemplo: Uso de secretos con .NET
public class DatabaseConfig
{
    private readonly IConfiguration _configuration;

    public DatabaseConfig(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        var connectionString = _configuration.GetConnectionString("DefaultConnection");
        var dbPassword = _configuration["Database:Password"]; // Inyectado desde secret manager

        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(connectionString));
    }
}
```

#### Herramientas de Gestión de Secretos
- **Azure Key Vault**: Gestión nativa en Azure
- **AWS Secrets Manager**: Gestión nativa en AWS
- **HashiCorp Vault**: Gestión de secretos open source
- **Kubernetes Secrets**: Para aplicaciones en K8s
- **Docker Secrets**: Para aplicaciones containerizadas

### 4. Variables de Entorno

#### Propósito
Configurar aplicaciones según el entorno de ejecución.

#### Estructura Recomendada
```bash
# Ejemplo: Variables de entorno por capa
# Infraestructura
DATABASE_URL=postgresql://localhost:5432/mydb
REDIS_URL=redis://localhost:6379
KAFKA_BROKERS=localhost:9092

# Aplicación
APP_ENV=production
APP_DEBUG=false
APP_LOG_LEVEL=INFO

# Seguridad
JWT_SECRET=your-secret-key
API_KEY=your-api-key

# Negocio
FEATURE_NEW_UI=true
MAX_USERS_PER_PAGE=50
PAYMENT_GATEWAY=stripe
```

#### Gestión por Entorno
- **Development**: Configuración local para desarrollo
- **Testing**: Configuración para pruebas automatizadas
- **Staging**: Configuración similar a producción
- **Production**: Configuración optimizada y segura

## Patrones de Implementación

### 1. Configuración por Capas

#### Estructura Jerárquica
```json
// Ejemplo: Configuración en capas con .NET
// appsettings.json (defaults)
{
  "App": {
    "Feature": {
      "NewUi": false,
      "UserPagination": false
    },
    "Security": {
      "JwtExpiration": 3600
    },
    "Database": {
      "PoolSize": 10
    }
  }
}

// appsettings.Development.json (development overrides)
{
  "App": {
    "Feature": {
      "NewUi": true
    },
    "Database": {
      "PoolSize": 5
    }
  }
}

// appsettings.Production.json (production overrides)
{
  "App": {
    "Security": {
      "JwtExpiration": 1800
    },
    "Database": {
      "PoolSize": 20
    }
  }
}
```

### 2. Configuración Dinámica

#### Hot Reload
```csharp
// Ejemplo: Configuración dinámica con .NET
public class UserService
{
    private readonly IOptionsMonitor<UserServiceOptions> _options;

    public UserService(IOptionsMonitor<UserServiceOptions> options)
    {
        _options = options;
    }

    public async Task<List<User>> GetUsersAsync()
    {
        var currentOptions = _options.CurrentValue;

        if (currentOptions.UserPaginationEnabled)
        {
            return await GetUsersWithPaginationAsync(currentOptions.MaxUsersPerPage);
        }
        else
        {
            return await GetAllUsersAsync();
        }
    }
}

public class UserServiceOptions
{
    public bool UserPaginationEnabled { get; set; }
    public int MaxUsersPerPage { get; set; }
}
```

### 3. Validación de Configuración

#### Validación con .NET 8
```csharp
// Ejemplo: Validación de configuración
public class AppConfig
{
    [Required]
    [Range(1, 100)]
    public int MaxUsersPerPage { get; set; }

    [Required]
    [RegularExpression("^(true|false)$")]
    public string FeatureNewUi { get; set; }

    [ValidateObject]
    public SecurityConfig Security { get; set; }
}

public class SecurityConfig
{
    [Required]
    public string JwtSecret { get; set; }

    [Range(1, 24)]
    public int TokenExpirationHours { get; set; }
}

public class ConfigurationValidator
{
    private readonly ILogger<ConfigurationValidator> _logger;

    public ConfigurationValidator(ILogger<ConfigurationValidator> logger)
    {
        _logger = logger;
    }

    public void ValidateCriticalConfig(IConfiguration configuration)
    {
        // Validar configuración crítica
        var appConfig = configuration.GetSection("App").Get<AppConfig>();

        if (appConfig == null)
        {
            throw new InvalidOperationException("App configuration is missing");
        }

        var validationResults = new List<ValidationResult>();
        var validationContext = new ValidationContext(appConfig);

        if (!Validator.TryValidateObject(appConfig, validationContext, validationResults, true))
        {
            var errors = string.Join(", ", validationResults.Select(v => v.ErrorMessage));
            throw new InvalidOperationException($"Configuration validation failed: {errors}");
        }

        _logger.LogInformation("Configuration validation completed successfully");
    }
}
```

### Auditoría de Cambios
```csharp
// Ejemplo: Auditoría de cambios de configuración
public class ConfigurationAuditService
{
    private readonly ILogger<ConfigurationAuditService> _logger;
    private readonly IAuditService _auditService;

    public ConfigurationAuditService(
        ILogger<ConfigurationAuditService> logger,
        IAuditService auditService)
    {
        _logger = logger;
        _auditService = auditService;
    }

    public void HandleConfigurationChange(ConfigurationChangeEvent changeEvent)
    {
        _logger.LogInformation("Configuration changed: {Key} = {Value} by user: {User}",
            changeEvent.Key, changeEvent.Value, changeEvent.User);

        // Enviar a sistema de auditoría
        _auditService.RecordConfigurationChange(changeEvent);
    }
}

public class ConfigurationChangeEvent
{
    public string Key { get; set; }
    public string Value { get; set; }
    public string User { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}
```

## Monitoreo y Observabilidad

### Métricas de Configuración
```yaml
# Ejemplo: Métricas con Micrometer
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,configprops
  endpoint:
    health:
      show-details: when-authorized
  metrics:
    export:
      prometheus:
        enabled: true
```

### Métricas Clave
- **Feature Flag Usage**: Uso de feature flags por funcionalidad
- **Configuration Changes**: Frecuencia de cambios de configuración
- **Secret Rotation**: Rotación de secretos y credenciales
- **Configuration Errors**: Errores de configuración y validación

### Alertas Recomendadas
- **Configuration Validation Failed**: Fallo en validación de configuración
- **Secret Expiration**: Secretos próximos a expirar
- **Feature Flag Conflicts**: Conflictos entre feature flags
- **Configuration Drift**: Desviación de configuración esperada

## Seguridad y Compliance

### Control de Acceso
```json
// Ejemplo: Control de acceso con .NET Identity
{
  "Authentication": {
    "JwtBearer": {
      "Authority": "https://auth.example.com",
      "Audience": "api.example.com",
      "RequireHttpsMetadata": true
    }
  },
  "Authorization": {
    "Policies": {
      "AdminOnly": {
        "RequireRole": "Admin"
      },
      "UserAccess": {
        "RequireRole": ["User", "Admin"]
      }
    }
  }
}
```

### Auditoría de Cambios
```csharp
// Ejemplo: Auditoría de cambios de configuración
public class ConfigurationAuditService
{
    private readonly ILogger<ConfigurationAuditService> _logger;
    private readonly IAuditService _auditService;

    public ConfigurationAuditService(
        ILogger<ConfigurationAuditService> logger,
        IAuditService auditService)
    {
        _logger = logger;
        _auditService = auditService;
    }

    public void HandleConfigurationChange(ConfigurationChangeEvent changeEvent)
    {
        _logger.LogInformation("Configuration changed: {Key} = {Value} by user: {User}",
            changeEvent.Key, changeEvent.Value, changeEvent.User);

        // Enviar a sistema de auditoría
        _auditService.RecordConfigurationChange(changeEvent);
    }
}

public class ConfigurationChangeEvent
{
    public string Key { get; set; }
    public string Value { get; set; }
    public string User { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}
```

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] Estrategia de configuración definida y documentada
- [ ] Feature flags implementados para funcionalidades críticas
- [ ] Gestión de secretos configurada y probada
- [ ] Configuración por entorno establecida
- [ ] Validación de configuración implementada
- [ ] Monitoreo de configuración configurado
- [ ] Documentación de configuración actualizada

### Para Cambios Existentes
- [ ] Impacto en configuración evaluado
- [ ] Feature flags creados para cambios significativos
- [ ] Secretos actualizados si es necesario
- [ ] Validación de configuración ejecutada
- [ ] Monitoreo de configuración actualizado

## Excepciones y Justificaciones

### Cuándo Usar Configuración Hardcoded
- **Configuración de bootstrap**: Configuración necesaria para arrancar
- **Configuración inmutable**: Valores que nunca cambian
- **Configuración de desarrollo**: Para desarrollo local rápido
- **Configuración de prueba**: Para tests unitarios

### Proceso de Excepción
1. **Justificación técnica**: Explicar por qué no usar configuración externa
2. **Análisis de riesgo**: Evaluar impacto en seguridad y mantenimiento
3. **Plan de migración**: Definir cómo migrar a configuración externa
4. **Aprobación**: Obtener aprobación del equipo de arquitectura
5. **Seguimiento**: Monitorear y revisar decisión periódicamente

## Referencias y Recursos

### Herramientas de Feature Flags
- [LaunchDarkly - Feature flag platform]
- [Unleash - Open source feature flags]
- [ConfigCat - Feature flags for multiple platforms]
- [Split.io - Feature flags and experimentation]

### Herramientas de Configuración
- **AWS Systems Manager Parameter Store**: Configuración centralizada en AWS
- **Azure App Configuration**: Servicio de configuración de Azure
- **HashiCorp Consul**: Service mesh y configuración distribuida
- **.NET Configuration**: Sistema nativo de configuración de .NET

### Herramientas de Secretos
- [AWS Secrets Manager - Secret management]
- [Azure Key Vault - Secret management]
- [HashiCorp Vault - Secret management]
- [Kubernetes Secrets - Secret management]

### Recursos Adicionales
- [12 Factor App - Configuration principles]
- [Feature Toggles - Martin Fowler]
- [Configuration Management - Best Practices]
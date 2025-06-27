# Seguridad Arquitectónica

## Propósito

Este lineamiento establece los principios, patrones y mejores prácticas de seguridad para proteger aplicaciones, datos y infraestructura. Se enfoca en implementar seguridad por diseño y defensa en profundidad en todos los niveles de la arquitectura.

> **Ver también**: [01 - Principios de Arquitectura](01-principios-arquitectura.md) para principios fundamentales de seguridad, [02 - Gobernanza y Compliance](02-gobernanza-y-compliance.md) para cumplimiento normativo

## Principios Fundamentales

### Security by Design
- **Seguridad desde el inicio**: Integrar seguridad en todas las fases del desarrollo
- **Defensa en profundidad**: Múltiples capas de seguridad
- **Principio de menor privilegio**: Acceso mínimo necesario
- **Zero Trust**: No confiar en nada, verificar todo
- **Fail Secure**: Fallar de forma segura

### Autenticación y Autorización
- **Autenticación centralizada**: Sistema único de autenticación
- **Autorización basada en roles**: RBAC (Role-Based Access Control)
- **Multi-factor authentication**: MFA para acceso crítico
- **Single Sign-On**: SSO para aplicaciones internas
- **Session management**: Gestión segura de sesiones

> **Ver también**: [04 - APIs y Contratos](04-apis-y-contratos.md) para implementación de autenticación en APIs

### Protección de Datos
- **Cifrado en tránsito**: TLS 1.3 para comunicaciones
- **Cifrado en reposo**: AES-256 para datos sensibles
- **Data classification**: Clasificación de datos por sensibilidad
- **Data masking**: Enmascaramiento de datos sensibles
- **Data retention**: Políticas de retención de datos

> **Ver también**: [07 - Datos y Persistencia](07-datos-y-persistencia.md) para estrategias de protección de datos

## Implementación con .NET 8

### Autenticación JWT
```csharp
// Ejemplo: Configuración de JWT con .NET 8
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = Configuration["Jwt:Issuer"],
                    ValidAudience = Configuration["Jwt:Audience"],
                    IssuerSigningKey = new SymmetricSecurityKey(
                        Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
                };
            });

        services.AddAuthorization(options =>
        {
            options.AddPolicy("AdminOnly", policy =>
                policy.RequireRole("Admin"));

            options.AddPolicy("UserAccess", policy =>
                policy.RequireRole("User", "Admin"));
        });
    }
}

// Controller con autorización
[ApiController]
[Route("api/v1/[controller]")]
[Authorize]
public class UsersController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "UserAccess")]
    public async Task<ActionResult<List<UserDto>>> GetUsers()
    {
        // Implementación
    }

    [HttpDelete("{id}")]
    [Authorize(Policy = "AdminOnly")]
    public async Task<ActionResult> DeleteUser(int id)
    {
        // Implementación
    }
}
```

### Identity y User Management
```csharp
// Ejemplo: Identity con .NET 8
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
}

public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Configuraciones de seguridad adicionales
        builder.Entity<ApplicationUser>(entity =>
        {
            entity.Property(e => e.Email).IsRequired();
            entity.HasIndex(e => e.Email).IsUnique();
        });
    }
}

// User Service con validaciones de seguridad
public class UserService : IUserService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ILogger<UserService> _logger;

    public async Task<Result<ApplicationUser>> CreateUserAsync(CreateUserRequest request)
    {
        // Validar contraseña fuerte
        var passwordValidator = new PasswordValidator<ApplicationUser>();
        var result = await passwordValidator.ValidateAsync(_userManager, null, request.Password);

        if (!result.Succeeded)
        {
            return Result<ApplicationUser>.Failure("Password does not meet security requirements");
        }

        var user = new ApplicationUser
        {
            UserName = request.Email,
            Email = request.Email,
            FirstName = request.FirstName,
            LastName = request.LastName,
            CreatedAt = DateTime.UtcNow,
            IsActive = true
        };

        var createResult = await _userManager.CreateAsync(user, request.Password);

        if (createResult.Succeeded)
        {
            await _userManager.AddToRoleAsync(user, "User");
            _logger.LogInformation("User created successfully: {UserId}", user.Id);
            return Result<ApplicationUser>.Success(user);
        }

        return Result<ApplicationUser>.Failure(string.Join(", ", createResult.Errors));
    }
}
```

### Cifrado de Datos
```csharp
// Ejemplo: Servicio de cifrado
public interface IEncryptionService
{
    string Encrypt(string plainText);
    string Decrypt(string cipherText);
}

public class EncryptionService : IEncryptionService
{
    private readonly byte[] _key;
    private readonly byte[] _iv;

    public EncryptionService(IConfiguration configuration)
    {
        _key = Convert.FromBase64String(configuration["Encryption:Key"]);
        _iv = Convert.FromBase64String(configuration["Encryption:IV"]);
    }

    public string Encrypt(string plainText)
    {
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.IV = _iv;

        using var encryptor = aes.CreateEncryptor();
        using var msEncrypt = new MemoryStream();
        using var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write);
        using var swEncrypt = new StreamWriter(csEncrypt);

        swEncrypt.Write(plainText);
        swEncrypt.Close();

        return Convert.ToBase64String(msEncrypt.ToArray());
    }

    public string Decrypt(string cipherText)
    {
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.IV = _iv;

        using var decryptor = aes.CreateDecryptor();
        using var msDecrypt = new MemoryStream(Convert.FromBase64String(cipherText));
        using var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read);
        using var srDecrypt = new StreamReader(csDecrypt);

        return srDecrypt.ReadToEnd();
    }
}
```

## Seguridad de APIs

### Rate Limiting
```csharp
// Ejemplo: Rate limiting con ASP.NET Core
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;
    private readonly ILogger<RateLimitingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = GetClientId(context);
        var endpoint = context.Request.Path;

        var key = $"rate_limit:{clientId}:{endpoint}";
        var currentCount = await _cache.GetStringAsync(key);

        if (currentCount != null && int.Parse(currentCount) >= 100)
        {
            context.Response.StatusCode = 429; // Too Many Requests
            await context.Response.WriteAsync("Rate limit exceeded");
            return;
        }

        await _cache.SetStringAsync(key,
            (int.Parse(currentCount ?? "0") + 1).ToString(),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1)
            });

        await _next(context);
    }

    private string GetClientId(HttpContext context)
    {
        return context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString() ?? "anonymous";
    }
}
```

### Input Validation
```csharp
// Ejemplo: Validación de entrada con FluentValidation
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(255)
            .MustAsync(async (email, cancellation) =>
            {
                // Validar que el email no contenga caracteres peligrosos
                return !email.Contains("<script>") && !email.Contains("javascript:");
            })
            .WithMessage("Invalid email format");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]")
            .WithMessage("Password must contain uppercase, lowercase, number and special character");

        RuleFor(x => x.FirstName)
            .NotEmpty()
            .MaximumLength(100)
            .Matches(@"^[a-zA-Z\s]+$")
            .WithMessage("First name can only contain letters and spaces");
    }
}
```

## Seguridad de Base de Datos

### Connection Security
```csharp
// Ejemplo: Configuración segura de base de datos
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Usar connection string seguro
        var connectionString = Configuration.GetConnectionString("DefaultConnection");

        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseNpgsql(connectionString, npgsqlOptions =>
            {
                npgsqlOptions.EnableRetryOnFailure(
                    maxRetryCount: 3,
                    maxRetryDelay: TimeSpan.FromSeconds(30),
                    errorCodesToAdd: null);
            });
        });
    }
}

// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=mydb;Username=myuser;Password=***;SSL Mode=Require;Trust Server Certificate=true"
  }
}
```

### SQL Injection Prevention
```csharp
// Ejemplo: Prevención de SQL Injection con Entity Framework
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public async Task<User> GetUserByEmailAsync(string email)
    {
        // Usar parámetros para prevenir SQL injection
        return await _context.Users
            .FirstOrDefaultAsync(u => u.Email == email);
    }

    public async Task<List<User>> SearchUsersAsync(string searchTerm)
    {
        // Usar EF Core para prevenir SQL injection
        return await _context.Users
            .Where(u => u.FirstName.Contains(searchTerm) ||
                       u.LastName.Contains(searchTerm) ||
                       u.Email.Contains(searchTerm))
            .ToListAsync();
    }
}
```

## Seguridad de Infraestructura

### AWS Security
```csharp
// Ejemplo: Configuración segura de AWS
public class AWSSecurityConfiguration
{
    public void ConfigureAWSServices(IServiceCollection services, IConfiguration configuration)
    {
        // Configurar AWS con credenciales seguras
        services.AddAWSService<IAmazonS3>(new AWSOptions
        {
            Region = Amazon.RegionEndpoint.USEast1,
            Credentials = new EnvironmentVariablesAWSCredentials()
        });

        // Configurar AWS Secrets Manager
        services.AddSingleton<ISecretManager, AWSSecretManager>();
    }
}

public class AWSSecretManager : ISecretManager
{
    private readonly IAmazonSecretsManager _secretsManager;

    public async Task<string> GetSecretAsync(string secretName)
    {
        var request = new GetSecretValueRequest
        {
            SecretId = secretName
        };

        var response = await _secretsManager.GetSecretValueAsync(request);
        return response.SecretString;
    }
}
```

### HTTPS Configuration
```csharp
// Ejemplo: Configuración HTTPS
public class Startup
{
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (!env.IsDevelopment())
        {
            app.UseHsts();
        }

        app.UseHttpsRedirection();
        app.UseStaticFiles();

        app.UseRouting();

        app.UseAuthentication();
        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

## Auditoría y Logging

### Security Logging
```csharp
// Ejemplo: Logging de seguridad
public class SecurityLogger
{
    private readonly ILogger<SecurityLogger> _logger;

    public void LogAuthenticationAttempt(string username, bool success, string ipAddress)
    {
        _logger.LogInformation(
            "Authentication attempt - User: {Username}, Success: {Success}, IP: {IPAddress}",
            username, success, ipAddress);
    }

    public void LogAuthorizationFailure(string username, string resource, string action)
    {
        _logger.LogWarning(
            "Authorization failure - User: {Username}, Resource: {Resource}, Action: {Action}",
            username, resource, action);
    }

    public void LogDataAccess(string username, string dataType, string action)
    {
        _logger.LogInformation(
            "Data access - User: {Username}, DataType: {DataType}, Action: {Action}",
            username, dataType, action);
    }
}
```

### Audit Trail
```csharp
// Ejemplo: Audit trail con Entity Framework
public class AuditEntry
{
    public int Id { get; set; }
    public string UserId { get; set; }
    public string Action { get; set; }
    public string TableName { get; set; }
    public string KeyValues { get; set; }
    public string OldValues { get; set; }
    public string NewValues { get; set; }
    public DateTime Timestamp { get; set; }
}

public class AuditDbContext : DbContext
{
    public DbSet<AuditEntry> AuditEntries { get; set; }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var auditEntries = OnBeforeSaveChanges();
        var result = await base.SaveChangesAsync(cancellationToken);
        await OnAfterSaveChanges(auditEntries);
        return result;
    }

    private List<AuditEntry> OnBeforeSaveChanges()
    {
        ChangeTracker.DetectChanges();
        var auditEntries = new List<AuditEntry>();

        foreach (var entry in ChangeTracker.Entries())
        {
            if (entry.Entity is AuditEntry || entry.State == EntityState.Detached || entry.State == EntityState.Unchanged)
                continue;

            var auditEntry = new AuditEntry
            {
                TableName = entry.Entity.GetType().Name,
                Action = entry.State.ToString(),
                KeyValues = JsonSerializer.Serialize(entry.Properties.Where(p => p.Metadata.IsPrimaryKey()).ToDictionary(x => x.Metadata.Name, x => x.CurrentValue)),
                OldValues = entry.State == EntityState.Modified ? JsonSerializer.Serialize(entry.Properties.Where(p => p.IsModified).ToDictionary(x => x.Metadata.Name, x => x.OriginalValue)) : null,
                NewValues = entry.State == EntityState.Added || entry.State == EntityState.Modified ? JsonSerializer.Serialize(entry.Properties.ToDictionary(x => x.Metadata.Name, x => x.CurrentValue)) : null,
                Timestamp = DateTime.UtcNow
            };

            auditEntries.Add(auditEntry);
        }

        return auditEntries;
    }
}
```

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] Autenticación JWT configurada
- [ ] Autorización basada en roles implementada
- [ ] Validación de entrada configurada
- [ ] Cifrado de datos sensibles implementado
- [ ] HTTPS configurado
- [ ] Rate limiting implementado
- [ ] Logging de seguridad configurado
- [ ] Audit trail implementado
- [ ] Secret management configurado

### Para Cambios Existentes
- [ ] Análisis de seguridad completado
- [ ] Vulnerabilidades identificadas y mitigadas
- [ ] Configuración de seguridad actualizada
- [ ] Testing de seguridad ejecutado
- [ ] Documentación de seguridad actualizada

## Excepciones y Justificaciones

### Cuándo Relajar Seguridad
- **Entornos de desarrollo**: Configuraciones menos estrictas para desarrollo
- **APIs públicas**: Cuando se requiere acceso público
- **Integraciones legacy**: Cuando hay limitaciones técnicas
- **Testing**: Para facilitar pruebas automatizadas

### Proceso de Excepción
1. **Análisis de riesgo**: Evaluar impacto de relajar seguridad
2. **Mitigaciones**: Identificar medidas compensatorias
3. **Aprobación**: Obtener aprobación del equipo de seguridad
4. **Documentación**: Documentar excepción y justificación
5. **Revisión**: Revisar excepción periódicamente

## Referencias y Recursos

### Estándares y Frameworks
- [OWASP Top 10 - Web Application Security]
- [NIST Cybersecurity Framework]
- [ISO 27001 - Information Security Management]
- [GDPR - General Data Protection Regulation]

### Herramientas de Seguridad
- [SonarQube - Code Quality and Security]
- [OWASP ZAP - Web Application Security Scanner]
- [Snyk - Vulnerability Scanner]
- [Checkmarx - Static Application Security Testing]

### Recursos Adicionales
- [Microsoft Security Documentation]
- [AWS Security Best Practices]
- [OWASP Cheat Sheet Series]
- [Security Headers - Web Security]
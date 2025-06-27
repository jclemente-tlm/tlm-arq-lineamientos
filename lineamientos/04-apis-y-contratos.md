# APIs

## Visión General

Las APIs son la columna vertebral de la arquitectura moderna, facilitando la comunicación entre servicios, aplicaciones frontend y sistemas externos. Este lineamiento define las pautas para diseñar, desarrollar e implementar APIs RESTful robustas, escalables y seguras, alineadas con nuestro stack tecnológico (.NET 8, AWS, microservicios).

## Principios Fundamentales

### 1. Diseño First
- **API-First Design**: Diseñar APIs antes de implementar
- **OpenAPI/Swagger**: Documentación como código
- **Contract-First**: Definir contratos antes del desarrollo
- **Versioning Strategy**: Estrategia clara de versionado

### 2. RESTful Best Practices
- **Resource-Oriented**: Diseñar alrededor de recursos
- **Stateless**: Sin estado entre requests
- **Cacheable**: Aprovechar caching HTTP
- **Uniform Interface**: Interfaz consistente

### 3. Performance y Escalabilidad
- **Async/Await**: Operaciones asíncronas
- **Caching Strategy**: Caching en múltiples niveles
- **Rate Limiting**: Control de velocidad de requests
- **Pagination**: Paginación eficiente

## Stack Tecnológico de APIs

### Herramientas Principales
```yaml
Backend_Framework:
  - ASP.NET Core 8+
  - Minimal APIs (para endpoints simples)
  - Controllers (para APIs complejas)

Documentation:
  - Swagger/OpenAPI 3.0
  - NSwag para generación de clientes

Validation:
  - FluentValidation
  - Data Annotations

Serialization:
  - System.Text.Json
  - JSON.NET (cuando sea necesario)

Authentication:
  - JWT Bearer Tokens
  - OAuth 2.0 / OpenID Connect
  - AWS Cognito

Monitoring:
  - Application Insights
  - AWS CloudWatch
  - Prometheus + Grafana
```

## Implementación en .NET 8

### Minimal APIs
```csharp
// Program.cs - Minimal API Example
var builder = WebApplication.CreateBuilder(args);

// Configurar servicios
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });
builder.Services.AddAuthorization();
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();

var app = builder.Build();

// Configurar pipeline
app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthentication();
app.UseAuthorization();

// Definir endpoints
app.MapGet("/api/v1/users", async (
    IUserService userService,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? search = null) =>
{
    var result = await userService.GetUsersAsync(page, pageSize, search);
    return Results.Ok(new ApiResponse<PaginatedResult<UserDto>>
    {
        Data = result,
        Meta = new MetaData
        {
            Page = page,
            PageSize = pageSize,
            TotalCount = result.TotalCount,
            TotalPages = result.TotalPages
        }
    });
})
.WithName("GetUsers")
.WithOpenApi()
.RequireAuthorization("read:users");

app.MapPost("/api/v1/users", async (
    CreateUserRequest request,
    IUserService userService,
    IValidator<CreateUserRequest> validator) =>
{
    var validationResult = await validator.ValidateAsync(request);
    if (!validationResult.IsValid)
    {
        return Results.BadRequest(new ApiResponse<object>
        {
            Success = false,
            Errors = validationResult.Errors.Select(e => new ErrorDetail
            {
                Code = "VALIDATION_ERROR",
                Message = e.ErrorMessage,
                Field = e.PropertyName
            }).ToList()
        });
    }

    var user = await userService.CreateUserAsync(request);
    return Results.CreatedAtRoute("GetUser", new { id = user.Id }, new ApiResponse<UserDto>
    {
        Data = user
    });
})
.WithName("CreateUser")
.WithOpenApi()
.RequireAuthorization("write:users");

app.MapGet("/api/v1/users/{id:guid}", async (
    Guid id,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);
    if (user == null)
    {
        return Results.NotFound(new ApiResponse<object>
        {
            Success = false,
            Errors = new List<ErrorDetail>
            {
                new() { Code = "USER_NOT_FOUND", Message = "User not found" }
            }
        });
    }

    return Results.Ok(new ApiResponse<UserDto> { Data = user });
})
.WithName("GetUser")
.WithOpenApi()
.RequireAuthorization("read:users");

app.Run();
```

### Controller-Based APIs
```csharp
// Controllers/UsersController.cs
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
[ProducesResponseType(typeof(ApiResponse<PaginatedResult<UserDto>>), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ApiResponse<object>), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ApiResponse<object>), StatusCodes.Status401Unauthorized)]
[ProducesResponseType(typeof(ApiResponse<object>), StatusCodes.Status403Forbidden)]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;
    private readonly IValidator<CreateUserRequest> _validator;

    public UsersController(
        IUserService userService,
        ILogger<UsersController> logger,
        IValidator<CreateUserRequest> validator)
    {
        _userService = userService;
        _logger = logger;
        _validator = validator;
    }

    /// <summary>
    /// Obtiene una lista paginada de usuarios
    /// </summary>
    /// <param name="query">Parámetros de consulta</param>
    /// <returns>Lista paginada de usuarios</returns>
    [HttpGet]
    [Authorize(Policy = "read:users")]
    public async Task<ActionResult<ApiResponse<PaginatedResult<UserDto>>>> GetUsers(
        [FromQuery] GetUsersQuery query)
    {
        try
        {
            var result = await _userService.GetUsersAsync(query);

            return Ok(new ApiResponse<PaginatedResult<UserDto>>
            {
                Data = result,
                Meta = new MetaData
                {
                    Page = query.Page,
                    PageSize = query.PageSize,
                    TotalCount = result.TotalCount,
                    TotalPages = result.TotalPages
                }
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving users");
            return StatusCode(500, new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "INTERNAL_ERROR", Message = "An internal error occurred" }
                }
            });
        }
    }

    /// <summary>
    /// Obtiene un usuario por su ID
    /// </summary>
    /// <param name="id">ID del usuario</param>
    /// <returns>Datos del usuario</returns>
    [HttpGet("{id:guid}")]
    [Authorize(Policy = "read:users")]
    public async Task<ActionResult<ApiResponse<UserDto>>> GetUser(Guid id)
    {
        var user = await _userService.GetUserByIdAsync(id);
        if (user == null)
        {
            return NotFound(new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "USER_NOT_FOUND", Message = "User not found" }
                }
            });
        }

        return Ok(new ApiResponse<UserDto> { Data = user });
    }

    /// <summary>
    /// Crea un nuevo usuario
    /// </summary>
    /// <param name="request">Datos del usuario a crear</param>
    /// <returns>Usuario creado</returns>
    [HttpPost]
    [Authorize(Policy = "write:users")]
    public async Task<ActionResult<ApiResponse<UserDto>>> CreateUser(
        [FromBody] CreateUserRequest request)
    {
        var validationResult = await _validator.ValidateAsync(request);
        if (!validationResult.IsValid)
        {
            return BadRequest(new ApiResponse<object>
            {
                Success = false,
                Errors = validationResult.Errors.Select(e => new ErrorDetail
                {
                    Code = "VALIDATION_ERROR",
                    Message = e.ErrorMessage,
                    Field = e.PropertyName
                }).ToList()
            });
        }

        try
        {
            var user = await _userService.CreateUserAsync(request);

            return CreatedAtAction(nameof(GetUser), new { id = user.Id },
                new ApiResponse<UserDto> { Data = user });
        }
        catch (UserAlreadyExistsException ex)
        {
            return Conflict(new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "USER_ALREADY_EXISTS", Message = ex.Message }
                }
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating user");
            return StatusCode(500, new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "INTERNAL_ERROR", Message = "An internal error occurred" }
                }
            });
        }
    }

    /// <summary>
    /// Actualiza un usuario existente
    /// </summary>
    /// <param name="id">ID del usuario</param>
    /// <param name="request">Datos a actualizar</param>
    /// <returns>Usuario actualizado</returns>
    [HttpPut("{id:guid}")]
    [Authorize(Policy = "write:users")]
    public async Task<ActionResult<ApiResponse<UserDto>>> UpdateUser(
        Guid id,
        [FromBody] UpdateUserRequest request)
    {
        var validationResult = await _validator.ValidateAsync(request);
        if (!validationResult.IsValid)
        {
            return BadRequest(new ApiResponse<object>
            {
                Success = false,
                Errors = validationResult.Errors.Select(e => new ErrorDetail
                {
                    Code = "VALIDATION_ERROR",
                    Message = e.ErrorMessage,
                    Field = e.PropertyName
                }).ToList()
            });
        }

        try
        {
            var user = await _userService.UpdateUserAsync(id, request);
            if (user == null)
            {
                return NotFound(new ApiResponse<object>
                {
                    Success = false,
                    Errors = new List<ErrorDetail>
                    {
                        new() { Code = "USER_NOT_FOUND", Message = "User not found" }
                    }
                });
            }

            return Ok(new ApiResponse<UserDto> { Data = user });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating user {UserId}", id);
            return StatusCode(500, new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "INTERNAL_ERROR", Message = "An internal error occurred" }
                }
            });
        }
    }

    /// <summary>
    /// Elimina un usuario
    /// </summary>
    /// <param name="id">ID del usuario</param>
    /// <returns>Sin contenido</returns>
    [HttpDelete("{id:guid}")]
    [Authorize(Policy = "delete:users")]
    public async Task<ActionResult> DeleteUser(Guid id)
    {
        try
        {
            var deleted = await _userService.DeleteUserAsync(id);
            if (!deleted)
            {
                return NotFound(new ApiResponse<object>
                {
                    Success = false,
                    Errors = new List<ErrorDetail>
                    {
                        new() { Code = "USER_NOT_FOUND", Message = "User not found" }
                    }
                });
            }

            return NoContent();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deleting user {UserId}", id);
            return StatusCode(500, new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "INTERNAL_ERROR", Message = "An internal error occurred" }
                }
            });
        }
    }
}
```

## Modelos y DTOs

### Request/Response Models
```csharp
// Models/Requests/CreateUserRequest.cs
public class CreateUserRequest
{
    [Required]
    [StringLength(100)]
    public string FirstName { get; set; } = string.Empty;

    [Required]
    [StringLength(100)]
    public string LastName { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [StringLength(20, MinimumLength = 8)]
    public string Password { get; set; } = string.Empty;

    [Phone]
    public string? PhoneNumber { get; set; }

    public DateTime? DateOfBirth { get; set; }
}

// Models/Requests/UpdateUserRequest.cs
public class UpdateUserRequest
{
    [StringLength(100)]
    public string? FirstName { get; set; }

    [StringLength(100)]
    public string? LastName { get; set; }

    [EmailAddress]
    public string? Email { get; set; }

    [Phone]
    public string? PhoneNumber { get; set; }

    public DateTime? DateOfBirth { get; set; }
}

// Models/Requests/GetUsersQuery.cs
public class GetUsersQuery
{
    [Range(1, int.MaxValue)]
    public int Page { get; set; } = 1;

    [Range(1, 100)]
    public int PageSize { get; set; } = 20;

    public string? Search { get; set; }

    public string? SortBy { get; set; }

    public string? SortDirection { get; set; } = "asc";

    public string? Status { get; set; }

    public DateTime? CreatedFrom { get; set; }

    public DateTime? CreatedTo { get; set; }
}

// Models/Responses/UserDto.cs
public class UserDto
{
    public Guid Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public DateTime? DateOfBirth { get; set; }
    public string Status { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

// Models/Responses/ApiResponse.cs
public class ApiResponse<T>
{
    public bool Success { get; set; } = true;
    public T? Data { get; set; }
    public MetaData? Meta { get; set; }
    public List<ErrorDetail> Errors { get; set; } = new();
}

// Models/Responses/MetaData.cs
public class MetaData
{
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages { get; set; }
    public string? NextPageUrl { get; set; }
    public string? PreviousPageUrl { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string RequestId { get; set; } = string.Empty;
}

// Models/Responses/ErrorDetail.cs
public class ErrorDetail
{
    public string Code { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public string? Field { get; set; }
    public string? HelpUrl { get; set; }
}

// Models/Responses/PaginatedResult.cs
public class PaginatedResult<T>
{
    public List<T> Items { get; set; } = new();
    public int TotalCount { get; set; }
    public int TotalPages { get; set; }
    public int CurrentPage { get; set; }
    public int PageSize { get; set; }
    public bool HasNextPage { get; set; }
    public bool HasPreviousPage { get; set; }
}
```

## Validación

### FluentValidation
```csharp
// Validators/CreateUserRequestValidator.cs
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.FirstName)
            .NotEmpty()
            .MaximumLength(100)
            .Matches(@"^[a-zA-Z\s]+$")
            .WithMessage("First name must contain only letters and spaces");

        RuleFor(x => x.LastName)
            .NotEmpty()
            .MaximumLength(100)
            .Matches(@"^[a-zA-Z\s]+$")
            .WithMessage("Last name must contain only letters and spaces");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(255);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .MaximumLength(20)
            .Matches(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]")
            .WithMessage("Password must contain at least one uppercase letter, one lowercase letter, one number, and one special character");

        RuleFor(x => x.PhoneNumber)
            .Matches(@"^\+?[1-9]\d{1,14}$")
            .When(x => !string.IsNullOrEmpty(x.PhoneNumber))
            .WithMessage("Phone number must be in international format");

        RuleFor(x => x.DateOfBirth)
            .LessThan(DateTime.Today.AddYears(-13))
            .When(x => x.DateOfBirth.HasValue)
            .WithMessage("User must be at least 13 years old");
    }
}

// Validators/UpdateUserRequestValidator.cs
public class UpdateUserRequestValidator : AbstractValidator<UpdateUserRequest>
{
    public UpdateUserRequestValidator()
    {
        RuleFor(x => x.FirstName)
            .MaximumLength(100)
            .Matches(@"^[a-zA-Z\s]+$")
            .When(x => !string.IsNullOrEmpty(x.FirstName))
            .WithMessage("First name must contain only letters and spaces");

        RuleFor(x => x.LastName)
            .MaximumLength(100)
            .Matches(@"^[a-zA-Z\s]+$")
            .When(x => !string.IsNullOrEmpty(x.LastName))
            .WithMessage("Last name must contain only letters and spaces");

        RuleFor(x => x.Email)
            .EmailAddress()
            .MaximumLength(255)
            .When(x => !string.IsNullOrEmpty(x.Email));

        RuleFor(x => x.PhoneNumber)
            .Matches(@"^\+?[1-9]\d{1,14}$")
            .When(x => !string.IsNullOrEmpty(x.PhoneNumber))
            .WithMessage("Phone number must be in international format");

        RuleFor(x => x.DateOfBirth)
            .LessThan(DateTime.Today.AddYears(-13))
            .When(x => x.DateOfBirth.HasValue)
            .WithMessage("User must be at least 13 years old");
    }
}
```

## Autenticación y Autorización

### JWT Configuration
```csharp
// Program.cs - JWT Configuration
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.Authority = builder.Configuration["Auth:Authority"];
    options.Audience = builder.Configuration["Auth:Audience"];
    options.RequireHttpsMetadata = true;
    options.SaveToken = true;

    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ClockSkew = TimeSpan.Zero
    };

    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            context.Response.StatusCode = 401;
            context.Response.ContentType = "application/json";

            var result = JsonSerializer.Serialize(new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "AUTHENTICATION_FAILED", Message = "Invalid token" }
                }
            });

            return context.Response.WriteAsync(result);
        },

        OnChallenge = context =>
        {
            context.HandleResponse();
            context.Response.StatusCode = 401;
            context.Response.ContentType = "application/json";

            var result = JsonSerializer.Serialize(new ApiResponse<object>
            {
                Success = false,
                Errors = new List<ErrorDetail>
                {
                    new() { Code = "AUTHENTICATION_REQUIRED", Message = "Authentication required" }
                }
            });

            return context.Response.WriteAsync(result);
        }
    };
});

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("read:users", policy =>
        policy.RequireClaim("permission", "read:users"));

    options.AddPolicy("write:users", policy =>
        policy.RequireClaim("permission", "write:users"));

    options.AddPolicy("delete:users", policy =>
        policy.RequireClaim("permission", "delete:users"));

    options.AddPolicy("admin", policy =>
        policy.RequireClaim("role", "admin"));
});
```

## Rate Limiting

### Rate Limiting Implementation
```csharp
// Program.cs - Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));

    options.AddPolicy("authenticated", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name ?? "anonymous",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 1000,
                Window = TimeSpan.FromMinutes(1)
            }));

    options.AddPolicy("api", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Request.Headers["X-API-Key"].ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 10000,
                Window = TimeSpan.FromMinutes(1)
            }));

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        context.HttpContext.Response.ContentType = "application/json";

        var result = JsonSerializer.Serialize(new ApiResponse<object>
        {
            Success = false,
            Errors = new List<ErrorDetail>
            {
                new() { Code = "RATE_LIMIT_EXCEEDED", Message = "Too many requests" }
            }
        });

        await context.HttpContext.Response.WriteAsync(result, token);
    };
});

// Aplicar rate limiting en endpoints
app.MapGet("/api/v1/users", async (context) => { /* ... */ })
    .RequireRateLimiting("authenticated");
```

## Caching

### Response Caching
```csharp
// Program.cs - Caching Configuration
builder.Services.AddResponseCaching(options =>
{
    options.MaximumBodySize = 64 * 1024 * 1024; // 64MB
    options.SizeLimit = 100 * 1024 * 1024; // 100MB
});

builder.Services.AddMemoryCache();

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "API_";
});

// Aplicar caching en endpoints
[HttpGet("{id:guid}")]
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
[Authorize(Policy = "read:users")]
public async Task<ActionResult<ApiResponse<UserDto>>> GetUser(Guid id)
{
    // Implementation
}

// Cache personalizado
[HttpGet]
[ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "page", "pageSize", "search" })]
[Authorize(Policy = "read:users")]
public async Task<ActionResult<ApiResponse<PaginatedResult<UserDto>>>> GetUsers(
    [FromQuery] GetUsersQuery query)
{
    // Implementation
}
```

## Logging y Monitoreo

### Structured Logging
```csharp
// Middleware/RequestLoggingMiddleware.cs
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var requestId = context.TraceIdentifier;
        var startTime = DateTime.UtcNow;

        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["RequestId"] = requestId,
            ["UserId"] = context.User?.Identity?.Name ?? "anonymous",
            ["IPAddress"] = context.Connection.RemoteIpAddress?.ToString() ?? "unknown"
        });

        _logger.LogInformation("Request started: {Method} {Path}",
            context.Request.Method, context.Request.Path);

        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Request failed: {Method} {Path}",
                context.Request.Method, context.Request.Path);
            throw;
        }
        finally
        {
            var duration = DateTime.UtcNow - startTime;
            _logger.LogInformation("Request completed: {Method} {Path} - {StatusCode} in {Duration}ms",
                context.Request.Method, context.Request.Path, context.Response.StatusCode, duration.TotalMilliseconds);
        }
    }
}

// Program.cs - Logging Configuration
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "UserAPI")
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.AmazonCloudWatch(
        logGroup: "/aws/ecs/user-api",
        region: "us-east-1",
        logStreamNameProvider: new DefaultLogStreamProvider(),
        textFormatter: new JsonFormatter()
    ));
```

## Health Checks

### Health Checks Implementation
```csharp
// Program.cs - Health Checks
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck<RedisHealthCheck>("redis")
    .AddCheck<ExternalServiceHealthCheck>("external-service");

// Health Checks
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDbConnection _dbConnection;

    public DatabaseHealthCheck(IDbConnection dbConnection)
    {
        _dbConnection = dbConnection;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            using var connection = _dbConnection;
            await connection.OpenAsync(cancellationToken);

            using var command = connection.CreateCommand();
            command.CommandText = "SELECT 1";
            command.CommandTimeout = 5;

            var result = await command.ExecuteScalarAsync(cancellationToken);

            return result != null
                ? HealthCheckResult.Healthy("Database is responding")
                : HealthCheckResult.Unhealthy("Database query failed");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database connection failed", ex);
        }
    }
}

// Endpoints de health check
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    }
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false
});
```

## Versionado de APIs

### API Versioning Strategy
```csharp
// Program.cs - API Versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new MediaTypeApiVersionReader("version"));
});

builder.Services.AddVersionedApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Controllers con versionado
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public async Task<ActionResult<ApiResponse<PaginatedResult<UserDto>>>> GetUsersV1()
    {
        // Implementation for v1
    }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public async Task<ActionResult<ApiResponse<PaginatedResult<UserDtoV2>>>> GetUsersV2()
    {
        // Implementation for v2
    }
}
```

## Documentación OpenAPI

### Swagger Configuration
```csharp
// Program.cs - Swagger Configuration
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "User Management API",
        Version = "v1",
        Description = "API for managing users in the system",
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "api-support@company.com"
        },
        License = new OpenApiLicense
        {
            Name = "MIT",
            Url = new Uri("https://opensource.org/licenses/MIT")
        }
    });

    options.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "User Management API",
        Version = "v2",
        Description = "Enhanced API for managing users in the system"
    });

    // Configurar autenticación JWT
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme. Example: \"Authorization: Bearer {token}\"",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });

    // Incluir comentarios XML
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);

    // Configurar operaciones
    options.OperationFilter<SwaggerDefaultValues>();
});

// Configurar Swagger UI
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "User API v1");
    options.SwaggerEndpoint("/swagger/v2/swagger.json", "User API v2");
    options.RoutePrefix = "api-docs";
    options.DocumentTitle = "User Management API Documentation";
    options.DefaultModelsExpandDepth(-1);
});
```

## Mejores Prácticas

### 1. Diseño de URLs
- **Recursos en plural**: `/api/v1/users` no `/api/v1/user`
- **Nombres descriptivos**: `/api/v1/orders` no `/api/v1/o`
- **Subrecursos**: `/api/v1/users/{id}/orders`
- **Filtros en query params**: `/api/v1/users?status=active&role=admin`

### 2. Códigos de Estado HTTP
- **200 OK**: Operación exitosa
- **201 Created**: Recurso creado
- **204 No Content**: Operación exitosa sin respuesta
- **400 Bad Request**: Error de validación
- **401 Unauthorized**: Falta autenticación
- **403 Forbidden**: Acceso denegado
- **404 Not Found**: Recurso no encontrado
- **409 Conflict**: Conflicto de estado
- **422 Unprocessable Entity**: Error de validación semántica
- **429 Too Many Requests**: Rate limit excedido
- **500 Internal Server Error**: Error del servidor

### 3. Manejo de Errores
- **Consistencia**: Formato de error uniforme
- **Códigos específicos**: Códigos de error únicos
- **Mensajes claros**: Mensajes descriptivos
- **Documentación**: Enlaces a documentación

### 4. Performance
- **Paginación**: Siempre paginar listas grandes
- **Caching**: Implementar caching apropiado
- **Compresión**: Usar gzip/deflate
- **Async/Await**: Operaciones asíncronas

### 5. Seguridad
- **HTTPS**: Siempre usar HTTPS en producción
- **Validación**: Validar todas las entradas
- **Autenticación**: Implementar autenticación robusta
- **Autorización**: Control de acceso granular
- **Rate Limiting**: Proteger contra abuso

## Herramientas Recomendadas

### Desarrollo
- **Swagger/OpenAPI**: Documentación de APIs
- **Postman**: Testing de APIs
- **Insomnia**: Cliente REST alternativo
- **NSwag**: Generación de clientes

### Testing
- **xUnit**: Unit testing
- **Moq**: Mocking framework
- **FluentAssertions**: Assertions más legibles
- **TestServer**: Integration testing

### Monitoreo
- **Application Insights**: APM y métricas
- **AWS CloudWatch**: Métricas y logs
- **Prometheus**: Métricas personalizadas
- **Grafana**: Visualización

### Seguridad
- **OWASP ZAP**: Security testing
- **Checkmarx**: Static analysis
- **SonarQube**: Code quality
- **AWS WAF**: Web application firewall

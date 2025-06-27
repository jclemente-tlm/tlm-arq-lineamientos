# Microservicios

## Propósito

Este lineamiento define los principios, patrones y mejores prácticas para el diseño, desarrollo e implementación de microservicios. Se enfoca en crear servicios autónomos, escalables y mantenibles que aporten valor real al negocio.

## Principios Fundamentales

### Autonomía de Servicios
- **Independencia de despliegue**: Cada servicio puede desplegarse independientemente
- **Base de datos por servicio**: Cada servicio gestiona su propio almacenamiento
- **Ciclo de vida independiente**: Desarrollo, testing y operación independientes
- **Tecnología específica**: Cada servicio puede usar la tecnología más apropiada
- **Equipos autónomos**: Equipos independientes por servicio

### Comunicación Asíncrona
- **Event-driven architecture**: Comunicación basada en eventos
- **Message queues**: Colas de mensajes para comunicación asíncrona
- **Event sourcing**: Trazabilidad completa de cambios
- **CQRS**: Separación de comandos y consultas
- **Saga pattern**: Transacciones distribuidas

### Desacoplamiento
- **Interfaces bien definidas**: Contratos claros entre servicios
- **Versionado de APIs**: Gestión de versiones de interfaces
- **Service discovery**: Descubrimiento dinámico de servicios
- **API Gateway**: Punto de entrada unificado
- **Circuit breaker**: Prevención de cascada de fallos

## Arquitectura de Microservicios

### Patrón de Arquitectura
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway   │    │   Load Balancer │    │   Service Mesh  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  User Service   │    │ Payment Service │    │ Order Service   │
│   (.NET 8)      │    │   (.NET 8)      │    │   (.NET 8)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   PostgreSQL    │    │   PostgreSQL    │    │   PostgreSQL    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Implementación con .NET 8
```csharp
// Ejemplo: User Service con .NET 8
[ApiController]
[Route("api/v1/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly IEventPublisher _eventPublisher;

    public UsersController(IUserService userService, IEventPublisher eventPublisher)
    {
        _userService = userService;
        _eventPublisher = eventPublisher;
    }

    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser([FromBody] CreateUserRequest request)
    {
        var user = await _userService.CreateUserAsync(request);

        // Publicar evento para otros servicios
        await _eventPublisher.PublishAsync(new UserCreatedEvent
        {
            UserId = user.Id,
            Email = user.Email,
            Timestamp = DateTime.UtcNow
        });

        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}

// Event Publisher
public interface IEventPublisher
{
    Task PublishAsync<T>(T @event) where T : class;
}

public class RabbitMQEventPublisher : IEventPublisher
{
    private readonly IConnection _connection;
    private readonly IModel _channel;

    public async Task PublishAsync<T>(T @event) where T : class
    {
        var message = JsonSerializer.Serialize(@event);
        var body = Encoding.UTF8.GetBytes(message);

        _channel.BasicPublish(
            exchange: "user_events",
            routingKey: typeof(T).Name.ToLower(),
            basicProperties: null,
            body: body);
    }
}
```

## Patrones de Comunicación

### Síncrona (HTTP/REST)
```csharp
// Ejemplo: Comunicación síncrona entre servicios
public class PaymentServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<PaymentServiceClient> _logger;

    public PaymentServiceClient(HttpClient httpClient, ILogger<PaymentServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        try
        {
            var response = await _httpClient.PostAsJsonAsync("/api/v1/payments", request);
            response.EnsureSuccessStatusCode();

            return await response.Content.ReadFromJsonAsync<PaymentResult>();
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error communicating with payment service");
            throw new ServiceUnavailableException("Payment service is unavailable");
        }
    }
}
```

### Asíncrona (Event-Driven)
```csharp
// Ejemplo: Event-driven communication
public class OrderCreatedEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<OrderCreatedEventHandler> _logger;

    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        try
        {
            // Actualizar inventario
            await _inventoryService.ReserveItemsAsync(@event.OrderItems);

            // Enviar notificación
            await _notificationService.SendOrderConfirmationAsync(@event.UserId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing order created event");
            // Implementar dead letter queue o retry logic
        }
    }
}
```

## Patrones de Resiliencia

### Circuit Breaker
```csharp
// Ejemplo: Circuit Breaker con Polly
public class ResilientPaymentService
{
    private readonly IAsyncPolicy<PaymentResult> _circuitBreakerPolicy;

    public ResilientPaymentService()
    {
        _circuitBreakerPolicy = Policy<PaymentResult>
            .Handle<HttpRequestException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(30));
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return await _circuitBreakerPolicy.ExecuteAsync(async () =>
        {
            return await _paymentServiceClient.ProcessPaymentAsync(request);
        });
    }
}
```

### Retry Pattern
```csharp
// Ejemplo: Retry con backoff exponencial
public class RetryService
{
    private readonly IAsyncPolicy<PaymentResult> _retryPolicy;

    public RetryService()
    {
        _retryPolicy = Policy<PaymentResult>
            .Handle<HttpRequestException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (exception, timeSpan, retryCount, context) =>
                {
                    _logger.LogWarning(
                        "Retry {RetryCount} after {Delay}ms due to {Exception}",
                        retryCount, timeSpan.TotalMilliseconds, exception.Message);
                });
    }
}
```

## Service Discovery y API Gateway

### Service Discovery
```csharp
// Ejemplo: Service discovery con Consul
public class ConsulServiceDiscovery : IServiceDiscovery
{
    private readonly IConsulClient _consulClient;

    public async Task<string> GetServiceUrlAsync(string serviceName)
    {
        var queryResult = await _consulClient.Health.Service(serviceName);
        var healthyService = queryResult.Response.FirstOrDefault();

        if (healthyService == null)
            throw new ServiceNotFoundException($"Service {serviceName} not found");

        return $"http://{healthyService.Service.Address}:{healthyService.Service.Port}";
    }
}
```

### API Gateway
```csharp
// Ejemplo: API Gateway con Ocelot
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddOcelot()
            .AddConsul()
            .AddPolly();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseOcelot().Wait();
    }
}

// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/v1/users/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "user-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/users/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ]
}
```

## Base de Datos por Servicio

### Estrategia de Datos
```csharp
// Ejemplo: Database per service pattern
public class UserDbContext : DbContext
{
    public UserDbContext(DbContextOptions<UserDbContext> options) : base(options)
    {
    }

    public DbSet<User> Users { get; set; }
    public DbSet<UserProfile> UserProfiles { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configuración específica del servicio de usuarios
        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(255);
            entity.HasIndex(e => e.Email).IsUnique();
        });
    }
}

// Configuración en Program.cs
builder.Services.AddDbContext<UserDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("UserDb")));
```

### Event Sourcing
```csharp
// Ejemplo: Event sourcing para trazabilidad
public class UserEventStore
{
    private readonly DbContext _context;

    public async Task SaveEventAsync(string aggregateId, string eventType, object eventData)
    {
        var eventRecord = new EventRecord
        {
            AggregateId = aggregateId,
            EventType = eventType,
            EventData = JsonSerializer.Serialize(eventData),
            Timestamp = DateTime.UtcNow,
            Version = await GetNextVersionAsync(aggregateId)
        };

        _context.Events.Add(eventRecord);
        await _context.SaveChangesAsync();
    }

    public async Task<List<EventRecord>> GetEventsAsync(string aggregateId)
    {
        return await _context.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();
    }
}
```

## Monitoreo y Observabilidad

### Distributed Tracing
```csharp
// Ejemplo: Distributed tracing con OpenTelemetry
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddOpenTelemetry()
            .WithTracing(builder =>
            {
                builder.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddEntityFrameworkCoreInstrumentation()
                       .AddJaegerExporter();
            });
    }
}
```

### Health Checks
```csharp
// Ejemplo: Health checks específicos del servicio
public class UserServiceHealthCheck : IHealthCheck
{
    private readonly UserDbContext _context;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("Database is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database is not accessible", ex);
        }
    }
}
```

## Contenedores y Orquestación

### Docker para Microservicios
```dockerfile
# Ejemplo de Dockerfile para microservicio .NET 8
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["UserService/UserService.csproj", "UserService/"]
RUN dotnet restore "UserService/UserService.csproj"
COPY . .
WORKDIR "/src/UserService"
RUN dotnet build "UserService.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "UserService.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "UserService.dll"]
```

```yaml
# Ejemplo de docker-compose.yml para microservicios
version: '3.8'

services:
  user-service:
    build:
      context: .
      dockerfile: UserService/Dockerfile
    ports:
      - "5001:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__UserDb=Server=user-db;Database=userdb;User=sa;Password=Your_password123;
    depends_on:
      - user-db
    networks:
      - microservices-network

  payment-service:
    build:
      context: .
      dockerfile: PaymentService/Dockerfile
    ports:
      - "5002:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__PaymentDb=Server=payment-db;Database=paymentdb;User=sa;Password=Your_password123;
    depends_on:
      - payment-db
    networks:
      - microservices-network

  api-gateway:
    build:
      context: .
      dockerfile: ApiGateway/Dockerfile
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      - user-service
      - payment-service
    networks:
      - microservices-network

  user-db:
    image: postgres:15
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: sa
      POSTGRES_PASSWORD: Your_password123
    volumes:
      - user_data:/var/lib/postgresql/data
    networks:
      - microservices-network

  payment-db:
    image: postgres:15
    environment:
      POSTGRES_DB: paymentdb
      POSTGRES_USER: sa
      POSTGRES_PASSWORD: Your_password123
    volumes:
      - payment_data:/var/lib/postgresql/data
    networks:
      - microservices-network

  consul:
    image: consul:latest
    ports:
      - "8500:8500"
    command: agent -server -bootstrap-expect=1 -ui -client=0.0.0.0
    networks:
      - microservices-network

volumes:
  user_data:
  payment_data:

networks:
  microservices-network:
    driver: bridge
```

### Kubernetes para Orquestación
```yaml
# Ejemplo de deployment.yaml para microservicio
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:latest
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__UserDb
          valueFrom:
            secretKeyRef:
              name: user-db-secret
              key: connection-string
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### Multi-stage Builds para Optimización
```dockerfile
# Ejemplo de multi-stage build optimizado
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copiar archivos de proyecto
COPY ["UserService/UserService.csproj", "UserService/"]
COPY ["UserService.Tests/UserService.Tests.csproj", "UserService.Tests/"]
RUN dotnet restore "UserService/UserService.csproj"

# Copiar código fuente
COPY . .
WORKDIR "/src/UserService"

# Ejecutar tests
RUN dotnet test "../UserService.Tests/UserService.Tests.csproj" --no-restore

# Build y publish
RUN dotnet build "UserService.csproj" -c Release -o /app/build
RUN dotnet publish "UserService.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Instalar herramientas de diagnóstico
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY --from=build /app/publish .

# Usuario no-root para seguridad
RUN adduser --disabled-password --gecos '' appuser && chown -R appuser /app
USER appuser

EXPOSE 80
ENTRYPOINT ["dotnet", "UserService.dll"]
```

### Configuración de Seguridad
```yaml
# Ejemplo de SecurityContext en Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: user-service
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: tmp
        emptyDir: {}
      - name: varlog
        emptyDir: {}
```

## Checklist de Cumplimiento

### Para Nuevos Microservicios
- [ ] Justificación clara del valor de negocio
- [ ] Autonomía del servicio definida
- [ ] Contratos de API documentados
- [ ] Estrategia de comunicación definida
- [ ] Patrones de resiliencia implementados
- [ ] Base de datos independiente configurada
- [ ] Monitoreo y observabilidad configurados
- [ ] CI/CD pipeline implementado
- [ ] Testing automatizado configurado

### Para Migraciones
- [ ] Análisis de impacto completado
- [ ] Estrategia de migración definida
- [ ] Plan de rollback preparado
- [ ] Testing exhaustivo realizado
- [ ] Documentación actualizada
- [ ] Equipo capacitado

## Excepciones y Justificaciones

### Cuándo No Usar Microservicios
- **Aplicaciones pequeñas**: Cuando la complejidad no justifica la overhead
- **Equipos pequeños**: Cuando no hay suficientes desarrolladores
- **Requisitos simples**: Cuando un monolito es más apropiado
- **Restricciones de recursos**: Cuando los costos son prohibitivos

### Proceso de Decisión
1. **Análisis de beneficios**: Evaluar valor real de microservicios
2. **Evaluación de costos**: Considerar complejidad y overhead
3. **Aprobación**: Obtener aprobación del equipo de arquitectura
4. **Documentación**: Documentar decisión y justificación
5. **Revisión**: Revisar decisión periódicamente

## Referencias y Recursos

### Patrones y Mejores Prácticas
- [Microservices Patterns - Chris Richardson]
- [Building Microservices - Sam Newman]
- [Domain-Driven Design - Eric Evans]
- [Event Sourcing - Martin Fowler]

### Herramientas y Frameworks
- [ASP.NET Core - Microservices]
- [Ocelot - API Gateway]
- [Polly - Resilience and transient-fault-handling]
- [OpenTelemetry - Observability]

### Recursos Adicionales
- [Microservices.io - Patterns and Best Practices]
- [AWS Microservices Architecture]
- [Azure Microservices Architecture]
- [Google Cloud Microservices]
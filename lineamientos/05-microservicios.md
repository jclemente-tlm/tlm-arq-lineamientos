# Microservicios y Arquitecturas Serverless

## Propósito

Este lineamiento define los principios, patrones y mejores prácticas para el diseño, desarrollo e implementación de microservicios y arquitecturas serverless. Se enfoca en crear servicios autónomos, escalables y mantenibles que aporten valor real al negocio, ya sea mediante contenedores tradicionales o funciones serverless.

> **Ver también**: [04 - APIs y Contratos](04-apis-y-contratos.md) para comunicación entre servicios, [06 - Arquitectura Orientada a Eventos](06-arquitectura-orientada-eventos.md) para comunicación asíncrona

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

> **Ver también**: [06 - Arquitectura Orientada a Eventos](06-arquitectura-orientada-eventos.md) para patrones de eventos

### Desacoplamiento
- **Interfaces bien definidas**: Contratos claros entre servicios
- **Versionado de APIs**: Gestión de versiones de interfaces
- **Service discovery**: Descubrimiento dinámico de servicios
- **API Gateway**: Punto de entrada unificado
- **Circuit breaker**: Prevención de cascada de fallos

> **Ver también**: [14 - Patrones de Resiliencia](14-patrones-resiliencia.md) para circuit breakers y patrones de resiliencia

## Arquitecturas de Microservicios

### Patrón de Arquitectura Tradicional
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

### Patrón de Arquitectura Serverless
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway   │    │   Event Bridge  │    │   SQS/SNS       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  User Lambda    │    │ Payment Lambda  │    │ Order Lambda    │
│   (.NET 8)      │    │   (.NET 8)      │    │   (.NET 8)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   DynamoDB      │    │   DynamoDB      │    │   DynamoDB      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Implementación con .NET 8

#### Microservicio Tradicional
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

#### Función Serverless
```csharp
// Ejemplo: Lambda Function con .NET 8
public class UserApiFunction
{
    private readonly ILogger<UserApiFunction> _logger;
    private readonly IUserService _userService;
    private readonly IEventPublisher _eventPublisher;

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
    {
        try
        {
            _logger.LogInformation("Processing {Method} request to {Path}",
                request.HttpMethod, request.Path);

            return request.HttpMethod switch
            {
                "GET" => await HandleGetRequest(request),
                "POST" => await HandlePostRequest(request),
                "PUT" => await HandlePutRequest(request),
                "DELETE" => await HandleDeleteRequest(request),
                _ => new APIGatewayProxyResponse
                {
                    StatusCode = 405,
                    Body = "Method not allowed"
                }
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "API request failed");
            return new APIGatewayProxyResponse
            {
                StatusCode = 500,
                Body = "Internal server error"
            };
        }
    }

    private async Task<APIGatewayProxyResponse> HandlePostRequest(APIGatewayProxyRequest request)
    {
        var createUserRequest = JsonSerializer.Deserialize<CreateUserRequest>(request.Body);
        if (createUserRequest == null)
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 400,
                Body = "Invalid request body"
            };
        }

        var user = await _userService.CreateUserAsync(createUserRequest);

        // Publicar evento
        await _eventPublisher.PublishAsync(new UserCreatedEvent
        {
            UserId = user.Id,
            Email = user.Email,
            Timestamp = DateTime.UtcNow
        });

        return new APIGatewayProxyResponse
        {
            StatusCode = 201,
            Headers = new Dictionary<string, string>
            {
                ["Content-Type"] = "application/json",
                ["Location"] = $"/users/{user.Id}"
            },
            Body = JsonSerializer.Serialize(user)
        };
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

## Arquitecturas Serverless

### Ventajas y Desventajas

#### Ventajas
- **Escalado automático**: Escala de 0 a miles de instancias automáticamente
- **Pago por uso**: Solo pagas por el tiempo de ejecución real
- **Sin gestión de infraestructura**: No hay servidores que mantener
- **Despliegue simplificado**: Despliegue directo desde código
- **Alta disponibilidad**: Redundancia automática en múltiples AZs

#### Desventajas
- **Cold starts**: Latencia inicial en la primera invocación
- **Límites de ejecución**: Tiempo máximo de ejecución (15 minutos en AWS Lambda)
- **Vendor lock-in**: Dependencia del proveedor cloud
- **Debugging complejo**: Más difícil de debuggear que aplicaciones tradicionales
- **Monitoreo limitado**: Menos visibilidad que contenedores

### Patrones Serverless

#### 1. Event-Driven Processing
```csharp
// Ejemplo: Lambda Function para procesamiento de eventos
public class OrderProcessingFunction
{
    private readonly ILogger<OrderProcessingFunction> _logger;
    private readonly IOrderService _orderService;
    private readonly IEventBus _eventBus;

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
    {
        try
        {
            _logger.LogInformation("Processing order event: {RequestId}", context.AwsRequestId);

            // Deserializar evento
            var orderEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(request.Body);
            if (orderEvent == null)
            {
                return new APIGatewayProxyResponse
                {
                    StatusCode = 400,
                    Body = "Invalid event data"
                };
            }

            // Procesar orden
            var order = await _orderService.ProcessOrderAsync(orderEvent.OrderId);

            // Publicar evento de procesamiento completado
            await _eventBus.PublishAsync(new OrderProcessedEvent
            {
                OrderId = order.Id,
                ProcessedAt = DateTime.UtcNow,
                Status = order.Status
            });

            return new APIGatewayProxyResponse
            {
                StatusCode = 200,
                Body = JsonSerializer.Serialize(new { success = true, orderId = order.Id })
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process order event");

            return new APIGatewayProxyResponse
            {
                StatusCode = 500,
                Body = "Internal server error"
            };
        }
    }
}
```

#### 2. S3 Event Processing
```csharp
// Ejemplo: Lambda para procesar archivos S3
public class S3ProcessingFunction
{
    private readonly ILogger<S3ProcessingFunction> _logger;
    private readonly IFileProcessor _fileProcessor;

    public async Task FunctionHandler(S3Event s3Event, ILambdaContext context)
    {
        foreach (var record in s3Event.Records)
        {
            try
            {
                var bucketName = record.S3.Bucket.Name;
                var objectKey = record.S3.Object.Key;

                _logger.LogInformation("Processing file: {Bucket}/{Key}", bucketName, objectKey);

                // Procesar archivo
                await _fileProcessor.ProcessFileAsync(bucketName, objectKey);

                _logger.LogInformation("Successfully processed file: {Key}", objectKey);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process file: {Key}", record.S3.Object.Key);
                throw; // Re-lanzar para que SQS/SNS maneje el retry
            }
        }
    }
}
```

#### 3. DynamoDB Streams Processing
```csharp
// Ejemplo: Lambda para procesar streams de DynamoDB
public class DynamoDBStreamFunction
{
    private readonly ILogger<DynamoDBStreamFunction> _logger;
    private readonly IUserService _userService;

    public async Task FunctionHandler(DynamoDBEvent dynamoDbEvent, ILambdaContext context)
    {
        foreach (var record in dynamoDbEvent.Records)
        {
            try
            {
                if (record.EventName == "INSERT" || record.EventName == "MODIFY")
                {
                    var newImage = record.Dynamodb.NewImage;
                    var userId = newImage["Id"].S;

                    _logger.LogInformation("Processing user change: {UserId}, Event: {EventName}",
                        userId, record.EventName);

                    // Procesar cambio de usuario
                    await _userService.HandleUserChangeAsync(userId, record.EventName);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process DynamoDB record");
                throw;
            }
        }
    }
}
```

### Configuración y Despliegue

#### Serverless Framework
```yaml
# serverless.yml
service: user-service

provider:
  name: aws
  runtime: dotnet8
  region: us-east-1
  environment:
    ASPNETCORE_ENVIRONMENT: Production
    DYNAMODB_TABLE: ${self:service}-users-${opt:stage, 'dev'}

functions:
  userApi:
    handler: UserService::UserService.Functions::FunctionHandler
    events:
      - http:
          path: users/{proxy+}
          method: ANY
          cors: true
    environment:
      DYNAMODB_TABLE: ${self:provider.environment.DYNAMODB_TABLE}

  userEvents:
    handler: UserService::UserService.Functions::ProcessUserEvents
    events:
      - stream:
          type: dynamodb
          arn: !GetAtt UsersTable.StreamArn
          batchSize: 10
          startingPosition: LATEST

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.environment.DYNAMODB_TABLE}
        AttributeDefinitions:
          - AttributeName: Id
            AttributeType: S
        KeySchema:
          - AttributeName: Id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
        StreamSpecification:
          StreamEnabled: true
          StreamViewType: NEW_AND_OLD_IMAGES
```

#### AWS SAM
```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: dotnet8
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        ASPNETCORE_ENVIRONMENT: Production

Resources:
  UserApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./src/UserService/bin/Release/net8.0/
      Handler: UserService::UserService.Functions::FunctionHandler
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /users/{proxy+}
            Method: ANY

  UserEventsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./src/UserService/bin/Release/net8.0/
      Handler: UserService::UserService.Functions::ProcessUserEvents
      Events:
        DynamoDBEvent:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt UsersTable.StreamArn
            BatchSize: 10
            StartingPosition: LATEST

  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: users-table
      AttributeDefinitions:
        - AttributeName: Id
          AttributeType: S
      KeySchema:
        - AttributeName: Id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
      StreamSpecification:
        StreamEnabled: true
        StreamViewType: NEW_AND_OLD_IMAGES
```

### Optimización de Performance

#### Cold Start Mitigation
```csharp
// Ejemplo: Optimización de cold starts
public class OptimizedFunction
{
    // Variables estáticas para reutilizar entre invocaciones
    private static readonly HttpClient _httpClient = new HttpClient();
    private static readonly ILogger _logger = LoggerFactory.Create(builder =>
        builder.AddConsole()).CreateLogger<OptimizedFunction>();

    // Constructor estático para inicialización costosa
    static OptimizedFunction()
    {
        // Configurar HttpClient una sola vez
        _httpClient.Timeout = TimeSpan.FromSeconds(30);
        _httpClient.DefaultRequestHeaders.Add("User-Agent", "Lambda-Function");
    }

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
    {
        // Usar recursos estáticos para evitar recreación
        var response = await _httpClient.GetAsync("https://api.example.com/data");

        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Body = await response.Content.ReadAsStringAsync()
        };
    }
}
```

#### Provisioned Concurrency
```yaml
# Configuración de provisioned concurrency
functions:
  userApi:
    handler: UserService::UserService.Functions::FunctionHandler
    provisionedConcurrency: 10  # Mantener 10 instancias calientes
    reservedConcurrency: 100    # Límite de concurrencia
```

### Monitoreo y Observabilidad

#### CloudWatch Metrics
```csharp
// Ejemplo: Métricas personalizadas
public class MetricsFunction
{
    private readonly ILogger<MetricsFunction> _logger;

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
    {
        var startTime = DateTime.UtcNow;

        try
        {
            // Procesar request
            var result = await ProcessRequestAsync(request);

            // Registrar métricas de éxito
            await PutMetricAsync("Requests", 1, "Success");
            await PutMetricAsync("Duration", (DateTime.UtcNow - startTime).TotalMilliseconds, "Success");

            return new APIGatewayProxyResponse
            {
                StatusCode = 200,
                Body = JsonSerializer.Serialize(result)
            };
        }
        catch (Exception ex)
        {
            // Registrar métricas de error
            await PutMetricAsync("Requests", 1, "Error");
            await PutMetricAsync("Duration", (DateTime.UtcNow - startTime).TotalMilliseconds, "Error");

            throw;
        }
    }

    private async Task PutMetricAsync(string metricName, double value, string dimension)
    {
        var cloudWatch = new AmazonCloudWatchClient();
        var metricData = new PutMetricDataRequest
        {
            Namespace = "UserService",
            MetricData = new List<MetricDatum>
            {
                new MetricDatum
                {
                    MetricName = metricName,
                    Value = value,
                    Unit = StandardUnit.Count,
                    Dimensions = new List<Dimension>
                    {
                        new Dimension { Name = "Environment", Value = "Production" },
                        new Dimension { Name = "Status", Value = dimension }
                    }
                }
            }
        };

        await cloudWatch.PutMetricDataAsync(metricData);
    }
}
```

### Testing de Funciones Serverless

#### Unit Testing
```csharp
// Ejemplo: Test unitario para Lambda function
[TestClass]
public class UserApiFunctionTests
{
    private UserApiFunction _function;
    private Mock<IUserService> _userServiceMock;

    [TestInitialize]
    public void Setup()
    {
        _userServiceMock = new Mock<IUserService>();
        _function = new UserApiFunction(_userServiceMock.Object);
    }

    [TestMethod]
    public async Task FunctionHandler_ValidRequest_ReturnsSuccess()
    {
        // Arrange
        var request = new APIGatewayProxyRequest
        {
            HttpMethod = "POST",
            Body = JsonSerializer.Serialize(new CreateUserRequest { Name = "Test User" })
        };

        var context = new TestLambdaContext();

        _userServiceMock.Setup(x => x.CreateUserAsync(It.IsAny<CreateUserRequest>()))
            .ReturnsAsync(new User { Id = "123", Name = "Test User" });

        // Act
        var response = await _function.FunctionHandler(request, context);

        // Assert
        Assert.AreEqual(201, response.StatusCode);
        Assert.IsNotNull(response.Body);
    }
}
```

#### Integration Testing
```csharp
// Ejemplo: Test de integración con AWS SAM
[TestClass]
public class UserApiIntegrationTests
{
    private static Process _samProcess;

    [ClassInitialize]
    public static void Setup(TestContext context)
    {
        // Iniciar SAM local
        _samProcess = Process.Start(new ProcessStartInfo
        {
            FileName = "sam",
            Arguments = "local start-api --port 3000",
            UseShellExecute = false,
            RedirectStandardOutput = true
        });

        // Esperar a que esté listo
        Thread.Sleep(5000);
    }

    [TestMethod]
    public async Task UserApi_CreateUser_ReturnsSuccess()
    {
        // Arrange
        var client = new HttpClient();
        var request = new CreateUserRequest { Name = "Integration Test User" };

        // Act
        var response = await client.PostAsJsonAsync("http://localhost:3000/users", request);

        // Assert
        Assert.IsTrue(response.IsSuccessStatusCode);
        var user = await response.Content.ReadFromJsonAsync<User>();
        Assert.IsNotNull(user);
        Assert.AreEqual("Integration Test User", user.Name);
    }

    [ClassCleanup]
    public static void Cleanup()
    {
        _samProcess?.Kill();
    }
}
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

### Para Arquitecturas Serverless
- [ ] Análisis de cold starts realizado
- [ ] Configuración de provisioned concurrency evaluada
- [ ] Límites de tiempo de ejecución considerados
- [ ] Estrategia de monitoreo implementada
- [ ] Testing local configurado
- [ ] Costos estimados y monitoreados

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

### Cuándo No Usar Serverless
- **Cargas de trabajo constantes**: Cuando hay tráfico constante
- **Aplicaciones con estado**: Cuando se requiere estado persistente
- **Procesamiento de larga duración**: Cuando excede límites de tiempo
- **Requisitos de control total**: Cuando se necesita control completo de la infraestructura

### Proceso de Decisión
1. **Análisis de beneficios**: Evaluar valor real de microservicios/serverless
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
- [Serverless Architectures - Mike Roberts]

### Herramientas y Frameworks
- [ASP.NET Core - Microservices]
- [Ocelot - API Gateway]
- [Polly - Resilience and transient-fault-handling]
- [OpenTelemetry - Observability]
- [AWS Lambda - Serverless Computing]
- [Azure Functions - Serverless Computing]

### Recursos Adicionales
- [Microservices.io - Patterns and Best Practices]
- [AWS Microservices Architecture]
- [Azure Microservices Architecture]
- [Google Cloud Microservices]
- [Serverless Framework Documentation]
- [AWS SAM Documentation]
# Patrones de Resiliencia

## Propósito

Este lineamiento define los patrones y estrategias de resiliencia para construir sistemas robustos que puedan manejar fallos, latencia y carga de manera efectiva. El objetivo es garantizar la disponibilidad y confiabilidad de los servicios incluso en condiciones adversas.

## Principios Fundamentales

### Fail Fast
- **Detección temprana**: Identificar fallos rápidamente
- **Fallback graceful**: Proporcionar alternativas cuando sea posible
- **Degradación controlada**: Reducir funcionalidad en lugar de fallar completamente
- **Recuperación automática**: Restaurar servicio automáticamente cuando sea posible

### Isolation
- **Bulkhead pattern**: Aislar fallos entre componentes
- **Circuit breaker**: Prevenir cascada de fallos
- **Timeout management**: Evitar bloqueos indefinidos
- **Resource limits**: Limitar uso de recursos críticos

### Redundancy
- **Multiple instances**: Redundancia de servicios
- **Data replication**: Copias de datos críticos
- **Geographic distribution**: Distribución geográfica
- **Diverse providers**: Múltiples proveedores de servicios

## Patrones de Resiliencia

### 1. Circuit Breaker Pattern

#### Propósito
Prevenir que un servicio en fallo cause cascada de fallos en otros servicios.

#### Implementación
```csharp
// Ejemplo: Circuit Breaker con Polly
public class UserService
{
    private readonly IUserRepository _userRepository;
    private readonly IAsyncPolicy<User> _circuitBreakerPolicy;

    public UserService(IUserRepository userRepository)
    {
        _userRepository = userRepository;
        _circuitBreakerPolicy = Policy<User>
            .Handle<Exception>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (exception, duration) =>
                    Console.WriteLine($"Circuit breaker opened for {duration}"),
                onReset: () =>
                    Console.WriteLine("Circuit breaker reset"),
                onHalfOpen: () =>
                    Console.WriteLine("Circuit breaker half-open")
            );
    }

    public async Task<User> GetUserByIdAsync(string id)
    {
        return await _circuitBreakerPolicy.ExecuteAsync(async () =>
        {
            var user = await _userRepository.GetByIdAsync(id);
            return user ?? throw new UserNotFoundException(id);
        });
    }

    private async Task<User> GetUserFallbackAsync(string id)
    {
        // Fallback: retornar usuario cacheado o default
        return new User
        {
            Id = id,
            Name = "Usuario no disponible",
            Email = "n/a",
            Status = "UNAVAILABLE"
        };
    }
}
```

#### Estados del Circuit Breaker
- **CLOSED**: Funcionamiento normal
- **OPEN**: Bloquea requests, fallback inmediato
- **HALF_OPEN**: Permite algunos requests para probar recuperación

#### Configuración
```json
// Ejemplo: Configuración de Circuit Breaker en appsettings.json
{
  "Polly": {
    "CircuitBreaker": {
      "UserService": {
        "ExceptionsAllowedBeforeBreaking": 5,
        "DurationOfBreak": "00:00:30",
        "HandledEventsAllowedBeforeBreaking": 10
      }
    }
  }
}
```

### 2. Retry Pattern

#### Propósito
Reintentar operaciones que pueden fallar temporalmente.

#### Implementación
```csharp
// Ejemplo: Retry con backoff exponencial
public class PaymentService
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly IAsyncPolicy<PaymentResult> _retryPolicy;

    public PaymentService(IPaymentGateway paymentGateway)
    {
        _paymentGateway = paymentGateway;
        _retryPolicy = Policy<PaymentResult>
            .Handle<Exception>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (exception, timeSpan, retryCount, context) =>
                    Console.WriteLine($"Retry {retryCount} after {timeSpan.TotalSeconds}s")
            );
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        return await _retryPolicy.ExecuteAsync(async () =>
        {
            return await _paymentGateway.ProcessAsync(request);
        });
    }

    private async Task<PaymentResult> ProcessPaymentFallbackAsync(PaymentRequest request)
    {
        // Fallback: procesar offline o usar método alternativo
        return await QueuePaymentForLaterAsync(request);
    }
}
```

#### Estrategias de Retry
- **Fixed Delay**: Intervalo fijo entre reintentos
- **Exponential Backoff**: Intervalo que aumenta exponencialmente
- **Jitter**: Variación aleatoria en el intervalo
- **Maximum Attempts**: Límite de reintentos

#### Configuración
```json
// Ejemplo: Configuración de Retry
{
  "Polly": {
    "Retry": {
      "PaymentService": {
        "MaxAttempts": 3,
        "WaitDuration": "00:00:01",
        "EnableExponentialBackoff": true,
        "ExponentialBackoffMultiplier": 2
      }
    }
  }
}
```

### 3. Timeout Pattern

#### Propósito
Evitar que las operaciones se bloqueen indefinidamente.

#### Implementación
```csharp
// Ejemplo: Timeout con Task
public class ExternalServiceClient
{
    private readonly IExternalService _externalService;
    private readonly IAsyncPolicy<Response> _timeoutPolicy;

    public ExternalServiceClient(IExternalService externalService)
    {
        _externalService = externalService;
        _timeoutPolicy = Policy<Response>
            .Handle<Exception>()
            .TimeoutAsync(TimeSpan.FromSeconds(5));
    }

    public async Task<Response> CallExternalServiceAsync(Request request)
    {
        return await _timeoutPolicy.ExecuteAsync(async () =>
        {
            return await _externalService.ProcessAsync(request);
        });
    }

    public async Task<Response> CallWithTimeoutAsync(Request request)
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        return await _externalService.ProcessAsync(request, cts.Token);
    }
}
```

#### Configuración de Timeouts
- **Connection Timeout**: Tiempo para establecer conexión
- **Read Timeout**: Tiempo para leer respuesta
- **Write Timeout**: Tiempo para escribir datos
- **Global Timeout**: Timeout general de la operación

### 4. Bulkhead Pattern

#### Propósito
Aislar recursos y fallos entre diferentes partes del sistema.

#### Implementación
```csharp
// Ejemplo: Bulkhead con thread pools separados
public class BulkheadConfig
{
    public static IServiceCollection AddBulkheadServices(IServiceCollection services)
    {
        services.AddSingleton<IUserServiceExecutor>(provider =>
            new UserServiceExecutor(10));

        services.AddSingleton<IPaymentServiceExecutor>(provider =>
            new PaymentServiceExecutor(5));

        return services;
    }
}

public class UserService
{
    private readonly IUserRepository _userRepository;
    private readonly IUserServiceExecutor _executor;

    public UserService(IUserRepository userRepository, IUserServiceExecutor executor)
    {
        _userRepository = userRepository;
        _executor = executor;
    }

    public async Task<User> GetUserAsync(string id)
    {
        return await _executor.ExecuteAsync(async () =>
        {
            return await _userRepository.GetByIdAsync(id);
        });
    }
}

public interface IUserServiceExecutor
{
    Task<T> ExecuteAsync<T>(Func<Task<T>> operation);
}

public class UserServiceExecutor : IUserServiceExecutor
{
    private readonly SemaphoreSlim _semaphore;

    public UserServiceExecutor(int maxConcurrency)
    {
        _semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        await _semaphore.WaitAsync();
        try
        {
            return await operation();
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

#### Tipos de Bulkhead
- **Thread Pool Isolation**: Pools de threads separados
- **Process Isolation**: Procesos separados
- **Service Isolation**: Servicios independientes
- **Database Connection Pool**: Pools de conexión separados

### 5. Fallback Pattern

#### Propósito
Proporcionar alternativas cuando el servicio principal falla.

#### Implementación
```csharp
// Ejemplo: Fallback con múltiples niveles
public class RecommendationService
{
    private readonly IRecommendationEngine _recommendationEngine;
    private readonly ICacheService _cacheService;
    private readonly IRecommendationRepository _recommendationRepository;
    private readonly IAsyncPolicy<List<Recommendation>> _fallbackPolicy;

    public RecommendationService(
        IRecommendationEngine recommendationEngine,
        ICacheService cacheService,
        IRecommendationRepository recommendationRepository)
    {
        _recommendationEngine = recommendationEngine;
        _cacheService = cacheService;
        _recommendationRepository = recommendationRepository;

        _fallbackPolicy = Policy<List<Recommendation>>
            .Handle<Exception>()
            .FallbackAsync(async (context) =>
            {
                // Fallback 1: Cache local
                var cached = await _cacheService.GetRecommendationsAsync(context.UserId);
                if (cached.Any())
                {
                    return cached;
                }

                // Fallback 2: Recomendaciones populares
                return await GetPopularRecommendationsAsync();
            });
    }

    public async Task<List<Recommendation>> GetRecommendationsAsync(string userId)
    {
        return await _fallbackPolicy.ExecuteAsync(async () =>
        {
            return await _recommendationEngine.GetRecommendationsAsync(userId);
        });
    }

    private async Task<List<Recommendation>> GetPopularRecommendationsAsync()
    {
        return await _recommendationRepository.GetPopularRecommendationsAsync();
    }
}
```

#### Estrategias de Fallback
- **Cache**: Usar datos cacheados
- **Default Values**: Valores por defecto
- **Alternative Service**: Servicio alternativo
- **Degraded Mode**: Modo degradado con funcionalidad limitada

### 6. Rate Limiting Pattern

#### Propósito
Limitar el número de requests para proteger servicios.

#### Implementación
```csharp
// Ejemplo: Rate Limiting con Bucket4j
@Service
public class RateLimitedService {

    private readonly Bucket bucket = Bucket.builder()
        .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
        .build();

    public Response processRequest(Request request) {
        if (bucket.tryConsume(1)) {
            return processRequestInternal(request);
        } else {
            throw new RateLimitExceededException("Rate limit exceeded");
        }
    }
}
```

#### Tipos de Rate Limiting
- **Fixed Window**: Ventana fija de tiempo
- **Sliding Window**: Ventana deslizante
- **Token Bucket**: Bucket de tokens
- **Leaky Bucket**: Bucket con fuga

### 7. Health Check Pattern

#### Propósito
Monitorear la salud de los servicios y detectar problemas.

#### Implementación
```csharp
// Ejemplo: Health Check con Spring Boot Actuator
@Component
public class DatabaseHealthIndicator : IHealthIndicator
{
    private readonly DataSource _dataSource;

    public DatabaseHealthIndicator(DataSource dataSource)
    {
        _dataSource = dataSource;
    }

    public Health health()
    {
        try (var connection = _dataSource.getConnection())
        {
            if (connection.isValid(1000))
            {
                return Health.up()
                    .withDetail("database", "Available")
                    .withDetail("connection", "OK")
                    .build();
            }
            else
            {
                return Health.down()
                    .withDetail("database", "Unavailable")
                    .withDetail("connection", "Failed")
                    .build();
            }
        }
        catch (Exception e)
        {
            return Health.down()
                .withDetail("database", "Error")
                .withDetail("error", e.Message)
                .build();
        }
    }
}
```

## Configuración y Monitoreo

### Configuración Centralizada
```json
# Ejemplo: Configuración de resiliencia
{
  "Polly": {
    "CircuitBreaker": {
      "UserService": {
        "ExceptionsAllowedBeforeBreaking": 5,
        "DurationOfBreak": "00:00:30",
        "HandledEventsAllowedBeforeBreaking": 10
      }
    },
    "Retry": {
      "PaymentService": {
        "MaxAttempts": 3,
        "WaitDuration": "00:00:01",
        "EnableExponentialBackoff": true,
        "ExponentialBackoffMultiplier": 2
      }
    },
    "Timeout": {
      "ExternalService": {
        "TimeoutDuration": "00:00:05"
      }
    },
    "Bulkhead": {
      "UserService": {
        "MaxConcurrentCalls": 10,
        "MaxWaitDuration": "00:00:01"
      }
    }
  }
}
```

### Métricas de Resiliencia
```csharp
// Ejemplo: Métricas con Micrometer
using Microsoft.Extensions.EventBus.RabbitMQ;
using Microsoft.Extensions.Metrics;

public class ResilienceMetrics
{
    private readonly IEventBus _eventBus;
    private readonly IMetrics _metrics;

    public ResilienceMetrics(IEventBus eventBus, IMetrics metrics)
    {
        _eventBus = eventBus;
        _metrics = metrics;
    }

    [EventListener]
    public void OnCircuitBreakerStateChanged(CircuitBreakerOnStateTransitionEvent @event)
    {
        _metrics.Counter("circuit_breaker_state_changes",
            "name", @event.CircuitBreakerName,
            "state", @event.StateTransition.ToState.Name)
            .Increment();
    }
}
```

### Alertas Recomendadas
- **Circuit Breaker Open**: Circuit breaker abierto por más de 5 minutos
- **High Retry Rate**: Tasa de reintentos > 20%
- **Timeout Exceeded**: Timeouts > 10% de requests
- **Bulkhead Full**: Bulkhead al 90% de capacidad

## Testing de Resiliencia

### Chaos Engineering
```csharp
// Ejemplo: Test de resiliencia
using Xunit;
using FluentAssertions;

public class ResilienceTest
{
    [Fact]
    public void ShouldHandleServiceFailure()
    {
        // Simular fallo del servicio
        var response = Given()
            .When()
            .Get("/api/users/123")
            .Then()
            .StatusCode(200)
            .Body("status", "UNAVAILABLE");

        response.Should().NotBeNull();
        response.StatusCode.Should().Be(200);
        response.Body.Should().Contain("status", "UNAVAILABLE");
    }

    [Fact]
    public void ShouldRetryOnTemporaryFailure()
    {
        // Simular fallo temporal
        var response = Given()
            .When()
            .Post("/api/payments")
            .Then()
            .StatusCode(200);

        response.Should().NotBeNull();
        response.StatusCode.Should().Be(200);
    }
}
```

### Estrategias de Testing
- **Unit Testing**: Probar patrones individualmente
- **Integration Testing**: Probar interacciones entre servicios
- **Chaos Testing**: Simular fallos en producción
- **Load Testing**: Probar bajo carga

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] Patrones de resiliencia identificados y documentados
- [ ] Circuit breakers implementados para servicios críticos
- [ ] Estrategia de retry definida y configurada
- [ ] Timeouts configurados para todas las operaciones externas
- [ ] Fallbacks implementados para funcionalidades críticas
- [ ] Health checks configurados y monitoreados
- [ ] Testing de resiliencia implementado

### Para Cambios Existentes
- [ ] Impacto en resiliencia evaluado
- [ ] Patrones de resiliencia actualizados si es necesario
- [ ] Configuración de resiliencia revisada
- [ ] Testing de resiliencia ejecutado
- [ ] Monitoreo de resiliencia actualizado

## Excepciones y Justificaciones

### Cuándo No Aplicar Patrones de Resiliencia
- **Operaciones idempotentes**: Operaciones que pueden repetirse sin efectos
- **Operaciones críticas**: Operaciones que no pueden fallar
- **Operaciones síncronas**: Operaciones que requieren respuesta inmediata
- **Operaciones de bajo volumen**: Operaciones con tráfico mínimo

### Proceso de Excepción
1. **Análisis de riesgo**: Evaluar impacto de no aplicar patrones
2. **Alternativas**: Identificar otras formas de manejar fallos
3. **Monitoreo**: Implementar monitoreo adicional
4. **Documentación**: Documentar justificación y alternativas
5. **Revisión**: Revisar decisión periódicamente

## Referencias y Recursos

### Librerías de Resiliencia
- [Resilience4j - Circuit breaker para Java]
- [Hystrix - Circuit breaker (deprecated)]
- [Polly - Resilience patterns para .NET]
- [Circuit Breaker - Patrón de diseño]

### Herramientas de Testing
- [Chaos Monkey - Chaos engineering]
- [Gremlin - Chaos engineering platform]
- [Litmus - Chaos engineering para Kubernetes]
- [Chaos Toolkit - Framework de chaos engineering]

### Recursos Adicionales
- [Resilience Patterns - Martin Fowler]
- [Circuit Breaker Pattern - Netflix]
- [Chaos Engineering - Principles]
- [Resilience Engineering - Best Practices]
- [Resilience Engineering - Best Practices]
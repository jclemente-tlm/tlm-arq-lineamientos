# Patrones de Resiliencia y Disaster Recovery

## Propósito

Este lineamiento define los patrones y estrategias de resiliencia para construir sistemas robustos que puedan manejar fallos, latencia y carga de manera efectiva, así como los procedimientos de Disaster Recovery (DR) para garantizar la continuidad del negocio ante fallos catastróficos. El objetivo es garantizar la disponibilidad y confiabilidad de los servicios incluso en condiciones adversas.

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

### RTO y RPO
- **RTO (Recovery Time Objective)**: Tiempo máximo aceptable para restaurar servicios
- **RPO (Recovery Point Objective)**: Pérdida máxima de datos aceptable
- **Definición por servicio**: Cada servicio debe tener sus propios objetivos
- **Validación regular**: Probar objetivos mediante simulacros

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
```json
// Ejemplo: Configuración de timeouts
{
  "Timeouts": {
    "Database": "00:00:30",
    "ExternalApi": "00:00:10",
    "Cache": "00:00:05",
    "FileSystem": "00:00:15"
  }
}
```

### 4. Bulkhead Pattern

#### Propósito
Aislar fallos entre diferentes partes del sistema.

#### Implementación
```csharp
// Ejemplo: Bulkhead con semáforos
public class BulkheadService
{
    private readonly SemaphoreSlim _databaseSemaphore;
    private readonly SemaphoreSlim _externalApiSemaphore;
    private readonly SemaphoreSlim _fileSystemSemaphore;

    public BulkheadService()
    {
        _databaseSemaphore = new SemaphoreSlim(10, 10); // Máximo 10 conexiones
        _externalApiSemaphore = new SemaphoreSlim(5, 5); // Máximo 5 requests
        _fileSystemSemaphore = new SemaphoreSlim(3, 3); // Máximo 3 operaciones
    }

    public async Task<Result> ProcessWithBulkheadAsync(ProcessRequest request)
    {
        // Usar bulkhead específico según el tipo de operación
        var semaphore = request.Type switch
        {
            "database" => _databaseSemaphore,
            "api" => _externalApiSemaphore,
            "file" => _fileSystemSemaphore,
            _ => throw new ArgumentException("Invalid operation type")
        };

        await semaphore.WaitAsync();
        try
        {
            return await ProcessRequestAsync(request);
        }
        finally
        {
            semaphore.Release();
        }
    }
}
```

### 5. Fallback Pattern

#### Propósito
Proporcionar alternativas cuando el servicio principal falla.

#### Implementación
```csharp
// Ejemplo: Fallback con múltiples estrategias
public class RecommendationService
{
    private readonly IRecommendationEngine _primaryEngine;
    private readonly IRecommendationEngine _secondaryEngine;
    private readonly ICacheService _cacheService;

    public async Task<List<Recommendation>> GetRecommendationsAsync(string userId)
    {
        try
        {
            // Intentar con el motor principal
            return await _primaryEngine.GetRecommendationsAsync(userId);
        }
        catch (Exception ex)
        {
            // Fallback 1: Motor secundario
            try
            {
                return await _secondaryEngine.GetRecommendationsAsync(userId);
            }
            catch
            {
                // Fallback 2: Cache
                var cachedRecommendations = await _cacheService.GetAsync<List<Recommendation>>($"rec_{userId}");
                if (cachedRecommendations != null)
                {
                    return cachedRecommendations;
                }

                // Fallback 3: Recomendaciones por defecto
                return GetDefaultRecommendations();
            }
        }
    }

    private List<Recommendation> GetDefaultRecommendations()
    {
        return new List<Recommendation>
        {
            new Recommendation { Id = "default_1", Title = "Producto Popular" },
            new Recommendation { Id = "default_2", Title = "Oferta Especial" }
        };
    }
}
```

## Estrategias de Disaster Recovery

### 1. Backup Strategy

#### Tipos de Backup
```yaml
# Ejemplo: Estrategia de backup con AWS
backup_strategy:
  database:
    type: "automated"
    frequency: "daily"
    retention: "30 days"
    cross_region: true
    encryption: "AES-256"

  application_data:
    type: "incremental"
    frequency: "hourly"
    retention: "7 days"
    storage_class: "S3-IA"

  configuration:
    type: "versioned"
    frequency: "on_change"
    retention: "1 year"
    source_control: true
```

#### Estrategias de Backup
- **Full Backup**: Backup completo de todos los datos
- **Incremental Backup**: Solo cambios desde el último backup
- **Differential Backup**: Cambios desde el último backup completo
- **Continuous Backup**: Backup en tiempo real (CDC)

#### Validación de Backups
```bash
# Ejemplo: Script de validación de backup
#!/bin/bash
# Validar backup de PostgreSQL
pg_restore --list backup_file.dump | grep -q "table_name"
if [ $? -eq 0 ]; then
    echo "Backup validation successful"
else
    echo "Backup validation failed"
    exit 1
fi
```

### 2. Data Replication

#### Replicación Síncrona vs Asíncrona
```sql
-- Ejemplo: Configuración de replicación PostgreSQL
-- Replicación síncrona para datos críticos
ALTER SYSTEM SET synchronous_commit = on;
ALTER SYSTEM SET synchronous_standby_names = 'standby1,standby2';

-- Replicación asíncrona para datos no críticos
ALTER SYSTEM SET synchronous_commit = off;
```

#### Estrategias de Replicación
- **Synchronous**: Confirmación solo después de replicación
- **Asynchronous**: Confirmación inmediata, replicación posterior
- **Semi-synchronous**: Confirmación después de replicación a al menos un nodo
- **Near-synchronous**: Replicación con latencia mínima

### 3. Multi-Region Strategy

#### Arquitectura Multi-Región
```yaml
# Ejemplo: Configuración multi-región con AWS
regions:
  primary:
    name: "us-east-1"
    services:
      - "api-gateway"
      - "lambda"
      - "rds"
      - "dynamodb"

  secondary:
    name: "us-west-2"
    services:
      - "api-gateway"
      - "lambda"
      - "rds-read-replica"
      - "dynamodb-global-table"

  routing:
    strategy: "active-active"
    health_check: "/health"
    failover_time: "30s"
```

#### Patrones de Multi-Región
- **Active-Active**: Servicios activos en todas las regiones
- **Active-Passive**: Servicios activos en una región, pasivos en otras
- **Read Replicas**: Lecturas distribuidas, escrituras centralizadas
- **Global Tables**: Tablas replicadas globalmente (DynamoDB)

### 4. Database Recovery

#### Estrategias de Recuperación de Base de Datos
```sql
-- Ejemplo: Procedimiento de recuperación PostgreSQL
-- 1. Verificar estado del backup
SELECT pg_is_in_recovery();

-- 2. Restaurar desde backup
pg_restore --clean --if-exists --verbose backup_file.dump

-- 3. Verificar integridad
SELECT COUNT(*) FROM critical_table;
```

#### Tipos de Recuperación
- **Point-in-Time Recovery**: Recuperar a un momento específico
- **Full Recovery**: Recuperación completa desde backup
- **Incremental Recovery**: Recuperación incremental
- **Logical Recovery**: Recuperación de datos específicos

### 5. Application Recovery

#### Estrategias de Recuperación de Aplicaciones
```csharp
// Ejemplo: Health check con .NET 8
[ApiController]
[Route("api/[controller]")]
public class HealthController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> Get()
    {
        var health = new
        {
            Status = "Healthy",
            Timestamp = DateTime.UtcNow,
            Services = new
            {
                Database = await CheckDatabaseHealth(),
                Cache = await CheckCacheHealth(),
                ExternalApi = await CheckExternalApiHealth()
            }
        };

        return Ok(health);
    }
}
```

#### Componentes de Recuperación
- **Health Checks**: Verificación de salud de servicios
- **Circuit Breakers**: Prevención de cascada de fallos
- **Graceful Degradation**: Degradación controlada de funcionalidades
- **Auto-scaling**: Escalado automático según demanda

## Plan de Disaster Recovery

### 1. Análisis de Impacto

#### Business Impact Analysis (BIA)
```yaml
# Ejemplo: Análisis de impacto por servicio
services:
  user_service:
    criticality: "High"
    rto: "4 hours"
    rpo: "15 minutes"
    business_impact: "User registration and authentication"

  payment_service:
    criticality: "Critical"
    rto: "1 hour"
    rpo: "5 minutes"
    business_impact: "Financial transactions"

  notification_service:
    criticality: "Medium"
    rto: "8 hours"
    rpo: "1 hour"
    business_impact: "User communications"
```

### 2. Procedimientos de Recuperación

#### Runbook de Recuperación
```bash
#!/bin/bash
# Runbook de recuperación de base de datos

echo "Starting database recovery process..."

# 1. Verificar conectividad
if ! pg_isready -h $DB_HOST -p $DB_PORT; then
    echo "Database is not accessible"
    exit 1
fi

# 2. Detener aplicaciones
echo "Stopping applications..."
docker-compose down

# 3. Restaurar backup
echo "Restoring from backup..."
pg_restore --clean --if-exists --verbose $BACKUP_FILE

# 4. Verificar integridad
echo "Verifying data integrity..."
psql -h $DB_HOST -p $DB_PORT -d $DB_NAME -c "SELECT COUNT(*) FROM users;"

# 5. Reiniciar aplicaciones
echo "Restarting applications..."
docker-compose up -d

echo "Recovery process completed"
```

### 3. Testing de Disaster Recovery

#### Tipos de Testing
- **Tabletop Exercises**: Simulacros de escritorio
- **Partial Failover**: Pruebas de failover parcial
- **Full Failover**: Pruebas de failover completo
- **Data Recovery**: Pruebas de recuperación de datos

#### Métricas de Testing
```yaml
# Ejemplo: Métricas con Prometheus
metrics:
  recovery_time:
    description: "Time to recover from disaster"
    unit: "seconds"
    target: "< 3600"  # 1 hour

  data_loss:
    description: "Amount of data lost during recovery"
    unit: "records"
    target: "0"

  service_availability:
    description: "Service availability during recovery"
    unit: "percentage"
    target: "> 99.9"
```

### Métricas de Disaster Recovery

#### Métricas Clave
- **RTO Achievement**: % de veces que se cumple el RTO
- **RPO Achievement**: % de veces que se cumple el RPO
- **Recovery Success Rate**: % de recuperaciones exitosas
- **Testing Frequency**: Frecuencia de pruebas de DR

#### Implementación de Métricas
```csharp
// Ejemplo: Métricas con Prometheus
public class DisasterRecoveryMetrics
{
    private readonly Counter _recoveryAttempts;
    private readonly Counter _recoverySuccesses;
    private readonly Histogram _recoveryTime;
    private readonly Gauge _dataLoss;

    public DisasterRecoveryMetrics()
    {
        _recoveryAttempts = Metrics.CreateCounter("dr_attempts_total", "Total DR attempts");
        _recoverySuccesses = Metrics.CreateCounter("dr_successes_total", "Total successful DR");
        _recoveryTime = Metrics.CreateHistogram("dr_recovery_time_seconds", "DR recovery time");
        _dataLoss = Metrics.CreateGauge("dr_data_loss_records", "Data loss during recovery");
    }

    public void RecordRecoveryAttempt()
    {
        _recoveryAttempts.Inc();
    }

    public void RecordRecoverySuccess(TimeSpan recoveryTime, int dataLoss)
    {
        _recoverySuccesses.Inc();
        _recoveryTime.Observe(recoveryTime.TotalSeconds);
        _dataLoss.Set(dataLoss);
    }
}
```

## Checklist de Cumplimiento

### Para Patrones de Resiliencia
- [ ] Circuit breakers implementados en servicios críticos
- [ ] Estrategias de retry configuradas apropiadamente
- [ ] Timeouts definidos para todas las operaciones externas
- [ ] Bulkheads implementados para aislar fallos
- [ ] Fallbacks definidos para servicios críticos

### Para Disaster Recovery
- [ ] Estrategia de backup definida e implementada
- [ ] Replicación de datos configurada
- [ ] Arquitectura multi-región implementada
- [ ] Procedimientos de recuperación documentados
- [ ] Testing de DR programado regularmente

### Para Ambos
- [ ] Métricas de resiliencia y DR implementadas
- [ ] Alertas configuradas para eventos críticos
- [ ] Documentación actualizada
- [ ] Equipo entrenado en procedimientos

## Referencias y Recursos

### Herramientas de Resiliencia
- [Polly - Biblioteca de resiliencia para .NET]
- [Resilience4j - Biblioteca de resiliencia para Java]
- [Hystrix - Circuit breaker pattern]
- [Sentinel - Control de flujo y resiliencia]

### Herramientas de Disaster Recovery
- [AWS Backup - Servicio de backup automatizado]
- [Azure Site Recovery - Recuperación de desastres]
- [Veeam - Backup y recuperación]
- [Commvault - Gestión de datos empresarial]

### Documentos Relacionados
- [Observabilidad y Monitorización](09-observabilidad-y-monitorizacion.md) - Monitoreo de resiliencia
- [Performance y Optimización](10-performance-y-optimizacion.md) - Optimización de rendimiento
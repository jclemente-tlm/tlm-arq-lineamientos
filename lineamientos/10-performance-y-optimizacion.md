# Performance y Optimización

## Propósito

Este lineamiento establece los principios, métricas y mejores prácticas para garantizar que los sistemas cumplan con los requisitos de rendimiento, escalabilidad y eficiencia. El objetivo es proporcionar una experiencia de usuario óptima mientras se optimiza el uso de recursos.

## Principios Fundamentales

### Performance by Design
- **Optimización temprana**: Considerar performance desde el diseño inicial
- **Medición continua**: Monitorear métricas de rendimiento en todos los entornos
- **Optimización iterativa**: Mejorar performance de forma incremental
- **Balance costo-beneficio**: Optimizar donde tenga mayor impacto

### Escalabilidad Horizontal
- **Stateless services**: Diseñar servicios sin estado para facilitar escalado
- **Load balancing**: Distribuir carga entre múltiples instancias
- **Database sharding**: Particionar datos para distribuir carga
- **Caching distribuido**: Usar cache compartido entre instancias

### Eficiencia de Recursos
- **Resource pooling**: Reutilizar conexiones y recursos costosos
- **Lazy loading**: Cargar datos solo cuando sea necesario
- **Connection pooling**: Mantener pool de conexiones a bases de datos
- **Memory management**: Optimizar uso de memoria y evitar leaks

## Métricas de Performance

### Métricas de Usuario (User-Centric)
- **First Contentful Paint (FCP)**: < 1.8s
- **Largest Contentful Paint (LCP)**: < 2.5s
- **First Input Delay (FID)**: < 100ms
- **Cumulative Layout Shift (CLS)**: < 0.1
- **Time to Interactive (TTI)**: < 3.8s

### Métricas de Sistema
- **Response Time**: P95 < 500ms, P99 < 1000ms
- **Throughput**: Transacciones por segundo (TPS)
- **Error Rate**: < 1% de errores
- **Availability**: 99.9% uptime
- **Resource Utilization**: CPU < 80%, Memory < 85%

### Métricas de Negocio
- **Conversion Rate**: Impacto en conversiones
- **User Engagement**: Tiempo en página, páginas por sesión
- **Bounce Rate**: % de usuarios que abandonan
- **Revenue Impact**: Impacto en ingresos por mejoras de performance

## Patrones de Optimización

### 1. Caching Strategy

#### Niveles de Cache
```csharp
// Ejemplo: Estrategia de cache en capas
public class UserService
{
    private readonly IUserRepository _userRepository;
    private readonly IProfileRepository _profileRepository;
    private readonly IDistributedCache _cache;

    public UserService(
        IUserRepository userRepository,
        IProfileRepository profileRepository,
        IDistributedCache cache)
    {
        _userRepository = userRepository;
        _profileRepository = profileRepository;
        _cache = cache;
    }

    [Cacheable("user-cache")] // Cache de aplicación
    public async Task<User> GetUserByIdAsync(string id)
    {
        return await _userRepository.GetByIdAsync(id);
    }

    [Cacheable("user-profile-cache")] // Cache específico
    public async Task<UserProfile> GetUserProfileAsync(string id)
    {
        return await _profileRepository.GetByUserIdAsync(id);
    }
}
```

#### Tipos de Cache
- **Application Cache**: Cache en memoria de la aplicación
- **Distributed Cache**: Redis, Memcached
- **CDN Cache**: Cache de contenido estático
- **Database Cache**: Query cache, connection pooling
- **Browser Cache**: Cache del navegador

#### Estrategias de Invalidación
- **TTL (Time To Live)**: Invalidación por tiempo
- **LRU (Least Recently Used)**: Invalidación por uso
- **Event-driven**: Invalidación por eventos
- **Manual**: Invalidación explícita

### 2. Database Optimization

#### Query Optimization
```sql
-- Ejemplo: Query optimizada con índices
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_user_date ON orders(user_id, created_date);

-- Query optimizada
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.email = ?
AND o.created_date >= ?
ORDER BY o.created_date DESC
LIMIT 20;
```

#### Patrones de Acceso a Datos
- **Connection Pooling**: Reutilizar conexiones
- **Batch Operations**: Operaciones en lote
- **Read Replicas**: Separar lecturas de escrituras
- **Database Sharding**: Particionar datos
- **Query Optimization**: Usar índices apropiados

### 3. API Optimization

#### Response Optimization
```csharp
// Ejemplo: API optimizada con paginación y proyección
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet]
    public async Task<ActionResult<PaginatedResult<UserSummary>>> GetUsers(
        [FromQuery] int page = 0,
        [FromQuery] int size = 20,
        [FromQuery] string? fields = null)
    {
        // Proyección de campos
        var projectedFields = !string.IsNullOrEmpty(fields)
            ? fields.Split(',')
            : new[] { "id", "name", "email" };

        var result = await _userService.FindUsersAsync(page, size, projectedFields);
        return Ok(result);
    }
}
```

#### Técnicas de Optimización
- **Pagination**: Paginación de resultados
- **Field Projection**: Seleccionar solo campos necesarios
- **Compression**: Comprimir respuestas (gzip, brotli)
- **Response Caching**: Cache de respuestas HTTP
- **GraphQL**: Query específica del cliente

### 4. Frontend Optimization

#### Code Splitting
```typescript
// Ejemplo: Code splitting con React
import React, { lazy, Suspense } from 'react';

const UserDashboard = lazy(() => import('./UserDashboard'));
const AdminPanel = lazy(() => import('./AdminPanel'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserDashboard />
    </Suspense>
  );
}
```

#### Técnicas de Optimización
- **Code Splitting**: Dividir código en chunks
- **Tree Shaking**: Eliminar código no usado
- **Image Optimization**: Comprimir y optimizar imágenes
- **Lazy Loading**: Cargar recursos bajo demanda
- **Service Workers**: Cache offline

### 5. Microservices Optimization

#### Service Mesh
```yaml
# Ejemplo: Configuración de Istio para optimización
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 90
    - destination:
        host: user-service
        subset: v2
      weight: 10
```

#### Patrones de Optimización
- **Circuit Breaker**: Prevenir cascada de fallos
- **Retry with Backoff**: Reintentos exponenciales
- **Bulkhead**: Aislar fallos entre servicios
- **Rate Limiting**: Limitar requests por cliente
- **Load Balancing**: Distribuir carga entre instancias

## Monitoreo y Alerting

### Métricas a Monitorear
```json
// Ejemplo: Configuración de Prometheus
{
  "scrape_configs": [
    {
      "job_name": "user-service",
      "metrics_path": "/metrics",
      "scrape_interval": "15s",
      "static_configs": [
        {
          "targets": ["user-service:8080"]
        }
      ]
    }
  ]
}
```

### Dashboards Recomendados
- **Application Performance**: Response time, error rate, throughput
- **Infrastructure**: CPU, memory, disk, network
- **Business Metrics**: User engagement, conversion rates
- **Error Tracking**: Error frequency, error types

### Alertas Críticas
- **Response Time**: P95 > 500ms por 5 minutos
- **Error Rate**: > 5% por 2 minutos
- **Availability**: < 99% por 10 minutos
- **Resource Usage**: CPU > 90% por 5 minutos

## Performance Testing

### Tipos de Pruebas
```csharp
// Ejemplo: Test de performance con NBomber
using NBomber.Contracts;
using NBomber.CSharp;
using NBomber.Http.CSharp;

public class PerformanceTest
{
    [Fact]
    public void UserApi_LoadTest()
    {
        var httpClient = new HttpClient();

        var scenario = Scenario.Create("user_api_load_test", async context =>
        {
            var response = await httpClient.GetAsync("https://api.example.com/users");

            return response.IsSuccessStatusCode
                ? Response.Ok()
                : Response.Fail();
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
        );

        NBomberRunner
            .RegisterScenarios(scenario)
            .WithTestName("User API Performance Test")
            .WithTestSuite("API")
            .Run();
    }
}
```

### Estrategias de Testing
- **Load Testing**: Carga normal esperada
- **Stress Testing**: Carga máxima sostenible
- **Spike Testing**: Picos de carga repentinos
- **Endurance Testing**: Carga sostenida por tiempo prolongado
- **Scalability Testing**: Verificar escalabilidad horizontal

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] Requisitos de performance definidos y documentados
- [ ] Métricas de performance establecidas y monitoreadas
- [ ] Estrategia de cache definida e implementada
- [ ] Optimización de base de datos completada
- [ ] Testing de performance automatizado
- [ ] Alertas de performance configuradas
- [ ] Documentación de optimizaciones implementadas

### Para Cambios Existentes
- [ ] Impacto en performance evaluado
- [ ] Métricas de baseline establecidas
- [ ] Testing de performance ejecutado
- [ ] Optimizaciones implementadas si es necesario
- [ ] Monitoreo de performance actualizado

## Excepciones y Justificaciones

### Cuándo Priorizar Performance
- **Sistemas críticos**: Aplicaciones de misión crítica
- **Alto tráfico**: Sistemas con muchos usuarios concurrentes
- **Transacciones financieras**: Sistemas de pago y banca
- **Experiencia de usuario**: Aplicaciones donde UX es crítico

### Proceso de Optimización
1. **Medición**: Establecer métricas de baseline
2. **Análisis**: Identificar cuellos de botella
3. **Optimización**: Implementar mejoras
4. **Validación**: Medir impacto de optimizaciones
5. **Iteración**: Repetir proceso según sea necesario

## Referencias y Recursos

### Herramientas de Monitoreo
- [Prometheus - Métricas y alerting]
- [Grafana - Visualización de métricas]
- [New Relic - APM y monitoreo]
- [Datadog - Monitoreo completo]

### Herramientas de Testing
- [JMeter - Testing de carga]
- [Gatling - Testing de APIs]
- [K6 - Testing moderno]
- [Artillery - Testing de APIs]

### Frameworks de Optimización
- [Spring Boot Actuator - Métricas de aplicación]
- [Micrometer - Métricas de aplicación]
- [Hystrix - Circuit breaker]
- [Resilience4j - Patrones de resiliencia]

### Recursos Adicionales
- [Web.dev - Performance guidelines]
- [Google PageSpeed Insights]
- [GTmetrix - Análisis de performance web]
- [Lighthouse - Auditoría de performance]
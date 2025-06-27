# Arquitectura Orientada a Eventos (EDA)

## Propósito

Este lineamiento define los principios, patrones y mejores prácticas para implementar arquitectura orientada a eventos (Event-Driven Architecture). Se enfoca en crear sistemas desacoplados, escalables y resilientes mediante la comunicación basada en eventos.

## Principios Fundamentales

### Desacoplamiento
- **Productores independientes**: Los productores no conocen a los consumidores
- **Consumidores autónomos**: Los consumidores procesan eventos independientemente
- **Contratos de eventos**: Interfaces bien definidas entre componentes
- **Temporal decoupling**: Los eventos pueden ser procesados en diferentes momentos
- **Spatial decoupling**: Los componentes pueden estar en diferentes ubicaciones

### Escalabilidad
- **Procesamiento paralelo**: Múltiples consumidores pueden procesar eventos
- **Auto-scaling**: Escalado automático basado en volumen de eventos
- **Load distribution**: Distribución de carga entre consumidores
- **Event partitioning**: Particionamiento de eventos para paralelismo
- **Backpressure handling**: Manejo de presión de eventos

### Resiliencia
- **Idempotencia**: Los consumidores deben ser idempotentes
- **Dead Letter Queues**: Manejo de eventos fallidos
- **Retry policies**: Políticas de reintento configuradas
- **Circuit breakers**: Protección contra fallos en cascada
- **Event replay**: Capacidad de reprocesar eventos

## Patrones de Event-Driven Architecture

### Event Sourcing
```csharp
// Ejemplo: Event sourcing con .NET 8
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

public class User : AggregateRoot
{
    public Guid Id { get; private set; }
    public string Email { get; private set; }
    public string Name { get; private set; }
    public UserStatus Status { get; private set; }

    public User(string email, string name)
    {
        Id = Guid.NewGuid();
        Email = email;
        Name = name;
        Status = UserStatus.Active;

        AddDomainEvent(new UserCreatedEvent(Id, Email, Name));
    }

    public void UpdateProfile(string name)
    {
        Name = name;
        AddDomainEvent(new UserProfileUpdatedEvent(Id, Name));
    }

    public void Deactivate()
    {
        Status = UserStatus.Inactive;
        AddDomainEvent(new UserDeactivatedEvent(Id));
    }
}

// Event Store
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<IDomainEvent> events, int expectedVersion);
    Task<IEnumerable<IDomainEvent>> GetEventsAsync(Guid aggregateId);
}

public class EventStore : IEventStore
{
    private readonly DbContext _context;

    public async Task SaveEventsAsync(Guid aggregateId, IEnumerable<IDomainEvent> events, int expectedVersion)
    {
        var eventRecords = events.Select((e, index) => new EventRecord
        {
            AggregateId = aggregateId,
            EventType = e.GetType().Name,
            EventData = JsonSerializer.Serialize(e),
            Version = expectedVersion + index + 1,
            Timestamp = DateTime.UtcNow
        });

        _context.Events.AddRange(eventRecords);
        await _context.SaveChangesAsync();
    }

    public async Task<IEnumerable<IDomainEvent>> GetEventsAsync(Guid aggregateId)
    {
        var eventRecords = await _context.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();

        return eventRecords.Select(DeserializeEvent);
    }
}
```

### CQRS (Command Query Responsibility Segregation)
```csharp
// Ejemplo: CQRS con eventos
public class UserCommandHandler
{
    private readonly IEventStore _eventStore;
    private readonly IEventPublisher _eventPublisher;

    public async Task<Result<UserDto>> HandleAsync(CreateUserCommand command)
    {
        var user = new User(command.Email, command.Name);

        await _eventStore.SaveEventsAsync(user.Id, user.DomainEvents, -1);

        // Publicar eventos para proyecciones
        foreach (var domainEvent in user.DomainEvents)
        {
            await _eventPublisher.PublishAsync(domainEvent);
        }

        return Result<UserDto>.Success(new UserDto
        {
            Id = user.Id,
            Email = user.Email,
            Name = user.Name,
            Status = user.Status.ToString()
        });
    }
}

// Query Handler (Read Model)
public class UserQueryHandler
{
    private readonly IUserReadRepository _readRepository;

    public async Task<UserDto> HandleAsync(GetUserQuery query)
    {
        return await _readRepository.GetByIdAsync(query.UserId);
    }
}

// Event Handler para proyecciones
public class UserCreatedEventHandler : IEventHandler<UserCreatedEvent>
{
    private readonly IUserReadRepository _readRepository;

    public async Task HandleAsync(UserCreatedEvent @event)
    {
        var userProjection = new UserProjection
        {
            Id = @event.UserId,
            Email = @event.Email,
            Name = @event.Name,
            Status = "Active",
            CreatedAt = @event.Timestamp
        };

        await _readRepository.SaveAsync(userProjection);
    }
}
```

### Saga Pattern
```csharp
// Ejemplo: Saga para proceso de orden
public class OrderSaga : ISaga
{
    public Guid Id { get; set; }
    public string State { get; set; }
    public List<ISagaStep> Steps { get; set; } = new();

    public async Task ExecuteAsync()
    {
        foreach (var step in Steps)
        {
            try
            {
                await step.ExecuteAsync();
                State = step.GetType().Name;
            }
            catch (Exception ex)
            {
                await CompensateAsync(step);
                throw;
            }
        }
    }

    private async Task CompensateAsync(ISagaStep step)
    {
        if (step is ICompensableSagaStep compensableStep)
        {
            await compensableStep.CompensateAsync();
        }
    }
}

public class ReserveInventoryStep : ISagaStep, ICompensableSagaStep
{
    private readonly IInventoryService _inventoryService;
    private readonly OrderCreatedEvent _orderEvent;

    public async Task ExecuteAsync()
    {
        await _inventoryService.ReserveItemsAsync(_orderEvent.OrderItems);
    }

    public async Task CompensateAsync()
    {
        await _inventoryService.ReleaseItemsAsync(_orderEvent.OrderItems);
    }
}
```

## Implementación con .NET 8

### Event Publisher
```csharp
// Ejemplo: Event publisher con RabbitMQ
public interface IEventPublisher
{
    Task PublishAsync<T>(T @event) where T : class;
}

public class RabbitMQEventPublisher : IEventPublisher
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    private readonly ILogger<RabbitMQEventPublisher> _logger;

    public async Task PublishAsync<T>(T @event) where T : class
    {
        try
        {
            var message = JsonSerializer.Serialize(@event);
            var body = Encoding.UTF8.GetBytes(message);

            var properties = _channel.CreateBasicProperties();
            properties.Persistent = true;
            properties.MessageId = Guid.NewGuid().ToString();
            properties.Timestamp = new AmqpTimestamp(DateTimeOffset.UtcNow.ToUnixTimeSeconds());

            _channel.BasicPublish(
                exchange: "domain_events",
                routingKey: typeof(T).Name.ToLower(),
                mandatory: true,
                basicProperties: properties,
                body: body);

            _logger.LogInformation("Event {EventType} published with ID {MessageId}",
                typeof(T).Name, properties.MessageId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error publishing event {EventType}", typeof(T).Name);
            throw;
        }
    }
}
```

### Event Consumer
```csharp
// Ejemplo: Event consumer con RabbitMQ
public class RabbitMQEventConsumer : IEventConsumer
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<RabbitMQEventConsumer> _logger;

    public async Task StartConsumingAsync()
    {
        _channel.QueueDeclare(
            queue: "user_events_queue",
            durable: true,
            exclusive: false,
            autoDelete: false,
            arguments: null);

        _channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);

        var consumer = new EventingBasicConsumer(_channel);
        consumer.Received += async (model, ea) =>
        {
            try
            {
                var body = ea.Body.ToArray();
                var message = Encoding.UTF8.GetString(body);
                var eventType = ea.RoutingKey;

                await ProcessEventAsync(eventType, message);

                _channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing event");
                _channel.BasicNack(deliveryTag: ea.DeliveryTag, multiple: false, requeue: false);
            }
        };

        _channel.BasicConsume(queue: "user_events_queue", autoAck: false, consumer: consumer);
    }

    private async Task ProcessEventAsync(string eventType, string message)
    {
        var handlerType = typeof(IEventHandler<>).MakeGenericType(Type.GetType(eventType));
        var handler = _serviceProvider.GetService(handlerType);

        if (handler != null)
        {
            var eventData = JsonSerializer.Deserialize(message, Type.GetType(eventType));
            var handleMethod = handlerType.GetMethod("HandleAsync");
            await (Task)handleMethod.Invoke(handler, new[] { eventData });
        }
    }
}
```

### AWS EventBridge Integration
```csharp
// Ejemplo: AWS EventBridge con .NET
public class AWSEventBridgePublisher : IEventPublisher
{
    private readonly IAmazonEventBridge _eventBridge;
    private readonly ILogger<AWSEventBridgePublisher> _logger;

    public async Task PublishAsync<T>(T @event) where T : class
    {
        try
        {
            var eventDetail = JsonSerializer.Serialize(@event);
            var putEventsRequest = new PutEventsRequest
            {
                Entries = new List<PutEventsRequestEntry>
                {
                    new PutEventsRequestEntry
                    {
                        Source = "com.company.users",
                        DetailType = typeof(T).Name,
                        Detail = eventDetail,
                        EventBusName = "default"
                    }
                }
            };

            var response = await _eventBridge.PutEventsAsync(putEventsRequest);

            if (response.FailedEntryCount > 0)
            {
                throw new EventPublishException("Failed to publish event");
            }

            _logger.LogInformation("Event {EventType} published successfully", typeof(T).Name);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error publishing event {EventType}", typeof(T).Name);
            throw;
        }
    }
}
```

## Patrones de Eventos

### Domain Events
```csharp
// Ejemplo: Domain events
public interface IDomainEvent
{
    Guid Id { get; }
    DateTime Timestamp { get; }
}

public record UserCreatedEvent : IDomainEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime Timestamp { get; } = DateTime.UtcNow;
    public Guid UserId { get; }
    public string Email { get; }
    public string Name { get; }

    public UserCreatedEvent(Guid userId, string email, string name)
    {
        UserId = userId;
        Email = email;
        Name = name;
    }
}

public record UserProfileUpdatedEvent : IDomainEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime Timestamp { get; } = DateTime.UtcNow;
    public Guid UserId { get; }
    public string Name { get; }

    public UserProfileUpdatedEvent(Guid userId, string name)
    {
        UserId = userId;
        Name = name;
    }
}
```

### Integration Events
```csharp
// Ejemplo: Integration events para comunicación entre servicios
public record OrderCreatedIntegrationEvent
{
    public Guid OrderId { get; init; }
    public Guid UserId { get; init; }
    public List<OrderItem> Items { get; init; }
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; }
}

public record PaymentProcessedIntegrationEvent
{
    public Guid OrderId { get; init; }
    public Guid PaymentId { get; init; }
    public string Status { get; init; }
    public DateTime ProcessedAt { get; init; }
}
```

## Dead Letter Queue (DLQ)

### Implementación de DLQ
```csharp
// Ejemplo: Dead Letter Queue con RabbitMQ
public class DeadLetterQueueHandler
{
    private readonly IModel _channel;
    private readonly ILogger<DeadLetterQueueHandler> _logger;

    public void ConfigureDeadLetterQueue()
    {
        // Configurar exchange para DLQ
        _channel.ExchangeDeclare("dlq_exchange", ExchangeType.Direct, durable: true);

        // Configurar cola DLQ
        _channel.QueueDeclare(
            queue: "dlq_queue",
            durable: true,
            exclusive: false,
            autoDelete: false,
            arguments: null);

        _channel.QueueBind("dlq_queue", "dlq_exchange", "failed_events");

        // Configurar cola principal con DLQ
        var arguments = new Dictionary<string, object>
        {
            { "x-dead-letter-exchange", "dlq_exchange" },
            { "x-dead-letter-routing-key", "failed_events" }
        };

        _channel.QueueDeclare(
            queue: "user_events_queue",
            durable: true,
            exclusive: false,
            autoDelete: false,
            arguments: arguments);
    }

    public async Task ProcessDeadLetterQueueAsync()
    {
        var consumer = new EventingBasicConsumer(_channel);
        consumer.Received += async (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = Encoding.UTF8.GetString(body);
            var headers = ea.BasicProperties.Headers;

            _logger.LogWarning("Processing failed event: {Message}", message);

            // Implementar lógica de retry o notificación
            await HandleFailedEventAsync(message, headers);

            _channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
        };

        _channel.BasicConsume(queue: "dlq_queue", autoAck: false, consumer: consumer);
    }
}
```

## Monitoreo y Observabilidad

### Event Tracing
```csharp
// Ejemplo: Distributed tracing para eventos
public class EventTracingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<EventTracingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        using var activity = ActivitySource.StartActivity("ProcessEvent");
        activity?.SetTag("correlation.id", correlationId);

        _logger.LogInformation("Processing event with correlation ID: {CorrelationId}", correlationId);

        await _next(context);
    }
}
```

### Event Metrics
```csharp
// Ejemplo: Métricas de eventos con Prometheus
public class EventMetrics
{
    private readonly Counter _eventsPublished;
    private readonly Counter _eventsProcessed;
    private readonly Histogram _eventProcessingDuration;

    public EventMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("event_metrics");

        _eventsPublished = meter.CreateCounter<long>("events_published_total");
        _eventsProcessed = meter.CreateCounter<long>("events_processed_total");
        _eventProcessingDuration = meter.CreateHistogram<double>("event_processing_duration_seconds");
    }

    public void RecordEventPublished(string eventType)
    {
        _eventsPublished.Add(1, new KeyValuePair<string, object>("event_type", eventType));
    }

    public void RecordEventProcessed(string eventType, TimeSpan duration)
    {
        _eventsProcessed.Add(1, new KeyValuePair<string, object>("event_type", eventType));
        _eventProcessingDuration.Record(duration.TotalSeconds, new KeyValuePair<string, object>("event_type", eventType));
    }
}
```

## Checklist de Cumplimiento

### Para Nuevos Eventos
- [ ] Evento documentado con contrato claro
- [ ] Idempotencia implementada en consumidores
- [ ] Dead Letter Queue configurada
- [ ] Retry policies definidas
- [ ] Monitoreo y alertas configurados
- [ ] Versionado de eventos implementado
- [ ] Testing de eventos automatizado

### Para Migraciones
- [ ] Análisis de impacto completado
- [ ] Estrategia de migración definida
- [ ] Compatibilidad hacia atrás mantenida
- [ ] Testing exhaustivo realizado
- [ ] Documentación actualizada

## Excepciones y Justificaciones

### Cuándo No Usar EDA
- **Operaciones síncronas críticas**: Cuando se requiere respuesta inmediata
- **Transacciones complejas**: Cuando se necesita ACID compliance
- **Sistemas simples**: Cuando la complejidad no justifica la overhead
- **Requisitos de orden**: Cuando el orden de eventos es crítico

### Proceso de Decisión
1. **Análisis de beneficios**: Evaluar valor real de EDA
2. **Evaluación de complejidad**: Considerar overhead de implementación
3. **Aprobación**: Obtener aprobación del equipo de arquitectura
4. **Documentación**: Documentar decisión y justificación
5. **Revisión**: Revisar decisión periódicamente

## Referencias y Recursos

### Patrones y Mejores Prácticas
- [Event Sourcing - Martin Fowler]
- [CQRS Pattern - Martin Fowler]
- [Saga Pattern - Chris Richardson]
- [Event-Driven Architecture - Patterns]

### Herramientas y Frameworks
- [RabbitMQ - Message Broker]
- [Apache Kafka - Distributed Streaming]
- [AWS EventBridge - Event Bus]
- [Azure Event Grid - Event Service]

### Recursos Adicionales
- [Event Store - Event Sourcing Database]
- [MassTransit - Message Bus for .NET]
- [NServiceBus - Service Bus for .NET]
- [Event Sourcing and CQRS - Greg Young]
# Integración y ETL

## Visión General

La integración de sistemas y los procesos ETL (Extract, Transform, Load) son fundamentales para conectar aplicaciones, sincronizar datos y mantener la consistencia en arquitecturas distribuidas. En nuestro contexto, nos enfocamos en patrones modernos basados en APIs, eventos y herramientas cloud-native.

## Patrones de Integración

### 1. API-First Integration
```csharp
// API Gateway para Integración
[ApiController]
[Route("api/[controller]")]
public class IntegrationController : ControllerBase
{
    private readonly IIntegrationService _integrationService;
    private readonly ILogger<IntegrationController> _logger;

    [HttpPost("sync")]
    public async Task<IActionResult> SyncDataAsync([FromBody] SyncRequest request)
    {
        try
        {
            var result = await _integrationService.SyncDataAsync(request);
            return Ok(result);
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to sync data");
            return StatusCode(500, "Internal server error");
        }
    }

    [HttpGet("status/{jobId}")]
    public async Task<IActionResult> GetSyncStatusAsync(string jobId)
    {
        var status = await _integrationService.GetSyncStatusAsync(jobId);
        return Ok(status);
    }
}

// Integration Service
public class IntegrationService : IIntegrationService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IEventBus _eventBus;
    private readonly IDataValidationService _validationService;
    private readonly ILogger<IntegrationService> _logger;

    public async Task<SyncResult> SyncDataAsync(SyncRequest request)
    {
        var jobId = Guid.NewGuid().ToString();

        try
        {
            // 1. Validar datos de entrada
            await _validationService.ValidateAsync(request.Data);

            // 2. Extraer datos del sistema origen
            var sourceData = await ExtractDataAsync(request.SourceSystem, request.Filters);

            // 3. Transformar datos
            var transformedData = await TransformDataAsync(sourceData, request.Transformations);

            // 4. Cargar datos en sistema destino
            await LoadDataAsync(request.TargetSystem, transformedData);

            // 5. Publicar evento de sincronización
            await _eventBus.PublishAsync(new DataSyncCompletedEvent
            {
                JobId = jobId,
                SourceSystem = request.SourceSystem,
                TargetSystem = request.TargetSystem,
                RecordsProcessed = transformedData.Count,
                CompletedAt = DateTime.UtcNow
            });

            return new SyncResult
            {
                JobId = jobId,
                Status = "Completed",
                RecordsProcessed = transformedData.Count,
                CompletedAt = DateTime.UtcNow
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Sync failed for job {JobId}", jobId);

            await _eventBus.PublishAsync(new DataSyncFailedEvent
            {
                JobId = jobId,
                Error = ex.Message,
                FailedAt = DateTime.UtcNow
            });

            throw;
        }
    }

    private async Task<List<dynamic>> ExtractDataAsync(string sourceSystem, Dictionary<string, object> filters)
    {
        var client = _httpClientFactory.CreateClient(sourceSystem);
        var response = await client.GetAsync($"/api/data?{BuildQueryString(filters)}");

        if (!response.IsSuccessStatusCode)
        {
            throw new IntegrationException($"Failed to extract data from {sourceSystem}");
        }

        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<List<dynamic>>(content) ?? new List<dynamic>();
    }

    private async Task<List<dynamic>> TransformDataAsync(List<dynamic> sourceData, List<TransformationRule> transformations)
    {
        var transformedData = new List<dynamic>();

        foreach (var item in sourceData)
        {
            var transformedItem = new Dictionary<string, object>();

            foreach (var rule in transformations)
            {
                var value = ApplyTransformationRule(item, rule);
                transformedItem[rule.TargetField] = value;
            }

            transformedData.Add(transformedItem);
        }

        return transformedData;
    }

    private async Task LoadDataAsync(string targetSystem, List<dynamic> data)
    {
        var client = _httpClientFactory.CreateClient(targetSystem);
        var content = JsonSerializer.Serialize(data);

        var response = await client.PostAsync("/api/data",
            new StringContent(content, Encoding.UTF8, "application/json"));

        if (!response.IsSuccessStatusCode)
        {
            throw new IntegrationException($"Failed to load data to {targetSystem}");
        }
    }
}
```

### 2. Event-Driven Integration
```csharp
// Event Bus Implementation
public class EventBus : IEventBus
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    private readonly ILogger<EventBus> _logger;
    private readonly Dictionary<string, List<IIntegrationEventHandler>> _handlers;

    public EventBus(IConnection connection, ILogger<EventBus> logger)
    {
        _connection = connection;
        _channel = _connection.CreateModel();
        _logger = logger;
        _handlers = new Dictionary<string, List<IIntegrationEventHandler>>();
    }

    public async Task PublishAsync<T>(T @event) where T : IntegrationEvent
    {
        try
        {
            var eventName = @event.GetType().Name;
            var message = JsonSerializer.Serialize(@event);

            _channel.ExchangeDeclare("integration_events", ExchangeType.Topic, durable: true);
            _channel.QueueDeclare(eventName, durable: true, exclusive: false, autoDelete: false);
            _channel.QueueBind(eventName, "integration_events", eventName);

            var body = Encoding.UTF8.GetBytes(message);
            var properties = _channel.CreateBasicProperties();
            properties.Persistent = true;
            properties.MessageId = Guid.NewGuid().ToString();
            properties.Timestamp = new AmqpTimestamp(DateTimeOffset.UtcNow.ToUnixTimeSeconds());

            _channel.BasicPublish("integration_events", eventName, properties, body);

            _logger.LogInformation("Event {EventName} published", eventName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to publish event {EventName}", @event.GetType().Name);
            throw;
        }
    }

    public void Subscribe<T, TH>()
        where T : IntegrationEvent
        where TH : IIntegrationEventHandler<T>
    {
        var eventName = typeof(T).Name;
        var handlerType = typeof(TH);

        if (!_handlers.ContainsKey(eventName))
        {
            _handlers[eventName] = new List<IIntegrationEventHandler>();
        }

        _handlers[eventName].Add(Activator.CreateInstance<TH>());

        StartBasicConsume(eventName);
    }

    private void StartBasicConsume(string eventName)
    {
        var consumer = new EventingBasicConsumer(_channel);
        consumer.Received += Consumer_Received;

        _channel.BasicConsume(eventName, false, consumer);
    }

    private async void Consumer_Received(object sender, BasicDeliverEventArgs e)
    {
        var eventName = e.RoutingKey;
        var message = Encoding.UTF8.GetString(e.Body.Span);

        try
        {
            await ProcessEvent(eventName, message);
            _channel.BasicAck(e.DeliveryTag, false);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process event {EventName}", eventName);
            _channel.BasicNack(e.DeliveryTag, false, true);
        }
    }

    private async Task ProcessEvent(string eventName, string message)
    {
        if (_handlers.ContainsKey(eventName))
        {
            var subscriptions = _handlers[eventName];
            foreach (var subscription in subscriptions)
            {
                await subscription.HandleAsync(message);
            }
        }
    }
}

// Integration Events
public abstract class IntegrationEvent
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public DateTime CreationDate { get; set; } = DateTime.UtcNow;
}

public class DataSyncCompletedEvent : IntegrationEvent
{
    public string JobId { get; set; } = string.Empty;
    public string SourceSystem { get; set; } = string.Empty;
    public string TargetSystem { get; set; } = string.Empty;
    public int RecordsProcessed { get; set; }
    public DateTime CompletedAt { get; set; }
}

public class DataSyncFailedEvent : IntegrationEvent
{
    public string JobId { get; set; } = string.Empty;
    public string Error { get; set; } = string.Empty;
    public DateTime FailedAt { get; set; }
}

// Event Handlers
public interface IIntegrationEventHandler
{
    Task HandleAsync(string message);
}

public interface IIntegrationEventHandler<T> : IIntegrationEventHandler where T : IntegrationEvent
{
    Task HandleAsync(T @event);
}

public class DataSyncCompletedEventHandler : IIntegrationEventHandler<DataSyncCompletedEvent>
{
    private readonly ILogger<DataSyncCompletedEventHandler> _logger;
    private readonly INotificationService _notificationService;

    public async Task HandleAsync(DataSyncCompletedEvent @event)
    {
        _logger.LogInformation("Data sync completed: {JobId}, {RecordsProcessed} records processed",
            @event.JobId, @event.RecordsProcessed);

        await _notificationService.SendNotificationAsync(new Notification
        {
            Type = "DataSync",
            Message = $"Data sync completed successfully. {@event.RecordsProcessed} records processed.",
            Recipients = new[] { "admin@company.com" }
        });
    }

    public async Task HandleAsync(string message)
    {
        var @event = JsonSerializer.Deserialize<DataSyncCompletedEvent>(message);
        if (@event != null)
        {
            await HandleAsync(@event);
        }
    }
}
```

## ETL Moderno con AWS

### 1. AWS Glue ETL Jobs
```csharp
// AWS Glue ETL Job con .NET
using Amazon.Glue;
using Amazon.Glue.Model;
using Amazon.S3;
using Amazon.S3.Model;
using Newtonsoft.Json;
using System.Text;

public class GlueETLJob
{
    private readonly IAmazonGlue _glueClient;
    private readonly IAmazonS3 _s3Client;
    private readonly ILogger<GlueETLJob> _logger;

    public GlueETLJob(IAmazonGlue glueClient, IAmazonS3 s3Client, ILogger<GlueETLJob> logger)
    {
        _glueClient = glueClient;
        _s3Client = s3Client;
        _logger = logger;
    }

    public async Task ProcessCustomerDataAsync()
    {
        try
        {
            // Extraer datos desde S3
            var rawData = await ExtractDataFromS3Async("raw-data-bucket", "customer-data/");

            // Aplicar transformaciones
            var transformedData = TransformCustomerData(rawData);

            // Filtrar datos válidos
            var filteredData = FilterValidData(transformedData);

            // Escribir a S3 procesado
            await SaveProcessedDataToS3Async(filteredData, "processed-data-bucket", "customers/");

            // Escribir a Redshift (usando AWS SDK)
            await SaveToRedshiftAsync(filteredData);

            _logger.LogInformation("ETL job completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ETL job failed");
            throw;
        }
    }

    private async Task<List<CustomerData>> ExtractDataFromS3Async(string bucket, string prefix)
    {
        var request = new ListObjectsV2Request
        {
            BucketName = bucket,
            Prefix = prefix
        };

        var response = await _s3Client.ListObjectsV2Async(request);
        var customerData = new List<CustomerData>();

        foreach (var obj in response.S3Objects)
        {
            var getObjectRequest = new GetObjectRequest
            {
                BucketName = bucket,
                Key = obj.Key
            };

            using var response = await _s3Client.GetObjectAsync(getObjectRequest);
            using var reader = new StreamReader(response.ResponseStream);
            var content = await reader.ReadToEndAsync();

            var data = JsonConvert.DeserializeObject<List<CustomerData>>(content);
            customerData.AddRange(data);
        }

        return customerData;
    }

    private List<CustomerData> TransformCustomerData(List<CustomerData> rawData)
    {
        return rawData.Select(record =>
        {
            // Limpiar y validar datos
            if (!string.IsNullOrEmpty(record.Email))
            {
                record.Email = record.Email.ToLower().Trim();
            }

            if (!string.IsNullOrEmpty(record.Phone))
            {
                // Formatear número de teléfono
                var phone = record.Phone.Replace("-", "").Replace(" ", "");
                if (phone.Length == 10)
                {
                    record.Phone = $"({phone.Substring(0, 3)}) {phone.Substring(3, 3)}-{phone.Substring(6)}";
                }
            }

            // Agregar timestamp de procesamiento
            record.ProcessedAt = DateTime.UtcNow;

            return record;
        }).ToList();
    }

    private List<CustomerData> FilterValidData(List<CustomerData> data)
    {
        return data.Where(record =>
            !string.IsNullOrEmpty(record.Email) &&
            record.Email.Contains("@")).ToList();
    }

    private async Task SaveProcessedDataToS3Async(List<CustomerData> data, string bucket, string prefix)
    {
        var json = JsonConvert.SerializeObject(data, Formatting.Indented);
        var key = $"{prefix}{DateTime.UtcNow:yyyy/MM/dd}/processed_{Guid.NewGuid()}.json";

        var request = new PutObjectRequest
        {
            BucketName = bucket,
            Key = key,
            ContentBody = json,
            ContentType = "application/json"
        };

        await _s3Client.PutObjectAsync(request);
    }

    private async Task SaveToRedshiftAsync(List<CustomerData> data)
    {
        // Implementar conexión a Redshift usando AWS SDK o ADO.NET
        // Este es un ejemplo simplificado
        var connectionString = "your-redshift-connection-string";

        using var connection = new NpgsqlConnection(connectionString);
        await connection.OpenAsync();

        foreach (var customer in data)
        {
            var command = new NpgsqlCommand(
                "INSERT INTO customers (id, email, phone, name, processed_at) VALUES (@id, @email, @phone, @name, @processedAt)",
                connection);

            command.Parameters.AddWithValue("@id", customer.Id);
            command.Parameters.AddWithValue("@email", customer.Email);
            command.Parameters.AddWithValue("@phone", customer.Phone);
            command.Parameters.AddWithValue("@name", customer.Name);
            command.Parameters.AddWithValue("@processedAt", customer.ProcessedAt);

            await command.ExecuteNonQueryAsync();
        }
    }
}

public class CustomerData
{
    public string Id { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
    public string Name { get; set; }
    public DateTime ProcessedAt { get; set; }
}
```

### 2. AWS Step Functions para Orquestación
```json
{
  "Comment": "ETL Pipeline Orchestration",
  "StartAt": "ValidateInput",
  "States": {
    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validate-input",
      "Next": "ExtractData",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "Next": "HandleValidationError"
        }
      ]
    },
    "ExtractData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:extract-data",
      "Next": "TransformData",
      "Catch": [
        {
          "ErrorEquals": ["ExtractionError"],
          "Next": "HandleExtractionError"
        }
      ]
    },
    "TransformData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:transform-data",
      "Next": "LoadData",
      "Catch": [
        {
          "ErrorEquals": ["TransformationError"],
          "Next": "HandleTransformationError"
        }
      ]
    },
    "LoadData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:load-data",
      "Next": "NotifySuccess",
      "Catch": [
        {
          "ErrorEquals": ["LoadError"],
          "Next": "HandleLoadError"
        }
      ]
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:notify-success",
      "End": true
    },
    "HandleValidationError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:handle-error",
      "End": true
    },
    "HandleExtractionError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:handle-error",
      "End": true
    },
    "HandleTransformationError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:handle-error",
      "End": true
    },
    "HandleLoadError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:handle-error",
      "End": true
    }
  }
}
```

### 3. AWS Lambda Functions para ETL
```csharp
// Extract Lambda Function
public class ExtractFunction
{
    private readonly ILogger<ExtractFunction> _logger;
    private readonly IDataExtractor _dataExtractor;

    public async Task<ExtractResult> FunctionHandler(ExtractRequest request, ILambdaContext context)
    {
        try
        {
            _logger.LogInformation("Starting data extraction for source: {Source}", request.Source);

            var data = await _dataExtractor.ExtractAsync(request.Source, request.Filters);

            // Guardar datos extraídos en S3
            var s3Key = $"raw-data/{request.Source}/{DateTime.UtcNow:yyyy/MM/dd}/{Guid.NewGuid()}.json";
            await SaveToS3Async(s3Key, data);

            return new ExtractResult
            {
                Success = true,
                RecordsExtracted = data.Count,
                S3Location = s3Key,
                ExtractedAt = DateTime.UtcNow
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Extraction failed for source: {Source}", request.Source);
            throw;
        }
    }

    private async Task SaveToS3Async(string s3Key, List<dynamic> data)
    {
        var s3Client = new AmazonS3Client();
        var jsonData = JsonSerializer.Serialize(data);

        var putRequest = new PutObjectRequest
        {
            BucketName = Environment.GetEnvironmentVariable("RAW_DATA_BUCKET"),
            Key = s3Key,
            ContentBody = jsonData,
            ContentType = "application/json"
        };

        await s3Client.PutObjectAsync(putRequest);
    }
}

// Transform Lambda Function
public class TransformFunction
{
    private readonly ILogger<TransformFunction> _logger;
    private readonly IDataTransformer _dataTransformer;

    public async Task<TransformResult> FunctionHandler(TransformRequest request, ILambdaContext context)
    {
        try
        {
            _logger.LogInformation("Starting data transformation for {Records} records", request.RecordCount);

            // Leer datos desde S3
            var rawData = await LoadFromS3Async(request.S3Location);

            // Aplicar transformaciones
            var transformedData = await _dataTransformer.TransformAsync(rawData, request.Transformations);

            // Guardar datos transformados
            var processedS3Key = $"processed-data/{DateTime.UtcNow:yyyy/MM/dd}/{Guid.NewGuid()}.json";
            await SaveToS3Async(processedS3Key, transformedData);

            return new TransformResult
            {
                Success = true,
                RecordsTransformed = transformedData.Count,
                S3Location = processedS3Key,
                TransformedAt = DateTime.UtcNow
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Transformation failed");
            throw;
        }
    }
}

// Load Lambda Function
public class LoadFunction
{
    private readonly ILogger<LoadFunction> _logger;
    private readonly IDataLoader _dataLoader;

    public async Task<LoadResult> FunctionHandler(LoadRequest request, ILambdaContext context)
    {
        try
        {
            _logger.LogInformation("Starting data load to target: {Target}", request.Target);

            // Leer datos transformados desde S3
            var processedData = await LoadFromS3Async(request.S3Location);

            // Cargar datos al destino
            await _dataLoader.LoadAsync(request.Target, processedData);

            return new LoadResult
            {
                Success = true,
                RecordsLoaded = processedData.Count,
                LoadedAt = DateTime.UtcNow
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Load failed for target: {Target}", request.Target);
            throw;
        }
    }
}
```

## Data Validation y Quality

```csharp
// Data Validation Service
public class DataValidationService : IDataValidationService
{
    private readonly ILogger<DataValidationService> _logger;
    private readonly List<IValidationRule> _validationRules;

    public async Task<ValidationResult> ValidateAsync(List<dynamic> data)
    {
        var result = new ValidationResult
        {
            TotalRecords = data.Count,
            ValidRecords = 0,
            InvalidRecords = new List<ValidationError>()
        };

        foreach (var record in data)
        {
            var recordErrors = new List<string>();

            foreach (var rule in _validationRules)
            {
                if (!await rule.ValidateAsync(record))
                {
                    recordErrors.Add(rule.ErrorMessage);
                }
            }

            if (recordErrors.Any())
            {
                result.InvalidRecords.Add(new ValidationError
                {
                    Record = record,
                    Errors = recordErrors
                });
            }
            else
            {
                result.ValidRecords++;
            }
        }

        result.ValidationRate = (double)result.ValidRecords / result.TotalRecords;

        _logger.LogInformation("Validation completed: {ValidRecords}/{TotalRecords} records valid",
            result.ValidRecords, result.TotalRecords);

        return result;
    }
}

// Validation Rules
public interface IValidationRule
{
    string ErrorMessage { get; }
    Task<bool> ValidateAsync(dynamic record);
}

public class EmailValidationRule : IValidationRule
{
    public string ErrorMessage => "Invalid email format";

    public Task<bool> ValidateAsync(dynamic record)
    {
        if (record.email == null)
            return Task.FromResult(false);

        var email = record.email.ToString();
        var emailRegex = new Regex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$");

        return Task.FromResult(emailRegex.IsMatch(email));
    }
}

public class RequiredFieldValidationRule : IValidationRule
{
    private readonly string _fieldName;

    public RequiredFieldValidationRule(string fieldName)
    {
        _fieldName = fieldName;
        ErrorMessage = $"Field {fieldName} is required";
    }

    public string ErrorMessage { get; }

    public Task<bool> ValidateAsync(dynamic record)
    {
        var value = record[_fieldName];
        return Task.FromResult(value != null && !string.IsNullOrEmpty(value.ToString()));
    }
}
```

## Error Handling y Retry Logic

```csharp
// Retry Policy Service
public class RetryPolicyService
{
    private readonly ILogger<RetryPolicyService> _logger;

    public async Task<T> ExecuteWithRetryAsync<T>(Func<Task<T>> operation, RetryPolicy policy)
    {
        var attempt = 0;
        var lastException = new Exception();

        while (attempt < policy.MaxRetries)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (policy.RetryableExceptions.Any(e => e.IsInstanceOfType(ex)))
            {
                attempt++;
                lastException = ex;

                if (attempt >= policy.MaxRetries)
                {
                    _logger.LogError(ex, "Operation failed after {MaxRetries} attempts", policy.MaxRetries);
                    throw;
                }

                var delay = policy.GetDelay(attempt);
                _logger.LogWarning(ex, "Operation failed, retrying in {Delay}ms (attempt {Attempt}/{MaxRetries})",
                    delay.TotalMilliseconds, attempt, policy.MaxRetries);

                await Task.Delay(delay);
            }
        }

        throw lastException;
    }
}

public class RetryPolicy
{
    public int MaxRetries { get; set; } = 3;
    public List<Type> RetryableExceptions { get; set; } = new();
    public RetryStrategy Strategy { get; set; } = RetryStrategy.ExponentialBackoff;
    public TimeSpan BaseDelay { get; set; } = TimeSpan.FromSeconds(1);

    public TimeSpan GetDelay(int attempt)
    {
        return Strategy switch
        {
            RetryStrategy.Fixed => BaseDelay,
            RetryStrategy.Linear => TimeSpan.FromMilliseconds(BaseDelay.TotalMilliseconds * attempt),
            RetryStrategy.ExponentialBackoff => TimeSpan.FromMilliseconds(BaseDelay.TotalMilliseconds * Math.Pow(2, attempt - 1)),
            _ => BaseDelay
        };
    }
}

public enum RetryStrategy
{
    Fixed,
    Linear,
    ExponentialBackoff
}
```

## Monitoreo y Observabilidad

```csharp
// ETL Monitoring Service
public class ETLMonitoringService
{
    private readonly ILogger<ETLMonitoringService> _logger;
    private readonly IMetricsService _metricsService;
    private readonly IEventBus _eventBus;

    public async Task<ETLJobStatus> StartJobAsync(string jobId, ETLJobRequest request)
    {
        var status = new ETLJobStatus
        {
            JobId = jobId,
            Status = "Running",
            StartTime = DateTime.UtcNow,
            Request = request
        };

        try
        {
            // Registrar métricas
            _metricsService.IncrementCounter("etl_jobs_started", new Dictionary<string, string>
            {
                ["job_type"] = request.JobType,
                ["source"] = request.Source
            });

            // Publicar evento
            await _eventBus.PublishAsync(new ETLJobStartedEvent
            {
                JobId = jobId,
                JobType = request.JobType,
                Source = request.Source,
                StartedAt = DateTime.UtcNow
            });

            return status;
        }
        catch (Exception ex)
        {
            status.Status = "Failed";
            status.Error = ex.Message;
            status.EndTime = DateTime.UtcNow;

            _logger.LogError(ex, "ETL job {JobId} failed to start", jobId);
            throw;
        }
    }

    public async Task CompleteJobAsync(string jobId, ETLJobResult result)
    {
        var status = new ETLJobStatus
        {
            JobId = jobId,
            Status = "Completed",
            EndTime = DateTime.UtcNow,
            Result = result
        };

        // Registrar métricas
        _metricsService.RecordHistogram("etl_job_duration",
            (DateTime.UtcNow - result.StartTime).TotalSeconds,
            new Dictionary<string, string>
            {
                ["job_type"] = result.JobType,
                ["source"] = result.Source
            });

        _metricsService.IncrementCounter("etl_records_processed", result.RecordsProcessed,
            new Dictionary<string, string>
            {
                ["job_type"] = result.JobType,
                ["source"] = result.Source
            });

        // Publicar evento
        await _eventBus.PublishAsync(new ETLJobCompletedEvent
        {
            JobId = jobId,
            JobType = result.JobType,
            RecordsProcessed = result.RecordsProcessed,
            CompletedAt = DateTime.UtcNow
        });
    }
}
```

## Mejores Prácticas

### 1. Diseño de Integración
- Usar APIs RESTful para integraciones síncronas
- Implementar event-driven architecture para integraciones asíncronas
- Mantener contratos de API estables y versionados
- Implementar circuit breakers para resiliencia

### 2. ETL Moderno
- Usar herramientas cloud-native (AWS Glue, Lambda)
- Implementar data validation en cada etapa
- Usar streaming cuando sea posible
- Implementar data lineage y cataloging

### 3. Performance
- Procesar datos en lotes apropiados
- Usar caching para datos frecuentemente accedidos
- Implementar particionamiento de datos
- Optimizar consultas y transformaciones

### 4. Seguridad
- Encriptar datos en tránsito y en reposo
- Implementar autenticación y autorización
- Validar y sanitizar todos los datos
- Auditar todas las operaciones

### 5. Monitoreo
- Implementar métricas de performance
- Monitorear calidad de datos
- Alertar sobre fallos y anomalías
- Mantener logs detallados

## Herramientas Recomendadas

### Integración
- **AWS API Gateway**: API management
- **AWS EventBridge**: Event routing
- **RabbitMQ**: Message queuing
- **Apache Kafka**: Event streaming

### ETL
- **AWS Glue**: Serverless ETL
- **AWS Lambda**: Event-driven processing
- **AWS Step Functions**: Workflow orchestration
- **Apache Airflow**: Complex workflows

### Data Quality
- **Great Expectations**: Data validation
- **AWS Deequ**: Data quality checks
- **Apache Griffin**: Data quality monitoring
- **Custom validators**: Specific business rules

### Monitoreo
- **AWS CloudWatch**: Métricas y logs
- **Prometheus**: Métricas personalizadas
- **Grafana**: Visualización
- **AWS X-Ray**: Distributed tracing

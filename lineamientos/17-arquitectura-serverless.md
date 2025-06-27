# Arquitectura Serverless

## Visión General

La arquitectura serverless permite ejecutar código sin gestionar servidores, pagando solo por el tiempo de ejecución real. En nuestro contexto, nos enfocamos en AWS Lambda, API Gateway, y servicios serverless para construir aplicaciones escalables, mantenibles y costo-efectivas.

## Patrones de Arquitectura Serverless

### 1. Event-Driven Processing
```csharp
// Lambda Function para procesamiento de eventos
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

// Event Models
public class OrderCreatedEvent
{
    public string OrderId { get; set; } = string.Empty;
    public string CustomerId { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class OrderProcessedEvent
{
    public string OrderId { get; set; } = string.Empty;
    public DateTime ProcessedAt { get; set; }
    public string Status { get; set; } = string.Empty;
}
```

### 2. API Gateway + Lambda
```csharp
// API Lambda Function
public class UserApiFunction
{
    private readonly ILogger<UserApiFunction> _logger;
    private readonly IUserService _userService;
    private readonly IValidationService _validationService;

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

    private async Task<APIGatewayProxyResponse> HandleGetRequest(APIGatewayProxyRequest request)
    {
        var userId = ExtractUserId(request.Path);
        if (string.IsNullOrEmpty(userId))
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 400,
                Body = "User ID is required"
            };
        }

        var user = await _userService.GetUserAsync(userId);
        if (user == null)
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 404,
                Body = "User not found"
            };
        }

        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Headers = new Dictionary<string, string>
            {
                ["Content-Type"] = "application/json"
            },
            Body = JsonSerializer.Serialize(user)
        };
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

        // Validar datos
        var validationResult = await _validationService.ValidateAsync(createUserRequest);
        if (!validationResult.IsValid)
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 400,
                Body = JsonSerializer.Serialize(validationResult.Errors)
            };
        }

        var user = await _userService.CreateUserAsync(createUserRequest);

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

    private string? ExtractUserId(string path)
    {
        var segments = path.Split('/');
        return segments.Length > 2 ? segments[2] : null;
    }
}
```

### 3. S3 Event Processing
```csharp
// S3 Event Processing Lambda
public class S3EventProcessingFunction
{
    private readonly ILogger<S3EventProcessingFunction> _logger;
    private readonly IImageProcessingService _imageService;
    private readonly IS3Client _s3Client;

    public async Task FunctionHandler(S3Event s3Event, ILambdaContext context)
    {
        try
        {
            foreach (var record in s3Event.Records)
            {
                if (record.EventName == "ObjectCreated:Put")
                {
                    await ProcessNewFileAsync(record);
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process S3 event");
            throw;
        }
    }

    private async Task ProcessNewFileAsync(S3Event.S3EventNotificationRecord record)
    {
        var bucketName = record.S3.Bucket.Name;
        var objectKey = record.S3.Object.Key;

        _logger.LogInformation("Processing new file: {Bucket}/{Key}", bucketName, objectKey);

        // Verificar si es una imagen
        if (IsImageFile(objectKey))
        {
            await ProcessImageAsync(bucketName, objectKey);
        }
        else if (IsDocumentFile(objectKey))
        {
            await ProcessDocumentAsync(bucketName, objectKey);
        }
    }

    private async Task ProcessImageAsync(string bucketName, string objectKey)
    {
        // Descargar imagen
        var imageStream = await _s3Client.GetObjectAsync(bucketName, objectKey);

        // Procesar imagen (resize, compress, etc.)
        var processedImage = await _imageService.ProcessAsync(imageStream);

        // Subir imagen procesada
        var processedKey = $"processed/{Path.GetFileNameWithoutExtension(objectKey)}_processed.jpg";
        await _s3Client.PutObjectAsync(bucketName, processedKey, processedImage);

        _logger.LogInformation("Image processed and uploaded: {ProcessedKey}", processedKey);
    }

    private bool IsImageFile(string fileName)
    {
        var extension = Path.GetExtension(fileName).ToLower();
        return new[] { ".jpg", ".jpeg", ".png", ".gif", ".bmp" }.Contains(extension);
    }

    private bool IsDocumentFile(string fileName)
    {
        var extension = Path.GetExtension(fileName).ToLower();
        return new[] { ".pdf", ".doc", ".docx", ".txt" }.Contains(extension);
    }
}
```

### 4. DynamoDB Stream Processing
```csharp
// DynamoDB Stream Processing Lambda
public class DynamoDBStreamProcessingFunction
{
    private readonly ILogger<DynamoDBStreamProcessingFunction> _logger;
    private readonly INotificationService _notificationService;
    private readonly IAnalyticsService _analyticsService;

    public async Task FunctionHandler(DynamoDBEvent dynamoDbEvent, ILambdaContext context)
    {
        try
        {
            foreach (var record in dynamoDbEvent.Records)
            {
                await ProcessRecordAsync(record);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process DynamoDB stream");
            throw;
        }
    }

    private async Task ProcessRecordAsync(DynamoDBEvent.DynamodbStreamRecord record)
    {
        var eventName = record.EventName;
        var tableName = record.EventSourceArn.Split('/').Last();

        _logger.LogInformation("Processing {EventName} event for table {TableName}",
            eventName, tableName);

        switch (eventName)
        {
            case "INSERT":
                await HandleInsertAsync(record);
                break;
            case "MODIFY":
                await HandleModifyAsync(record);
                break;
            case "REMOVE":
                await HandleRemoveAsync(record);
                break;
        }
    }

    private async Task HandleInsertAsync(DynamoDBEvent.DynamodbStreamRecord record)
    {
        var newImage = record.Dynamodb.NewImage;

        if (newImage.ContainsKey("userId"))
        {
            var userId = newImage["userId"].S;

            // Enviar notificación de bienvenida
            await _notificationService.SendWelcomeEmailAsync(userId);

            // Registrar evento de analytics
            await _analyticsService.TrackUserRegistrationAsync(userId);
        }
    }

    private async Task HandleModifyAsync(DynamoDBEvent.DynamodbStreamRecord record)
    {
        var oldImage = record.Dynamodb.OldImage;
        var newImage = record.Dynamodb.NewImage;

        // Detectar cambios específicos
        if (oldImage.ContainsKey("status") && newImage.ContainsKey("status"))
        {
            var oldStatus = oldImage["status"].S;
            var newStatus = newImage["status"].S;

            if (oldStatus != newStatus)
            {
                var userId = newImage["userId"].S;
                await _notificationService.SendStatusChangeNotificationAsync(userId, oldStatus, newStatus);
            }
        }
    }

    private async Task HandleRemoveAsync(DynamoDBEvent.DynamodbStreamRecord record)
    {
        var oldImage = record.Dynamodb.OldImage;

        if (oldImage.ContainsKey("userId"))
        {
            var userId = oldImage["userId"].S;

            // Registrar evento de eliminación
            await _analyticsService.TrackUserDeletionAsync(userId);
        }
    }
}
```

## Optimización de Performance

### 1. Cold Start Optimization
```csharp
// Optimized Lambda Function
public class OptimizedFunction
{
    // Variables estáticas para reutilizar entre invocaciones
    private static readonly HttpClient _httpClient;
    private static readonly IConfiguration _configuration;
    private static readonly ILogger<OptimizedFunction> _logger;

    static OptimizedFunction()
    {
        // Inicializar recursos compartidos
        _httpClient = new HttpClient();
        _configuration = new ConfigurationBuilder()
            .AddEnvironmentVariables()
            .AddSystemsManager("/myapp/")
            .Build();

        var loggerFactory = LoggerFactory.Create(builder =>
            builder.AddConsole().SetMinimumLevel(LogLevel.Information));
        _logger = loggerFactory.CreateLogger<OptimizedFunction>();
    }

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
    {
        // Usar recursos ya inicializados
        var result = await ProcessRequestAsync(request);
        return result;
    }

    private async Task<APIGatewayProxyResponse> ProcessRequestAsync(APIGatewayProxyRequest request)
    {
        // Lógica de procesamiento
        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Body = "Success"
        };
    }
}
```

### 2. Connection Pooling
```csharp
// Database Connection Pool
public class DatabaseConnectionPool
{
    private static readonly SemaphoreSlim _semaphore = new SemaphoreSlim(10, 10); // Max 10 connections
    private static readonly List<IDbConnection> _connections = new();
    private static readonly object _lock = new object();

    public static async Task<IDbConnection> GetConnectionAsync()
    {
        await _semaphore.WaitAsync();

        lock (_lock)
        {
            if (_connections.Count > 0)
            {
                var connection = _connections[0];
                _connections.RemoveAt(0);
                return connection;
            }
        }

        // Crear nueva conexión si no hay disponibles
        return CreateNewConnection();
    }

    public static void ReturnConnection(IDbConnection connection)
    {
        if (connection.State == ConnectionState.Open)
        {
            lock (_lock)
            {
                if (_connections.Count < 10)
                {
                    _connections.Add(connection);
                }
                else
                {
                    connection.Dispose();
                }
            }
        }

        _semaphore.Release();
    }

    private static IDbConnection CreateNewConnection()
    {
        var connectionString = Environment.GetEnvironmentVariable("DATABASE_CONNECTION_STRING");
        var connection = new NpgsqlConnection(connectionString);
        connection.Open();
        return connection;
    }
}
```

## Gestión de Configuración y Secretos

```csharp
// Configuration Service
public class ConfigurationService
{
    private readonly IAmazonSimpleSystemsManagement _ssmClient;
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly ILogger<ConfigurationService> _logger;
    private readonly Dictionary<string, string> _cache = new();
    private readonly SemaphoreSlim _cacheLock = new SemaphoreSlim(1, 1);

    public async Task<string> GetParameterAsync(string parameterName)
    {
        // Verificar cache primero
        if (_cache.TryGetValue(parameterName, out var cachedValue))
        {
            return cachedValue;
        }

        await _cacheLock.WaitAsync();
        try
        {
            // Verificar cache nuevamente después del lock
            if (_cache.TryGetValue(parameterName, out cachedValue))
            {
                return cachedValue;
            }

            var request = new GetParameterRequest
            {
                Name = parameterName,
                WithDecryption = true
            };

            var response = await _ssmClient.GetParameterAsync(request);
            var value = response.Parameter.Value;

            // Cache por 5 minutos
            _cache[parameterName] = value;

            return value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get parameter {ParameterName}", parameterName);
            throw;
        }
        finally
        {
            _cacheLock.Release();
        }
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            var request = new GetSecretValueRequest
            {
                SecretId = secretName
            };

            var response = await _secretsManager.GetSecretValueAsync(request);
            return response.SecretString;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get secret {SecretName}", secretName);
            throw;
        }
    }
}
```

## Monitoreo y Observabilidad

```csharp
// Lambda Monitoring
public class LambdaMonitoringService
{
    private readonly ILogger<LambdaMonitoringService> _logger;
    private readonly IMetricsService _metricsService;

    public async Task TrackExecutionAsync(string functionName, Func<Task> execution)
    {
        var startTime = DateTime.UtcNow;
        var success = false;

        try
        {
            await execution();
            success = true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Lambda execution failed: {FunctionName}", functionName);
            throw;
        }
        finally
        {
            var duration = DateTime.UtcNow - startTime;

            // Registrar métricas
            _metricsService.RecordHistogram("lambda_duration", duration.TotalMilliseconds,
                new Dictionary<string, string> { ["function"] = functionName });

            _metricsService.IncrementCounter("lambda_invocations",
                new Dictionary<string, string>
                {
                    ["function"] = functionName,
                    ["success"] = success.ToString()
                });
        }
    }

    public async Task<T> TrackExecutionAsync<T>(string functionName, Func<Task<T>> execution)
    {
        var startTime = DateTime.UtcNow;
        var success = false;
        T result = default!;

        try
        {
            result = await execution();
            success = true;
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Lambda execution failed: {FunctionName}", functionName);
            throw;
        }
        finally
        {
            var duration = DateTime.UtcNow - startTime;

            _metricsService.RecordHistogram("lambda_duration", duration.TotalMilliseconds,
                new Dictionary<string, string> { ["function"] = functionName });

            _metricsService.IncrementCounter("lambda_invocations",
                new Dictionary<string, string>
                {
                    ["function"] = functionName,
                    ["success"] = success.ToString()
                });
        }
    }
}
```

## Serverless Framework Configuration

```yaml
# serverless.yml
service: my-serverless-app

provider:
  name: aws
  runtime: dotnet8
  region: us-east-1
  memorySize: 512
  timeout: 30
  environment:
    STAGE: ${opt:stage, 'dev'}
    REGION: ${self:provider.region}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - s3:GetObject
        - s3:PutObject
        - s3:DeleteObject
      Resource: "arn:aws:s3:::my-bucket/*"
    - Effect: Allow
      Action:
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
        - dynamodb:Query
        - dynamodb:Scan
      Resource: "arn:aws:dynamodb:${self:provider.region}:*:table/my-table"
    - Effect: Allow
      Action:
        - ssm:GetParameter
        - ssm:GetParameters
      Resource: "arn:aws:ssm:${self:provider.region}:*:parameter/myapp/*"
    - Effect: Allow
      Action:
        - secretsmanager:GetSecretValue
      Resource: "arn:aws:secretsmanager:${self:provider.region}:*:secret:myapp/*"

functions:
  userApi:
    handler: MyApp::UserApiFunction::FunctionHandler
    events:
      - http:
          path: users/{proxy+}
          method: ANY
          cors: true
    environment:
      TABLE_NAME: users-${self:provider.stage}
    memorySize: 256
    timeout: 10

  orderProcessor:
    handler: MyApp::OrderProcessingFunction::FunctionHandler
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10
    environment:
      TABLE_NAME: orders-${self:provider.stage}
    memorySize: 1024
    timeout: 60

  imageProcessor:
    handler: MyApp::S3EventProcessingFunction::FunctionHandler
    events:
      - s3:
          bucket: my-images-bucket
          event: s3:ObjectCreated:*
          rules:
            - suffix: .jpg
            - suffix: .png
    environment:
      PROCESSED_BUCKET: my-processed-images-bucket
    memorySize: 2048
    timeout: 300

  dynamoDbProcessor:
    handler: MyApp::DynamoDBStreamProcessingFunction::FunctionHandler
    events:
      - stream:
          type: dynamodb
          arn: !GetAtt UsersTable.StreamArn
          batchSize: 100
          startingPosition: LATEST
    environment:
      NOTIFICATION_TOPIC: !Ref NotificationTopic
    memorySize: 512
    timeout: 30

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: users-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        StreamSpecification:
          StreamEnabled: true
          StreamViewType: NEW_AND_OLD_IMAGES

    OrderQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: orders-${self:provider.stage}
        VisibilityTimeoutSeconds: 60
        MessageRetentionPeriod: 1209600

    NotificationTopic:
      Type: AWS::SNS::Topic
      Properties:
        TopicName: notifications-${self:provider.stage}

    NotificationSubscription:
      Type: AWS::SNS::Subscription
      Properties:
        TopicArn: !Ref NotificationTopic
        Protocol: email
        Endpoint: admin@company.com
```

## Mejores Prácticas

### 1. Diseño de Funciones
- Mantener funciones pequeñas y especializadas
- Usar single responsibility principle
- Implementar idempotencia
- Manejar errores apropiadamente

### 2. Performance
- Optimizar cold starts
- Usar connection pooling
- Implementar caching
- Minimizar dependencias

### 3. Seguridad
- Usar IAM roles mínimos
- Encriptar datos sensibles
- Validar inputs
- Implementar rate limiting

### 4. Monitoreo
- Implementar métricas personalizadas
- Configurar alertas
- Usar distributed tracing
- Monitorear costos

### 5. Testing
- Implementar tests unitarios
- Usar tests de integración
- Simular eventos AWS
- Implementar chaos testing

## Casos de Uso Recomendados

### Apropiados para Serverless
- APIs RESTful
- Procesamiento de eventos
- ETL y data processing
- Notificaciones
- Webhooks
- Cron jobs
- Image/video processing

### No Apropiados para Serverless
- Aplicaciones con estado
- Procesamiento CPU intensivo
- Aplicaciones con latencia crítica
- Sistemas legacy complejos
- Aplicaciones con dependencias pesadas

## Herramientas Recomendadas

### Desarrollo
- **AWS SAM**: Serverless Application Model
- **Serverless Framework**: Multi-cloud
- **Terraform**: Infrastructure as Code
- **AWS CDK**: TypeScript/JavaScript

### Testing
- **AWS SAM Local**: Local testing
- **Moto**: AWS mocking
- **LocalStack**: Local AWS stack
- **Jest**: Unit testing

### Monitoreo
- **AWS CloudWatch**: Métricas y logs
- **AWS X-Ray**: Distributed tracing
- **Datadog**: APM y métricas
- **New Relic**: Performance monitoring

### CI/CD
- **AWS CodePipeline**: Pipeline automation
- **GitHub Actions**: CI/CD workflows
- **Jenkins**: Build automation
- **AWS CodeBuild**: Build service

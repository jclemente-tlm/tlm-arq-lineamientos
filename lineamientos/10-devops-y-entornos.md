# DevOps y Entornos

## Visión General

DevOps es una práctica que combina desarrollo de software (Dev) y operaciones de TI (Ops) para acortar el ciclo de vida del desarrollo de sistemas y proporcionar entrega continua de valor. En nuestro contexto, DevOps abarca automatización, infraestructura como código, monitoreo continuo y cultura de colaboración.

## Principios Fundamentales

### 1. Automatización Completa
- **CI/CD Pipelines**: Automatizar construcción, pruebas y despliegue de todos los artefactos
- **Infraestructura como Código (IaC)**: Gestionar infraestructura mediante código versionado
- **Testing Automatizado**: Integrar pruebas unitarias, de integración y E2E en el pipeline
- **Despliegues Automatizados**: Eliminar intervención manual en despliegues

### 2. Entornos Consistentes
- **Reproducibilidad**: Todos los entornos deben ser idénticos en configuración
- **Versionado**: Control de versiones para configuraciones y dependencias
- **Aislamiento**: Separación clara entre entornos de desarrollo, testing y producción
- **Gestión de Configuración**: Centralizar y versionar configuraciones

### 3. Seguridad Integrada
- **Shift Left Security**: Integrar seguridad desde el inicio del desarrollo
- **Análisis Estático**: Validación automática de código y configuración
- **Gestión de Secretos**: Manejo seguro de credenciales y configuraciones sensibles
- **Compliance Automatizado**: Verificación continua de cumplimiento normativo

## Stack Tecnológico DevOps

### Herramientas Principales

#### CI/CD Pipelines
```yaml
# Ejemplo de Azure DevOps Pipeline para .NET 8
trigger:
  branches:
    include:
    - main
    - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  dotNetVersion: '8.0.x'

stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    - task: UseDotNet@2
      inputs:
        version: $(dotNetVersion)

    - task: DotNetCoreCLI@2
      inputs:
        command: 'restore'
        projects: '$(solution)'

    - task: DotNetCoreCLI@2
      inputs:
        command: 'build'
        projects: '$(solution)'
        arguments: '--configuration $(buildConfiguration)'

    - task: DotNetCoreCLI@2
      inputs:
        command: 'test'
        projects: '**/*Tests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'

    - task: SonarQubePrepare@4
      inputs:
        SonarQube: 'SonarQube'
        scannerMode: 'MSBuild'
        projectKey: '$(Build.Repository.Name)'
        projectName: '$(Build.Repository.Name)'

    - task: SonarQubeAnalyze@4

    - task: SonarQubePublish@4
      inputs:
        pollingTimeoutSec: '300'

    - task: PublishBuildArtifacts@1
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: 'drop'
```

#### Infraestructura como Código (Terraform)
```hcl
# Ejemplo de configuración Terraform para AWS
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC para microservicios
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    Project     = var.project_name
  }
}

# Subnets privadas para microservicios
resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-private-subnet-${count.index + 1}"
    Environment = var.environment
    Type        = "Private"
  }
}

# ECS Cluster para microservicios
resource "aws_ecs_cluster" "main" {
  name = "${var.environment}-${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "postgresql" {
  identifier = "${var.environment}-${var.project_name}-db"

  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.db_instance_class

  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_max_allocated_storage
  storage_encrypted     = true

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  deletion_protection = var.environment == "prod" ? true : false

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

#### Gestión de Configuración (AWS Systems Manager Parameter Store)
```csharp
// Ejemplo de configuración en .NET 8
public class ConfigurationService
{
    private readonly IAmazonSimpleSystemsManagement _ssmClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<ConfigurationService> _logger;

    public ConfigurationService(
        IAmazonSimpleSystemsManagement ssmClient,
        IConfiguration configuration,
        ILogger<ConfigurationService> logger)
    {
        _ssmClient = ssmClient;
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<string> GetParameterAsync(string parameterName)
    {
        try
        {
            var request = new GetParameterRequest
            {
                Name = parameterName,
                WithDecryption = true
            };

            var response = await _ssmClient.GetParameterAsync(request);
            return response.Parameter.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving parameter {ParameterName}", parameterName);
            throw;
        }
    }

    public async Task<Dictionary<string, string>> GetParametersByPathAsync(string path)
    {
        var parameters = new Dictionary<string, string>();
        string nextToken = null;

        do
        {
            var request = new GetParametersByPathRequest
            {
                Path = path,
                Recursive = true,
                WithDecryption = true,
                NextToken = nextToken
            };

            var response = await _ssmClient.GetParametersByPathAsync(request);

            foreach (var parameter in response.Parameters)
            {
                parameters[parameter.Name] = parameter.Value;
            }

            nextToken = response.NextToken;
        } while (!string.IsNullOrEmpty(nextToken));

        return parameters;
    }
}
```

### Herramientas de Análisis y Calidad

#### SonarQube Integration
```yaml
# SonarQube Quality Gate en Pipeline
- task: SonarQubePublish@4
  inputs:
    pollingTimeoutSec: '300'

- task: SonarQubeQualityGate@4
  inputs:
    pollingTimeoutSec: '300'
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

#### Checkov para IaC Security
```yaml
# Checkov en Pipeline
- script: |
    pip install checkov
    checkov -d . --framework terraform --output cli --output junitxml --output-file-path ./checkov-results
  displayName: 'Run Checkov Security Scan'
  condition: succeededOrFailed()

- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'JUnit'
    testResultsFiles: '**/checkov-results.xml'
    mergeTestResults: true
    testRunTitle: 'Checkov Security Scan'
  condition: succeededOrFailed()
```

## Estrategia de Entornos

### Estructura de Entornos

#### 1. Desarrollo (Development)
- **Propósito**: Desarrollo activo y pruebas unitarias
- **Características**:
  - Despliegues automáticos desde ramas feature
  - Datos sintéticos o copias de producción anonimizadas
  - Configuraciones mínimas de recursos
  - Acceso amplio para desarrolladores

#### 2. Integración (Integration)
- **Propósito**: Pruebas de integración y validación de features
- **Características**:
  - Despliegues automáticos desde ramas develop
  - Datos de prueba estructurados
  - Configuraciones similares a staging
  - Acceso controlado para QA y desarrolladores

#### 3. Staging/Pre-producción
- **Propósito**: Validación final antes de producción
- **Características**:
  - Despliegues automáticos desde ramas release
  - Datos de producción anonimizados
  - Configuraciones idénticas a producción
  - Acceso restringido para stakeholders

#### 4. Producción
- **Propósito**: Ambiente de usuarios finales
- **Características**:
  - Despliegues controlados desde main
  - Datos reales de usuarios
  - Configuraciones optimizadas para rendimiento
  - Acceso altamente restringido

### Gestión de Configuración por Entorno

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "Database": {
    "ConnectionString": "Host=localhost;Database=dev_db;Username=dev_user;Password=dev_pass",
    "EnableDetailedErrors": true,
    "EnableSensitiveDataLogging": true
  },
  "AWS": {
    "Region": "us-east-1",
    "UseLocalStack": true
  },
  "FeatureFlags": {
    "EnableNewUI": true,
    "EnableBetaFeatures": true
  }
}
```

```json
// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "Database": {
    "ConnectionString": "{{DB_CONNECTION_STRING}}",
    "EnableDetailedErrors": false,
    "EnableSensitiveDataLogging": false
  },
  "AWS": {
    "Region": "{{AWS_REGION}}",
    "UseLocalStack": false
  },
  "FeatureFlags": {
    "EnableNewUI": false,
    "EnableBetaFeatures": false
  }
}
```

## Estrategia de Despliegue

### Blue-Green Deployment
```yaml
# Ejemplo de Blue-Green Deployment en AWS ECS
- task: AWSEBDeployment@1
  inputs:
    awsCredentials: 'AWS-Credentials'
    regionName: 'us-east-1'
    applicationName: '$(ApplicationName)'
    environmentName: '$(EnvironmentName)'
    deploymentType: 'blue-green'
    deploymentOption: 'with-traffic-control'
    deploymentTimeout: 'PT1H'
    rollbackEnabled: 'true'
```

### Canary Deployment
```yaml
# Canary Deployment con AWS CodeDeploy
- task: AWSCodeDeployDeployment@1
  inputs:
    awsCredentials: 'AWS-Credentials'
    regionName: 'us-east-1'
    applicationName: '$(ApplicationName)'
    deploymentGroupName: '$(DeploymentGroupName)'
    deploymentType: 'in-place'
    deploymentRevisionSource: 's3'
    s3Bucket: '$(S3Bucket)'
    s3Key: '$(S3Key)'
    deploymentWaitTime: 'PT30M'
```

### Rollback Strategy
```csharp
// Implementación de Rollback en .NET
public class DeploymentService
{
    private readonly ILogger<DeploymentService> _logger;
    private readonly IConfiguration _configuration;

    public async Task<bool> DeployAsync(string version, string environment)
    {
        try
        {
            _logger.LogInformation("Iniciando despliegue de versión {Version} en {Environment}", version, environment);

            // 1. Backup de configuración actual
            await CreateBackupAsync(environment);

            // 2. Despliegue de nueva versión
            var deploymentResult = await PerformDeploymentAsync(version, environment);

            // 3. Health checks
            if (!await PerformHealthChecksAsync(environment))
            {
                _logger.LogWarning("Health checks fallaron, iniciando rollback");
                await RollbackAsync(environment);
                return false;
            }

            // 4. Smoke tests
            if (!await PerformSmokeTestsAsync(environment))
            {
                _logger.LogWarning("Smoke tests fallaron, iniciando rollback");
                await RollbackAsync(environment);
                return false;
            }

            _logger.LogInformation("Despliegue exitoso de versión {Version}", version);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error durante el despliegue");
            await RollbackAsync(environment);
            return false;
        }
    }

    private async Task RollbackAsync(string environment)
    {
        _logger.LogInformation("Iniciando rollback en {Environment}", environment);

        // Implementar lógica de rollback específica
        // - Restaurar configuración anterior
        // - Revertir cambios de base de datos
        // - Notificar a stakeholders
    }
}
```

## Monitoreo y Observabilidad

### Health Checks
```csharp
// Health Checks en .NET 8
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHealthChecks()
            .AddCheck<DatabaseHealthCheck>("database")
            .AddCheck<ExternalServiceHealthCheck>("external-service")
            .AddCheck<RedisHealthCheck>("redis")
            .AddCheck<DiskSpaceHealthCheck>("disk-space");
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseHealthChecks("/health", new HealthCheckOptions
        {
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
            ResultStatusCodes =
            {
                [HealthStatus.Healthy] = StatusCodes.Status200OK,
                [HealthStatus.Degraded] = StatusCodes.Status200OK,
                [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
            }
        });
    }
}
```

### Logging Centralizado
```csharp
// Configuración de Serilog para AWS CloudWatch
public class Program
{
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog((context, services, configuration) => configuration
                .ReadFrom.Configuration(context.Configuration)
                .ReadFrom.Services(services)
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.AmazonCloudWatch(
                    logGroup: "/aws/ecs/myapp",
                    region: "us-east-1",
                    logStreamNameProvider: new DefaultLogStreamProvider(),
                    textFormatter: new JsonFormatter()
                ))
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

## Seguridad en DevOps

### Gestión de Secretos
```csharp
// Integración con AWS Secrets Manager
public class SecretsManagerService
{
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly ILogger<SecretsManagerService> _logger;

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
            _logger.LogError(ex, "Error retrieving secret {SecretName}", secretName);
            throw;
        }
    }
}
```

### Container Security
```dockerfile
# Dockerfile seguro para .NET 8
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Usuario no-root
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser

FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["MyApp.csproj", "./"]
RUN dotnet restore "MyApp.csproj"
COPY . .
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
USER appuser
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## Métricas y KPIs

### Métricas de Pipeline
- **Build Success Rate**: > 95%
- **Deployment Frequency**: Múltiples veces por día
- **Lead Time**: < 2 horas desde commit hasta producción
- **Mean Time to Recovery (MTTR)**: < 1 hora
- **Change Failure Rate**: < 5%

### Métricas de Calidad
- **Code Coverage**: > 80%
- **SonarQube Quality Gate**: 100% pass rate
- **Security Vulnerabilities**: 0 críticas/altas
- **Technical Debt**: < 5% del código base

## Mejores Prácticas

### 1. Automatización Completa
- Automatizar todos los procesos manuales
- Implementar gates automáticos en el pipeline
- Usar feature flags para control de despliegues

### 2. Seguridad Integrada
- Escanear dependencias regularmente
- Validar configuración de infraestructura
- Implementar secret scanning en el pipeline

### 3. Monitoreo Continuo
- Implementar alertas proactivas
- Monitorear métricas de negocio
- Usar distributed tracing

### 4. Cultura DevOps
- Promover colaboración entre Dev y Ops
- Implementar blameless post-mortems
- Compartir responsabilidades de operación

### 5. Documentación
- Mantener runbooks actualizados
- Documentar procedimientos de rollback
- Crear diagramas de arquitectura actualizados

## Herramientas Recomendadas

### CI/CD
- **Azure DevOps**: Para proyectos .NET
- **GitHub Actions**: Para proyectos open source
- **Jenkins**: Para integraciones complejas
- **GitLab CI**: Para proyectos GitLab

### Infraestructura
- **Terraform**: Para multi-cloud
- **AWS CDK**: Para proyectos AWS nativos
- **Pulumi**: Para programación en lenguajes generales
- **Ansible**: Para configuración de servidores

### Monitoreo
- **AWS CloudWatch**: Para servicios AWS
- **Prometheus + Grafana**: Para métricas personalizadas
- **ELK Stack**: Para logs centralizados
- **Jaeger**: Para distributed tracing

### Seguridad
- **SonarQube**: Análisis de código
- **Checkov**: Validación de IaC
- **Snyk**: Vulnerabilidades de dependencias
- **Trivy**: Vulnerabilidades de containers

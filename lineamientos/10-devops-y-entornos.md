# DevOps y Entornos

## Propósito

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

##### GitHub Actions
```yaml
# Ejemplo de GitHub Actions para .NET 8
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOTNET_VERSION: '8.0.x'
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Necesario para SonarQube

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore --configuration Release

    - name: Run tests
      run: dotnet test --no-build --verbosity normal --configuration Release --collect:"XPlat Code Coverage"

    - name: SonarQube Analysis
      uses: sonarqube-quality-gate-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ env.SONAR_TOKEN }}
      with:
        scannerHome: ${{ github.workspace }}/sonar-scanner
        args: >
          -Dsonar.projectKey=my-project
          -Dsonar.sources=.
          -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }}
          -Dsonar.login=${{ env.SONAR_TOKEN }}

    - name: Checkov Security Scan
      uses: bridgecrewio/checkov-action@master
      with:
        directory: .
        framework: terraform,kubernetes,dockerfile
        output_format: sarif
        output_file_path: checkov-results.sarif

    - name: Upload Checkov results
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: checkov-results.sarif

  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-test

    steps:
    - uses: actions/checkout@v4

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'my-app:latest'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: trivy-results.sarif

  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-app
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

##### Jenkins
```groovy
// Ejemplo de Jenkinsfile para .NET 8
pipeline {
    agent any

    environment {
        DOTNET_VERSION = '8.0.x'
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup .NET') {
            steps {
                sh 'dotnet --version'
                sh 'dotnet restore'
            }
        }

        stage('Build') {
            steps {
                sh 'dotnet build --configuration Release --no-restore'
            }
        }

        stage('Test') {
            steps {
                sh 'dotnet test --configuration Release --no-build --collect:"XPlat Code Coverage"'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        dotnet sonarscanner begin /k:"my-project" /d:sonar.host.url="http://sonarqube:9000" /d:sonar.login="${SONAR_TOKEN}"
                        dotnet build --configuration Release
                        dotnet sonarscanner end /d:sonar.login="${SONAR_TOKEN}"
                    '''
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh 'checkov -d . --output sarif --output-file-path checkov-results.sarif'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    docker.build("my-app:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://your-registry.com', 'registry-credentials') {
                        docker.image("my-app:${env.BUILD_NUMBER}").push()
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

#### Contenedores y Orquestación

##### Docker
```dockerfile
# Ejemplo de Dockerfile para .NET 8
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "./"]
RUN dotnet restore "MyApp.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```yaml
# Ejemplo de docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=myapp;User=sa;Password=Your_password123;
    depends_on:
      - db
    networks:
      - app-network

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: sa
      POSTGRES_PASSWORD: Your_password123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  sonarqube:
    image: sonarqube:latest
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonar
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    depends_on:
      - db
    networks:
      - app-network

volumes:
  postgres_data:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:

networks:
  app-network:
    driver: bridge
```

#### Análisis de Calidad y Seguridad

##### SonarQube
```xml
<!-- Ejemplo de sonar-project.properties -->
sonar.projectKey=my-project
sonar.projectName=My Application
sonar.projectVersion=1.0

sonar.sources=.
sonar.exclusions=**/bin/**,**/obj/**,**/wwwroot/**,**/node_modules/**

sonar.tests=.
sonar.test.inclusions=**/*Tests/**/*.cs,**/*Test/**/*.cs

sonar.cs.opencover.reportsPaths=**/coverage.opencover.xml
sonar.coverage.exclusions=**/*Tests/**/*,**/*Test/**/*,**/Program.cs,**/Startup.cs

sonar.cpd.cs.minimumTokens=100
sonar.cpd.exclusions=**/*Tests/**/*,**/*Test/**/*

# Configuración de reglas específicas para .NET
sonar.cs.roslyn.ignoreIssues=false
sonar.cs.roslyn.reportIssuesOnly=false
```

##### Checkov
```yaml
# Ejemplo de configuración Checkov para infraestructura
# .checkov.yaml
framework:
  - terraform
  - kubernetes
  - dockerfile
  - helm

output: sarif
output-file-path: checkov-results.sarif

skip-path:
  - .terraform
  - node_modules

skip-check:
  - CKV_AWS_130  # VPC Flow Logs
  - CKV_AWS_126  # VPC Default Security Group

compact: true
directory:
  - .
  - infrastructure
  - k8s
  - docker
```

```hcl
# Ejemplo de Terraform con mejores prácticas de seguridad
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# VPC con logging habilitado
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# Flow logs para auditoría
resource "aws_flow_log" "main" {
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
}

# Security group con reglas restrictivas
resource "aws_security_group" "app" {
  name_prefix = "${var.environment}-app-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-app-sg"
    Environment = var.environment
  }
}
```

#### Monitoreo y Observabilidad

##### Prometheus y Grafana
```yaml
# Ejemplo de configuración Prometheus
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'dotnet-app'
    static_configs:
      - targets: ['localhost:5000']
    metrics_path: '/metrics'
    scrape_interval: 5s

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - localhost:9093
```

```csharp
// Ejemplo de métricas personalizadas en .NET
public class MetricsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly Counter _requestCounter;
    private readonly Histogram _requestDuration;

    public MetricsMiddleware(RequestDelegate next)
    {
        _next = next;
        _requestCounter = Metrics.CreateCounter("http_requests_total", "Total HTTP requests", new CounterConfiguration
        {
            LabelNames = new[] { "method", "endpoint", "status" }
        });
        _requestDuration = Metrics.CreateHistogram("http_request_duration_seconds", "HTTP request duration", new HistogramConfiguration
        {
            LabelNames = new[] { "method", "endpoint" }
        });
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            sw.Stop();

            _requestCounter.WithLabels(
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode.ToString()
            ).Inc();

            _requestDuration.WithLabels(
                context.Request.Method,
                context.Request.Path
            ).Observe(sw.Elapsed.TotalSeconds);
        }
    }
}
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

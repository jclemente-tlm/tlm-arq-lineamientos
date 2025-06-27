# Disaster Recovery

## Propósito

Este lineamiento establece las estrategias y procedimientos de Disaster Recovery (DR) para garantizar la continuidad del negocio ante fallos catastróficos, desastres naturales o incidentes de seguridad. El objetivo es minimizar el tiempo de inactividad y la pérdida de datos.

## Principios Fundamentales

### RTO y RPO
- **RTO (Recovery Time Objective)**: Tiempo máximo aceptable para restaurar servicios
- **RPO (Recovery Point Objective)**: Pérdida máxima de datos aceptable
- **Definición por servicio**: Cada servicio debe tener sus propios objetivos
- **Validación regular**: Probar objetivos mediante simulacros

### Redundancia Geográfica
- **Multi-región**: Distribución de servicios en múltiples regiones
- **Multi-AZ**: Uso de múltiples zonas de disponibilidad
- **Active-Active**: Servicios activos en múltiples ubicaciones
- **Active-Passive**: Configuración de respaldo en standby

### Automatización
- **Recuperación automática**: Automatizar procesos de recuperación
- **Failover automático**: Cambio automático a sistemas de respaldo
- **Backup automatizado**: Backups programados y verificados
- **Testing automatizado**: Validación automática de procedimientos

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
    business_impact: "Payment processing and transactions"

  reporting_service:
    criticality: "Medium"
    rto: "24 hours"
    rpo: "1 hour"
    business_impact: "Business intelligence and reporting"
```

### 2. Procedimientos de Recuperación

#### Runbook de Recuperación
```bash
#!/bin/bash
# Runbook: Recuperación de base de datos PostgreSQL

echo "Starting database recovery procedure..."

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

# 6. Verificar salud
echo "Checking application health..."
curl -f http://localhost/health || exit 1

echo "Recovery completed successfully"
```

### 3. Testing y Validación

#### Plan de Testing
```yaml
# Ejemplo: Plan de testing de DR
testing_schedule:
  monthly:
    - "Full DR simulation"
    - "Database recovery test"
    - "Application failover test"

  quarterly:
    - "Multi-region failover"
    - "Data integrity validation"
    - "Performance testing post-recovery"

  annually:
    - "Complete DR exercise"
    - "Team training and validation"
    - "Documentation review and update"
```

#### Métricas de Testing
- **Recovery Time**: Tiempo real vs RTO objetivo
- **Data Loss**: Pérdida real vs RPO objetivo
- **Service Availability**: Disponibilidad post-recuperación
- **Performance Impact**: Impacto en rendimiento

## Monitoreo y Alerting

### Métricas de Disaster Recovery
```yaml
# Ejemplo: Métricas con Prometheus
metrics:
  backup_status:
    - "backup_last_successful_timestamp"
    - "backup_duration_seconds"
    - "backup_size_bytes"

  replication_lag:
    - "replication_lag_seconds"
    - "replication_status"
    - "replication_errors_total"

  recovery_readiness:
    - "dr_site_health"
    - "dr_site_sync_status"
    - "dr_site_performance"
```

### Alertas Críticas
- **Backup Failure**: Fallo en backup por más de 24 horas
- **Replication Lag**: Lag de replicación > 5 minutos
- **DR Site Unavailable**: Sitio de DR no disponible
- **Data Corruption**: Detección de corrupción de datos

## Automatización de DR

### Infrastructure as Code
```terraform
# Ejemplo: Configuración de DR con Terraform
resource "aws_rds_cluster" "primary" {
  cluster_identifier = "primary-cluster"
  engine             = "aurora-postgresql"
  availability_zones = ["us-east-1a", "us-east-1b"]

  backup_retention_period = 30
  copy_tags_to_snapshot  = true

  tags = {
    Environment = "production"
    DR_Strategy = "multi-region"
  }
}

resource "aws_rds_cluster" "dr" {
  cluster_identifier = "dr-cluster"
  engine             = "aurora-postgresql"
  availability_zones = ["us-west-2a", "us-west-2b"]

  backup_retention_period = 30
  copy_tags_to_snapshot  = true

  tags = {
    Environment = "dr"
    DR_Strategy = "multi-region"
  }
}
```

### Automatización de Failover
```csharp
// Ejemplo: Script de failover automático
using Amazon.Route53;
using Amazon.Route53.Model;
using Amazon.EC2;
using Amazon.EC2.Model;

public class DisasterRecoveryService
{
    private readonly IAmazonRoute53 _route53Client;
    private readonly IAmazonEC2 _ec2Client;
    private readonly ILogger<DisasterRecoveryService> _logger;

    public DisasterRecoveryService(
        IAmazonRoute53 route53Client,
        IAmazonEC2 ec2Client,
        ILogger<DisasterRecoveryService> logger)
    {
        _route53Client = route53Client;
        _ec2Client = ec2Client;
        _logger = logger;
    }

    public async Task ExecuteFailoverAsync()
    {
        try
        {
            _logger.LogInformation("Starting automatic failover procedure");

            // 1. Verificar salud del sitio primario
            if (!await CheckPrimaryHealthAsync())
            {
                _logger.LogWarning("Primary site is unhealthy, initiating failover");

                // 2. Activar región secundaria
                await ActivateSecondaryRegionAsync();

                // 3. Actualizar registros DNS
                await UpdateDnsRecordsAsync();

                // 4. Verificar salud de servicios
                await VerifyServicesHealthAsync();

                // 5. Notificar completado
                await SendNotificationAsync("Failover completed");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failover procedure failed");
            throw;
        }
    }

    private async Task<bool> CheckPrimaryHealthAsync()
    {
        try
        {
            var describeInstancesRequest = new DescribeInstancesRequest
            {
                Filters = new List<Filter>
                {
                    new Filter("tag:Environment", new List<string> { "production" }),
                    new Filter("instance-state-name", new List<string> { "running", "pending" })
                }
            };

            var response = await _ec2Client.DescribeInstancesAsync(describeInstancesRequest);

            var totalInstances = response.Reservations.Sum(r => r.Instances.Count);
            var runningInstances = response.Reservations
                .SelectMany(r => r.Instances)
                .Count(i => i.State.Name.Value == "running");

            return totalInstances > 0 && (double)runningInstances / totalInstances >= 0.8;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to check primary health");
            return false;
        }
    }

    private async Task ActivateSecondaryRegionAsync()
    {
        var startInstancesRequest = new StartInstancesRequest
        {
            InstanceIds = new List<string> { "i-1234567890abcdef0" }
        };

        await _ec2Client.StartInstancesAsync(startInstancesRequest);
    }

    private async Task UpdateDnsRecordsAsync()
    {
        var changeRequest = new ChangeResourceRecordSetsRequest
        {
            HostedZoneId = "Z1234567890ABCDEF",
            ChangeBatch = new ChangeBatch
            {
                Changes = new List<Change>
                {
                    new Change
                    {
                        Action = ChangeAction.UPSERT,
                        ResourceRecordSet = new ResourceRecordSet
                        {
                            Name = "api.example.com",
                            Type = RRType.A,
                            TTL = 300,
                            ResourceRecords = new List<ResourceRecord>
                            {
                                new ResourceRecord { Value = "10.0.1.100" }
                            }
                        }
                    }
                }
            }
        };

        await _route53Client.ChangeResourceRecordSetsAsync(changeRequest);
    }

    private async Task VerifyServicesHealthAsync()
    {
        using var httpClient = new HttpClient();
        var healthEndpoints = new[]
        {
            "http://secondary-api.example.com/health",
            "http://secondary-api.example.com/health/ready",
            "http://secondary-api.example.com/health/live"
        };

        foreach (var endpoint in healthEndpoints)
        {
            var response = await httpClient.GetAsync(endpoint);
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception($"Health check failed for {endpoint}");
            }
        }
    }

    private async Task SendNotificationAsync(string message)
    {
        // Implementar notificación al equipo (email, Slack, etc.)
        _logger.LogInformation("Sending notification: {Message}", message);

        // Ejemplo: Enviar email usando AWS SES
        // await _sesClient.SendEmailAsync(...);
    }
}
```

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] Análisis de impacto de negocio completado
- [ ] Objetivos RTO/RPO definidos y documentados
- [ ] Estrategia de backup implementada y probada
- [ ] Replicación de datos configurada
- [ ] Procedimientos de recuperación documentados
- [ ] Testing de DR programado y ejecutado
- [ ] Monitoreo y alertas configurados
- [ ] Equipo entrenado en procedimientos de DR

### Para Cambios Existentes
- [ ] Impacto en DR evaluado
- [ ] Procedimientos de DR actualizados
- [ ] Testing de DR ejecutado
- [ ] Documentación de DR actualizada
- [ ] Equipo notificado de cambios

## Excepciones y Justificaciones

### Cuándo Reducir DR
- **Sistemas de desarrollo**: Entornos de desarrollo y testing
- **Sistemas no críticos**: Sistemas con bajo impacto de negocio
- **Prototipos**: Sistemas experimentales
- **Costos prohibitivos**: Cuando el costo de DR excede el beneficio

### Proceso de Excepción
1. **Análisis de riesgo**: Evaluar impacto de no implementar DR
2. **Alternativas**: Identificar otras formas de mitigar riesgos
3. **Aprobación**: Obtener aprobación del equipo de arquitectura
4. **Documentación**: Documentar justificación y alternativas
5. **Revisión**: Revisar decisión periódicamente

## Referencias y Recursos

### Herramientas de DR
- [AWS Disaster Recovery - Soluciones de DR]
- [Azure Site Recovery - Recuperación de sitios]
- [Google Cloud DR - Disaster Recovery]
- [Veeam - Backup y DR]

### Frameworks y Estándares
- [ISO 22301 - Business Continuity Management]
- [NIST Cybersecurity Framework - DR]
- [ITIL - Service Continuity Management]
- [COBIT - Business Continuity]

### Recursos Adicionales
- [AWS Well-Architected Framework - DR]
- [Azure Architecture Center - DR]
- [Google Cloud Architecture Framework - DR]
- [Disaster Recovery Best Practices]
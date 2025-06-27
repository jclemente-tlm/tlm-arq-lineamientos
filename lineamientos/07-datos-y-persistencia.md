# Datos y Persistencia

## Propósito

Este lineamiento define los principios, patrones y mejores prácticas para el diseño, implementación y gestión de bases de datos y persistencia de datos. Se enfoca en PostgreSQL como base de datos principal y la migración gradual desde Oracle.

## Principios Fundamentales

### Estrategia de Base de Datos
- **PostgreSQL como estándar**: Base de datos relacional principal para nuevos desarrollos
- **DynamoDB para NoSQL**: Casos de uso específicos que requieren NoSQL
- **Evitar proliferación**: Minimizar el número de motores de base de datos
- **Migración gradual**: Transición controlada desde Oracle a PostgreSQL
- **Documentación completa**: Esquemas y modelos bien documentados

### Diseño de Datos
- **Normalización**: Diseñar esquemas normalizados y eficientes
- **Versionado**: Control de versiones para esquemas de base de datos
- **Migraciones automatizadas**: Scripts de migración versionados y automatizados
- **Backups automáticos**: Estrategia de respaldo y recuperación
- **Cifrado**: Protección de datos sensibles en reposo y tránsito

## Migración de Oracle a PostgreSQL

### Estrategia de Migración
```sql
-- Ejemplo: Migración de tipos de datos Oracle a PostgreSQL
-- Oracle
CREATE TABLE users (
    id NUMBER(10) PRIMARY KEY,
    email VARCHAR2(255) NOT NULL,
    created_date DATE DEFAULT SYSDATE,
    status VARCHAR2(20) DEFAULT 'ACTIVE'
);

-- PostgreSQL equivalente
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'ACTIVE'
);
```

### Herramientas de Migración
```bash
# Ejemplo: Script de migración con ora2pg
ora2pg --type TABLE \
       --source oracle_connection \
       --target postgres_connection \
       --schema public \
       --table users

# Ejemplo: Migración de datos con pgloader
pgloader oracle://user:pass@host:port/db \
         postgresql://user:pass@host:port/db
```

### Configuración de Entity Framework
```csharp
// Ejemplo: Configuración para PostgreSQL con .NET 8
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseNpgsql(Configuration.GetConnectionString("DefaultConnection"),
                npgsqlOptions =>
                {
                    npgsqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 3,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorCodesToAdd: null);

                    npgsqlOptions.MigrationsAssembly("YourApp.Migrations");
                });
        });
    }
}

// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=mydb;Username=myuser;Password=***;SSL Mode=Require;Trust Server Certificate=true"
  }
}
```

## Diseño de Esquemas

### Modelado de Entidades
```csharp
// Ejemplo: Entidades con Entity Framework Core
public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsActive { get; set; }

    // Relaciones
    public virtual ICollection<Order> Orders { get; set; }
    public virtual UserProfile Profile { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public string PhoneNumber { get; set; }
    public string Address { get; set; }
    public DateTime? DateOfBirth { get; set; }

    // Foreign Key
    public virtual User User { get; set; }
}

// Configuración de Entity Framework
public class ApplicationDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<UserProfile> UserProfiles { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configuración de User
        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(255);
            entity.HasIndex(e => e.Email).IsUnique();
            entity.Property(e => e.FirstName).HasMaxLength(100);
            entity.Property(e => e.LastName).HasMaxLength(100);
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");
            entity.Property(e => e.IsActive).HasDefaultValue(true);
        });

        // Configuración de UserProfile
        modelBuilder.Entity<UserProfile>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.HasOne(e => e.User)
                  .WithOne(e => e.Profile)
                  .HasForeignKey<UserProfile>(e => e.UserId)
                  .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

### Migraciones Automatizadas
```csharp
// Ejemplo: Migración con Entity Framework
public partial class AddUserProfileTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "UserProfiles",
            columns: table => new
            {
                Id = table.Column<int>(type: "integer", nullable: false)
                    .Annotation("Npgsql:ValueGenerationStrategy", NpgsqlValueGenerationStrategy.IdentityByDefaultColumn),
                UserId = table.Column<int>(type: "integer", nullable: false),
                PhoneNumber = table.Column<string>(type: "character varying(20)", maxLength: 20, nullable: true),
                Address = table.Column<string>(type: "text", nullable: true),
                DateOfBirth = table.Column<DateTime>(type: "timestamp with time zone", nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_UserProfiles", x => x.Id);
                table.ForeignKey(
                    name: "FK_UserProfiles_Users_UserId",
                    column: x => x.UserId,
                    principalTable: "Users",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });

        migrationBuilder.CreateIndex(
            name: "IX_UserProfiles_UserId",
            table: "UserProfiles",
            column: "UserId",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "UserProfiles");
    }
}
```

## Optimización de Consultas

### Índices Estratégicos
```sql
-- Ejemplo: Índices para optimización
-- Índice único para email
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Índice compuesto para búsquedas frecuentes
CREATE INDEX idx_users_name_status ON users(first_name, last_name, is_active);

-- Índice parcial para usuarios activos
CREATE INDEX idx_users_active ON users(id) WHERE is_active = true;

-- Índice para fechas
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

### Consultas Optimizadas
```csharp
// Ejemplo: Repository con consultas optimizadas
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public async Task<List<User>> GetActiveUsersAsync(int page, int pageSize)
    {
        return await _context.Users
            .Where(u => u.IsActive)
            .Include(u => u.Profile)
            .OrderBy(u => u.CreatedAt)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
    }

    public async Task<User> GetUserWithOrdersAsync(int userId)
    {
        return await _context.Users
            .Include(u => u.Profile)
            .Include(u => u.Orders)
                .ThenInclude(o => o.OrderItems)
            .FirstOrDefaultAsync(u => u.Id == userId);
    }

    public async Task<List<User>> SearchUsersAsync(string searchTerm)
    {
        return await _context.Users
            .Where(u => u.IsActive &&
                       (u.FirstName.Contains(searchTerm) ||
                        u.LastName.Contains(searchTerm) ||
                        u.Email.Contains(searchTerm)))
            .ToListAsync();
    }
}
```

## DynamoDB para Casos NoSQL

### Configuración de DynamoDB
```csharp
// Ejemplo: Configuración de DynamoDB con .NET
public class DynamoDBConfiguration
{
    public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        services.AddAWSService<IAmazonDynamoDB>(new AWSOptions
        {
            Region = Amazon.RegionEndpoint.USEast1
        });

        services.AddSingleton<IDynamoDBContext, DynamoDBContext>();
    }
}

// Modelo de DynamoDB
[DynamoDBTable("UserSessions")]
public class UserSession
{
    [DynamoDBHashKey]
    public string UserId { get; set; }

    [DynamoDBRangeKey]
    public string SessionId { get; set; }

    [DynamoDBProperty]
    public DateTime CreatedAt { get; set; }

    [DynamoDBProperty]
    public DateTime ExpiresAt { get; set; }

    [DynamoDBProperty]
    public Dictionary<string, string> Attributes { get; set; }
}

// Repository para DynamoDB
public class DynamoDBUserSessionRepository : IUserSessionRepository
{
    private readonly IDynamoDBContext _context;

    public async Task<UserSession> GetSessionAsync(string userId, string sessionId)
    {
        return await _context.LoadAsync<UserSession>(userId, sessionId);
    }

    public async Task SaveSessionAsync(UserSession session)
    {
        await _context.SaveAsync(session);
    }

    public async Task DeleteSessionAsync(string userId, string sessionId)
    {
        await _context.DeleteAsync<UserSession>(userId, sessionId);
    }
}
```

## Backup y Recuperación

### Estrategia de Backup
```bash
# Ejemplo: Script de backup automático para PostgreSQL
#!/bin/bash

# Configuración
DB_NAME="mydb"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql"

# Crear backup
pg_dump -h localhost -U postgres -d $DB_NAME > $BACKUP_FILE

# Comprimir backup
gzip $BACKUP_FILE

# Eliminar backups antiguos (mantener últimos 30 días)
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

# Verificar backup
if [ $? -eq 0 ]; then
    echo "Backup completed successfully: ${BACKUP_FILE}.gz"
else
    echo "Backup failed"
    exit 1
fi
```

### Recuperación de Datos
```bash
# Ejemplo: Script de recuperación
#!/bin/bash

DB_NAME="mydb"
BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

# Restaurar backup
gunzip -c $BACKUP_FILE | psql -h localhost -U postgres -d $DB_NAME

if [ $? -eq 0 ]; then
    echo "Restore completed successfully"
else
    echo "Restore failed"
    exit 1
fi
```

## Cifrado de Datos

### Cifrado en Reposo
```csharp
// Ejemplo: Cifrado de datos sensibles
public class EncryptedUser
{
    public int Id { get; set; }
    public string Email { get; set; }

    [Encrypted]
    public string SSN { get; set; }

    [Encrypted]
    public string CreditCardNumber { get; set; }
}

// Atributo de cifrado
[AttributeUsage(AttributeTargets.Property)]
public class EncryptedAttribute : Attribute
{
}

// Interceptor de Entity Framework para cifrado
public class EncryptionInterceptor : SaveChangesInterceptor
{
    private readonly IEncryptionService _encryptionService;

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;

        foreach (var entry in context.ChangeTracker.Entries())
        {
            foreach (var property in entry.Properties)
            {
                if (property.Metadata.PropertyInfo?.GetCustomAttribute<EncryptedAttribute>() != null)
                {
                    if (property.CurrentValue != null)
                    {
                        property.CurrentValue = _encryptionService.Encrypt(property.CurrentValue.ToString());
                    }
                }
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

## Auditoría de Datos

### Audit Trail
```csharp
// Ejemplo: Auditoría de cambios en datos
public class AuditEntry
{
    public int Id { get; set; }
    public string TableName { get; set; }
    public string KeyValues { get; set; }
    public string OldValues { get; set; }
    public string NewValues { get; set; }
    public string Action { get; set; }
    public string UserId { get; set; }
    public DateTime Timestamp { get; set; }
}

public class AuditDbContext : DbContext
{
    public DbSet<AuditEntry> AuditEntries { get; set; }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var auditEntries = OnBeforeSaveChanges();
        var result = await base.SaveChangesAsync(cancellationToken);
        await OnAfterSaveChanges(auditEntries);
        return result;
    }

    private List<AuditEntry> OnBeforeSaveChanges()
    {
        ChangeTracker.DetectChanges();
        var auditEntries = new List<AuditEntry>();

        foreach (var entry in ChangeTracker.Entries())
        {
            if (entry.Entity is AuditEntry || entry.State == EntityState.Detached || entry.State == EntityState.Unchanged)
                continue;

            var auditEntry = new AuditEntry
            {
                TableName = entry.Entity.GetType().Name,
                Action = entry.State.ToString(),
                KeyValues = JsonSerializer.Serialize(entry.Properties.Where(p => p.Metadata.IsPrimaryKey()).ToDictionary(x => x.Metadata.Name, x => x.CurrentValue)),
                OldValues = entry.State == EntityState.Modified ? JsonSerializer.Serialize(entry.Properties.Where(p => p.IsModified).ToDictionary(x => x.Metadata.Name, x => x.OriginalValue)) : null,
                NewValues = entry.State == EntityState.Added || entry.State == EntityState.Modified ? JsonSerializer.Serialize(entry.Properties.ToDictionary(x => x.Metadata.Name, x => x.CurrentValue)) : null,
                Timestamp = DateTime.UtcNow
            };

            auditEntries.Add(auditEntry);
        }

        return auditEntries;
    }
}
```

## Monitoreo y Performance

### Métricas de Base de Datos
```csharp
// Ejemplo: Métricas de performance con Prometheus
public class DatabaseMetrics
{
    private readonly Counter _queryCount;
    private readonly Histogram _queryDuration;
    private readonly Counter _connectionCount;

    public DatabaseMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("database_metrics");

        _queryCount = meter.CreateCounter<long>("database_queries_total");
        _queryDuration = meter.CreateHistogram<double>("database_query_duration_seconds");
        _connectionCount = meter.CreateCounter<long>("database_connections_total");
    }

    public void RecordQuery(string queryType, TimeSpan duration)
    {
        _queryCount.Add(1, new KeyValuePair<string, object>("query_type", queryType));
        _queryDuration.Record(duration.TotalSeconds, new KeyValuePair<string, object>("query_type", queryType));
    }
}
```

## Checklist de Cumplimiento

### Para Nuevos Proyectos
- [ ] PostgreSQL configurado como base de datos principal
- [ ] Esquemas de base de datos documentados
- [ ] Migraciones automatizadas configuradas
- [ ] Estrategia de backup implementada
- [ ] Cifrado de datos sensibles configurado
- [ ] Auditoría de datos implementada
- [ ] Monitoreo de performance configurado
- [ ] Testing de base de datos automatizado

### Para Migraciones
- [ ] Análisis de impacto completado
- [ ] Estrategia de migración definida
- [ ] Plan de rollback preparado
- [ ] Testing exhaustivo realizado
- [ ] Documentación actualizada
- [ ] Equipo capacitado

## Excepciones y Justificaciones

### Cuándo Usar Otros Motores
- **DynamoDB**: Para casos de uso NoSQL específicos
- **Redis**: Para cache y sesiones
- **Oracle**: Para sistemas legacy que requieren migración gradual
- **SQLite**: Para desarrollo local y testing

### Proceso de Excepción
1. **Análisis técnico**: Evaluar necesidad de motor alternativo
2. **Justificación**: Documentar razones técnicas
3. **Aprobación**: Obtener aprobación del equipo de arquitectura
4. **Plan de migración**: Definir plan para migración futura
5. **Revisión**: Revisar decisión periódicamente

## Referencias y Recursos

### Documentación Oficial
- [PostgreSQL Documentation]
- [Entity Framework Core Documentation]
- [AWS DynamoDB Documentation]
- [Oracle to PostgreSQL Migration Guide]

### Herramientas de Migración
- [ora2pg - Oracle to PostgreSQL Migration]
- [pgloader - Database Migration Tool]
- [AWS Database Migration Service]
- [Azure Database Migration Service]

### Recursos Adicionales
- [PostgreSQL Performance Tuning]
- [Entity Framework Performance]
- [Database Design Best Practices]
- [Data Migration Strategies]
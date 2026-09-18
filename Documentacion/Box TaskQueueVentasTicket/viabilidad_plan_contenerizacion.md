=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** Box TaskQueueVentasTicket

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Almacenamiento y Estado (Factor 6):** `[Bloqueante]` El uso de SQLite (`AlertMonitor.db`) en el sistema de archivos del host de Windows prohíbe el comportamiento efímero nativo. Al reiniciarse el contenedor, se perderá la base de datos de alertas. Se requiere obligatoriamente **externalizar la base de datos** (ej. migrar SQLite a PostgreSQL/SQL Server) o bien configurar un **Persistent Volume Claim (PVC)** que monte un volumen persistente de red en el contenedor.
2. **Configuración (Factor 3):** `[No Cumple]` App.config con secretos hardcodeados debe portarse a `appsettings.json` o variables de entorno.
3. **Logs (Factor 11):** `[No Cumple]` Redirección a `stdout`.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO DESPUÉS (.NET 8 WORKER CON CONFIGURACIÓN DE VOLUMEN)
```csharp
public class AlertMonitorWorker : BackgroundService
{
    private readonly IAlertUnitOfWork _unitOfWork;
    private readonly ILogger<AlertMonitorWorker> _logger;
    private readonly AppConfig _config;

    public AlertMonitorWorker(IAlertUnitOfWork unitOfWork, IOptions<AppConfig> config, ILogger<AlertMonitorWorker> logger)
    {
        _unitOfWork = unitOfWork;
        _config = config.Value;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Iniciando Monitor de Alertas de Venta de Tickets.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                _logger.LogInformation("Ejecutando conciliación analítica de inventario...");
                await _unitOfWork.Alerts.ProcessTicketSalesAuditAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Falla en ciclo de análisis de inventario de tickets.");
            }

            await Task.Delay(TimeSpan.FromMinutes(_config.CheckIntervalMinutes), stoppingToken);
        }
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
La suite de alertas forma parte del contexto de **"Audit & Compliance Bounded Context"**. Para desacoplar el monitoreo masivo del ERP, cada vez que se emita un ticket, el punto de venta (POS) debe disparar una transacción a un bus de eventos (Kafka/RabbitMQ). Este microservicio consumirá dichos eventos y validará anomalías reactivamente en tiempo real.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.AlertMonitorService.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.AlertMonitorService.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.AlertMonitorService.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.AlertMonitorService.dll"]
```

#### `docker-compose.yml` (Con Volumen Persistente para SQLite)
```yaml
version: '3.8'

services:
  alert-monitor-agent:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: alert-monitor-agent
    environment:
      - AppConfig__CheckIntervalMinutes=10
      # Apunta SQLite al volumen de almacenamiento persistente montado en el contenedor
      - ConnectionStrings__AlertMonitorDb=Data Source=/data/AlertMonitor.db;Version=3;
    volumes:
      - alert-data:/data
    restart: always

volumes:
  alert-data:
    driver: local
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy AlertMonitor
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/alert-monitor-agent:${{ github.sha }} .
```
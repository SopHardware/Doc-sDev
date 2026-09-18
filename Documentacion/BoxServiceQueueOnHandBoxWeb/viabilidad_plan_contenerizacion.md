=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceQueueOnHandBoxWeb

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Backing Services (Factor 4):** `[Riesgo Crítico]` La aplicación satura la base de datos origen. En K8s, bajo autoscaling, esto podría multiplicar las conexiones y tirar el ERP.
2. **Telemetría (Factor 11):** `[No Cumple]` Debe removerse Windows EventLog en favor de Salida Estándar (`stdout`).
3. **Procesos (Factor 6):** `[Cumple]` Agente *Stateless*.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO CON BACKGROUND SERVICE)
```csharp
public class OnHandSyncWorker : BackgroundService
{
    private readonly IOnHandRepository _repository;
    private readonly ILogger<OnHandSyncWorker> _logger;
    private readonly AppConfig _config;

    public OnHandSyncWorker(IOnHandRepository repository, IOptions<AppConfig> config, ILogger<OnHandSyncWorker> logger)
    {
        _repository = repository;
        _config = config.Value;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var stockData = await _repository.GetDirtyStockAsync(stoppingToken); // Query optimizado (Solo Delta)
                if (stockData.Any())
                {
                    await ProcessStockUpdateAsync(stockData, stoppingToken);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Falla en sincronización de OnHand");
            }
            // Retardo seguro controlado. No hay superposición (No se usa Timer)
            await Task.Delay(TimeSpan.FromSeconds(_config.PollingIntervalSeconds), stoppingToken);
        }
    }

    private async Task ProcessStockUpdateAsync(List<StockItem> stockData, CancellationToken cancellationToken)
    {
        using var semaphore = new SemaphoreSlim(_config.MaxConcurrentRequests);
        var tasks = stockData.Chunk(500).Select(async chunk =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                await _repository.SendToBoxWebApiAsync(chunk.ToList(), cancellationToken);
            }
            finally
            {
                semaphore.Release();
            }
        });
        await Task.WhenAll(tasks);
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
Eliminar el *Polling*. Adoptar **Change Data Capture (CDC)** (como Debezium) directamente conectado al SQL Server de Epicor, capturando mutaciones en `PartBin` y publicándolas en Kafka, donde este agente escuchará y retransmitirá a BoxWeb.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.OnHand.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.OnHand.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.OnHand.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.OnHand.dll"]
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy OnHand Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/onhand-agent:${{ github.sha }} .
```
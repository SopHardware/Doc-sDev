=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceQueuePartRepartos (DataCollector)

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` App.config debe eliminarse y reemplazarse por variables de entorno para cumplir con 12-Factor App.
2. **Concurrencia (Factor 8):** `[Riesgo]` La falta de asincronismo e inyección de HttpClient Factory propiciará *Socket Exhaustion* bajo alta concurrencia.
3. **Telemetría y Logs (Factor 11):** `[No Cumple]` Redirección de logs a `stdout` JSON es obligatoria.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO CON SEMÁFORO)
```csharp
public class RepartosQueueManager : IRepartosQueueManager
{
    private readonly IRepartosRepository _repository;
    private readonly ILogger<RepartosQueueManager> _logger;
    private readonly AppConfig _config;

    public RepartosQueueManager(IRepartosRepository repository, IOptions<AppConfig> config, ILogger<RepartosQueueManager> logger)
    {
        _repository = repository;
        _config = config.Value;
        _logger = logger;
    }

    public async Task ProcessAsync(CancellationToken cancellationToken)
    {
        var sourceData = await _repository.GetPendingRepartosAsync();
        if (!sourceData.Any()) return;

        using var semaphore = new SemaphoreSlim(_config.MaxConcurrentRequests);
        var tasks = sourceData.Chunk(_config.BatchSize).Select(async chunk =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                await _repository.SendRepartosToOrchestratorAsync(chunk.ToList());
                await _repository.UpdateLocalStatusAsync(chunk.Select(x => x.SyncID).ToList(), 2);
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
Abstraer bajo el contexto de **"Logistics & Delivery Bounded Context"**. El servicio debe convertirse en un consumidor de eventos logísticos de colas RabbitMQ, reduciendo la dependencia directa a consultas directas del ERP de Epicor.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.RepartosColector.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.RepartosColector.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.RepartosColector.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.RepartosColector.dll"]
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy RepartosColector
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/repartos-colector-agent:${{ github.sha }} .
```
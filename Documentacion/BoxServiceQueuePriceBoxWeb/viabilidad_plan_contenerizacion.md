=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceQueuePriceBoxWeb

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` App.config debe eliminarse y portarse a variables de entorno inyectadas por K8s Secrets.
2. **Concurrencia (Factor 8):** `[Riesgo]` La falta de asincronismo y la instanciación directa de HttpClient Factory propiciará *Socket Exhaustion* bajo alta concurrencia.
3. **Logs (Factor 11):** `[No Cumple]` Redirección de logs a `stdout` JSON es obligatoria para Kubernetes.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO DESPUÉS (.NET 8 WORKER CON BATCHING SEGURO)
```csharp
public class PriceQueueManager : IPriceQueueManager
{
    private readonly IPriceRepository _repository;
    private readonly ILogger<PriceQueueManager> _logger;
    private readonly AppConfig _config;

    public PriceQueueManager(IPriceRepository repository, IOptions<AppConfig> config, ILogger<PriceQueueManager> logger)
    {
        _repository = repository;
        _config = config.Value;
        _logger = logger;
    }

    public async Task ProcessAsync(CancellationToken cancellationToken)
    {
        var priceUpdates = await _repository.GetModifiedPricesAsync(cancellationToken);
        if (!priceUpdates.Any()) return;

        _logger.LogInformation("Sincronizando {Count} combinaciones de precios.", priceUpdates.Count);

        using var semaphore = new SemaphoreSlim(_config.MaxConcurrentRequests);
        var tasks = priceUpdates.Chunk(_config.BatchSize).Select(async chunk =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                await _repository.SendPricesToBoxWebApiAsync(chunk.ToList(), cancellationToken);
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
Aislar bajo el Bounded Context de **"Pricing & Catalog Context"**. La actualización de precios debe desacoplarse utilizando colas de mensajes (RabbitMQ/Kafka) para sincronizar BoxWeb de manera pasiva y reactiva (modelo push), protegiendo la base de datos de Epicor ante consultas masivas recurrentes.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.PriceBoxWeb.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.PriceBoxWeb.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.PriceBoxWeb.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.PriceBoxWeb.dll"]
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy PriceBoxWeb Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/price-boxweb-agent:${{ github.sha }} .
```
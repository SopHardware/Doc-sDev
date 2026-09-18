=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceQueueMaestroArticulos

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` Anclado a `ConfigurationManager` en .NET Framework.
2. **Concurrencia (Factor 8):** `[Bloqueante]` El uso de `Task.WaitAll` detiene los hilos y compromete la elasticidad del contenedor, forzando consumos de CPU artificiales (Idle Wait).
3. **Telemetría y Logs (Factor 11):** `[No Cumple]` Uso repetitivo de `EventLog` de Windows.
4. **Dependencias y Estado:** `[Cumple]` La aplicación lee y publica (Pipeline ETL básico), por lo que es Stateless.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
public async Task ProAsync()
{
    await Task.Run(() =>
    {
        List<PartSyncEntity> DataSource = part.Query("exec boxito.BOX_SP_BulkInsertParts");
        if (DataSource.Count <= 0) return;

        List<Task> ToDo = new List<Task>();
        Dictionary<string, List<PartSyncEntity>> byPlant = DataSource.GroupBy(g => g.Sucursal).ToDictionary(d => d.Key, d => d.ToList());
        foreach (var plant in byPlant)
        {
            int pageSize = 1000, pageNumber = 0;
            List<PartSyncEntity> partial = null;

            while ((partial = plant.Value.Skip(pageNumber * pageSize).Take(pageSize).ToList()) != null)
            {
                if (partial.Count <= 0) break;
                ToDo.Add(ToParnet(ServiceConfig.Company, plant.Key, partial));
                pageNumber++;
            }
        }
        Task.WaitAll(ToDo.ToArray()); // RIESGO: Bloqueo Síncrono de Hilo
    });
}
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER CON PARALELISMO SEGURO)
```csharp
public class PartQueueManager : IPartQueueManager
{
    private readonly IPartRepository _repository;
    private readonly ILogger<PartQueueManager> _logger;
    private readonly AppConfig _config;

    public PartQueueManager(IPartRepository repository, IOptions<AppConfig> config, ILogger<PartQueueManager> logger)
    {
        _repository = repository;
        _config = config.Value;
        _logger = logger;
    }

    public async Task ProcessAsync(CancellationToken cancellationToken)
    {
        var dataSource = await _repository.GetBulkPartsAsync("exec boxito.BOX_SP_BulkInsertParts");
        if (!dataSource.Any()) return;

        var byPlant = dataSource.GroupBy(g => g.Sucursal).ToDictionary(d => d.Key, d => d.ToList());
        var tasks = new List<Task>();
        using var semaphore = new SemaphoreSlim(_config.MaxConcurrentRequests); // Throttling

        foreach (var plant in byPlant)
        {
            var chunks = plant.Value.Chunk(_config.BatchSize); // Uso optimizado de LINQ Chunk (.NET 6+)
            foreach (var chunk in chunks)
            {
                tasks.Add(Task.Run(async () =>
                {
                    await semaphore.WaitAsync(cancellationToken);
                    try
                    {
                        await _repository.SendToParnetAsync(_config.Company, plant.Key, chunk.ToList());
                    }
                    finally
                    {
                        semaphore.Release();
                    }
                }, cancellationToken));
            }
        }
        await Task.WhenAll(tasks); // Seguro y asíncrono puro
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
El agente "Cola de Artículos" debe migrarse a una arquitectura dirigida por eventos (Event-Driven). En lugar de usar un SP masivo `BOX_SP_BulkInsertParts` cíclicamente, el ERP debe emitir mensajes a **Kafka/RabbitMQ** cada vez que un artículo cambie, y este Worker solo los consumirá y despachará.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.MaestroArticulos.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.MaestroArticulos.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.MaestroArticulos.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.MaestroArticulos.dll"]
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy MaestroArticulos
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/maestro-articulos-agent:${{ github.sha }} .
```
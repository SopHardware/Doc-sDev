=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceQueuePartAllocTOs

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` Debe implementarse inyección de `IOptions<T>` usando variables de entorno.
2. **Telemetría y Logs (Factor 11):** `[No Cumple]` Dependencia propietaria de Windows.
3. **Concurrencia (Factor 8):** `[Riesgo]` La escalabilidad de hilos es deficiente.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO DESPUÉS (.NET 8 WORKER Y LINQ CHUNK OPTIMIZADO)
```csharp
public class PartAllocQueueManager : IPartAllocQueueManager
{
    private readonly IPartAllocRepository _repository;
    private readonly ILogger<PartAllocQueueManager> _logger;

    public PartAllocQueueManager(IPartAllocRepository repository, ILogger<PartAllocQueueManager> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task ProcessAsync(CancellationToken cancellationToken)
    {
        var dataSource = await _repository.GetPendingAllocationsAsync();
        if (!dataSource.Any()) return;

        var byPlant = dataSource.GroupBy(g => g.Plant).ToDictionary(d => d.Key, d => d.ToList());
        using var semaphore = new SemaphoreSlim(4); // Control estricto de concurrencia de red

        var tasks = byPlant.SelectMany(plant => 
            plant.Value.Chunk(500).Select(async chunk => 
            {
                await semaphore.WaitAsync(cancellationToken);
                try
                {
                    await _repository.SendAllocationsToApiAsync(plant.Key, chunk.ToList());
                }
                finally
                {
                    semaphore.Release();
                }
            })
        );

        await Task.WhenAll(tasks);
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
Este agente forma parte del Bounded Context **"Inventory & Warehouse (WMS)"**. Para hacerlo Cloud-Native, el ERP debería inyectar eventos de reserva de stock directamente a una cola **RabbitMQ**. El agente se convertiría en un consumidor pasivo de esa cola en Kubernetes.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.PartAllocTOs.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.PartAllocTOs.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.PartAllocTOs.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.PartAllocTOs.dll"]
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy PartAllocTOs Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/part-alloc-agent:${{ github.sha }} .
```
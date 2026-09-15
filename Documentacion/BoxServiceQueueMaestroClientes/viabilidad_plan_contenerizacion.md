=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceQueueMaestroClientes

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` `App.config` debe ser eliminado.
2. **Concurrencia (Factor 8):** `[Riesgo]` La escalabilidad de hilos es precaria. Debe migrar a `IHostedService` en .NET 8.
3. **Telemetría y Logs (Factor 11):** `[No Cumple]` Acoplamiento con Windows EventLog.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
// Iteración manual ineficiente y bloqueo de hilos
int pageSize = 1000, pageNumber = 0;
List<CustomerSyncEntity> partial = null;
while ((partial = plant.Value.Skip(pageNumber * pageSize).Take(pageSize).ToList()) != null)
{
    if (partial.Count <= 0) break;
    ToDo.Add(ToParnet(Company, plant.Key, partial));
    pageNumber++;
}
Task.WaitAll(ToDo.ToArray());
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER - LINQ CHUNK OPTIMIZADO)
```csharp
public class CustomerQueueManager : ICustomerQueueManager
{
    private readonly ICustomerRepository _repository;
    private readonly ILogger<CustomerQueueManager> _logger;

    public CustomerQueueManager(ICustomerRepository repository, ILogger<CustomerQueueManager> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task ProcessAsync(CancellationToken cancellationToken)
    {
        var dataSource = await _repository.GetPendingCustomersAsync();
        if (!dataSource.Any()) return;

        var byPlant = dataSource.GroupBy(g => g.Plant).ToDictionary(d => d.Key, d => d.ToList());
        using var semaphore = new SemaphoreSlim(5); // Control de flujos concurrentes

        var tasks = byPlant.SelectMany(plant => 
            plant.Value.Chunk(1000).Select(async chunk => 
            {
                await semaphore.WaitAsync(cancellationToken);
                try
                {
                    await _repository.SendToOrchestratorAsync(plant.Key, chunk.ToList());
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
Fusionar esta lógica con las cargas iniciales en un Bounded Context **"Customer Management"**. Implementar Kafka/Event Grid para evitar consultas constantes a la base de datos (polling) y procesar de manera reactiva (push).

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.MaestroClientes.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.MaestroClientes.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.MaestroClientes.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.MaestroClientes.dll"]
```

### FASE 4: PIPELINE CI/CD Y SEGURIDAD
```yaml
name: Deploy MaestroClientes
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/maestro-clientes-agent:${{ github.sha }} .
```
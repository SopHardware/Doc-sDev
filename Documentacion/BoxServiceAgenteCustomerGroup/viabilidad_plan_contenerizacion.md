=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceAgenteCustomerGroup

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES ESTRICTAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` Uso de `ConfigurationManager` en XML planos. Debe modificarse para cargar dinámicamente mediante variables de entorno bajo .NET 8.
2. **Concurrencia (Factor 8):** `[Bloqueante]` El método `ToOrigin` encolado por `ThreadPool` realiza una llamada asíncrona desatendida (*Fire and Forget*):
   ```csharp
   public void ToOrigin(object model)
   {
       List<object> oModel = (List<object>)model;
       Task oTask = custumerGroupRespository.OrigenAsync(oModel); // No tiene wait ni await
   }
   ```
   Esto expone la ejecución a fallas indetectables y fugas de hilos de red en un contenedor con límites de recursos en Kubernetes.
3. **Logs (Factor 11):** `[No Cumple]` Dependencia de telemetría local de Windows (`EventLog` heredado a través de `EventsLog`). Debe redireccionarse a `stdout`/`stderr` en JSON.
4. **Estado (Factor 6):** `[Cumple]` Proceso transaccional *stateless* (sin estado persistente local).

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
public async Task ProcesoAsync()
{
    await Task.Run(() =>
    {
        List<Customer> oOrigen = custumerGroupRespository.GetBySP("exec Boxito.SP_POS_Trae_CustomerGroups");
        foreach (Customer oCust in oOrigen)
            ToDestiny(oCust); // Síncrono bloqueante en ciclo

        List<Customer> oDestino = custumerGroupRespository.GetLista().Where(x => x.ExportStatus == 0).ToList();
        Dictionary<string, List<Customer>> byPlant = oDestino.GroupBy(g => g.Plant).ToDictionary(d => d.Key, d => d.ToList());
        foreach (var Plant in byPlant)
        {
            List<object> oModel = new List<object>() { Plant.Key, Plant.Value };
            ThreadPool.QueueUserWorkItem(ToOrigin, oModel); // Encolado sin límite de concurrencia
        }
    });
}

public void ToOrigin(object model)
{
    List<object> oModel = (List<object>)model;
    Task oTask = custumerGroupRespository.OrigenAsync(oModel); // Fire-and-forget incontrolable
}
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO CON SEMÁFORO)
```csharp
public class CustomerGroupSyncManager : ICustomerGroupSyncManager
{
    private readonly ICustomerGroupRepository _repository;
    private readonly AppConfig _config;
    private readonly ILogger<CustomerGroupSyncManager> _logger;

    public CustomerGroupSyncManager(ICustomerGroupRepository repository, IOptions<AppConfig> config, ILogger<CustomerGroupSyncManager> logger)
    {
        _repository = repository;
        _config = config.Value;
        _logger = logger;
    }

    public async Task ProcessSyncAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Iniciando recolección de grupos de clientes.");
        var sourceData = await _repository.GetBySpAsync("exec Boxito.SP_POS_Trae_CustomerGroups");

        // Guardado paralelo controlado para evitar saturar base de datos
        using (var semaphore = new SemaphoreSlim(_config.MaxConcurrentDbConnections))
        {
            var saveTasks = sourceData.Select(async customer =>
            {
                await semaphore.WaitAsync(cancellationToken);
                try
                {
                    var data = new DataSource { ForeignSysRowID = customer.ForeignSysRowID };
                    await _repository.InsertAsync(customer, data);
                }
                finally
                {
                    semaphore.Release();
                }
            });
            await Task.WhenAll(saveTasks);
        }

        // Envío asíncrono controlado con semáforo hacia el orquestador
        var pendingData = await _repository.GetPendingSyncGroupsAsync();
        var byPlant = pendingData.GroupBy(g => g.Plant).ToDictionary(d => d.Key, d => d.ToList());

        using (var apiSemaphore = new SemaphoreSlim(_config.MaxConcurrentApiRequests))
        {
            var syncTasks = byPlant.Select(async plantGroup =>
            {
                await apiSemaphore.WaitAsync(cancellationToken);
                try
                {
                    var model = new List<object> { plantGroup.Key, plantGroup.Value };
                    await _repository.OrigenAsync(model); // Await real
                }
                finally
                {
                    apiSemaphore.Release();
                }
            });
            await Task.WhenAll(syncTasks);
        }
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
Abstraer en el contexto **"Customer Master Data (Bounded Context)"**. Este agente se convertirá en un microservicio ligero que consume de un clúster de mensajería (Kafka/RabbitMQ) el evento de mutación de un grupo de clientes, eliminando procesos "pull" masivos por intervalos.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.CustomerGroup.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.CustomerGroup.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.CustomerGroup.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.CustomerGroup.dll"]
```

### FASE 4: PIPELINE CI/CD, SEGURIDAD Y HEALTHCHECKS
```yaml
name: Deploy CustomerGroup Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/customer-group-agent:${{ github.sha }} .
```
*(Monitoreo liveness/readiness en Kubernetes validando el archivo de Heartbeat temporal).*
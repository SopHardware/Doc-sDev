=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceAgenteCondicionPago

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES ESTRICTAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` La dependencia de `ConfigurationManager` en .NET Framework impide inyectar configuraciones dinámicamente mediante variables de entorno nativas de Docker/K8s.
2. **Concurrencia (Factor 8):** `[Bloqueante]` El uso intensivo de bloqueos `.Wait()` y `ThreadPool` provocará una inanición de hilos (*thread pool starvation*) en entornos contenerizados con recursos limitados.
3. **Logs (Factor 11):** `[No Cumple]` La telemetría anclada al Event Viewer de Windows aislará los logs, impidiendo que herramientas como Fluentd/Prometheus puedan ingerirlos.
4. **Estado (Factor 6):** `[Cumple]` La lógica es *stateless*, posibilitando la escalabilidad horizontal una vez refactorizada la conectividad asíncrona.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
public async void ToOrigin(object model)
{
    Thread.CurrentThread.IsBackground = false;
    List<object> oModel = (List<object>)model;
    string oPlant = oModel[0].ToString();
    List<Tbl_Terms> oDestino = (List<Tbl_Terms>)oModel[1];
    
    // Riesgo: Inicialización costosa en cada iteración
    ProcessRepository process = new ProcessRepository(ServiceName, IsServices);
    
    // Riesgo: Bloqueo asíncrono destructivo (Sync-over-Async)
    bool success = await process.DestinyAsync(oModel);
}

// BUG EN ENTIDAD (Tbl_Terms.cs)
public override bool Equals(object obj)
{
    var other = obj as Tbl_Terms;
    if (other == null) return false;
    // CRÍTICO: Operador != en lugar de == provoca falsos negativos
    return this.TermsCode != other.TermsCode && this.Description != other.Description; 
}
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO)
```csharp
// CORRECCIÓN ENTIDAD
public override bool Equals(object obj)
{
    if (obj is not Tbl_Terms other) return false;
    return this.TermsCode == other.TermsCode && this.Description == other.Description;
}

// SERVICIO DESACOPLADO Y ASÍNCRONO
public class TermsSyncManager : ITermsSyncManager
{
    private readonly IProcessRepository _repository;
    private readonly ILogger<TermsSyncManager> _logger;

    public TermsSyncManager(IProcessRepository repository, ILogger<TermsSyncManager> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task ProcessSyncAsync(CancellationToken cancellationToken)
    {
        var destinationData = await _repository.GetPendingTermsAsync();
        var byPlant = destinationData.GroupBy(g => g.Plant).ToDictionary(d => d.Key, d => d.ToList());

        // Concurrencia controlada para evitar Thread Pool Starvation
        using var semaphore = new SemaphoreSlim(5); 
        var apiTasks = byPlant.Select(async plantData =>
        {
            await semaphore.WaitAsync(cancellationToken);
            try
            {
                var modelToSend = new List<object> { plantData.Key, plantData.Value };
                await _repository.DestinyAsync(modelToSend);
            }
            finally
            {
                semaphore.Release();
            }
        });
        
        await Task.WhenAll(apiTasks);
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
Implementar un contexto **"Financial Terms Bounded Context"**. Aislar la sincronización de Condiciones de Pago en un API/Worker autónomo que reciba eventos desde Epicor a través de un bus de mensajes (RabbitMQ) en lugar de consultar iterativamente la base de datos local.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.CondicionPago.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.CondicionPago.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.CondicionPago.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.CondicionPago.dll"]
```

### FASE 4: PIPELINE CI/CD, SEGURIDAD Y HEALTHCHECKS
```yaml
name: Deploy CondicionPago Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/condicionpago-agent:${{ github.sha }} .
```
*(Sondas Liveness para Kubernetes validarán la existencia del archivo temporal de Heartbeat generado por el Worker).*
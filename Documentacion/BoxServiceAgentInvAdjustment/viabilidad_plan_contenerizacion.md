# Plan de Modernización y Viabilidad de Contenedores

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**Dictamen:** **VIABLE CON CONDICIONES**

La aplicación actual en su estado "Tal cual" (As-Is) **NO ES VIABLE** para ser ejecutada en contenedores Linux estándar debido a su dependencia del .NET Framework 4.7.2, `ServiceBase` (Servicio de Windows) y el uso de `EventLog`. Podría ejecutarse en *Windows Containers*, pero estos son pesados, lentos y anti-patrones para microservicios modernos.

Para lograr viabilidad hacia contenedores Linux ligeros en Kubernetes, se debe cumplir con la **Condición de Migración:** Portar el proyecto a **.NET 8 Worker Service**. 

**Evaluación 12-Factor App:**
❌ **Configuración:** Almacenada en código (`App.config`), debe moverse a variables de entorno.
❌ **Logs:** Enviados a Event Viewer, deben enrutarse a STDOUT (Consola).
❌ **Concurrencia:** Diseño actual no permite levantar múltiples contenedores sin duplicar procesamiento (requiere RabbitMQ / Service Bus en el futuro).
✅ **Backing Services:** Conexiones a DB y APIs tratadas como recursos conectables.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Migración a Worker Service y DI)

Se debe refactorizar el núcleo para eliminar `ConfigurationManager` y acoplamientos.

**CÓDIGO ANTES (Acoplado):**
```csharp
// Dentro de ServiceAgentInvAdjustment.cs
CompanyParnet = ConfigurationManager.AppSettings["CompanyParnet"];
// ...
management = new InvAdjustmentManagement(ServiceNameAgent, CompanyParnet, WareHouseParnet, true);

// Dentro de InvAdjustmentManagement.cs
public void WriteEntry(string msg, EventLogEntryType entryType = EventLogEntryType.Information)
{
    // ...
    EventLog.WriteEntry(msg, entryType);
}
```

**CÓDIGO DESPUÉS (.NET 8 Worker Service + ILogger + IOptions):**
```csharp
// Program.cs (.NET 8)
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.Configure<AgentSettings>(hostContext.Configuration.GetSection("AgentSettings"));
        services.AddTransient<IInvAdjustmentManagement, InvAdjustmentManagement>();
        services.AddHostedService<Worker>(); // Remplaza a ServiceBase
    })
    .Build();
host.Run();

// Worker.cs
public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private readonly IInvAdjustmentManagement _management;

    public Worker(ILogger<Worker> logger, IInvAdjustmentManagement management)
    {
        _logger = logger;
        _management = management;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Ejecutando proceso de ajuste a las: {time}", DateTimeOffset.Now);
            await _management.CreateAndProcessTaskListAdjustment(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(4), stoppingToken); // Reemplaza Timer
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
Actualmente el agente actúa como un orquestador cronometrado (CronJob/Worker). Para que escale horizontalmente (ej: Múltiples bodegas al mismo tiempo):
1. **Separar el Productor del Consumidor:** Un microservicio (Productor) extrae los identificadores de la Vista SQL y los coloca en una cola (RabbitMQ, Kafka o Azure Service Bus).
2. **Workers Escalables (Consumidores):** El código actual (`BoxServiceAgentInvAdjustment`) se suscribe a la cola, toma un registro de forma exclusiva, lo procesa y lo envía al API del orquestador Parnet, evitando procesamientos dobles.

### Fase 3: Dockerización

**Dockerfile (Multi-stage para .NET 8 Worker):**
```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BoxServiceAgentInvAdjustment/BoxServiceAgentInvAdjustment.csproj", "BoxServiceAgentInvAdjustment/"]
RUN dotnet restore "BoxServiceAgentInvAdjustment/BoxServiceAgentInvAdjustment.csproj"
COPY . .
WORKDIR "/src/BoxServiceAgentInvAdjustment"
RUN dotnet build "BoxServiceAgentInvAdjustment.csproj" -c Release -o /app/build

# Publish Stage
FROM build AS publish
RUN dotnet publish "BoxServiceAgentInvAdjustment.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Final Image Stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
# No exponer puertos ya que es un proceso en background (Worker)
ENTRYPOINT ["dotnet", "BoxServiceAgentInvAdjustment.dll"]
```

**.dockerignore:**
```text
**/.classpath
**/.dockerignore
**/.env
**/.git
**/.gitignore
**/.project
**/.settings
**/.toolstarget
**/.vs
**/.vscode
**/*.*proj.user
**/*.dbmdl
**/*.jfm
**/bin
**/obj
**/logs
```

**docker-compose.yml (Entorno Local):**
```yaml
version: '3.8'
services:
  agent-inv-adjustment:
    build: 
      context: .
      dockerfile: BoxServiceAgentInvAdjustment/Dockerfile
    environment:
      - AgentSettings__CompanyParnet=BOXITO
      - AgentSettings__IntervaloEjecucion=4
      - ConnectionStrings__cnxepicorboxito=${DB_CONNECTION_STRING}
      - SnapShot__Authorization=${API_TOKEN}
    restart: always
```

### Fase 4: Pipeline CI/CD (GitHub Actions) y K8s Probes

**.github/workflows/ci-cd-docker.yml:**
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "DEV.VELA", "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
        
    - name: Restore dependencies
      run: dotnet restore ./BoxServiceAgentInvAdjustment/BoxServiceAgentInvAdjustment.csproj
      
    - name: Build
      run: dotnet build ./BoxServiceAgentInvAdjustment/BoxServiceAgentInvAdjustment.csproj --configuration Release --no-restore
      
    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./BoxServiceAgentInvAdjustment/Dockerfile
        push: true
        tags: tunombre/box-agent-inv:${{ github.sha }}
```

**Sondas en Kubernetes (Liveness/Readiness):**
Para un Worker Service sin API HTTP expuesta, ASP.NET Core 8 introduce Health Checks para Workers escribiendo un archivo en disco o exponiendo un puerto ínfimo. Si se expone un puerto mínimo (ej: 8080) para el endpoint `/health`:

```yaml
# En deployment.yaml de K8s
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 15
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

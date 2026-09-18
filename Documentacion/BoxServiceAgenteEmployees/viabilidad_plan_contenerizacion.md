# Viabilidad y Plan de Modernización hacia Contenedores: BoxServiceAgenteEmployees

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**DICTAMEN FINAL: VIABLE CON CONDICIONES**

El código en su estado actual (.NET Framework 4.7.2 + `ServiceBase` de Windows) **NO ES VIABLE** para ser ejecutado en contenedores Linux de forma nativa ni cumple con los principios de una *12-Factor App*. 

Sin embargo, el paradigma de la aplicación (un proceso en background o *daemon* que se ejecuta por intervalos) es altamente compatible con Kubernetes (ej. mediante *CronJobs* o un pod con ciclo de vida persistente). 
**Condición Obligatoria para Contenerización:** Migrar el proyecto de `Windows Service (.NET Framework)` a un `Worker Service (.NET 6 o superior)` (Multiplataforma).

### Análisis de 12-Factor App:
- **I. Codebase:** Cumple.
- **III. Config:** **Falla.** Configuraciones fuertemente acopladas a `App.config`. Deben migrarse a Variables de Entorno.
- **VIII. Concurrency:** Parcial. El modelo de concurrencia actual acopla `Threadpool` manual, requiere modelo async/await puro de .NET moderno.
- **XI. Logs:** **Falla.** Uso acoplado de `EventLog` de Windows. En contenedores, los logs deben escribirse como flujos de eventos (event streams) hacia `stdout`/`stderr`.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Migración a Worker Service y Config/Logs)

Se debe reescribir la entrada de la aplicación y la inyección del logger para cumplir con los factores III y XI.

**CÓDIGO ANTES (Acoplado a Windows y App.config):**
```csharp
// Program.cs
static void Main() {
    ServiceBase[] ServicesToRun = new ServiceBase[] { new ServiceAgenteEmployees() };
    ServiceBase.Run(ServicesToRun);
}

// ServiceAgenteEmployees.cs (Lectura manual)
ServiceNameAgent = ConfigurationManager.AppSettings.Get("ServiceName");
EventLog.WriteEntry(ServiceNameAgent, "Inicia", EventLogEntryType.Information, 101, 1);
```

**CÓDIGO DESPUÉS (.NET 6/8+ Worker Service - Preparado para Contenedores):**
```csharp
// Program.cs (.NET 6+)
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddHostedService<Worker>();
        // DI para Repositorios
        services.AddTransient<IEmployeeRepository, EmployeeRepository>();
    })
    .ConfigureLogging(logging =>
    {
        logging.ClearProviders();
        logging.AddConsole(); // Flujo stdout para Docker/K8s (Factor XI)
    })
    .Build();

await host.RunAsync();
```

```csharp
// Worker.cs
public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private readonly IConfiguration _configuration;

    public Worker(ILogger<Worker> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Las variables se inyectarán vía Variables de Entorno (Factor III)
        var intervalo = _configuration.GetValue<int>("IntervaloEjecucion");

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Ejecutando proceso de Empleados en: {time}", DateTimeOffset.Now);
            // Lógica de Managements.ProcesoAsync()
            await Task.Delay(TimeSpan.FromMinutes(intervalo), stoppingToken);
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
Actualmente el servicio sincroniza `Employees` (PerCon). La arquitectura monolítica y las carpetas `_SRV_POST_EMPLOYEE` (obsoletas) indican que es candidato a ser un microservicio independiente (ej. `employee-sync-service`).
1. **Desacoplar la BD:** Aislar la capa de lectura (EpicorTest) y la de escritura/estado (EpicorBoxito).
2. **Resiliencia HTTP:** Envolver la llamada HTTP al Orquestador Parnet con librerías de resiliencia como **Polly** (Retry y Circuit Breaker), eliminando el frágil manejo de estado actual.

### Fase 3: Dockerización

**Dockerfile Multi-stage (Optimizado para .NET Worker):**
```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
# Copiar csproj y restaurar dependencias
COPY ["BoxServiceAgenteEmployees.csproj", "./"]
RUN dotnet restore "BoxServiceAgenteEmployees.csproj"
# Copiar todo el código y construir
COPY . .
RUN dotnet build "BoxServiceAgenteEmployees.csproj" -c Release -o /app/build
RUN dotnet publish "BoxServiceAgenteEmployees.csproj" -c Release -o /app/publish

# Run Stage
FROM mcr.microsoft.com/dotnet/runtime:8.0
WORKDIR /app
COPY --from=build /app/publish .

# Usuario no root por seguridad
RUN adduser --disabled-password --gecos "" appuser
USER appuser

ENTRYPOINT ["dotnet", "BoxServiceAgenteEmployees.dll"]
```

**.dockerignore:**
```text
bin/
obj/
.git/
.vscode/
*.user
*.suo
```

**docker-compose.yml (Para desarrollo local):**
```yaml
version: '3.8'
services:
  employee-sync:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - ConnectionStrings__conexion=Data Source=db;Initial Catalog=EpicorBoxito;User ID=sa;Password=tuPassword!
      - OrquestadorApi=http://orquestador-mock:8080/
      - IntervaloEjecucion=2
    restart: unless-stopped
```

### Fase 4: Pipeline CI/CD (GitHub Actions) y Sondas K8s

**GitHub Actions (ci.yml):**
```yaml
name: CI/CD Pipeline para Employee Sync Service

on:
  push:
    branches: [ "main" ]

jobs:
  build_and_push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
        
    - name: Restore dependencies
      run: dotnet restore
      
    - name: Build
      run: dotnet build --no-restore --configuration Release
      
    # Asumiendo que existen pruebas unitarias, aunque la auditoría detectó que no las hay.
    # - name: Test
    #   run: dotnet test --no-build --verbosity normal
      
    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v3
      with:
        context: .
        push: true
        tags: tunamespace/employee-sync:latest
```

**Sondas Liveness/Readiness para Kubernetes (Deployment):**
Como este es un proceso Worker (Background Service) sin endpoints HTTP, las sondas de Kubernetes deben configurarse ejecutando un comando interno o implementando un servidor HTTP ligero (HealthCheck). En .NET 6+, se puede inyectar `Microsoft.Extensions.Diagnostics.HealthChecks`.

```yaml
# Fragmento del manifiesto Deployment.yaml de K8s
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 15
          periodSeconds: 20
```
*(Nota: El Worker deberá tocar o actualizar el archivo `/tmp/healthy` en cada iteración del bucle para indicar a Kubernetes que no está colgado).*
# Viabilidad y Plan de Contenerización
**Aplicación:** BoxServiceAgenteCustomerMotofit (Clientes Motofit)

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)
**Estado:** **VIABLE CON CONDICIONES (REQUIERE REFACTORIZACIÓN)**

**Justificación según 12-Factor App:**
*   **III. Configuraciones:** Falla. Actualmente usa `App.config` fuertemente acoplado al Framework. En un contenedor, debe usar Variables de Entorno.
*   **XI. Logs:** Falla. Intenta usar el `EventLog` de Windows y la Consola de forma incorrecta. Docker y K8s esperan flujos continuos hacia `stdout`/`stderr`.
*   **VIII. Concurrencia (Procesos sin estado):** Pasa parcialmente, el servicio hace polling y no guarda estado, pero al heredar de `ServiceBase` restringe su ejecución a entornos con el Gestor de Servicios de Windows (SCM), lo que hace ineficiente y pesado su uso nativo en contenedores Linux.

**Condición Obligatoria:** Migrar la base del código de un proyecto Windows Service (.NET Framework 4.7.2) a un **Worker Service** en .NET Core / .NET 8 (Multiplataforma) antes de crear la imagen de Docker.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Migración a Worker Service)

**Código ANTES (Dependiente de Windows):**
```csharp
// Program.cs
static void Main() {
    ServiceBase[] ServicesToRun = new ServiceBase[] { new ServiceAgente() };
    ServiceBase.Run(ServicesToRun);
}

// ServiceAgente.cs
public partial class ServiceAgente : ServiceBase {
    protected override void OnStart(string[] args) {
        Timer timer = new Timer();
        timer.Interval = 180000;
        timer.Elapsed += new ElapsedEventHandler(ProcesosAgente);
        timer.Start();
    }
}
```

**Código DESPUÉS (.NET 8 Worker Service + Dependency Injection):**
```csharp
// Program.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddHostedService<Worker>();
        services.AddTransient<ISucursalManagement, SucursalManagement>();
        services.AddTransient<IClienteRepository, ClienteRepository>();
    })
    .Build()
    .Run();

// Worker.cs
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System.Threading;
using System.Threading.Tasks;

public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private readonly ISucursalManagement _sucursalManagement;

    public Worker(ILogger<Worker> logger, ISucursalManagement sucursalManagement)
    {
        _logger = logger;
        _sucursalManagement = sucursalManagement;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Ejecutando proceso a las: {time}", DateTimeOffset.Now);
            
            // Lógica de negocio
            _sucursalManagement.AddSucursal();
            
            // Retardo seguro que previene reentrada
            await Task.Delay(180000, stoppingToken);
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
Actualmente el agente maneja Sucursales, Clientes y Embarques. 
1. **Desacoplamiento de Módulos:** Crear interfaces separadas para cada Management.
2. **Event-Driven:** En lugar de ejecutar todos en serie cada 3 minutos, evaluar suscribirse a eventos desde Epicor ERP si es soportado (Webhooks/Colas RabbitMQ).
3. **Escalabilidad:** Al migrar a K8s, si el volumen de clientes es alto, se pueden desplegar réplicas del worker leyendo de una cola de mensajes en lugar de extraer masivamente de la BD.

### Fase 3: Dockerización

**Archivo: `Dockerfile` (Multi-stage para imagen liviana de Linux)**
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BoxAgenteMotoFit/BoxAgenteMotoFit.csproj", "BoxAgenteMotoFit/"]
RUN dotnet restore "BoxAgenteMotoFit/BoxAgenteMotoFit.csproj"
COPY . .
WORKDIR "/src/BoxAgenteMotoFit"
RUN dotnet build "BoxAgenteMotoFit.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "BoxAgenteMotoFit.csproj" -c Release -o /app/publish

# Final runtime image
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
# Configuración via variables de entorno
ENV ConnectionStrings__conexion="Server=db_host;Database=EpicorERPTest;User Id=sa;Password=tu_password;"
ENTRYPOINT ["dotnet", "BoxAgenteMotoFit.dll"]
```

**Archivo: `docker-compose.yml`**
```yaml
version: '3.8'
services:
  agente-motofit:
    build: 
      context: .
      dockerfile: Dockerfile
    environment:
      - ConnectionStrings__conexion=${DB_CONNECTION}
      - AppSettings__UrlOrquestador=${URL_ORQUESTADOR}
      - AppSettings__Authorization=${AUTH_TOKEN}
    restart: always
```

**Archivo: `.dockerignore`**
```text
bin/
obj/
.vs/
*.user
*.suo
TestResults/
```

### Fase 4: Pipeline CI/CD (GitHub Actions) y Sondas K8s

**Archivo: `.github/workflows/deploy.yml`**
```yaml
name: CI/CD Docker Build & Push

on:
  push:
    branches: [ "DEV.VELA" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x

    - name: Build and Test
      run: |
        dotnet build --configuration Release
        # dotnet test

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: ./Componentes-Observer/Agentes/Parnet-Orquestador-Base/BoxAgenteMotoFit/AgenteMotoFit
        push: true
        tags: empresa/agente-motofit:${{ github.sha }}, empresa/agente-motofit:latest
```

**Sondas para Kubernetes (Liveness / Readiness):**
Como los Worker Services no exponen puertos HTTP por defecto, se recomienda agregar la librería `Microsoft.Extensions.Diagnostics.HealthChecks` y exponer un micro puerto HTTP para que K8s sepa si el servicio sigue vivo.
```yaml
# En el Deployment de K8s:
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```
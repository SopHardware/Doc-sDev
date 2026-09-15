# Plan de Modernización hacia Contenedores
**Aplicación:** BoxServiceAgenteVendor (Proveedores)

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES
**Dictamen:** **NO VIABLE EN SU ESTADO ACTUAL (Requiere Refactorización Profunda)**

**Justificación basada en 12-Factor App:**
1. **Configuraciones (Factor 3):** Actualmente utiliza `ConfigurationManager.AppSettings` (presumiblemente un `App.config`), lo cual no es compatible de forma nativa y limpia con variables de entorno de contenedores (Docker/K8s).
2. **Dependencias del SO (Factor 9 / Disposability):** Hereda de `ServiceBase` e instala hooks nativos (`ProjectInstaller.cs`) obligando a su ejecución exclusiva en Windows. Escribe directamente en el visor de eventos de Windows (`EventLog.WriteEntry`), impidiendo la salida estándar a la consola que requieren los contenedores.
3. **Concurrencia (Factor 8):** El manejo inestable de hilos junto con NHibernate causará fugas de memoria o caídas abruptas dentro del contenedor que engañarán al orquestador (K8s).

**Transición:** Es viable condicionada a su re-escritura hacia el modelo **Worker Service** de .NET 6/8.

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Migración a Worker Service)

**CÓDIGO ANTES (Service.cs actual, atado a Windows):**
```csharp
public partial class ServiceAgente : ServiceBase
{
    protected override void OnStart(string[] args)
    {
        Timer timer = new Timer();
        timer.Interval = 1000 * 60 * 60; // 1 Hora
        timer.Elapsed += new ElapsedEventHandler(ProcesosAgente);
        timer.Start();
    }
    public async void ProcesosAgente(object sender, ElapsedEventArgs e)
    {
        EventLog.WriteEntry("Inicio Importación");
        VendorManagement vendor = new VendorManagement();
        // ...
    }
}
```

**CÓDIGO DESPUÉS (.NET 8 Worker Service con Inyección de Dependencias):**
```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System.Threading;
using System.Threading.Tasks;

namespace BoxServiceAgenteVendor.Worker
{
    public class VendorSyncWorker : BackgroundService
    {
        private readonly ILogger<VendorSyncWorker> _logger;
        private readonly IVendorManagement _vendorManagement;

        public VendorSyncWorker(ILogger<VendorSyncWorker> logger, IVendorManagement vendorManagement)
        {
            _logger = logger;
            _vendorManagement = vendorManagement;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                _logger.LogInformation("Inicio Importación de Proveedores: {time}", DateTimeOffset.Now);
                try
                {
                    await _vendorManagement.ProcessAsync(stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error durante la sincronización");
                }
                // Esperar 1 hora
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
1. **Desacoplamiento de NHibernate:** Encapsular NHibernate y registrar `ISessionFactory` como Singleton en el contenedor IoC, inyectando `ISession` por Scope temporal o manejar la sesión dentro del Repository explícitamente para procesos en background largos.
2. **Eliminación de EventLog:** Configurar `Serilog` para volcar logs a `Console` estructurados en formato JSON.
3. **Gestión de Secretos:** Migrar connection strings a variables de entorno inyectadas nativamente por `.NET Core ConfigurationProvider`.

### Fase 3: Dockerización

**Dockerfile (Multi-stage build para Linux Alpine):**
```dockerfile
# Etapa de construcción
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["BoxServiceAgenteVendor/BoxServiceAgenteVendor.csproj", "BoxServiceAgenteVendor/"]
RUN dotnet restore "BoxServiceAgenteVendor/BoxServiceAgenteVendor.csproj"
COPY . .
WORKDIR "/src/BoxServiceAgenteVendor"
RUN dotnet build "BoxServiceAgenteVendor.csproj" -c Release -o /app/build

# Etapa de publicación
FROM build AS publish
RUN dotnet publish "BoxServiceAgenteVendor.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa final (Runtime)
FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Usuario no root por seguridad
RUN addgroup -g 1000 appgroup && adduser -u 1000 -G appgroup -D appuser
USER appuser

ENTRYPOINT ["dotnet", "BoxServiceAgenteVendor.dll"]
```

**.dockerignore:**
```text
.git
.vs
bin/
obj/
*.user
*.suo
appsettings.Development.json
```

**docker-compose.yml (Entorno Local/Pruebas):**
```yaml
version: '3.8'
services:
  agente-vendor:
    build: 
      context: .
      dockerfile: BoxServiceAgenteVendor/Dockerfile
    environment:
      - ConnectionStrings__EpicorDb=Server=server;Database=epicor;User Id=user;Password=pass;
      - ConnectionStrings__ParnetDb=Server=server;Database=parnet;User Id=user;Password=pass;
    restart: always
```

### Fase 4: Pipeline CI/CD y Preparación para Kubernetes

**GitHub Actions YAML (Pipeline CI):**
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "DEV.VELA" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Set up .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'

    - name: Build
      run: dotnet build --configuration Release

    # ⚠️ Nota: Información no proporcionada sobre pruebas unitarias.
    # - name: Test
    #   run: dotnet test --no-build --verbosity normal

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: BoxServiceAgenteVendor/Dockerfile
        push: true
        tags: micuenta/agente-vendor:latest
```

**Sondas en Kubernetes (Probes):**
Como los `Worker Services` puros no exponen puertos HTTP, la sonda de liveness se recomienda habilitar mediante el paquete `Microsoft.Extensions.Diagnostics.HealthChecks` y exponiendo un puerto ligero en el background, o mediante el chequeo del proceso:
```yaml
# En deployment.yaml de k8s
livenessProbe:
  exec:
    command:
    - pgrep
    - dotnet
  initialDelaySeconds: 15
  periodSeconds: 20
```
Alternativamente, exponer un mini endpoint de salud en el puerto `8080` (preferido en K8s modernos).
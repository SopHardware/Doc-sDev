# Dictamen y Plan de Modernización hacia Contenedores

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**DICTAMEN: VIABLE CON CONDICIONES (REQUIERE MIGRACIÓN)**

**Justificación basada en 12-Factor App:**
Actualmente, el proyecto está fuertemente acoplado al ecosistema de Windows (hereda de `ServiceBase` y utiliza `EventLog`). Para ser contenedorizado de forma eficiente (idealmente en contenedores Linux para reducir costos y overhead), la aplicación **debe migrarse** de .NET Framework a **.NET Core / .NET 6+** utilizando la plantilla de **Worker Service**.

*   **I. Codebase:** Cumple. Es un único repositorio.
*   **III. Config:** No cumple. Utiliza `App.config` (XML). Debe migrarse a variables de entorno y `appsettings.json`.
*   **VI. Processes (Stateless):** Cumple parcialmente. El proceso en sí no guarda estado en memoria entre ejecuciones del Timer, depende de la BD.
*   **XI. Logs:** No cumple. Escribe en Windows Event Viewer. Los contenedores exigen que los logs fluyan a `stdout`/`stderr`.

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Código C#)
Migrar el proyecto a un **Worker Service** en .NET 6/8 y desacoplar el logging.

**CÓDIGO ANTES (Acoplado a Windows):**
```csharp
// Service.cs actual
public partial class Service : ServiceBase
{
    protected override void OnStart(string[] args)
    {
        double.TryParse(ConfigurationManager.AppSettings["IntervaloEjecucion"], out double intervalo);
        EventLog.WriteEntry(ServiceNameAgent, "Inicio", EventLogEntryType.Information);
        // ... timer start
    }
}
```

**CÓDIGO DESPUÉS (Listo para contenedores):**
```csharp
// Worker.cs (.NET 6+)
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using System.Threading;
using System.Threading.Tasks;

namespace BoxServiceAgenteResurtido
{
    public class Worker : BackgroundService
    {
        private readonly ILogger<Worker> _logger;
        private readonly IConfiguration _configuration;
        private readonly IRestorPartsManagement _management;

        public Worker(ILogger<Worker> logger, IConfiguration configuration, IRestorPartsManagement management)
        {
            _logger = logger;
            _configuration = configuration;
            _management = management;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            int intervaloMinutos = _configuration.GetValue<int>("IntervaloEjecucionMinutos", 4);
            _logger.LogInformation("Servicio iniciado. Intervalo: {intervalo}", intervaloMinutos);

            while (!stoppingToken.IsCancellationRequested)
            {
                _logger.LogInformation("Ejecutando proceso a las: {time}", DateTimeOffset.Now);
                try
                {
                    await _management.ProAsync(stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error durante la ejecución del proceso.");
                }
                
                await Task.Delay(TimeSpan.FromMinutes(intervaloMinutos), stoppingToken);
            }
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
El servicio actual ya opera como un *Worker de Sincronización* (cron job continuo). No requiere dividirse más, pero sí requiere separar la capa HTTP (Orquestador) de la capa de acceso a datos (Repositorio). Se recomienda implementarlo como un CronJob en Kubernetes en lugar de un deployment continuo, dejando que K8s maneje la calendarización si el intervalo es estático, o mantenerlo como un Pod constante si requiere intervalos menores a 1 minuto.

### Fase 3: Dockerización

**Archivo: `Dockerfile` (Multi-stage)**
```dockerfile
# Etapa de compilacion
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BoxServiceAgenteResurtido/BoxServiceAgenteResurtido.csproj", "BoxServiceAgenteResurtido/"]
# ⚠️ Información no proporcionada en la entrada: Se asume estructura de solución estándar y que las dependencias de proyectos (BO, Net, etc.) se copian aquí.
RUN dotnet restore "BoxServiceAgenteResurtido/BoxServiceAgenteResurtido.csproj"
COPY . .
WORKDIR "/src/BoxServiceAgenteResurtido"
RUN dotnet build "BoxServiceAgenteResurtido.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "BoxServiceAgenteResurtido.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa de produccion
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENV IntervaloEjecucionMinutos=4
ENTRYPOINT ["dotnet", "BoxServiceAgenteResurtido.dll"]
```

**Archivo: `docker-compose.yml` (Para desarrollo local)**
```yaml
version: '3.8'
services:
  agenteresurtido:
    build: 
      context: .
      dockerfile: BoxServiceAgenteResurtido/Dockerfile
    environment:
      - ConnectionStrings__conexionOrigen=Server=myServerAddress;Database=EpicorBoxito;User Id=myUsername;Password=myPassword;
      - ConnectionStrings__conexionDestino=Server=myServerAddress;Database=SyncDB;User Id=myUsername;Password=myPassword;
      - IntervaloEjecucionMinutos=4
    restart: unless-stopped
```

**Archivo: `.dockerignore`**
```
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
**/packages
```

### Fase 4: Pipeline CI/CD y Sondas Kubernetes

**Archivo: `.github/workflows/docker-build-push.yml` (GitHub Actions)**
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "main", "DEV.VELA" ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/agenteresurtido

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: BoxServiceAgenteResurtido/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

**Sondas para Kubernetes (Fragmento YAML Deployment):**
Dado que es un Background Worker, no expone un puerto HTTP por defecto para liveness/readiness. Para solucionarlo en Kubernetes, se debe agregar el paquete de HealthChecks de .NET para exponer un puerto ligero, o usar comandos en el contenedor.

```yaml
# Sondas requerirán exponer un mini servidor web en el Worker (IHostBuilder.ConfigureWebHostDefaults)
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
```
*(Nota: ⚠️ Información no proporcionada en la entrada: Si el servicio se mantiene como puro Worker de Framework sin endpoints HTTP, las sondas de K8s tendrían que basarse en revisar el tiempo de modificación de un archivo en disco que el Worker actualice en cada ciclo usando `exec`).*
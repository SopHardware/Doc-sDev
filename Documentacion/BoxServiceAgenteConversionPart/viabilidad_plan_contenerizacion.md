# Dictamen de Viabilidad y Plan de Contenerización
**Aplicación:** BoxServiceAgenteConversionPart

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES

**Resultado:** **VIABLE CON CONDICIONES**

**Justificación basada en "12-Factor App":**
1. **Codebase:** Se cuenta con un repositorio centralizado, pero el código es `.NET Framework 4.7.2`. Para que la dockerización sea liviana y nativa en Linux, **ES CONDICIÓN OBLIGATORIA** actualizar a `.NET 8 (Worker Service)`. Dockerizar .NET Framework 4.7.2 requeriría Windows Containers, los cuales son pesados, lentos y limitados en Kubernetes.
2. **Dependencies:** Los paquetes (NHibernate, NewtonSoft) son compatibles con .NET Standard/Core, por lo que la migración es factible.
3. **Config:** Actualmente en `App.config` (xml). Debe migrarse a Variables de Entorno (`Environment Variables`).
4. **Logs:** Actualmente escribe en el EventLog de Windows. Debe migrarse a `STDOUT/STDERR` estructurado.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Refactorización a Worker Service)
El primer paso es transformar el proyecto heredado a un modelo moderno con `IHostedService` y desacoplar responsabilidades.

**Código ANTES (Service.cs heredado acoplado a SO):**
```csharp
public class Service : ServiceBase
{
    protected override void OnStart(string[] args)
    {
        // Timer de sistema que no frena la ejecución
        Timer timer = new Timer();
        timer.Interval = 4 * 60 * 1000;
        timer.Elapsed += new ElapsedEventHandler(ProcesosAgente);
        timer.Start();
    }
}
```

**Código DESPUÉS (BackgroundService en .NET 8 con DI):**
```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.DependencyInjection;

public class ConversionPartWorker : BackgroundService
{
    private readonly ILogger<ConversionPartWorker> _logger;
    private readonly IServiceScopeFactory _scopeFactory;

    public ConversionPartWorker(ILogger<ConversionPartWorker> logger, IServiceScopeFactory scopeFactory)
    {
        _logger = logger;
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Servicio de conversión iniciado.");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            using (var scope = _scopeFactory.CreateScope())
            {
                var management = scope.ServiceProvider.GetRequiredService<IConversionManagement>();
                try 
                {
                    await management.ProcesoAsync(stoppingToken);
                }
                catch(Exception ex)
                {
                    _logger.LogError(ex, "Error crítico en ciclo de procesamiento.");
                }
            }
            
            // Intervalo tomado de variables de entorno
            await Task.Delay(TimeSpan.FromMinutes(4), stoppingToken);
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
1. **Desacoplar HTTP de Datos:** El `ConversionPartRepository` debe dividirse en:
   - `ConversionDbRepository` (Exclusivo para NHibernate).
   - `OrquestadorApiClient` (Exclusivo para HTTP).
2. **Patrón Saga / Outbox:** Enviar eventos al Orquestador primero y, de ser exitoso, marcar la actualización local. Evitar dos sesiones de BD distribuidas de forma manual.

### Fase 3: Dockerización

Una vez migrado a `.NET 8`, utilizaremos un build Multi-stage ligero.

**Archivo: `.dockerignore`**
```text
bin/
obj/
.vs/
*.suo
*.user
Logs/
```

**Archivo: `Dockerfile`**
```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BoxServiceAgenteConversionPart.csproj", "./"]
RUN dotnet restore "BoxServiceAgenteConversionPart.csproj"
COPY . .
RUN dotnet build "BoxServiceAgenteConversionPart.csproj" -c Release -o /app/build

# Publish Stage
FROM build AS publish
RUN dotnet publish "BoxServiceAgenteConversionPart.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime Stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Variables requeridas en entorno
ENV OrquestadorApi=""
ENV Authorization=""
ENV x-api-key=""
ENV ConnectionStrings__conexionOrigen=""
ENV ConnectionStrings__conexionDestino=""

ENTRYPOINT ["dotnet", "BoxServiceAgenteConversionPart.dll"]
```

**Archivo: `docker-compose.yml` (Para desarrollo local)**
```yaml
version: '3.8'
services:
  agente-conversion:
    build: 
      context: .
      dockerfile: Dockerfile
    environment:
      - OrquestadorApi=http://10.40.3.30:8080/
      - Authorization=${API_AUTH_TOKEN}
      - x-api-key=${API_KEY}
      - ConnectionStrings__conexionOrigen=Server=10.40.3.72;Database=EpicorERPPilot;User Id=interfacesparnet;Password=${DB_PASS};
      - ConnectionStrings__conexionDestino=Server=10.40.3.72;Database=EpicorBoxito;User Id=interfacesparnet;Password=${DB_PASS};
    restart: always
```

### Fase 4: Pipeline CI/CD y Kubernetes

**Archivo: `.github/workflows/ci-cd.yml` (Pipeline de GitHub Actions)**
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "DEV.VELA" ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/agente-conversion-part

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: ./AgenteConversionParte/SRV_POST_CONVERSIONSPART/
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}, ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

**Sondas para Kubernetes (Liveness / Readiness):**
Para que K8s evalúe la salud de un Worker Service (que no expone puerto HTTP por defecto), se recomienda agregar un endpoint mínimo (HealthCheck) vía IHost, o que el servicio escriba un archivo de timestamp `/tmp/healthy` cada vez que el ciclo del `while` se completa exitosamente.

```yaml
# Fragmento del Deployment.yaml de Kubernetes
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 30
  periodSeconds: 60
```
*(Nota: El WorkerService debe modificarse para tocar este archivo en cada loop exitoso).*
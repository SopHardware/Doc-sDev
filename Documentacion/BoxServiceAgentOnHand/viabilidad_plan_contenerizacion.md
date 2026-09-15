# 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)
**DICTAMEN: NO VIABLE (EN SU ESTADO ACTUAL)**

El código actual hereda de `System.ServiceProcess.ServiceBase` (Windows Service clásico) y utiliza `System.Diagnostics.EventLog` interactuando de forma exclusiva con el sistema operativo Windows. Adicionalmente, depende de `ConfigurationManager.AppSettings` (clásico de .NET Framework/XML config). 

Para poder contenerizar la aplicación bajo estándares Cloud-Native (Linux Containers en Kubernetes/Docker), es estrictamente necesario ejecutar una refactorización previa (Remediación) hacia el modelo moderno de Worker Service (C# 10+, .NET 6 o superior). ⚠️ *Nota: Se asume que el marco de trabajo destino es al menos .NET 6 o .NET 8, ya que la entrada es un proyecto acoplado a .NET Framework Legacy.*

**Justificación 12-Factor App:**
1. **Configuración (Violación):** Configuración atada a archivos estáticos locales `App.config`. Debe delegarse a variables de entorno inyectadas en tiempo de ejecución.
2. **Logs (Violación):** Escribe exclusivamente en el Visor de Eventos de Windows (`EventsLog.cs`). Debe emitir flujos de eventos (streams) a `stdout` / `stderr`.
3. **Backing Services (Violación):** Acoplamiento de configuración en código y cadenas quemadas sin un mecanismo claro de sobreescritura externa.

# 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Migración a Worker Service)

**Código ANTES (Windows Service acoplado):**
```csharp
using System.ServiceProcess;
using System.Diagnostics;
namespace BoxServiceAgentOnHand {
    public partial class ServiceAgentOnHand : ServiceBase {
        protected override void OnStart(string[] args) {
            EventLog.WriteEntry("Agent", "Iniciando servicio", EventLogEntryType.Information);
            // Timer y lanzamiento...
        }
    }
}
```

**Código DESPUÉS (.NET 6+ Worker Service):**
```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace BoxServiceAgentOnHand
{
    public class OnHandWorker : BackgroundService
    {
        private readonly ILogger<OnHandWorker> _logger;
        private readonly IPartWhseManagement _management;
        private readonly AgentSettings _settings;

        public OnHandWorker(ILogger<OnHandWorker> logger, IPartWhseManagement management, IOptions<AgentSettings> settings)
        {
            _logger = logger;
            _management = management;
            _settings = settings.Value;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("Servicio OnHand iniciado en: {time}", DateTimeOffset.Now);
            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    // Delegar el procesamiento al manager orquestador
                    await _management.CreateAndProcessTaskListSnapShot();
                }
                catch(Exception ex)
                {
                    _logger.LogError(ex, "Error crítico en orquestación de snapshot.");
                }
                
                await Task.Delay(TimeSpan.FromMinutes(_settings.IntervaloEjecucionMinutos), stoppingToken);
            }
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios
1. **Capa de Extracción (ETL):** El acoplamiento a las BBDD de Epicor y Boxito a través de FluentNHibernate es pesado. Podría separarse en un microservicio de Extracción asíncrono, dejando al Worker orquestador como puro despachador de eventos.
2. **Capa de Envío Reactiva:** En lugar de saturar con `Task.WhenAll` consultas sincrónicas masivas, el despachador de sucursales (`Plant`) debería publicar mensajes en un Message Broker (RabbitMQ o Kafka). Otros Workers consumirían dichos mensajes procesando las llamadas a `Orquestador API` y escalando horizontalmente según la carga.

### Fase 3: Dockerización

**Dockerfile (Multi-stage build para .NET 6/8 Linux):**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Restauración de paquetes
COPY ["BoxServiceAgentOnHand.csproj", "./"]
RUN dotnet restore "BoxServiceAgentOnHand.csproj"

# Copia de código y publicación
COPY . .
RUN dotnet publish "BoxServiceAgentOnHand.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .

# Usuario sin privilegios
USER app
ENTRYPOINT ["dotnet", "BoxServiceAgentOnHand.dll"]
```

**.dockerignore:**
```text
bin/
obj/
.git/
.vs/
*.user
*.suo
```

**docker-compose.yml:**
```yaml
version: '3.8'
services:
  agent-onhand:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - ConnectionStrings__cnxepicorerp=Server=db1;Database=erp;User Id=sa;Password=secret;
      - ConnectionStrings__cnxepicorboxito=Server=db2;Database=boxito;User Id=sa;Password=secret;
      - AgentSettings__IntervaloEjecucionMinutos=10
    restart: always
```

### Fase 4: Pipeline CI/CD y Sondas K8s

**GitHub Actions (ci-cd.yaml):**
```yaml
name: CI/CD Docker Build and Push

on:
  push:
    branches: [ "main", "DEV.VELA" ]

jobs:
  build:
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
      run: dotnet build --no-restore -c Release
      
    - name: Test
      run: dotnet test --no-build --verbosity normal
      
    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: miempresa/agentonhand:${{ github.sha }}
```

**Sondas Kubernetes (Liveness/Readiness):**
*⚠️ Información no proporcionada en la entrada: Como el código original es un servicio en segundo plano, no levanta puertos. Para soportar sondas K8s, es necesario exponer un puerto en el nuevo Worker.*
Se implementa habilitando un pequeño host de Health Checks (`UseHealthChecks`) en el Worker de .NET Core.

```yaml
# Fragmento del Deployment en K8s para las sondas
livenessProbe:
  httpGet:
    path: /health/alive
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 15
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

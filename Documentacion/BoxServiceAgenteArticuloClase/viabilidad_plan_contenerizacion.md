# DICTAMEN Y HOJA DE RUTA DE CONTENERIZACIÓN
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**Dictamen Explícito:** `[VIABLE CON CONDICIONES]`

**Justificación Técnica Estricta (12-Factor App):**
1.  **Estado Efímero:** El agente parece no requerir guardar archivos locales, lo cual es positivo (Stateless), pero su dependencia del Visor de Eventos de Windows para el estado operativo es un bloqueante. En un contenedor efímero, el *EventLog* no existe (o si se emula en Windows Containers, muere al caer el Pod).
2.  **Configuraciones:** El código exige leer archivos físicos (`App.config`). Docker/Kubernetes inyectan configuraciones mediante variables de entorno (ENV) y ConfigMaps. Esto obliga a reestructurar la capa de inicialización.
3.  **Acoplamiento de Procesos:** La clase `ServiceBase` espera señales de control de los servicios de Windows. Los contenedores Docker envían señales POSIX (`SIGTERM`). Si el agente no se reescribe, Docker no podrá apagar el contenedor con gracia (Graceful Shutdown) y corromperá transacciones en la base de datos al enviar un `SIGKILL` forzado.

⚠️ *Condición absoluta:* La aplicación actual (WinExe, .NET 4.7.2) no debe contenerizarse en `Windows Server Core` (antipatrón de gran tamaño y lenta inicialización). Requiere una refactorización previa a `.NET 8 (Linux Alpine)` detallada en la Fase 1.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria del Código (Refactorización a BackgroundService)

**ANTES (Código con deuda técnica, acoplado y atado a Windows):**
```csharp
using System;
using System.Configuration;
using System.Diagnostics;
using System.ServiceProcess;
using System.Threading.Tasks;
using System.Timers;
using Managements; // Acoplamiento directo a implementaciones

namespace BoxServiceAgenteArticuloClase
{
    public partial class ServiceAgenteArticuloClase : ServiceBase
    {
        public string ServiceNameAgent { get; set; }
        public string Company { get; set; }
        private Task TaskGral { get; set; }
        private Timer timer;

        public ServiceAgenteArticuloClase() { InitializeComponent(); }

        protected override void OnStart(string[] args)
        {
            ServiceNameAgent = ConfigurationManager.AppSettings.Get("ServiceName");
            Company = ConfigurationManager.AppSettings.Get("Company");
            timer = new Timer();
            timer.Interval = 240000;
            timer.Elapsed += new ElapsedEventHandler(ProcesosAgente);
            timer.Start();
        }

        public void ProcesosAgente(object sender, ElapsedEventArgs e)
        {
            if (TaskGral == null || TaskGral.IsCompleted)
            {
                try
                {
                    // Violación DIP - Acoplamiento Duro
                    PartClassManagement p = new PartClassManagement(Company, ServiceNameAgent, true);
                    TaskGral = p.ProAsync();
                }
                catch (Exception ex)
                {
                    // Destrucción del stacktrace y dependencia local
                    EventLog.WriteEntry(ServiceNameAgent, ex.Message, EventLogEntryType.Error, 101, 1);
                }
            }
        }
    }
}
```

**DESPUÉS (Código Cloud-Native, SOLID, listo para contenedores Linux - .NET 8):**
```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using System;
using System.Threading;
using System.Threading.Tasks;
// IPartClassManagement debe ser extraída de Managements.PartClassManagement
using Managements.Interfaces; 

namespace BoxAgenteCloud
{
    public class BoxAgenteWorker : BackgroundService
    {
        private readonly ILogger<BoxAgenteWorker> _logger;
        private readonly IPartClassManagement _partClassManagement;
        private readonly string _company;
        private readonly string _serviceName;
        private readonly int _intervaloMs;

        // Inyección de dependencias y configuración por Variables de Entorno
        public BoxAgenteWorker(
            ILogger<BoxAgenteWorker> logger, 
            IPartClassManagement partClassManagement, 
            IConfiguration config)
        {
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
            _partClassManagement = partClassManagement ?? throw new ArgumentNullException(nameof(partClassManagement));
            
            _company = config.GetValue<string>("AGENT_COMPANY") 
                       ?? throw new InvalidOperationException("La variable AGENT_COMPANY es requerida.");
            _serviceName = config.GetValue<string>("AGENT_SERVICENAME", "BoxAgenteV2");
            _intervaloMs = config.GetValue<int>("AGENT_INTERVAL_MS", 240000);
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("Agente {ServiceName} iniciando operaciones para la empresa {Company}", _serviceName, _company);

            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    _logger.LogDebug("Iniciando sincronización de clases de artículo...");
                    // Delegación controlada y sensible a la señal de apagado SIGTERM
                    await _partClassManagement.ProAsync(stoppingToken);
                    _logger.LogDebug("Sincronización finalizada exitosamente.");
                }
                catch (TaskCanceledException)
                {
                    _logger.LogWarning("Operación cancelada por el orquestador (Apagado ordenado).");
                }
                catch (Exception ex)
                {
                    // Registro estructurado con el StackTrace completo a stdout
                    _logger.LogError(ex, "Fallo crítico en el ciclo del agente {ServiceName}", _serviceName);
                }

                // Espera asíncrona segura, liberando el hilo al thread pool
                await Task.Delay(_intervaloMs, stoppingToken);
            }

            _logger.LogInformation("El servicio {ServiceName} se ha detenido con gracia.", _serviceName);
        }
    }
}
```
*(Se asume la creación en `Program.cs` del `Host.CreateDefaultBuilder(args)` registrando este Worker y la implementación de `IPartClassManagement`).*

### Fase 2: Estrategia de Descomposición

El Agente actual es un proceso batch programado. Desde la perspectiva del Domain-Driven Design (DDD), pertenece al contexto de **Sincronización de Catálogos / Inventario**. No es necesario romper esto en múltiples microservicios si su única función es migrar Artículos. Lo correcto es aislar la lógica de dominio (el cómo conectarse a Epicor/Parnet) en una librería separada (`AgenteDominio.dll`) y usar el Worker Service únicamente como disparador de tiempo (Orquestador).

### Fase 3: Dockerización y Orquestación Local

**Dockerfile (Multi-Stage Build optimizado, Linux Alpine):**
```dockerfile
# Etapa 1: Build y Publish
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# Restablecimiento en capas para aprovechar la caché de Docker
COPY ["BoxAgenteCloud.csproj", "./"]
RUN dotnet restore "BoxAgenteCloud.csproj"

# Compilación final
COPY . .
RUN dotnet publish "BoxAgenteCloud.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa 2: Runtime ligero
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app
COPY --from=build /app/publish .

# Seguridad: Evitar correr como root en el contenedor
RUN addgroup -S agentgroup && adduser -S agentuser -G agentgroup
USER agentuser

# Configuración 12-Factor
ENV DOTNET_ENVIRONMENT=Production
ENV AGENT_INTERVAL_MS="240000"

ENTRYPOINT ["dotnet", "BoxAgenteCloud.dll"]
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

**docker-compose.yml (Entorno de Desarrollo Aislado):**
```yaml
version: '3.8'

services:
  box-agente-articulo:
    build:
      context: .
      dockerfile: Dockerfile
    image: box-agente-articulo:local
    container_name: box-agente-articulo-dev
    restart: unless-stopped
    environment:
      - DOTNET_ENVIRONMENT=Development
      - AGENT_COMPANY=EmpresaTest
      - AGENT_SERVICENAME=BoxAgenteDocker
      - AGENT_INTERVAL_MS=60000
      # Secciones inyectadas que sobreescriben configuraciones
      - ConnectionStrings__EpicorDB=Server=db-mock;Database=EpicorDB;User Id=sa;Password=SuperSecurePassword123!;TrustServerCertificate=True;
    depends_on:
      db-mock:
        condition: service_healthy
    networks:
      - agent-net

  db-mock:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sqlserver-mock
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=SuperSecurePassword123!
    ports:
      - "1433:1433"
    healthcheck:
      test: ["CMD", "/opt/mssql-tools/bin/sqlcmd", "-U", "sa", "-P", "SuperSecurePassword123!", "-Q", "SELECT 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - agent-net
    volumes:
      - sql-data:/var/opt/mssql

volumes:
  sql-data:

networks:
  agent-net:
    driver: bridge
```

### Fase 4: Pipeline CI/CD, Seguridad y Healthchecks

Para que Kubernetes pueda evaluar la salud de este Worker (que no tiene puertos expuestos por defecto), se debe instalar `Microsoft.Extensions.Diagnostics.HealthChecks` y exponer un puerto TCP/HTTP mínimo.

**Definición de Pipeline Completa (GitHub Actions - `.github/workflows/ci-cd.yml`):**
```yaml
name: CI/CD Pipeline - BoxAgenteArticulo

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup .NET 8
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x

      - name: Restore dependencies
        run: dotnet restore

      - name: SAST Scanner (Snyk)
        uses: snyk/actions/dotnet@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: code test
          args: --severity-threshold=high

      - name: Run Unit Tests
        run: dotnet test --no-restore --verbosity normal --collect:"XPlat Code Coverage"

      - name: Build Application
        run: dotnet build --no-restore -c Release

  docker-build-push:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}/boxagente:${{ github.sha }},ghcr.io/${{ github.repository }}/boxagente:latest
```
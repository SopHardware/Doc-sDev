# 2. Dictamen de Viabilidad de Contenerización - BoxServiceQueueConversionPart

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES
**DICTAMEN: VIABLE CON CONDICIONES (Requiere Refactorización Profunda)**

El estado actual del proyecto se cataloga como un **monolito de acoplamiento rígido con el Sistema Operativo (Windows Service)**. Compilado bajo .NET Framework 4.7.2 y apoyándose profundamente en servicios nativos (`System.ServiceProcess.ServiceBase` y `System.Diagnostics.EventLog`), el proyecto actual NO es nativo para nube, y viola múltiples principios de la **12-Factor App**:

- **Fallo 12-Factor (Logs):** Usa Windows Event Viewer en lugar de escribir a `stdout` (Consola). Un contenedor Docker estándar asume que los logs salen por la consola estándar.
- **Fallo 12-Factor (Dependencies & Config):** Los nombres de bases de datos, las configuraciones en `ConfigurationManager` (`app.config`), la falta de control por variables de entorno y la ausencia de Inyección de Dependencias impiden que el ejecutable sea verdaderamente agnóstico al entorno (Dev/QA/Prod).

Para ser ejecutable en Docker/Kubernetes, **obligatoriamente** debe migrarse la capa de hosting (de `Windows Service` a `Console App` estándar, idealmente pasándolo a .NET 6/.NET 8 con `Worker Services`, o usando un contenedor `mcr.microsoft.com/dotnet/framework/runtime:4.8-windowsservercore` como puente si no se migra de framework, lo cual es altamente ineficiente e incrementa el peso de la imagen a +5GB).

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Corregir ISession Thread-Safety y SRP)

**ANTES (Código con Anti-Patrones en ConversionPartRepository.cs):**
```csharp
public class ConversionPartRepository : RespositoryBase<dtSP>, IProcess
{
    // ❌ Error: Instancias globales compartidas entre hilos (No Thread-Safe)
    public ConversionPartRepository(string serviceName = "", bool isService = true)
    {
        sessionOrigen = ConexionBDEpicorERPTest.OpenSession();
        sessionDestino = ConexionBD.OpenSession();
    }
    
    public void Destiny(object model) {
         // ❌ Error: Llamada HTTP incrustada en Repositorio
         // ❌ Error: Sync-over-Async (.Wait())
         Task.Run(async () => {
             var oResponse = await OrquestadorHelp.SendOrquetadorAsync(request);
         }).Wait();
    }
}
```

**DESPUÉS (Refactorización a Clean Architecture / SOLID / Async):**
```csharp
// 1. Servicio de Sincronización Independiente (Separar HTTP de BD)
public class ConversionSyncService 
{
    private readonly IConversionPartRepository _repo;
    private readonly IOrquestadorClient _orquestador;
    
    // Inyección de Dependencias (IoC)
    public ConversionSyncService(IConversionPartRepository repo, IOrquestadorClient orquestador)
    {
        _repo = repo;
        _orquestador = orquestador;
    }
    
    public async Task ProcessSyncAsync(string plant, List<dtSP> partConversion)
    {
        // 1. Enviar HTTP (Agnóstico a la base de datos)
        var request = new ConvRequest { Plant = plant, ConversionsPart = MapToDatos(partConversion) };
        var response = await _orquestador.SendOrquestadorAsync(request);
        
        // 2. Insertar a BD
        await _repo.UpdateSyncStatusAsync(partConversion, response.ConversionParts);
    }
}

// 2. Repositorio Limpio (Unit of Work y manejo local de sesión)
public class ConversionPartRepository : IConversionPartRepository
{
    public async Task UpdateSyncStatusAsync(List<dtSP> parts, object responsePart)
    {
        // ✅ Sesión Local. Thread-safe por llamada.
        using (var session = ConexionBDEpicorFactory.OpenSession())
        using (var tx = session.BeginTransaction())
        {
            try {
                // ... lógica de actualización NHibernate ...
                await tx.CommitAsync();
            }
            catch {
                await tx.RollbackAsync();
                throw;
            }
        }
    }
}
```

### Fase 2: Estrategia de Descomposición a Microservicios / Worker Service
Para salir de `ServiceBase` y prepararse para Kubernetes, el proyecto se debe refactorizar a un `BackgroundService` (.NET Core / .NET 8).

1. Crear nuevo proyecto: `dotnet new worker -n Box.Service.ConversionPart.Worker`
2. Migrar la lógica de `clManagement` al método `ExecuteAsync` del Worker.
3. Cambiar `EventLog` por el estándar `Microsoft.Extensions.Logging.ILogger` (imprime por consola / stdout).
4. Configurar el Worker para inyectar dependencias y leer el AppSettings / Variables de Entorno.

### Fase 3: Dockerización

**1. Archivo `.dockerignore`:**
```text
.dockerignore
.env
.git
.gitignore
.vs
.vscode
*/bin
*/obj
*.suo
```

**2. Archivo `Dockerfile` (Multi-stage, asumiendo migración a .NET 8 Worker Service. ⚠️ NOTA: Si se mantiene .NET Framework 4.7.2 requerirá contenedores Windows, lo cual encarece infraestructura):**

```dockerfile
# Etapa de Compilación
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["Box.Service.ConversionPart.Worker/Box.Service.ConversionPart.Worker.csproj", "Box.Service.ConversionPart.Worker/"]
# Restaurar paquetes
RUN dotnet restore "Box.Service.ConversionPart.Worker/Box.Service.ConversionPart.Worker.csproj"
COPY . .
WORKDIR "/src/Box.Service.ConversionPart.Worker"
RUN dotnet build "Box.Service.ConversionPart.Worker.csproj" -c Release -o /app/build

# Etapa de Publicación
FROM build AS publish
RUN dotnet publish "Box.Service.ConversionPart.Worker.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa Final (Imagen ligera de runtime)
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Usuario sin privilegios por seguridad
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

ENTRYPOINT ["dotnet", "Box.Service.ConversionPart.Worker.dll"]
```

**3. Archivo `docker-compose.yml` (Entorno de desarrollo local):**
```yaml
version: '3.8'

services:
  conversionpart-collector:
    build: 
      context: .
      dockerfile: Box.Service.ConversionPart.Worker/Dockerfile
    environment:
      - ConnectionStrings__conexionDestino=Server=tcp:dbsrv,1433;Database=Intermedia;User ID=sa;Password=tu_password;
      - ConnectionStrings__conexionEpicor=Server=tcp:epicorsrv,1433;Database=EpicorERP;User ID=sa;Password=tu_password;
      - AppSettings__IntervaloEjecucion=4
      - AppSettings__OrquestadorUrl=http://api-orquestador:5000/
    restart: unless-stopped
```

### Fase 4: Pipeline CI/CD (GitHub Actions) y Sondas K8s

**Pipeline de Construcción y Push a Registry (`.github/workflows/docker-ci.yml`):**
```yaml
name: CI/CD Docker Pipeline

on:
  push:
    branches: [ "main", "DEV.VELA" ]

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
        
    - name: Restore dependencies
      run: dotnet restore ./Box.Service.ConversionPart.Worker/Box.Service.ConversionPart.Worker.csproj
      
    - name: Build
      run: dotnet build ./Box.Service.ConversionPart.Worker/Box.Service.ConversionPart.Worker.csproj --no-restore -c Release

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Box.Service.ConversionPart.Worker/Dockerfile
        push: true
        tags: boxito/conversionpart-worker:latest, boxito/conversionpart-worker:${{ github.sha }}
```

**Sondas para Kubernetes (Liveness/Readiness):**
Dado que es un proceso Background, se sugiere habilitar un pequeño WebHost en el puerto 8080 (usando HealthChecks de ASP.NET Core) para que Kubernetes pueda monitorizar si el loop de trabajo está vivo.

```yaml
# Extracto del deployment en K8s
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

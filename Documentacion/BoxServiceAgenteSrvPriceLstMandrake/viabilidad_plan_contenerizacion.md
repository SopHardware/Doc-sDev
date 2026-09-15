# Dictamen y Plan de Modernización hacia Contenedores

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES
**Estado:** VIABLE CON CONDICIONES (Refactorización Obligatoria a .NET Core/8)

El aplicativo `BoxServiceAgenteSrvPriceLstMandrake` en su estado puro está desarrollado sobre **.NET Framework 4.7.2** interactuando directamente con primitivas restrictivas como `ServiceBase` y `EventLog`. Desplegar esto bajo contenedores de Windows resulta en imágenes masivas (+5GB), arranques lentos y altos costos de cómputo.
Bajo la filosofía **12-Factor App**, el proyecto actual falla estrepitosamente:
*   **I. Codebase:** Es viable.
*   **III. Config:** Fallido. Credenciales quemadas en `App.config` vs variables de entorno.
*   **VI. Processes:** Fallido. Ejecuta lógicas de Hilos sin control, conservando estados volátiles e inconsistentes en memoria.
*   **XI. Logs:** Fallido. Emisión exclusiva al visor de eventos de Windows en lugar de al flujo `stdout`.

**Decisión:** El plan es contenerizar bajo Linux (Alpine o Debian) usando Docker/Kubernetes. Esto exige transformar el proyecto actual de *Windows Service (.NET 4.7.2)* a un **Worker Service (.NET 8)**.

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Control de Hilos)
Antes de siquiera pensar en .NET 8 o Docker, el código actual debe estabilizarse eliminando las condiciones de carrera y bloqueos mágicos.

**CÓDIGO ANTES (PriceLstManagement.cs - Método ProcesoAsync):**
```csharp
// Ejecucion de los hilos
await Task.Run(() =>
{
    procList.ForEach(proc =>
    {
        ThreadPool.QueueUserWorkItem(Execute, proc);
    });
});

Thread.Sleep(16000); // <- DESTRUYE ESTABILIDAD
RegisterToEventViewer("Finaliza procesamiento AgentMandrakote");
```

**CÓDIGO DESPUÉS (.NET C# - Concurrencia Controlada):**
```csharp
// Ejecución controlada y asíncrona real
var throttler = new SemaphoreSlim(initialCount: 5); // Limita a 5 llamadas concurrentes

var tasks = procList.Select(async proc =>
{
    await throttler.WaitAsync();
    try
    {
        await proc.Execute();
    }
    catch (Exception procEx)
    {
        RegisterToEventViewer($"Falla en subproceso: {procEx.Message}");
    }
    finally
    {
        throttler.Release();
    }
});

await Task.WhenAll(tasks); // Espera inteligente, sin Sleep
RegisterToEventViewer("Finaliza procesamiento AgentMandrakote de forma íntegra");
```

### Fase 2: Estrategia de Descomposición en Microservicios
1.  **Migración de Host:** Mover lógica de negocio de `ServiceBase` a `Microsoft.Extensions.Hosting.BackgroundService` en .NET 8.
2.  **Configuraciones Cloud-Native:** Eliminar `App.config`. Adoptar `IConfiguration` para mapeo de variables de entorno, de modo que `appsettings.json` pueda ser sobreescrito en los pods de K8s.
3.  **Logs de Salida:** Implementar `ILogger<Worker>` delegando logs al proveedor de consola estándar, permitiendo que K8s (Fluentd/Promtail) los procese de manera centralizada.
4.  **Gestión de Base de Datos:** Centralizar las dependencias mediante Inyección de Dependencias (DI) para inyectar correctamente contextos o repositorios por "Scope", asegurando el cierre automático de las sesiones abiertas contra las BD de las sucursales.

### Fase 3: Dockerización

*Archivo `Dockerfile` (Multi-stage build para Worker Service .NET 8)*
```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# Se asume una estructura moderna tras la migración a .NET 8
COPY ["BoxWorkerMandrake/BoxWorkerMandrake.csproj", "BoxWorkerMandrake/"]
# ⚠️ Información no proporcionada en la entrada: Se asume que dependencias
# como 'Commons' o 'BO' serán convertidas a netstandard2.0/net8.0
COPY ["Commons/Commons.csproj", "Commons/"]
COPY ["BO/BO.csproj", "BO/"]
RUN dotnet restore "BoxWorkerMandrake/BoxWorkerMandrake.csproj"

COPY . .
WORKDIR "/src/BoxWorkerMandrake"
RUN dotnet build "BoxWorkerMandrake.csproj" -c Release -o /app/build

# Publish Stage
FROM build AS publish
RUN dotnet publish "BoxWorkerMandrake.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Final Image
FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Usuario no root para seguridad de contenedores
RUN adduser -D nonrootuser
USER nonrootuser

ENTRYPOINT ["dotnet", "BoxWorkerMandrake.dll"]
```

*Archivo `docker-compose.yml`*
```yaml
version: '3.8'

services:
  agent-mandrake:
    build: 
      context: .
      dockerfile: Dockerfile
    environment:
      # Las credenciales ahora provienen de variables seguras, no quemadas
      - ConnectionStrings__conexion=Server=10.40.3.72;Database=EpicorBoxito;User Id=ucustom;Password=${DB_PASSWORD};
      - ApiSettings__UrlOrquestador=http://10.40.3.30/
      - ApiSettings__Authorization=${API_TOKEN}
      - TimerSettings__IntervalMinutes=60
    restart: always
```

*Archivo `.dockerignore`*
```
.git
.vs
.vscode
bin/
obj/
packages/
*.suo
*.user
```

### Fase 4: Pipeline CI/CD (GitHub Actions) y Sondas para Kubernetes

*Archivo `.github/workflows/ci-cd.yml` (YAML Completo)*
```yaml
name: CI/CD Pipeline Agent Mandrake

on:
  push:
    branches: [ "DEV.VELA", "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3
    
    - name: Setup .NET 8
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
        
    - name: Restore dependencies
      run: dotnet restore ./BoxWorkerMandrake/BoxWorkerMandrake.csproj
      
    - name: Build Application
      run: dotnet build ./BoxWorkerMandrake/BoxWorkerMandrake.csproj --no-restore -c Release
      
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.REGISTRY_USER }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: micorporacion/agent-mandrake:${{ github.sha }}, micorporacion/agent-mandrake:latest
```

*Sondas K8s (Fragmento para deployment.yaml)*
Ya que es un proceso en background, configuramos sondas Liveness usando un comando CLI integrado (si no se expone servidor web interno).
```yaml
          livenessProbe:
            exec:
              command:
              - /bin/sh
              - -c
              - ps aux | grep "BoxWorkerMandrake.dll" || exit 1
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
```
*(Se recomienda en la modernización a .NET 8 habilitar `Microsoft.Extensions.Diagnostics.HealthChecks` para habilitar sondeos puramente HTTP en el puerto 8080).*

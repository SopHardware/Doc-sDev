# Viabilidad y Plan de Modernización hacia Contenedores

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES
**Dictamen: NO VIABLE EN ESTADO ACTUAL (Requiere Migración a .NET Core / .NET 8 Worker Service)**

La aplicación actual está construida sobre **.NET Framework 4.7.2** heredando de `ServiceBase` (Windows Service) y acoplada a la API de **Windows EventLog**. Esto impide su ejecución nativa en contenedores Linux estándar (Docker/Kubernetes). Si bien podría contenerizarse usando imágenes base de Windows Server Core (muy pesadas, lentas y costosas), contradice los lineamientos de arquitectura Cloud-Native. 

Para que sea verdaderamente **VIABLE**, debe transicionar a un proyecto `.NET Worker Service` moderno cross-platform, lo cual le permitirá cumplir con el estándar **12-Factor App**:
- **III. Configuración:** Extraer `App.config` hacia Variables de Entorno (`.env`).
- **XI. Logs:** Abandonar EventLog en favor de flujos de salida estándar (`STDOUT`).
- **VI. Procesos:** Convertir en un proceso en segundo plano sin dependencias de sistema operativo huésped.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Código C#)
Solucionar el problema de *Socket Exhaustion* y extraer configuración de entorno.

**CÓDIGO ANTES (`HttpClientHelp.cs` actual):**
```csharp
public class HttpClientHelp
{
    public static async Task<PartResponse> SendOrquestadorAsync(PartRequest request)
    {
        HttpClient httpClient = new HttpClient(); // ERROR: Anti-patrón Socket Exhaustion
        // ...
        using (var response = await httpClient.SendAsync(httpRequestMessage))
        // ...
    }
}
```

**CÓDIGO DESPUÉS (`HttpClientHelp.cs` remediado para .NET 8):**
```csharp
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;
using System;

namespace Repositorys.Configurations
{
    public class HttpClientHelp
    {
        // Solución estática o vía IHttpClientFactory en DI
        private static readonly HttpClient _httpClient = new HttpClient { Timeout = TimeSpan.FromMinutes(30) };

        public static async Task<PartResponse> SendOrquestadorAsync(PartRequest request)
        {
            // Migrar a variables de entorno para evitar App.config harcodeado
            string apiUri = Environment.GetEnvironmentVariable("ORQUESTADOR_API_URL") 
                            ?? throw new Exception("ORQUESTADOR_API_URL no configurada");
            
            string apiKey = Environment.GetEnvironmentVariable("API_KEY");

            try
            {
                var httpRequestMessage = new HttpRequestMessage(HttpMethod.Post, apiUri);
                httpRequestMessage.Headers.Add("x-api-key", apiKey);
                
                string oJson = JsonSerializer.Serialize(request);
                httpRequestMessage.Content = new StringContent(oJson, Encoding.UTF8, "application/json");

                using (var response = await _httpClient.SendAsync(httpRequestMessage))
                {
                    response.EnsureSuccessStatusCode();
                    var strRead = await response.Content.ReadAsStringAsync();
                    var respuesta = JsonSerializer.Deserialize<PartResponse>(strRead);
                    return respuesta;
                }
            }
            catch (Exception ex)
            {
                return new PartResponse { Status = 3, Message = ex.Message, Error = true, MessageError = ex.Message };
            }
        }
    }
}
```

### Fase 2: Estrategia de Descomposición
1. Crear un nuevo proyecto `dotnet new worker -n BoxServiceAgentPartWorker`.
2. Migrar la lógica de NHibernate y Mapeos.
3. Remplazar `EventLog` por el `ILogger<T>` nativo de .NET 8 configurado para imprimir en Consola.
4. Remplazar `System.Timers.Timer` por un `BackgroundService` con `Task.Delay` en un bucle asíncrono para el ciclo de vida del agente.

### Fase 3: Dockerización (Docker Multi-stage)

**Dockerfile completo:**
```dockerfile
# Stage 1: Base runtime
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS base
WORKDIR /app

# Stage 2: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BoxServiceAgentPartWorker/BoxServiceAgentPartWorker.csproj", "BoxServiceAgentPartWorker/"]
RUN dotnet restore "BoxServiceAgentPartWorker/BoxServiceAgentPartWorker.csproj"
COPY . .
WORKDIR "/src/BoxServiceAgentPartWorker"
RUN dotnet build "BoxServiceAgentPartWorker.csproj" -c Release -o /app/build

# Stage 3: Publish
FROM build AS publish
RUN dotnet publish "BoxServiceAgentPartWorker.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 4: Final Image
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Usuario no root por seguridad
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

ENTRYPOINT ["dotnet", "BoxServiceAgentPartWorker.dll"]
```

**.dockerignore:**
```text
bin/
obj/
.git/
.vscode/
*.user
*.suo
*.md
```

**docker-compose.yml:**
```yaml
version: '3.8'
services:
  box-agent-part:
    build: 
      context: .
      dockerfile: BoxServiceAgentPartWorker/Dockerfile
    environment:
      - ORQUESTADOR_API_URL=http://10.40.3.19:44313/api/Parts/AddOrUpdateBulk
      - API_KEY=${API_KEY_SECRET}
      - DB_CONNECTION_BOXITO=${DB_BOXITO_SECRET}
      - DB_CONNECTION_LIVE=${DB_LIVE_SECRET}
      - INTERVAL_MINUTES=4
    restart: unless-stopped
    # Logging nativo Docker, ideal para enviar a ELK / Datadog
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### Fase 4: Pipeline CI/CD y Kubernetes

**GitHub Actions YAML (Construcción y publicación de contenedor):**
```yaml
name: Build and Publish Docker Image

on:
  push:
    branches: [ "DEV.VELA" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and Push Image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: BoxServiceAgentPartWorker/Dockerfile
        push: true
        tags: organization/box-agent-part:latest, organization/box-agent-part:${{ github.sha }}
```

**Sondas Liveness/Readiness para K8s:**
Dado que es un `Worker Service` sin puertos HTTP expuestos de forma predeterminada, se requiere habilitar endpoints de salud.
Se recomienda inyectar `Microsoft.Extensions.Diagnostics.HealthChecks` y exponer el puerto `8080`.

Ejemplo fragmento Deployment K8s:
```yaml
          livenessProbe:
            httpGet:
              path: /health/liveness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health/readiness
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```
⚠️ *Información no proporcionada en la entrada:* No se especifica en la arquitectura actual la existencia de un listener HTTP en el Agente. Para soportar sondas de K8s, el Worker Service deberá registrar los servicios de endpoints (HealthChecks) mínimos.
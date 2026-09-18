# Viabilidad y Plan de Modernización hacia Contenedores
**Aplicación:** BoxServiceProviderConversionPart (Provider)

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES
**Dictamen:** VIABLE CON CONDICIONES (Requiere Refactorización Previa)

**Justificación según 12-Factor App:**
* **I. Codebase:** Viable. El código está en un repositorio centralizado.
* **II. Dependencies:** Viable. Dependencias como `EFCore.BulkExtensions` y `Newtonsoft.Json` se manejan por NuGet.
* **III. Config:** ⚠️ *Información no proporcionada en la entrada.* Se asume que las cadenas de conexión se manejan por variables de entorno u `appsettings.json`, pero el uso de `ConnectionHelper.GetConnection(request.Company, request.Plant)` sugiere lógica hardcodeada o búsqueda custom en base de datos. Esto debe externalizarse a variables de entorno inyectadas al contenedor.
* **IV. Backing services:** Viable. SQL Server se trata como recurso adjunto.
* **VI. Processes:** Requiere atención. Los métodos estáticos y contextos autogestionados (`new EFCoreContext(connection)`) dentro del flujo de la petición pueden causar fugas de memoria y agotar el Connection Pool en un entorno efímero como Docker.
* **IX. Disposability:** Crítico. El uso de transacciones larguísimas o Timeouts de base de datos de 960,000 segundos impedirá que los contenedores se apaguen de forma grácil (Graceful Shutdown) en Kubernetes, provocando corrupción de datos o interrupciones severas.

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Refactorización)

**Código ANTES (Problemático):**
```csharp
// Fragmento original en ConversionPartController
[HttpPost]
[Route("api/ConversionPart/AddOrUpdateBulk")]
public async Task<IHttpActionResult> AddOrUpdateBulk(ConversionPartRequestBulk request)
{
    try
    {
        Model.IsValidOnThrow(request);
        string connection = ConnectionHelper.GetConnection(request.Company, request.Plant);
        // Llamada a método estático - Acoplamiento Fuerte
        var response = await ConversionPartService.CreateOrUpdateBulk(request, connection);
        return Ok(response);
    }
    catch (Exception ex)
    {
        // Exposición de vulnerabilidades al cliente
        return BadRequest(ex.Message + ". InnerException:\n" + ex.InnerException?.Message);
    }
}
```

**Código DESPUÉS (Listo para Contenedores):**
```csharp
// Fragmento refactorizado en ConversionPartController con DI
[HttpPost]
[Route("api/ConversionPart/AddOrUpdateBulk")]
public async Task<IHttpActionResult> AddOrUpdateBulk(ConversionPartRequestBulk request)
{
    try
    {
        Model.IsValidOnThrow(request);
        // La conexión debería venir por configuración / DI en el DbContext, no en vuelo
        var response = await _conversionPartService.CreateOrUpdateBulk(request, request.Company, request.Plant);
        return Ok(response);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error processing bulk update");
        return InternalServerError(new Exception("An unexpected error occurred processing your request."));
    }
}
```

*Nota: Se requiere refactorizar el servicio para inyectar `EFCoreContext` por constructor (Scoped) en lugar de hacer `new EFCoreContext()` manual, y eliminar el `ctx.Database.SetCommandTimeout(960000)`.*

### Fase 2: Estrategia de Descomposición en Microservicios
1. **Aislamiento de Dominio:** Extraer la lógica de "Conversión de Partes" a una API independiente, desvinculando cualquier dependencia directa de la base de datos monolítica de "PARNET".
2. **Comunicación Asíncrona:** El método "Bulk" puede tardar bastante. Transicionar de una respuesta síncrona HTTP 200 a un modelo HTTP 202 (Accepted) con procesamiento en Background Workers y Message Brokers (RabbitMQ o Kafka).

### Fase 3: Dockerización

**Dockerfile Multi-stage (Optimizado para .NET Core/.NET 5+):**
```dockerfile
# Etapa de Construcción
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["ParnetRestAPI/ParnetRestAPI.csproj", "ParnetRestAPI/"]
COPY ["Services/Services.csproj", "Services/"]
COPY ["EFCore/EFCore.csproj", "EFCore/"]
COPY ["BO/BO.csproj", "BO/"]
COPY ["Commons/Commons.csproj", "Commons/"]
RUN dotnet restore "ParnetRestAPI/ParnetRestAPI.csproj"
COPY . .
WORKDIR "/src/ParnetRestAPI"
RUN dotnet build "ParnetRestAPI.csproj" -c Release -o /app/build

# Etapa de Publicación
FROM build AS publish
RUN dotnet publish "ParnetRestAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa Final (Runtime)
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ParnetRestAPI.dll"]
```
*⚠️ Información no proporcionada en la entrada: La versión exacta de .NET (se asume .NET 6.0 como ejemplo de API moderna con base en los imports y uso, si usa .NET Framework clásico requerirá contenedores Windows, lo cual desaconsejamos fuertemente).*

**.dockerignore:**
```text
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
**/bld
**/core
**/docker-compose.yml
**/Dockerfile*
**/node_modules
**/npm-debug.log
**/obj
**/secrets.env
**/tests
```

**docker-compose.yml:**
```yaml
version: '3.8'
services:
  conversionpart-api:
    image: boxito/conversionpart-api:latest
    build:
      context: .
      dockerfile: ParnetRestAPI/Dockerfile
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=sql_server;Database=ParnetDB;User Id=sa;Password=YourPassword123!;
    restart: unless-stopped
    depends_on:
      - sql_server

  sql_server:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1433:1433"
```

### Fase 4: Pipeline CI/CD y Kubernetes

**Pipeline CI/CD (GitHub Actions - .github/workflows/docker-build.yml):**
```yaml
name: CI/CD Pipeline para ConversionPart API

on:
  push:
    branches: [ "DEV.VELA", "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '6.0.x'
        
    - name: Restore dependencies
      run: dotnet restore ./ParnetRestAPI/ParnetRestAPI.sln
      
    - name: Build
      run: dotnet build ./ParnetRestAPI/ParnetRestAPI.sln --no-restore --configuration Release
      
    - name: Test
      run: dotnet test ./ParnetRestAPI/ParnetRestAPI.sln --no-build --verbosity normal

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./ParnetRestAPI/Dockerfile
        push: true
        tags: boxito/conversionpart-api:${{ github.sha }}
```

**Sondas para Kubernetes (Liveness y Readiness):**
```yaml
# Fragmento Deployment.yaml K8s
livenessProbe:
  httpGet:
    path: /health/liveness
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 15
readinessProbe:
  httpGet:
    path: /health/readiness
    port: 80
  initialDelaySeconds: 15
  periodSeconds: 10
```
*Requisito:* Implementar `Microsoft.Extensions.Diagnostics.HealthChecks` en el pipeline de la aplicación para habilitar los endpoints `/health/liveness` y `/health/readiness` que validen la conexión al DbContext.

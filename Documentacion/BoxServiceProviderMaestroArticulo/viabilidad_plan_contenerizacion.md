# Dictamen y Plan de Modernización hacia Contenedores

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**Dictamen:** **VIABLE CON CONDICIONES**

El componente evaluado (ASP.NET Web API heredado interactuando con EF Core) es viable para ser contenerizado (usando imágenes de Windows Containers o migrándolo completamente a .NET Core / .NET 5+ para Linux Containers). Sin embargo, requiere modificaciones previas basadas en los principios **12-Factor App**:

1. **Configuración (Config):** Actualmente el código lee conexiones de `ConnectionHelper.GetConnection` que parecen depender de estado global o configuraciones externas locales. Debe adaptarse para inyectarse puramente vía variables de entorno.
2. **Dependencias (Dependencies):** Al crear instancias manuales de DbContext dentro de métodos estáticos, se viola la inyección estructural. Esto dificulta el manejo de recursos en un contenedor inmutable de larga vida.
3. **Logs:** Actualmente se usan utilerías estáticas (`LogHelper.Write`). Debe transicionar hacia STDOUT/STDERR para que el motor de contenedores o Fluentd/Promtail capturen los logs.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Refactorización Inyección de Dependencias)

**ANTES (Código Actual en `PartsController.cs` y `PartService.cs`):**
```csharp
// PartsController.cs
[HttpPost]
[Route("api/Parts/AddOrUpdate")]
public async Task<IHttpActionResult> AddOrUpdate(PartRequest request)
{
    try
    {
        string connection = ConnectionHelper.GetConnection(request.Company, request.Plant);
        var response = await PartService.CreateOrUpdate(request.Parts, request.Plant, connection);
        return Ok(response);
    }
    catch (Exception ex)
    {
        return BadRequest(ex.Message);
    }
}

// PartService.cs (Estático y DbContext Manual)
public class PartService 
{
    public static async Task<PartResponse> CreateOrUpdate(List<PartModel> request, string plant, string connection)
    {
        EFCoreContext ctx = new EFCoreContext(connection);
        // ... Logica ...
    }
}
```

**DESPUÉS (Código Refactorizado con Inyección de Dependencias):**
```csharp
// IPartService.cs
public interface IPartService 
{
    Task<PartResponse> CreateOrUpdateAsync(List<PartModel> request, string plant, string connectionString);
}

// PartService.cs (No estático)
public class PartService : IPartService 
{
    private readonly ILogger<PartService> _logger;

    public PartService(ILogger<PartService> logger)
    {
        _logger = logger;
    }

    public async Task<PartResponse> CreateOrUpdateAsync(List<PartModel> request, string plant, string connectionString)
    {
        // Se recomienda que la conexión provenga de un DbContextFactory inyectado o Scoped Service
        using (var ctx = new EFCoreContext(connectionString)) 
        {
             // Logica
             // _logger.LogInformation("Inicio de proceso");
        }
    }
}

// PartsController.cs
[ApiController]
[Route("api/[controller]")]
public class PartsController : ControllerBase
{
    private readonly IPartService _partService;
    private readonly IConfiguration _configuration;
    private readonly ILogger<PartsController> _logger;

    public PartsController(IPartService partService, IConfiguration configuration, ILogger<PartsController> logger)
    {
        _partService = partService;
        _configuration = configuration;
        _logger = logger;
    }

    [HttpPost("AddOrUpdate")]
    public async Task<IActionResult> AddOrUpdate([FromBody] PartRequest request)
    {
        try
        {
            // Obtener cadena via configuración / variables de entorno
            string connection = _configuration.GetConnectionString("DefaultConnection");
            var response = await _partService.CreateOrUpdateAsync(request.Parts, request.Plant, connection);
            return Ok(response);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Ocurrió un error al procesar Parts");
            return StatusCode(500, "Error interno del servidor");
        }
    }
}
```

### Fase 2: Estrategia de Descomposición en Microservicios

Si la API actual maneja múltiples dominios (Parts, Customers, Vendors, Tickets, etc.), se recomienda un enfoque de Estrangulamiento (Strangler Fig Pattern):
1. Separar los controladores relacionados a catálogo de Artículos (`PartsController`, `PartClassesController`, etc.) hacia un microservicio dedicado: `ProductCatalogService`.
2. Extraer las entidades EFCore estrictamente vinculadas a inventario/partes.
3. Desplegar de forma independiente detrás de un API Gateway (e.g., Ocelot, Kong o Azure API Management).

### Fase 3: Dockerización

**Dockerfile (Multi-stage build para .NET Web API):**
*(Asumiendo actualización a .NET 6/7/8 o un entorno compatible en su evolución natural)*

```dockerfile
# Etapa de construcción
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["ParnetRestAPI/ParnetRestAPI.csproj", "ParnetRestAPI/"]
# Restaurar dependencias
RUN dotnet restore "ParnetRestAPI/ParnetRestAPI.csproj"
COPY . .
WORKDIR "/src/ParnetRestAPI"
RUN dotnet build "ParnetRestAPI.csproj" -c Release -o /app/build

# Etapa de publicación
FROM build AS publish
RUN dotnet publish "ParnetRestAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa final (Runtime)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Usuario sin privilegios
RUN addgroup --system --gid 1000 appgroup && \
    adduser --system --uid 1000 --ingroup appgroup --home /app appuser
USER appuser

EXPOSE 8080
ENV ASPNETCORE_URLS=http+:8080

ENTRYPOINT ["dotnet", "ParnetRestAPI.dll"]
```

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
**/obj
**/logs
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  parnet-api:
    image: parnet-api:latest
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=MaestroDb;User Id=sa;Password=Your_password123;TrustServerCertificate=True;
    depends_on:
      - sqlserver

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=Your_password123
    ports:
      - "1433:1433"
```

### Fase 4: Pipeline CI/CD y Kubernetes (K8s)

**GitHub Actions YAML (.github/workflows/ci-cd.yml):**
```yaml
name: CI/CD Pipeline para Parnet API

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/parnet-api

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'

      - name: Build and Test
        run: |
          dotnet restore
          dotnet build --configuration Release --no-restore
          dotnet test --no-build --verbosity normal

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest,${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

**Sondas Liveness y Readiness para Kubernetes (deployment.yaml extract):**
```yaml
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
*(⚠️ Información no proporcionada en la entrada: Se asume que en el nuevo refactor se agregará el middleware de `AddHealthChecks()` a la API para soportar `/health/live` y `/health/ready`).*
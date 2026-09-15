# Plan de Viabilidad y Modernización hacia Contenedores
**Aplicación:** BoxServiceProviderMaestroClientes (MaestrosApi)

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**DICTAMEN ACTUAL: NO VIABLE (Para ecosistemas Linux / Kubernetes Estándar)**

**Justificación basada en 12-Factor App:**
1. **Codebase:** El proyecto actual es una aplicación basada en **.NET Framework 4.7.2** (`<TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>`) acoplada a dependencias exclusivas de Windows como `System.Web`, y diseñada para alojarse en IIS Express / IIS (visible en el `.csproj` y `Global.asax`). Las imágenes de contenedores Linux no soportan .NET Framework.
2. **Config (Factor 3 - Almacenar configuración en el entorno):** Violado. La configuración (`Web.config`) contiene credenciales en texto plano y depende fuertemente de archivos estáticos no basados en el entorno.
3. **Disposability (Factor 9):** El uso extensivo de objetos manuales de base de datos sin un contenedor de DI robusto, y el uso de tablas temporales SQL por hilo de petición, dificulta una escalabilidad ágil.

*Nota:* Podría considerarse "Viable con Condiciones" si se emplea una infraestructura heredada de Windows Containers, pero dado el alto coste computacional, el tamaño enorme de las imágenes Windows (varios GBs) y el desuso de esta práctica frente a Kubernetes estándar, el Comité descarta esta ruta a favor de un refactor.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

Para lograr la contenerización en clústeres Kubernetes nativos, se debe migrar la API a .NET 8 (Core).

### Fase 1: Remediación Obligatoria (Migración y Desacoplamiento)
El primer paso es la migración del stack y la inclusión de Inyección de Dependencias nativa.

**Código ANTES (Acoplado - .NET Framework 4.7.2):**
```csharp
public class CustumerController : ApiController
{
    [Route("api/MaestrosApi/Epicor/Custumer")]
    [HttpGet]
    public IHttpActionResult GetCustumerEI()
    {
        using (CustumerUoW uow = new CustumerUoW(new EpicorBoxitoDbContext()))
        {
            List<Customer> items = uow.GetCustomers(new Customer(), 0, 10);
            return Ok(items);
        }   
    }
}
```

**Código DESPUÉS (Desacoplado - .NET 8 / ASP.NET Core):**
```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/MaestrosApi")]
public class CustumerController : ControllerBase
{
    private readonly ICustumerUoW _uow;

    // Inyección de Dependencias por Constructor
    public CustumerController(ICustumerUoW uow)
    {
        _uow = uow;
    }

    [HttpGet("Epicor/Custumer")]
    public IActionResult GetCustumerEI([FromQuery] int page = 0, [FromQuery] int size = 10)
    {
        var items = _uow.GetCustomers(new Customer(), page, size);
        return Ok(items);
    }
}

// Configuración en Program.cs
// builder.Services.AddScoped<IDbContext, EpicorBoxitoDbContext>();
// builder.Services.AddScoped<ICustumerUoW, CustumerUoW>();
```

### Fase 2: Estrategia de Descomposición en Microservicios
La aplicación `MaestrosApi` contiene controladores dispares (`VendorController`, `CustumerController`, `PartController`, `CondicionPagoController`). 
1. **Bounded Contexts:** Separar lógicamente en microservicios independientes:
   - `CustomerService` (Customer, CustomerGroup, ClienteRegular).
   - `ProductService` (Part, PartClass, PartWhse).
   - `Finance/VendorService` (Vendor, CondicionPago).
2. **Eliminación de Tablas Temporales en DB:** Modificar los `DataAccess` (Ej. `CustumerGetEDA.cs`) para eliminar la dependencia de TempDB y que cada microservicio sea idempotente y escalable horizontalmente.

### Fase 3: Dockerización
Una vez migrado a .NET 8, proveemos la infraestructura Docker.

**Dockerfile Multi-stage (Optimizado para Producción)**
```dockerfile
# Etapa de construcción
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MaestrosAPI/MaestrosAPI.csproj", "MaestrosAPI/"]
# ⚠️ Información no proporcionada sobre Nugets privados, asumiendo repositorios públicos
RUN dotnet restore "MaestrosAPI/MaestrosAPI.csproj"
COPY . .
WORKDIR "/src/MaestrosAPI"
RUN dotnet build "MaestrosAPI.csproj" -c Release -o /app/build
RUN dotnet publish "MaestrosAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa de producción
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MaestrosAPI.dll"]
```

**.dockerignore**
```
**/.git
**/.vs
**/.vscode
**/bin/
**/obj/
**/appsettings.Development.json
**/*.user
**/*.suo
```

**docker-compose.yml (Entorno Local)**
```yaml
version: '3.8'
services:
  maestros-api:
    build: 
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      # Inyectar secreto desde variables de entorno
      - ConnectionStrings__DefaultConnection=${DB_CONNECTION_STRING}
```

### Fase 4: Pipeline CI/CD y Preparación para Kubernetes

**GitHub Actions YAML (Integración y Despliegue Continuo)**
```yaml
name: CI/CD Pipeline Maestros API

on:
  push:
    branches: [ "DEV.VELA", "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
        
    - name: Restore, Build and Test
      run: |
        dotnet restore MaestrosAPI/MaestrosAPI.csproj
        dotnet build MaestrosAPI/MaestrosAPI.csproj --no-restore -c Release
        # dotnet test (Faltan proyectos de pruebas en la estructura actual)

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
        
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: corporativo/maestros-api:${{ github.sha }}, corporativo/maestros-api:latest
```

**Sondas para Kubernetes (Liveness & Readiness)**
En Kubernetes, se configurará el `deployment.yaml` con las siguientes validaciones de salud (asumiendo que en .NET 8 se implementa middleware de HealthChecks):

```yaml
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```
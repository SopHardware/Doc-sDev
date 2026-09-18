# Viabilidad y Plan de Contenerización: BoxServiceProviderPartAllocTOs (OrderAllocAPI)

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**Dictamen:** **VIABLE CON CONDICIONES (REQUIERE REFACTORIZACIÓN PREVIA)**

Actualmente, la aplicación está construida sobre **.NET Framework 4.6**, lo cual limita su despliegue nativo a contenedores de Windows (Windows Containers). Si bien esto es técnicamente posible, los contenedores Windows son pesados (varios GBs), de arranque lento y menos soportados en ecosistemas Kubernetes estándar frente a contenedores Linux. Para una adopción verdaderamente Cloud-Native y cumplir con los estándares de orquestación modernos, es mandatorio un upgrade a **.NET 8 (Core)**.

**Análisis según factores clave de 12-Factor App:**
1. **Configuraciones (Factor III):** *Incumplido.* Las configuraciones, contraseñas y base URLs están "hardcodeadas" en el `Web.config`. En contenedores, estas deben inyectarse mediante Variables de Entorno.
2. **Logs (Factor XI):** *Incumplido.* Utiliza `EventLog` de Windows. En contenedores, los logs deben tratarse como un flujo de eventos (stdout/stderr) para que Docker/K8s los capturen.
3. **Backing Services (Factor IV):** *Parcial.* La base de datos y la API de Epicor están separadas, pero el acoplamiento a las credenciales locales dificulta el cambio fluido de entornos (Dev/QA/Prod).

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Refactorización)
El primer paso es resolver los riesgos críticos identificados y preparar el código para su inyección de dependencias.

**Código ANTES (ApiBaseController.cs original - Inseguro):**
```csharp
protected IHttpActionResult BadRequest(Exception ex)
{
    var _response = Request.CreateResponse(HttpStatusCode.BadRequest, ex);
    _response.Headers.Add("X-API-Version", $"{AseemblyName}/{Version}");
    return base.ResponseMessage(_response);
}
```

**Código DESPUÉS (ApiBaseController.cs corregido - Seguro):**
```csharp
protected IHttpActionResult BadRequest(Exception ex)
{
    // Solo registrar internamente, NO devolver 'ex' al cliente
    // _logger.Error(ex, "Error no controlado"); 
    
    var errorResponse = new {
        Message = "Ha ocurrido un error al procesar la solicitud.",
        TrackId = this.TrackId // ID de seguimiento para buscar en logs
    };

    var _response = Request.CreateResponse(HttpStatusCode.BadRequest, errorResponse);
    _response.Headers.Add("X-API-Version", $"{AseemblyName}/{Version}");
    return base.ResponseMessage(_response);
}
```

**Código ANTES (OrderAllocController.cs original - Sin DI):**
```csharp
[HttpPost]
[Route("api/OrderAlloc/ApproveTransferOrder")]
public async Task<IHttpActionResult> ApproveTransferOrder(ApproveTransferOrderRequestModel model)
{
    try
    {
        using (ApproveTransferOrderUoW approveTransferOrderUoW = new ApproveTransferOrderUoW())
        {
            approveTransferOrderUoW.PrepareService(model, GuidExt.ParseOrCreate(model.Id));
            var results = await approveTransferOrderUoW.Start();
            return Ok(results);
        }
    }
    catch (Exception ex)
    {
        return BadRequest(ex);
    }
}
```

**Código DESPUÉS (OrderAllocController.cs - Con DI y limpieza):**
```csharp
public class OrderAllocController : ApiBaseController
{
    private readonly IApproveTransferOrderUoW _approveTransferUoW;

    // Inyección a través del constructor
    public OrderAllocController(IApproveTransferOrderUoW approveTransferUoW)
    {
        _approveTransferUoW = approveTransferUoW;
    }

    [HttpPost]
    [Route("api/OrderAlloc/ApproveTransferOrder")]
    public async Task<IHttpActionResult> ApproveTransferOrder(ApproveTransferOrderRequestModel model)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }

        try
        {
            _approveTransferUoW.PrepareService(model, GuidExt.ParseOrCreate(model.Id));
            var results = await _approveTransferUoW.Start();
            return Ok(results);
        }
        catch (Exception ex)
        {
            return BadRequest(ex);
        }
    }
}
```
*(Nota: Para .NET 4.6, requiere configurar UnityConfig.cs o similar. Si se migra a .NET 8, usa `builder.Services.AddScoped<IApproveTransferOrderUoW, ...>();`)*

### Fase 2: Estrategia de Descomposición en Microservicios
Actualmente es un componente coordinado, pero para escalar debe operar de forma independiente:
1. **Migración de Framework:** Actualizar el proyecto completo (y su dependencia `EventHandler`) a **.NET 8**. Esto transformará el `Global.asax` y `Web.config` en un limpio `Program.cs` y `appsettings.json`.
2. **Contextos Delimitados:** Si `OrderAllocAPI` sólo aprueba órdenes (Transfer y Gains), debe empaquetarse en un solo microservicio, comunicándose con la base de datos `ObserverBD` vía Dapper o EF Core, y hacia Epicor vía `HttpClientFactory`.
3. **Comunicación Asíncrona (Opcional Futuro):** En lugar de que el cliente espere pasivamente un HTTP POST pesado, evaluar recibir el comando, publicarlo en un Message Broker (RabbitMQ/Kafka) y retornar `202 Accepted`.

### Fase 3: Dockerización (Asumiendo migración a .NET 8)

**Dockerfile (Multi-stage para optimizar peso, basado en Linux):**
```dockerfile
# Etapa 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copiar csproj y restaurar dependencias
COPY ["OrderAllocAPI/OrderAllocAPI.csproj", "OrderAllocAPI/"]
# ⚠️ Información no proporcionada: Asumimos que los proyectos dependientes como EventHandler también se copian
# COPY ["EventHandler/EventHandler.csproj", "EventHandler/"]
RUN dotnet restore "OrderAllocAPI/OrderAllocAPI.csproj"

# Copiar el resto y compilar
COPY . .
WORKDIR "/src/OrderAllocAPI"
RUN dotnet build "OrderAllocAPI.csproj" -c Release -o /app/build
RUN dotnet publish "OrderAllocAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Etapa 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Inyectar variables de entorno críticas
ENV ConnectionStrings__DefaultConnection="Server=10.40.3.84;Database=ObserverBD;User Id=${DB_USER};Password=${DB_PASS};"

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "OrderAllocAPI.dll"]
```

**.dockerignore:**
```text
bin/
obj/
App_Data/
.vs/
*.user
*.suo
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  orderalloc-api:
    build: 
      context: .
      dockerfile: OrderAllocAPI/Dockerfile
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - DB_USER=${DB_USER_SECRET}
      - DB_PASS=${DB_PASS_SECRET}
      - Epicor__ApiKey=${EPICOR_API_KEY}
    restart: unless-stopped
```

### Fase 4: Pipeline CI/CD y Preparación para Kubernetes

**GitHub Actions YAML (`.github/workflows/deploy.yml`):**
```yaml
name: CI/CD Pipeline OrderAllocAPI

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'

    - name: Restore dependencies
      run: dotnet restore ./OrderAllocAPI/OrderAllocAPI.csproj

    - name: Build
      run: dotnet build ./OrderAllocAPI/OrderAllocAPI.csproj --no-restore -c Release

    # ⚠️ No hay pruebas unitarias detectadas en el repositorio, pero aquí iría el comando:
    # - name: Test
    #   run: dotnet test ./OrderAllocAPI.Tests --no-build -c Release

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./OrderAllocAPI/Dockerfile
        push: true
        tags: micorporacion/orderallocapi:${{ github.sha }}, micorporacion/orderallocapi:latest
```

**Sondas para Kubernetes (Liveness & Readiness):**
Dentro del `deployment.yaml` de Kubernetes, se deben configurar las siguientes sondas utilizando los HealthChecks nativos de .NET 8 (o endpoints custom si se mantiene .NET 4.6):

```yaml
        livenessProbe:
          httpGet:
            path: /health/live
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 15
```
*(Nota: El endpoint de readiness deberá validar la conexión a la base de datos `ObserverBD` y la disponibilidad de la red hacia Epicor ERP).*
# Viabilidad de Plan de Contenerización (Docker / Kubernetes)

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES
**DICTAMEN: VIABLE CON CONDICIONES (REQUIERE REFACTORIZACIÓN)**

La aplicación es un Servicio de Windows basado en **.NET Framework 4.7.2**. Contenerizar servicios de Windows nativos es posible usando *Windows Containers*, pero es una práctica obsoleta, muy pesada (imágenes >4GB) y poco compatible con clústeres modernos de Kubernetes basados en Linux. 
Para ser verdaderamente viable bajo los preceptos de **12-Factor App**, se exige una migración a **.NET Core / .NET (6/8)** como un tipo de proyecto `Worker Service` (Linux compatible).

**Justificación 12-Factor App:**
*   **III. Configuración:** La config se guarda en `App.config`, se debe cambiar a Variables de Entorno (`Environment Variables`).
*   **VIII. Concurrencia:** La escalabilidad está comprometida por el manejo en hilos manual y cuello de botella de `NHibernateHelper`.
*   **XI. Logs:** Actualmente se usa `EventLog.WriteEntry` atado a Windows. Debe cambiarse a salida `stdout/stderr` (Console) para que Docker/K8s los capture.

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Código ANTES y DESPUÉS en C#)

**Problema (P0): Anti-patrón de NHibernateHelper**

**ANTES (Problema de CPU/Memoria):**
```csharp
// Connection\NHibernateHelper.cs (Malo)
public class NHibernateHelper : IDisposable
{
    private ISessionFactory erpContext;
    private ISessionFactory epicorBoxitoContext;

    public NHibernateHelper() // <-- Se ejecuta cada vez
    {
        var erpConfig = Fluently.Configure()...BuildConfiguration();
        erpContext = erpConfig.BuildSessionFactory(); // <-- Operación de ~2-3 segundos
        // ...
    }
}
```

**DESPUÉS (Patrón Singleton Resolutor):**
```csharp
// Connection\NHibernateHelper.cs (Corregido)
public class NHibernateHelper 
{
    private static ISessionFactory _erpContext;
    private static ISessionFactory _epicorBoxitoContext;
    private static readonly object _lock = new object();

    public static ISessionFactory ErpContext 
    {
        get {
            if (_erpContext == null) {
                lock(_lock) {
                    if (_erpContext == null) {
                        _erpContext = Fluently.Configure()
                            .Database(MsSqlConfiguration.MsSql2008.ConnectionString(Environment.GetEnvironmentVariable("DB_EPICORLIVE") ?? ConfigurationManager.ConnectionStrings["EpicorLive"].ConnectionString))
                            .Mappings(m => m.FluentMappings.AddFromAssemblyOf<CustomerRegularMap>())
                            .BuildSessionFactory();
                    }
                }
            }
            return _erpContext;
        }
    }

    public static ISessionFactory EpicorBoxitoContext
    {
        get {
            if (_epicorBoxitoContext == null) {
                lock(_lock) {
                    if (_epicorBoxitoContext == null) {
                        _epicorBoxitoContext = Fluently.Configure()
                            .Database(MsSqlConfiguration.MsSql2008.ConnectionString(Environment.GetEnvironmentVariable("DB_EPICORBOXITO") ?? ConfigurationManager.ConnectionStrings["EpicorBoxito"].ConnectionString))
                            .Mappings(m => m.FluentMappings.AddFromAssemblyOf<CustomerRegularMap>())
                            .BuildSessionFactory();
                    }
                }
            }
            return _epicorBoxitoContext;
        }
    }

    public static ISession OpenSessionErp() => ErpContext.OpenSession();
    public static ISession OpenSessionEpicorBoxito() => EpicorBoxitoContext.OpenSession();
}
```
*⚠️ Nota: En el código DESPUÉS se elimina la implementación de `IDisposable` a nivel de clase de fábrica, ya que los SessionFactories deben vivir a lo largo del ciclo de vida de la app.*

### Fase 2: Estrategia de Descomposición en Microservicios
1.  **Migración de Framework:** Crear un nuevo proyecto tipo `BackgroundService` (`Worker Service`) en **.NET 8**.
2.  **Abstracción de Logging:** Remover uso de `EventLog.WriteEntry`. Incorporar `Microsoft.Extensions.Logging` y configurar logs en consola (`stdout`).
3.  **Inyección de Dependencias Explicita:** Registrar `CustumerRegularRespository` como `Transient` o `Scoped` en `Program.cs`.

### Fase 3: Dockerización

**Archivo `.dockerignore` completo:**
```text
.dockerignore
.env
.git
.gitignore
.vs
.vscode
bin/
obj/
*.user
*.suo
```

**Archivo `Dockerfile` Multi-stage completo (.NET 8 Background Service):**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["BoxServiceAgenteCustomerRegulars/BoxServiceAgenteCustomerRegulars.csproj", "BoxServiceAgenteCustomerRegulars/"]
COPY ["BO/BO.csproj", "BO/"]
COPY ["Mappers/Mappers.csproj", "Mappers/"]
COPY ["Repositorys/Repositorys.csproj", "Repositorys/"]
COPY ["Connection/Connection.csproj", "Connection/"]
RUN dotnet restore "BoxServiceAgenteCustomerRegulars/BoxServiceAgenteCustomerRegulars.csproj"
COPY . .
WORKDIR "/src/BoxServiceAgenteCustomerRegulars"
RUN dotnet build "BoxServiceAgenteCustomerRegulars.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "BoxServiceAgenteCustomerRegulars.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
# Las configuraciones ahora son variables de entorno, no App.config
ENV APIParnet="http://10.40.3.19:44313/api/CustomerRegulars/AddOrUpdate"
ENTRYPOINT ["dotnet", "BoxServiceAgenteCustomerRegulars.dll"]
```

**Archivo `docker-compose.yml` completo:**
```yaml
version: '3.8'
services:
  agent-customer-regulars:
    image: myregistry.azurecr.io/boxito/agente-customer-regulars:latest
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - DB_EPICORLIVE=Data Source=10.40.3.72;Initial Catalog=EpicorERP;User ID=interfacesparnet;Password=${DB_PASSWORD};
      - DB_EPICORBOXITO=Data Source=10.40.3.72;Initial Catalog=EpicorBoxito;User ID=interfacesparnet;Password=${DB_PASSWORD};
      - IntervaloEjecucion=1.5
      - TZ=America/Mexico_City
    restart: unless-stopped
```

### Fase 4: Pipeline CI/CD y Sondajes Kubernetes

**Sondas para Kubernetes (Liveness/Readiness):**
*⚠️ Información no proporcionada en la entrada sobre endpoints web (es un Worker). Para habilitar sondas en un Worker, se debe exponer un Endpoint ligero de estado.*
```yaml
# Fragmento del deployment K8s
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  initialDelaySeconds: 15
  periodSeconds: 20
```
*(El Worker Service debería escribir (touch) a `/tmp/healthy` en cada iteración exitosa del `ProcesoAsync`).*

**Archivo `.github/workflows/docker-build-deploy.yml` completo:**
```yaml
name: CI/CD Agente Customer Regulars

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

    - name: Login to Azure Container Registry
      uses: docker/login-action@v2
      with:
        registry: myregistry.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}

    - name: Build and push Docker Image
      uses: docker/build-push-action@v3
      with:
        context: ./Componentes-Base/Agentes/Parnet-Orquestador-Base/BoxAgenteMaestroParnet/AgenteCustomerRegular
        file: ./Componentes-Base/Agentes/Parnet-Orquestador-Base/BoxAgenteMaestroParnet/AgenteCustomerRegular/Dockerfile
        push: true
        tags: myregistry.azurecr.io/boxito/agente-customer-regulars:${{ github.sha }}

    - name: Deploy to Kubernetes
      uses: Azure/k8s-deploy@v4
      with:
        manifests: |
          k8s/deployment.yaml
        images: |
          myregistry.azurecr.io/boxito/agente-customer-regulars:${{ github.sha }}
        imagepullsecrets: |
          acr-secret
```
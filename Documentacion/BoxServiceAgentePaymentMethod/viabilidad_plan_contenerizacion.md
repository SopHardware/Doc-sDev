# Dictamen y Plan de Modernización hacia Contenedores
**Aplicación:** BoxServiceAgentePaymentMethod
**Comité Consultor:** Arquitecto de Software Cloud-Native, Líder de DevSecOps, Auditor Principal de Ciberseguridad

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

**Resolución Final:** **VIABLE CON CONDICIONES**

**Justificación basada en "12-Factor App":**
El proyecto, tal como está, **NO ES VIABLE** para ser contenedorizado en entornos de alta densidad (Linux/Kubernetes) debido a las siguientes violaciones a los principios 12-Factor:
*   *Factor II (Dependencies)* y *Factor IX (Disposability)*: Está ligado fuertemente al ecosistema de Windows (`ServiceBase`) e inicia procesos de fondo en el `ThreadPool` que se interrumpirían abruptamente al recibir una señal de terminación (SIGTERM) en Kubernetes.
*   *Factor XI (Logs)*: Almacena logs en el Windows Event Log de manera monolítica, en lugar de escribirlos en la salida estándar (`stdout`) para su agregación.
*   *Factor III (Config)*: Depende de un `app.config` ligado al framework subyacente.

Para lograr viabilidad se exige **refactorizar el Servicio de Windows a un .NET Worker Service (C# 6.0/8.0)**, lo cual elimina las dependencias a Windows, expone los logs al `stdout`, y permite correr nativamente en un contenedor Linux.

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### Fase 1: Remediación Obligatoria (Seguridad y Concurrencia)

Antes de iniciar la modernización de plataforma, se debe resolver la Inyección de SQL.

**CÓDIGO ANTES (TblSyncRepository.cs):**
```csharp
public void Run()
{
    session = ConexionDB.OpenSession();
    ITransaction txDestino = session.BeginTransaction();
    try
    {
        // VULNERABILIDAD CRÍTICA DE SQL INJECTION
        string query = "UPDATE [SAS1005].[dbo].[Tbl_Sync_FormaPago] set ExportEstatus = "+ staus + " WHERE Sucursal = "+ payment.Sucursal + " AND Clave = '"+payment.Clave+"' ";
        txDestino.Begin();
        session.CreateSQLQuery(query).ExecuteUpdate();
        txDestino.Commit();
        EventLog.WriteEntry("BoxServiceAgentePaymentMethod", "MARCADO A 2!", EventLogEntryType.SuccessAudit, 101, 1);
    }
    // ... catch logic ...
}
```

**CÓDIGO DESPUÉS (TblSyncRepository.cs refactorizado):**
```csharp
public void Run()
{
    // Se recomienda que la sesión se inyecte vía constructor en la nueva arquitectura
    using (var session = ConexionDB.OpenSession())
    using (var txDestino = session.BeginTransaction())
    {
        try
        {
            string query = "UPDATE [SAS1005].[dbo].[Tbl_Sync_FormaPago] " +
                           "SET ExportEstatus = :estatus " +
                           "WHERE Sucursal = :sucursal AND Clave = :clave";

            session.CreateSQLQuery(query)
                   .SetParameter("estatus", staus)
                   .SetParameter("sucursal", payment.Sucursal)
                   .SetParameter("clave", payment.Clave)
                   .ExecuteUpdate();

            txDestino.Commit();
            // Refactor posterior requerido: Cambiar a ILogger
            Console.WriteLine($"[Success] Marcado a {staus} para Sucursal: {payment.Sucursal}, Clave: {payment.Clave}"); 
        }
        catch (Exception ex)
        {
            txDestino.Rollback();
            Console.WriteLine($"[Error] ERROR MARCADO: {ex.Message}");
            throw; // Evitar tragar la excepción silenciosamente
        }
    }
}
```

*⚠️ Información no proporcionada en la entrada:* No se especifica la versión destino del framework para .NET, se asume .NET 8 LTS para la migración. No se proporciona código detallado de las librerías `Orquestador.Api.Services` (se asume que son compatibles con .NET Standard o Core).

### Fase 2: Estrategia de Descomposición en Microservicios
1.  **Migración de Host:** Mover el proyecto a una plantilla de `Worker Service` de .NET (ej. `dotnet new worker`). Esto sustituye la clase `ServiceBase` y el `Timer` por un `BackgroundService`.
2.  **Inyección de Dependencias:** Registrar `NHibernate` y los Repositorios en el contenedor de servicios de .NET (`IServiceCollection`).
3.  **Logs como Eventos:** Reemplazar `EventLog` con `Microsoft.Extensions.Logging.ILogger` configurado para imprimir en Consola (vital para herramientas como fluentd/Promtail en K8s).

### Fase 3: Dockerización

**Dockerfile (Multi-stage para .NET 8):**
```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app

# Copiar csproj y restaurar dependencias
COPY ["BoxServiceAgentePaymentMethod/BoxServiceAgentePaymentMethod.csproj", "BoxServiceAgentePaymentMethod/"]
# ⚠️ Información no proporcionada: Se asume que las librerías BO, Mappers, etc. están en el mismo repo o son NuGets. 
# Si son librerías locales, deben añadirse los comandos COPY correspondientes aquí.
RUN dotnet restore "BoxServiceAgentePaymentMethod/BoxServiceAgentePaymentMethod.csproj"

# Copiar el resto del código y compilar
COPY . .
WORKDIR "/app/BoxServiceAgentePaymentMethod"
RUN dotnet build "BoxServiceAgentePaymentMethod.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "BoxServiceAgentePaymentMethod.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Run
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Variables de entorno por defecto
ENV IntervaloEjecucion=4
ENV Company="DEFAULT_COMPANY"

# Punto de entrada
ENTRYPOINT ["dotnet", "BoxServiceAgentePaymentMethod.dll"]
```

**.dockerignore:**
```text
.vs/
.vscode/
bin/
obj/
*.user
*.suo
```

**docker-compose.yml:**
```yaml
version: '3.8'
services:
  payment-agent:
    build: 
      context: .
      dockerfile: BoxServiceAgentePaymentMethod/Dockerfile
    environment:
      - IntervaloEjecucion=4
      - Authorization=${ORQUESTADOR_AUTH}
      - x-api-key=${ORQUESTADOR_API_KEY}
      - ConnectionStrings__DataBase=${DB_CONNECTION_STRING}
    restart: unless-stopped
```

### Fase 4: Pipeline CI/CD y Preparación para Kubernetes

**GitHub Actions (CI/CD Pipeline - main.yml):**
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ "DEV.VELA", "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
        
    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: BoxServiceAgentePaymentMethod/Dockerfile
        push: true
        tags: organization/payment-agent:latest
```

**Sondas para Kubernetes (Liveness/Readiness):**
Dado que es un Worker Service (sin puertos HTTP por defecto), se recomienda añadir el paquete de `Microsoft.Extensions.Diagnostics.HealthChecks` y habilitar un endpoint de status ligero por el puerto 8080:

```yaml
# En el deployment de K8s:
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```
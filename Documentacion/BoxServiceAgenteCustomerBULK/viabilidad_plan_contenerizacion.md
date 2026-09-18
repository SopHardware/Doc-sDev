=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceAgenteCustomerBULK

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES ESTRICTAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[Bloqueante]` Uso intensivo de `App.config` para almacenar cadenas de conexión, secretos y API Keys. En Docker, esto expone las credenciales de producción en la capa de la imagen.
2. **Backing Services (Factor 4):** `[Riesgo Crítico]` La reconstrucción continua de la factoría de NHibernate (SessionFactories) dentro del loop transaccional causará *Memory Leaks* masivos, provocando OOMKilled (Out of Memory) rápidos en K8s.
3. **Telemetría y Logs (Factor 11):** `[No Cumple]` Eventos enviados a la API de Windows que deben ser reimplementados hacia `stdout`.
4. **Estado (Factor 6):** `[Cumple]` La lógica es procesal y no mantiene un estado local (solo memoria caché que debe mitigarse).

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
// FUGA MASIVA DE MEMORIA Y CONEXIONES (NHibernateHelper)
public class NHibernateHelper
{
    // Creado dinámicamente en cada ciclo en lugar de ser un Singleton Estático
    public ISession OpenSession()
    {
        // Esto compila la factoría de mapeos por cada cliente procesado!
        ISessionFactory sessionFactory = Fluently.Configure()
            .Database(MsSqlConfiguration.MsSql2012.ConnectionString(...))
            .Mappings(m => m.FluentMappings.AddFromAssemblyOf<Tbl_Sync_Customer>())
            .BuildSessionFactory();
            
        return sessionFactory.OpenSession();
    }
}

// BUG EN ENTIDAD (Tbl_Sync_Customer.cs)
public override bool Equals(object obj)
{
    var other = obj as Tbl_Sync_Customer;
    // CRÍTICO: Evalúa desigualdad provocando que NHibernate falle en actualizar la caché L1/L2
    return this.CustID != other.CustID; 
}
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO CON SINGLETON)
```csharp
// CORRECCIÓN ENTIDAD
public override bool Equals(object obj)
{
    if (obj is not Tbl_Sync_Customer other) return false;
    return this.CustID == other.CustID;
}
public override int GetHashCode() => CustID?.GetHashCode() ?? 0;

// SINGLETON CORRECTO DE NHIBERNATE
public sealed class NHibernateHelper
{
    private static readonly Lazy<ISessionFactory> _sessionFactory = new Lazy<ISessionFactory>(() =>
    {
        var connectionString = Environment.GetEnvironmentVariable("DB_CONNECTION_STRING");
        return Fluently.Configure()
            .Database(MsSqlConfiguration.MsSql2012.ConnectionString(connectionString))
            .Mappings(m => m.FluentMappings.AddFromAssemblyOf<Tbl_Sync_Customer>())
            .BuildSessionFactory();
    });

    public static ISession OpenSession() => _sessionFactory.Value.OpenSession();
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
Implementar un contexto **"Customer Master Data Management (MDM)"**. Separar las cargas BULK (Masivas) de las cargas Delta/Regulares para usar herramientas de ETL asíncronas como Apache Kafka Connect, evitando el procesamiento fila por fila para grandes volúmenes.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.CustomerBULK.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.CustomerBULK.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.CustomerBULK.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.CustomerBULK.dll"]
```

### FASE 4: PIPELINE CI/CD, SEGURIDAD Y HEALTHCHECKS
```yaml
name: Deploy Customer BULK Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    # Análisis SAST para detectar secretos hardcodeados
    - uses: snyk/actions/dotnet@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
    - run: docker build -t boxitohub/customer-bulk-agent:${{ github.sha }} .
```
*(Se debe parametrizar Kubernetes `Secret` para inyectar credenciales seguras vía `envFrom`).*
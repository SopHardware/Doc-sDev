=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceAgentInitialCharge

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Configuración (Factor 3):** `[No Cumple]` Cargar las claves API `Authorization` y `XApiKey` desde `App.config` es incompatible con contenedores. Debe portarse a variables de entorno para cumplir con 12-Factor App.
2. **Concurrencia (Factor 8):** `[Cumple]` Al ser una herramienta de carga inicial que se ejecuta una sola vez, funciona idealmente como un **Kubernetes Job** de corta duración (*stateless Batch Job*), en lugar de un Daemon permanente.
3. **Telemetría y Logs (Factor 11):** `[No Cumple]` No debe registrar logs de manera local; las salidas de streams deben enviarse directamente a `stdout` utilizando `ILogger`.
4. **Manejo de Conexiones (Factor 4):** `[Riesgo Crítico]` La instanciación incontrolada de `HttpClient` provocará el agotamiento de sockets. Es un requisito bloquear la migración hasta centralizar el ciclo de vida del cliente HTTP.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
public static async Task<Tuple<CustomerResponse, string>> ParnetPostCustomer(CustomerRequest request, string help)
{
    string apiUri = $"{ConfigurationManager.AppSettings["OrquestadorApi"]}/execute";
    string oJson = JsonConvert.SerializeObject(request);

    try
    {
        HttpRequestMessage httpRequestMessage = new HttpRequestMessage();
        httpRequestMessage.Headers.Add("Authorization", ConfigurationManager.AppSettings["Authorization"]);
        httpRequestMessage.Headers.Add("x-api-key", ConfigurationManager.AppSettings["XApiKey"]);
        httpRequestMessage.RequestUri = new Uri(apiUri);
        httpRequestMessage.Method = HttpMethod.Post;
        httpRequestMessage.Content = new StringContent(oJson, Encoding.UTF8, "application/json");

        // RIESGO: Instanciación directa de HttpClient
        HttpClient httpClient = new HttpClient();
        httpClient.Timeout = new TimeSpan(0, 30, 0); // Timeout destructivo

        using (var response = await httpClient.SendAsync(httpRequestMessage))
        using (var result = await response.Content.ReadAsStreamAsync())
        using (var stream = new StreamReader(result))
        {
            string strRead = await stream.ReadToEndAsync();
            return new Tuple<CustomerResponse, string>(JsonConvert.DeserializeObject<CustomerResponse>(strRead), "200");
        }
    }
    catch (Exception ex)
    {
        throw ex;
    }
}
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO CON CLIENTE HTTP CENTRALIZADO)
```csharp
public class InitialChargeSyncManager : IInitialChargeSyncManager
{
    private readonly HttpClient _httpClient; // Inyectado como Singleton mediante HttpClientFactory
    private readonly ILogger<InitialChargeSyncManager> _logger;
    private readonly ApiSettings _settings;

    public InitialChargeSyncManager(HttpClient httpClient, IOptions<ApiSettings> settings, ILogger<InitialChargeSyncManager> logger)
    {
        _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _settings = settings?.Value ?? throw new ArgumentNullException(nameof(settings));
    }

    public async Task<CustomerResponse> ParnetPostCustomerAsync(CustomerRequest request, string endpointHelp, CancellationToken cancellationToken)
    {
        var apiUri = $"{_settings.OrquestadorApiUrl}/execute";
        var jsonBody = new Dictionary<string, object>
        {
            { "alias", endpointHelp },
            { "params", null },
            { "headers", null },
            { "body", request }
        };

        var requestContent = new StringContent(JsonConvert.SerializeObject(jsonBody), Encoding.UTF8, "application/json");

        using var requestMessage = new HttpRequestMessage(HttpMethod.Post, apiUri);
        requestMessage.Headers.Add("Authorization", _settings.AuthorizationToken);
        requestMessage.Headers.Add("x-api-key", _settings.XApiKey);
        requestMessage.Headers.Add("version", _settings.Version);
        requestMessage.Headers.Add("help", endpointHelp);
        requestMessage.Content = requestContent;

        try
        {
            _logger.LogInformation("Enviando petición de carga masiva para: {Help}", endpointHelp);
            using var response = await _httpClient.SendAsync(requestMessage, cancellationToken);
            response.EnsureSuccessStatusCode();

            using var contentStream = await response.Content.ReadAsStreamAsync(cancellationToken);
            using var streamReader = new StreamReader(contentStream);
            var rawResponse = await streamReader.ReadToEndAsync();

            var deserialized = JsonConvert.DeserializeObject<CustomerResponseHTTPClient>(rawResponse);
            return new CustomerResponse 
            { 
                Status = deserialized?.Respuesta.Status ?? 3, 
                Customers = deserialized?.Respuesta.Customers, 
                Message = deserialized?.Respuesta.Message 
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Falla crítica en carga masiva para {Help}", endpointHelp);
            return new CustomerResponse { Status = 3, Message = ex.Message, Customers = null };
        }
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
El agente de Carga Inicial debe considerarse un componente de **Migración/Inicialización transitoria (Job)**. No debe ser un servicio en ejecución continua. Se desacopla en un **Kubernetes Job** que corre únicamente durante el aprovisionamiento de una nueva sucursal o reinstalación de base de datos local.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.InitialCharge.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.InitialCharge.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.InitialCharge.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.InitialCharge.dll"]
```

### FASE 4: PIPELINE CI/CD, SEGURIDAD Y HEALTHCHECKS
```yaml
name: Deploy InitialCharge Job
on:
  push:
    branches: [ "master" ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - run: docker build -t boxitohub/initial-charge-job:${{ github.sha }} .
```
*(Para K8s, al ser un Job, no requiere liveness probe HTTP persistente; el orquestador valida la salud controlando el Exit Code del contenedor).*
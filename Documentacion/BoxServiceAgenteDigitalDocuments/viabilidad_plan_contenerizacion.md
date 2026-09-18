=============================================================================
### ARCHIVO 2: viabilidad_plan_contenerizacion.md
=============================================================================

# PLAN DE VIABILIDAD DE CONTENERIZACIÓN Y MODERNIZACIÓN CLOUD-NATIVE
**Comité Consultor de Élite: Arquitectura Cloud-Native, DevSecOps y Ciberseguridad**
**Aplicación:** BoxServiceAgenteDigitalDocuments

---

## 2.1 DICTAMEN DE VIABILIDAD PARA CONTENEDORES (DOCKER / KUBERNETES)

### **Dictamen:** `[VIABLE CON CONDICIONES CRÍTICAS]`

#### **Justificación Técnica (Basada en 12-Factor App):**
1. **Almacenamiento y Estado (Factor 6):** `[No Cumple]` La aplicación escribe archivos temporales XML y PDF en rutas duras de Windows (ej. `C:\XD`). Para operar en contenedores efímeros (Docker/K8s), debe reescribirse para usar flujos en memoria (`MemoryStream`) o directorios temporales de Linux configurados mediante variables de entorno (ej. `/tmp/`), evitando persistir nada localmente.
2. **Backing Services (Factor 4):** `[Riesgo Crítico]` La aplicación interactúa con tres bases de datos SQL de manera transaccional combinando llamadas HTTP a BoxDrive API. Mantener transacciones abiertas de base de datos mientras se hace una transferencia de red HTTP (Multipart Upload de PDFs) es un antipatrón severo que causará bloqueos (*lock contention*) masivos.
3. **Concurrencia (Factor 8):** `[Bloqueante]` El uso indiscriminado de `.Wait()` sobre peticiones `HttpClient` generará un estrangulamiento de sockets (*socket exhaustion*) y un bloqueo de hilos de ejecución en Docker, donde los límites de hilos virtuales y recursos de CPU son controlados.

---

## 2.2 PLAN DE ACCIÓN Y HOJA DE RUTA DE TRANSICIÓN

### FASE 1: REMEDIACIÓN OBLIGATORIA DEL CÓDIGO (REFACTORIZACIÓN)

#### CÓDIGO ANTES (DEUDA CRÍTICA EN C#)
```csharp
public async Task ImportBoxitoToApiDocumetos()
{
    // Transacción de base de datos abarcando llamadas de red
    using (var transaction = _dbContext.Database.BeginTransaction())
    {
        var pendingDocs = _dbContext.TblSyncDocs.Where(x => x.ExportStatus == 0).ToList();
        foreach (var doc in pendingDocs)
        {
            // Escribe a ruta dura C:\XD
            string pathXml = $@"C:\XD\{doc.Folio}.xml";
            File.WriteAllText(pathXml, doc.XmlContent);

            // Genera PDF localmente
            string pathPdf = $@"C:\XD\{doc.Folio}.pdf";
            PDFHelper.Generate(pathXml, pathPdf);

            // LLAMADA HTTP BLOQUEANTE DENTRO DE TRANSACCIÓN SQL
            using (var client = new HttpClient()) // Socket exhaustion
            {
                var content = new MultipartFormDataContent();
                content.Add(new ByteArrayContent(File.ReadAllBytes(pathPdf)), "file", $"{doc.Folio}.pdf");
                var response = client.PostAsync("http://10.40.3.18/api/upload", content).Result; // .Result / .Wait() bloqueante
                
                if (response.IsSuccessStatusCode)
                {
                    doc.ExportStatus = 2;
                    _dbContext.SaveChanges();
                }
            }
            File.Delete(pathXml);
            File.Delete(pathPdf);
        }
        transaction.Commit(); // Fin de la larguísima transacción
    }
}
```

#### CÓDIGO DESPUÉS (.NET 8 WORKER SEGURO, MEMORY STREAM Y SINGLETON HTTPCLIENT)
```csharp
public class DigitalDocumentsSyncManager : IDigitalDocumentsSyncManager
{
    private readonly HttpClient _httpClient; // HttpClient inyectado por IoC (Singleton)
    private readonly IDocumentsRepository _repository;
    private readonly ILogger<DigitalDocumentsSyncManager> _logger;
    private readonly AppConfig _config;

    public DigitalDocumentsSyncManager(HttpClient httpClient, IDocumentsRepository repository, IOptions<AppConfig> config, ILogger<DigitalDocumentsSyncManager> logger)
    {
        _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _config = config?.Value ?? throw new ArgumentNullException(nameof(config));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task ProcessDocumentsAsync(CancellationToken cancellationToken)
    {
        // 1. Obtener registros de la base de datos sin abrir transacciones prolongadas
        var pendingDocs = await _repository.GetPendingDocumentsAsync();
        _logger.LogInformation("Procesando {Count} documentos de manera asíncrona.", pendingDocs.Count);

        using (var semaphore = new SemaphoreSlim(_config.MaxConcurrentUploads))
        {
            var tasks = pendingDocs.Select(async doc =>
            {
                await semaphore.WaitAsync(cancellationToken);
                try
                {
                    // 2. Procesamiento en memoria pura utilizando Streams (sin tocar disco C:\XD)
                    using var xmlStream = new MemoryStream(Encoding.UTF8.GetBytes(doc.XmlContent));
                    using var pdfStream = new MemoryStream();
                    
                    await PDFHelper.GenerateToStreamAsync(xmlStream, pdfStream, cancellationToken);
                    pdfStream.Position = 0;

                    // 3. Envío HTTP asíncrono no bloqueante
                    using var content = new MultipartFormDataContent();
                    var fileContent = new StreamContent(pdfStream);
                    content.Add(fileContent, "file", $"{doc.Folio}.pdf");

                    var response = await _httpClient.PostAsync($"{_config.BoxDriveApiUrl}/upload", content, cancellationToken);

                    if (response.IsSuccessStatusCode)
                    {
                        // 4. Actualizar estado de manera transaccional corta y aislada
                        await _repository.UpdateExportStatusAsync(doc.SyncID, 2);
                        _logger.LogInformation("Documento {Folio} cargado exitosamente.", doc.Folio);
                    }
                    else
                    {
                        _logger.LogError("Falla al cargar documento {Folio}. HTTP Status: {Status}", doc.Folio, response.StatusCode);
                    }
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error procesando el documento {Folio}", doc.Folio);
                }
                finally
                {
                    semaphore.Release();
                }
            });

            await Task.WhenAll(tasks);
        }
    }
}
```

### FASE 2: ESTRATEGIA DE DESCOMPOSICIÓN EN MICROSERVICIOS
Abstraer en el contexto **"Digital Document & CFDI Engine Context"**. Se encargará exclusivamente del procesamiento de archivos, PDF e inyección a BoxDrive. El microservicio funcionará como un consumidor asíncrono que reaccionará a un evento `InvoiceCreated` en una cola (RabbitMQ/NATS), procesando el PDF y XML de manera reactiva e inmediata, eliminando el "polling" por intervalos de minutos.

### FASE 3: DOCKERIZACIÓN Y ORQUESTACIÓN LOCAL

#### `Dockerfile` (Multi-stage Build)
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY ["Boxito.Agentes.DigitalDocuments.csproj", "./"]
RUN dotnet restore "Boxito.Agentes.DigitalDocuments.csproj"
COPY . .
RUN dotnet publish "Boxito.Agentes.DigitalDocuments.csproj" -c Release -o /app/publish -p:PublishTrimmed=true

FROM mcr.microsoft.com/dotnet/runtime:8.0-alpine AS final
WORKDIR /app
# Instalar dependencias requeridas para la generación de PDFs y fuentes tipográficas en Alpine Linux
RUN apk add --no-cache icu-libs fontconfig ttf-dejavu
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build --chown=appuser:appgroup /app/publish .
ENTRYPOINT ["dotnet", "Boxito.Agentes.DigitalDocuments.dll"]
```

### FASE 4: PIPELINE CI/CD, SEGURIDAD Y HEALTHCHECKS
```yaml
name: Deploy DigitalDocuments Agent
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-dotnet@v3
      with: { dotnet-version: '8.0.x' }
    - run: dotnet test
    - name: Docker Build and Scan
      run: |
        docker build -t boxitohub/digital-documents-agent:${{ github.sha }} .
```
*(Sondas Liveness y Readiness para K8s apuntarán al mecanismo de Heartbeat por archivo temporal en `/tmp/heartbeat`).*
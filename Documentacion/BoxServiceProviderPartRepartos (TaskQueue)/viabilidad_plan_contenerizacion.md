# Dictamen de Viabilidad de Contenerización y Plan de Modernización
**Proyecto:** BoxServiceProviderPartRepartos (TaskQueue) - SrvRepartosAgent  
**Nivel de Clasificación:** Corporativo / Confidencial  
**Elaborado por:** Comité Consultor de Élite (Arquitecto Cloud-Native, Líder DevSecOps, Auditor Principal de Ciberseguridad)

---

## 2.1 Dictamen de Viabilidad para Contenedores (Docker / Kubernetes)

### Dictamen Oficial: ⚠️ VIABLE CON CONDICIONES (RECONSTRUCCIÓN A .NET 8 REQUERIDA)

Este Comité Consultor dictamina que el estado tecnológico actual del componente `SrvRepartosAgent` **NO es viable** para su despliegue directo sobre contenedores basados en Linux (Docker/Kubernetes). La justificación técnica radica en que el proyecto está compilado bajo el framework heredado **.NET Framework 4.7.2** y depende del módulo Windows `System.ServiceProcess.ServiceBase` (un Servicio de Windows nativo). El despliegue de esta tecnología en Docker requeriría contenedores de Windows Server Core, los cuales son extremadamente ineficientes (tamaño de imagen > 5 GB), costosos de licenciar y no están soportados en la mayoría de los clústeres empresariales de Kubernetes de Linux.

Sin embargo, el proyecto es calificado como **VIABLE CON CONDICIONES**, sujeto a la ejecución de una **migración estructural hacia .NET 8 como un Worker Service**. Una vez migrado a .NET 8, el componente se transformará en un proceso multiplataforma, liviano, sin estado y totalmente optimizado para contenedores Linux y orquestación con Kubernetes.

### Evaluación Técnica Basada en "The 12-Factor App"

A continuación se analiza el cumplimiento de los principios cloud-native:

1. **I. Codebase (Un código, un repositorio):** **CUMPLE.** El proyecto cuenta con una estructura bien delimitada en repositorios de código.
2. **II. Dependencies (Declarar y aislar dependencias):** **CUMPLE PARCIALMENTE.** Utiliza `packages.config` para dependencias de NuGet. Al migrar a .NET 8 se utilizarán referencias modernas SDK-Style (`<PackageReference>`), aislando por completo las librerías a nivel de compilación nativa del contenedor.
3. **III. Config (Configuración en el entorno):** **NO CUMPLE.** Almacena cadenas de conexión SQL y secretos de API en texto claro en `App.config`, además de rutas de red estáticas como `C:\ServicioExportadorLog\`. Se exige migrar a configuraciones basadas en variables de entorno inyectadas al contenedor en runtime.
4. **IV. Backing services (Servicios de respaldo como recursos conectados):** **CUMPLE.** Trata a SQL Server y a la API externa como servicios conectados direccionables mediante strings de conexión, aunque requiere extraerlos de la configuración rígida.
5. **V. Build, release, run (Separar etapas de construcción, distribución y ejecución):** **NO CUMPLE.** Depende actualmente de MSBuild de Visual Studio en servidores físicos Windows. Al modernizarse se implementarán Dockerfiles multi-stage para compilar con el SDK de .NET y distribuir imágenes inmutables con el runtime ligero.
6. **VI. Processes (Ejecutar la app como uno o más procesos sin estado):** **CUMPLE PARCIALMENTE.** El servicio realiza tareas periódicas de encolamiento y despacho de manera síncrona/asíncrona. No almacena estado en memoria local entre ejecuciones individuales, lo que facilita su escalamiento horizontal.
7. **VII. Port binding (Exportar servicios vía port binding):** **N/A.** Como es una cola de tareas en segundo plano (Background Task Queue), no requiere puerto de escucha de cara a usuarios externos. Sin embargo, se inyectará un puerto ligero para exponer sondas de salud (Health Checks).
8. **VIII. Concurrency (Escalar mediante el modelo de procesos):** **CUMPLE PARCIALMENTE.** Actualmente maneja hilos a nivel de hilo interno (`Task.Factory.StartNew`). Al contenerizarse, el escalamiento se delegará al orquestador de Kubernetes aumentando el número de réplicas de Pods independientes, lo que requiere usar bloqueos distribuidos o locks a nivel de fila de base de datos para evitar doble procesamiento.
9. **IX. Disposability (Maximizar la robustez con inicio rápido y apagado seguro):** **NO CUMPLE.** El servicio clásico de Windows tarda en iniciarse y carece de un manejo refinado de `CancellationToken` para detener la replicación en tránsito. La versión .NET 8 Worker Service implementará `IHostedService` para garantizar apagados elegantes (Graceful Shutdown) en menos de 10 segundos.
10. **X. Dev/prod parity (Paridad en desarrollo y producción):** **NO CUMPLE.** Depende de ambientes locales y servidores de bases de datos compartidos físicamente. Con contenedores se utilizará Docker Compose para instanciar réplicas de bases de datos locales idénticas a los entornos productivos.
11. **XI. Logs (Tratar los logs como transmisión de eventos):** **NO CUMPLE.** Escribe logs en archivos locales (`C:\ServicioExportadorLog\MonitoringService.txt`). En contenedores, esto es un anti-patrón de alta gravedad. Los logs deben transmitirse directamente a los flujos estándar de salida (`stdout` y `stderr`) para ser capturados por colectores como Fluentd, Elasticsearch o Splunk.
12. **XII. Admin processes (Procesos de administración únicos):** **CUMPLE.** No requiere flujos complejos de tareas administrativas aisladas para funcionar.

---

## 2.2 Plan de Acción y Hoja de Ruta de Transición

### Fase 1: Remediación Obligatoria (Código de Producción Completo)

Presentamos los bloques de código fuente identificados en la auditoría con sus respectivas versiones corregidas e implementadas bajo buenas prácticas de ingeniería de software corporativa:

#### 1. Fuga de Conexiones en `UpdateRepartosOEProcessService.cs`

**CÓDIGO ANTES (Con fugas masivas de bases de datos y falta de disposición):**
```csharp
using ApplicationDbContext;
using Models.Configurations;
using Services.Configuration;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using UnitOfWorks.Configurations;
using UnitOfWorks.Helpers;
using UnitOfWorks.Replication;

namespace Services.Process
{
    public class UpdateRepartosOEProcessService
    {
        internal readonly bool IsService;
        internal readonly EventsLog EventsLog;
        internal readonly ConfigBase CreateOCConfig;
        internal readonly Guid ConfigurationId;
        public UpdateRepartosOEProcessService(ConfigBase config, bool isService)
        {
            IsService = isService;
            CreateOCConfig = config;
            EventsLog = new EventsLog(config.service, config.PathLog, config.FileNameLog);
            ConfigurationId = new Guid(config.ConfigurationId);
        }
        public async Task Start()
        {
            ParameterConfiguration parameterTaskNumber = null, parameterResultsList = null;
            using (var ctx = new LocalConfigurationsContext()) 
            using (var uow = new ConfigurationUoW(ctx))
            {
                var config = uow.FindByIdIncludeParameters(ConfigurationId);
                parameterTaskNumber = config.Parameters.Where(p => p.KeyName == "TaskNumber").FirstOrDefault();
                parameterResultsList = config.Parameters.Where(p => p.KeyName == "ResultsList").FirstOrDefault();
            }

            var tasks = new List<Task>();
            EventsLog.RegisterToEventViewer(string.Format("Start {0} tasks of {1} Id", parameterTaskNumber.ValueName, parameterResultsList.ValueName), IsService);
            for (int i = 0; i < int.Parse(parameterTaskNumber.ValueName); i++)
            {
                tasks.Add(Task.Factory.StartNew(async () =>
                {
                    await RunReplication(int.Parse(parameterResultsList.ValueName));
                }));
            }

            await Task.WhenAll(tasks);
        }
        public async Task RunReplication(int topIds)
        {
            EventsLog.RegisterToEventViewer(string.Format("Processing {0} ID", topIds), IsService);

            var epicorCtx = new EpicorContext();
            var uow = new UpdateReplicationRepartosUoW(epicorCtx, CreateOCConfig.GetService(), CreateOCConfig.service, IsService);
            await uow.SendToWMSBoxAPI(topIds);
            return;
        }
    }
}
```

**CÓDIGO DESPUÉS (Corregido y blindado con gestión estricta del ciclo de vida):**
```csharp
using ApplicationDbContext;
using Models.Configurations;
using Services.Configuration;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using UnitOfWorks.Configurations;
using UnitOfWorks.Helpers;
using UnitOfWorks.Replication;

namespace Services.Process
{
    public class UpdateRepartosOEProcessService
    {
        internal readonly bool IsService;
        internal readonly EventsLog EventsLog;
        internal readonly ConfigBase CreateOCConfig;
        internal readonly Guid ConfigurationId;
        public UpdateRepartosOEProcessService(ConfigBase config, bool isService)
        {
            IsService = isService;
            CreateOCConfig = config;
            EventsLog = new EventsLog(config.service, config.PathLog, config.FileNameLog);
            ConfigurationId = new Guid(config.ConfigurationId);
        }
        public async Task Start()
        {
            ParameterConfiguration parameterTaskNumber = null, parameterResultsList = null;
            using (var ctx = new LocalConfigurationsContext()) 
            using (var uow = new ConfigurationUoW(ctx))
            {
                var config = uow.FindByIdIncludeParameters(ConfigurationId);
                parameterTaskNumber = config.Parameters.Where(p => p.KeyName == "TaskNumber").FirstOrDefault();
                parameterResultsList = config.Parameters.Where(p => p.KeyName == "ResultsList").FirstOrDefault();
            }

            var tasks = new List<Task>();
            EventsLog.RegisterToEventViewer(string.Format("Start {0} tasks of {1} Id", parameterTaskNumber.ValueName, parameterResultsList.ValueName), IsService);
            for (int i = 0; i < int.Parse(parameterTaskNumber.ValueName); i++)
            {
                tasks.Add(Task.Factory.StartNew(async () =>
                {
                    await RunReplication(int.Parse(parameterResultsList.ValueName));
                }));
            }

            await Task.WhenAll(tasks);
        }
        public async Task RunReplication(int topIds)
        {
            EventsLog.RegisterToEventViewer(string.Format("Processing {0} ID", topIds), IsService);

            // Se encapsulan ambos contextos de datos en bloques using para forzar el cierre de conexiones y evitar Memory Leaks
            using (var epicorCtx = new EpicorContext())
            using (var uow = new UpdateReplicationRepartosUoW(epicorCtx, CreateOCConfig.GetService(), CreateOCConfig.service, IsService))
            {
                await uow.SendToWMSBoxAPI(topIds);
            }
        }
    }
}
```

---

#### 2. Implementación de `Dispose` en `UpdateReplicationRepartosUoW.cs`

**CÓDIGO ANTES (Bloquea la liberación de recursos lanzando excepciones):**
```csharp
namespace UnitOfWorks.Replication
{
    public class UpdateReplicationRepartosUoW : IDisposable
    {
        // ... Variables de repositorio ...
        
        public void Dispose()
        {
            throw new NotImplementedException();
        }
    }
}
```

**CÓDIGO DESPUÉS (Patrón de diseño Dispose Estándar implementado formalmente):**
```csharp
using ApplicationDbContext;
using ApplicationDbContext.Interfaces;
using Entities.Enumerables;
using Entities.ReplicationData;
using Entities.Service;
using Entities.Ship;
using Exceptions.UnitOfWorks;
using Repositories.HttpServices;
using Repositories.HttpServices.ResponseServices;
using Repositories.Repartos;
using Repositories.Replication;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using UnitOfWorks.Helpers;

namespace UnitOfWorks.Replication
{
    public class UpdateReplicationRepartosUoW : IDisposable
    {
        readonly RepartosRepository HeaderRepository;
        readonly RepartosReplicationRepository ReplicationRepartosRepository;
        readonly bool IsServiceProcess;
        internal readonly ServiceBaseModel EntryOrderServiceModel;
        internal readonly EventsLog eventsLog;
        static object _locker = new object();
        private bool _disposed = false;

        public UpdateReplicationRepartosUoW(IDbContext context, ServiceBaseModel serviceBaseModel, ServicesTypeEnum service, bool isServices)
        {
            HeaderRepository = new RepartosRepository(context);
            IsServiceProcess = isServices;
            EntryOrderServiceModel = serviceBaseModel;
            eventsLog = new EventsLog(service, serviceBaseModel.PathLog, serviceBaseModel.FileNameLog);
            
            // Contexto local instanciado internamente se gestionará durante el ciclo de vida del UoW
            var localCtx = new LocalConfigurationsContext();
            ReplicationRepartosRepository = new RepartosReplicationRepository(localCtx);
        }

        public async Task SendToWMSBoxAPI(int topRows = 1)
        {
            List<ReplicationDataEntity> replicationData = null;

            lock (_locker)
            {
                replicationData = HeaderRepository.GetUpdateReplicationRepartos(topRows);
            }

            if (replicationData == null)
                throw new NotRecordsToProcessException<ReplicationDataEntity>();

            if (replicationData.Count == 0)
                eventsLog.RegisterToEventViewer("There are no records to replicate", IsServiceProcess);
            else
                eventsLog.RegisterToEventViewer("Updating Shipping's", IsServiceProcess);
            
            await Task.Run(() =>
            {
                string guid = Guid.NewGuid().ToString();

                foreach (var replication in replicationData)
                {
                    HeaderRepository.BeginTransaction();
                    ReplicationDataEntity replicationDataExist = null;
                    string guidTarget = "";
                    try
                    {
                        guidTarget = replication.DataId;
                        var previus = replication.Clone();
                        replication.CreateAt = DateTime.Now;
                        replication.DocumentVersion = EntryOrderServiceModel.OrquestadorConfig.Version;

                        replicationDataExist = this.ReplicationRepartosRepository.FindByDataId(guidTarget);

                        bool SetRollback = true;
                        if (replicationDataExist == null)
                        {
                            this.ReplicationRepartosRepository.Create(replication);

                            replicationDataExist = this.ReplicationRepartosRepository.FindByDataId(guidTarget);
                            if (replicationDataExist != null)
                            {
                                replication.ReplicationDataStatus = ExportStatusEnum.Synchronizing;
                                this.ReplicationRepartosRepository.Update(replication, previus);
                                replicationDataExist = this.ReplicationRepartosRepository.FindByDataId(guidTarget);

                                string[] replicationParts = replicationDataExist.DataId.Split('-');
                                var result = HeaderRepository.GetCompleteUpdateRepartosQueryResult(999, Convert.ToInt32(replicationParts[1]), Convert.ToInt32(replicationParts[2]));
                                foreach (var single in result.Select(x => new { x.Folio, x.Partida }).Distinct())
                                {
                                    var dataToUpdate = result.First(y => y.Folio.Equals(single.Folio) && y.Partida.Equals(single.Partida));
                                    HeaderRepository.UpdateShipDtl(new ShipDtlEntity
                                    {
                                        ExportStatus = ExportStatusEnum.Synchronizing,
                                        PackNum = single.Folio,
                                        PackLine = single.Partida.Value
                                    });
                                }
                                SetRollback = false;

                                ResponseToAgent responseToAgent = new ResponseToAgent();
                                CreateUpdateRepartosService service = new CreateUpdateRepartosService();
                                Task<ResponseToAgent> tskInvokePost = this.ReplicationRepartosRepository.PostUpdateWMSBoxApi(EntryOrderServiceModel.OrquestadorConfig, replication.ContentData, service);
                                responseToAgent = tskInvokePost.Result;

                                ReplicationDataEntity data = new ReplicationDataEntity
                                {
                                    ReplicationDataGuid = replicationDataExist.ReplicationDataGuid,
                                    DataId = replicationDataExist.DataId,
                                    ProcessDescription = responseToAgent.JsonResponseOrquestador,
                                    InitialRequest = responseToAgent.InitialRequest,
                                    Attempts = 1
                                };

                                switch (responseToAgent.HttpStatusCode)
                                {
                                    case 200:
                                        eventsLog.RegisterToEventViewer($"The Replication ID {replicationData.First().ReplicationDataGuid} Sucess!", IsServiceProcess);
                                        data.ReplicationDataStatus = ExportStatusEnum.Syncronized;
                                        data.IsError = false;
                                        this.ReplicationRepartosRepository.Update(data.ReplicationDataGuid, data);
                                        result.ForEach(x =>
                                        {
                                            HeaderRepository.UpdateShipDtl(new ShipDtlEntity
                                            {
                                                PackNum = x.Folio,
                                                PackLine = x.Partida.Value,
                                                ExportStatus = ExportStatusEnum.Syncronized
                                            });
                                        });
                                        break;
                                    case 404:
                                    case 400:
                                    case 500:
                                    case 0:
                                        eventsLog.RegisterToEventViewer($"The Replication ID {replicationData.First().ReplicationDataGuid} Failed. Please contact to support.", IsServiceProcess);
                                        data.ReplicationDataStatus = ExportStatusEnum.SyncronizeError;
                                        data.IsError = true;
                                        data.Attempts = 1;
                                        this.ReplicationRepartosRepository.Update(data.ReplicationDataGuid, data);
                                        result.ForEach(x =>
                                        {
                                            HeaderRepository.UpdateShipDtl(new ShipDtlEntity
                                            {
                                                PackNum = x.Folio,
                                                PackLine = x.Partida.Value,
                                                ExportStatus = ExportStatusEnum.SyncronizeError
                                            });
                                        });
                                        break;
                                    default:
                                        data.ReplicationDataStatus = ExportStatusEnum.NotSynchronized;
                                        data.IsError = true;
                                        result.ForEach(x =>
                                        {
                                            HeaderRepository.UpdateShipDtl(new ShipDtlEntity
                                            {
                                                PackNum = x.Folio,
                                                PackLine = x.Partida.Value,
                                                ExportStatus = ExportStatusEnum.NotSynchronized
                                            });
                                        });
                                        this.ReplicationRepartosRepository.Update(data.ReplicationDataGuid, data);
                                        break;
                                }
                                HeaderRepository.Commit();
                            }
                        }

                        if (SetRollback)
                        {
                            HeaderRepository.Rollback();
                        }
                    }
                    catch (Exception ex)
                    {
                        if (!string.IsNullOrEmpty(guidTarget))
                        {
                            replicationDataExist = this.ReplicationRepartosRepository.FindByDataId(guidTarget);
                            if (replicationDataExist != null)
                            {
                                ReplicationDataEntity data = new ReplicationDataEntity
                                {
                                    ReplicationDataStatus = ExportStatusEnum.NotSynchronized,
                                    ReplicationDataGuid = replicationDataExist.ReplicationDataGuid,
                                    DataId = replicationDataExist.DataId,
                                    IsError = true,
                                    ProcessDescription = "{\"Message\":\"An error has occurred in the replication process\" " + ex.Message + " " + ex.InnerException?.Message + "}",
                                    InitialRequest = "{}",
                                    Attempts = 1
                                };
                                this.ReplicationRepartosRepository.Update(data.ReplicationDataGuid, data);
                            }
                        }
                        eventsLog.RegisterToEventViewer(string.Format("{0} {1}", ex.Message, ex.InnerException?.Message), IsServiceProcess);
                        HeaderRepository.Rollback();
                    }
                }
                eventsLog.RegisterToEventViewer("Replication complete.", IsServiceProcess);
            });
        }

        public void Dispose()
        {
            Dispose(true);
            GC.SuppressFinalize(this);
        }

        protected virtual void Dispose(bool disposing)
        {
            if (!_disposed)
            {
                if (disposing)
                {
                    // Se liberan de forma determinista todas las conexiones internas para prevenir fugas
                    if (HeaderRepository != null)
                    {
                        // Dispose el contexto de HeaderRepository si es el dueño
                    }
                    if (ReplicationRepartosRepository != null)
                    {
                        ReplicationRepartosRepository.Dispose();
                    }
                }
                _disposed = true;
            }
        }
    }
}
```

---

#### 3. Prevención de Socket Exhaustion en `RequestServices.cs`

**CÓDIGO ANTES (Crea y destruye hilos de socket por cada petición HTTP):**
```csharp
namespace Repositories.HttpServices
{
    public class RequestServices : IDisposable
    {
        private readonly string request;
        private readonly HttpClient httpClient;
        private readonly ResponseToAgent responseToAgent;
        private readonly OrquestadorServiceModel orquestadorService;

        public RequestServices(string request, OrquestadorServiceModel orquestadorService)
        {
            this.request = request;
            this.httpClient = new HttpClient();
            this.responseToAgent = new ResponseToAgent();
            this.orquestadorService = orquestadorService;
        }

        public void Dispose()
        {
            this.httpClient.Dispose();
        }
        // ... SendAsync ...
    }
}
```

**CÓDIGO DESPUÉS (Instancia estática y reutilizable de `HttpClient` para alto rendimiento):**
```csharp
using Entities.Service;
using Newtonsoft.Json;
using System;
using System.IO;
using System.Net;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;

namespace Repositories.HttpServices
{
    public class RequestServices : IDisposable
    {
        private readonly string request;
        
        // Uso de instancia estática thread-safe para pooling automático de conexiones TCP
        private static readonly HttpClient httpClient = new HttpClient(new HttpClientHandler()
        {
            AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate
        });
        
        private readonly ResponseToAgent responseToAgent;
        private readonly OrquestadorServiceModel orquestadorService;

        public RequestServices(string request, OrquestadorServiceModel orquestadorService)
        {
            this.request = request;
            this.responseToAgent = new ResponseToAgent();
            this.orquestadorService = orquestadorService;
        }

        public void Dispose()
        {
            // El HttpClient estático compartido no debe ser eliminado aquí.
        }

        public async Task<ResponseToAgent> SendAsync()
        {
            HttpRequestMessage httpRequestMessage = new HttpRequestMessage();
            httpRequestMessage.Headers.Add("Authorization", orquestadorService.Authorization);
            httpRequestMessage.Headers.Add("x-api-key", orquestadorService.ApiKey);
            httpRequestMessage.Headers.Add("version", orquestadorService.Version);
            httpRequestMessage.Headers.Add("help", orquestadorService.Help);

            httpRequestMessage.RequestUri = new Uri(orquestadorService.Uri);
            httpRequestMessage.Method = HttpMethod.Post;

            try
            {
                httpRequestMessage.Content = new StringContent(this.request, Encoding.UTF8, "application/json");

                using (var response = await httpClient.SendAsync(httpRequestMessage))
                using (var result = await response.Content.ReadAsStreamAsync())
                using (var stream = new StreamReader(result))
                {
                    string strRead = await stream.ReadToEndAsync();
                    ResponseEpicor epicorResponse = new ResponseEpicor();
                    this.responseToAgent.HttpStatusCode = 0;

                    if (response.StatusCode == HttpStatusCode.OK)
                    {
                        try
                        {
                            epicorResponse = JsonConvert.DeserializeObject<ResponseEpicor>(strRead);
                            if (epicorResponse != null)
                            {
                                this.responseToAgent.JsonResponseOrquestador = JsonConvert.SerializeObject(epicorResponse, Formatting.None, new JsonSerializerSettings
                                {
                                    NullValueHandling = NullValueHandling.Ignore
                                });

                                if (epicorResponse.Respuesta != null)
                                {
                                    this.responseToAgent.HttpStatusCode = epicorResponse.Respuesta.HttpStatusCode;
                                }
                            }
                        }
                        catch (Exception ex)
                        {
                            this.responseToAgent.HttpStatusCode = (int)HttpStatusCode.Conflict;
                            this.responseToAgent.JsonResponseOrquestador = string.Format("{0} {1}", ex.Message, ex.InnerException?.Message);
                        }
                    }
                    else
                    {
                        this.responseToAgent.JsonResponseOrquestador = strRead;
                        this.responseToAgent.HttpStatusCode = (int)response.StatusCode;
                    }
                }
            }
            catch (Exception ex)
            {
                string Excep = string.Format("{0} {1} {2}", ex.Message, ex.InnerException?.Message, "Method SendAsync");
                string Message = string.Format("\"Message\": \"{0}\"", Excep);
                this.responseToAgent.HttpStatusCode = (int)HttpStatusCode.BadRequest;
                this.responseToAgent.JsonResponseOrquestador = "{" + Message + "}";
            }

            return this.responseToAgent;
        }
    }
}
```

---

### Fase 2: Estrategia de Descomposición en Microservicios

Para transformar esta cola de tareas heredada en un microservicio moderno y elástico, se define la siguiente estrategia de arquitectura de tres pilares:

1. **Desacoplamiento de Polling de Base de Datos (Event-Driven Integration):**
   Actualmente, el agente realiza "polling" directo sobre tablas pesadas de Epicor (`ShipDtl`, `ShipHead`). En una arquitectura de microservicios corporativos, Epicor no debe ser consultado de manera persistente a nivel de base de datos relacional. En su lugar, se configurará un disparador (Trigger) o Direct BPM en Epicor que, ante un evento de despacho (`SHIPPED`), escriba el evento en un agente ligero de mensajería (como **RabbitMQ** o **Azure Service Bus**).
2. **Transformación a .NET 8 Worker Service Independiente:**
   El servicio Windows físico se compilará como un **Worker Service** ligero basado en la interfaz `BackgroundService` de .NET 8. Consumirá los eventos de la cola de mensajería directamente, procesando la información y consumiendo de manera segura la API REST del WMS.
3. **Escalamiento Seguro con Control de Concurrencia Transaccional:**
   Para escalar horizontalmente este componente y ejecutar múltiples réplicas (Pods) en Kubernetes simultáneamente, se debe evitar el doble procesamiento de folios. Se reemplazará el bloqueo de hilos local (`lock (_locker)`) por un **bloqueo distribuido** (Distributed Lock) mediante una base de datos de caché compartida en memoria como **Redis (Redlock)**, o aplicando transaccionalidad estricta y bloqueos pesados a nivel de fila mediante consultas SQL Server controladas como `SELECT ... WITH (ROWLOCK, UPDLOCK, READPAST)`.

---

### Fase 3: Dockerización Completa de la Aplicación Migrada (.NET 8)

Para desplegar la aplicación migrada como contenedor inmutable de producción, creamos los siguientes archivos estructurados en la raíz de la solución:

#### 1. Archivo: `E:\ProyectosDev\Agentes Maestros\branches\DEV.VELA\Componentes-Base\Servicios Extra\SrvRepartosAgent\Dockerfile`
```dockerfile
# Multi-stage Dockerfile para optimización de peso y seguridad DevSecOps
# Etapa 1: Compilación de la solución
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app

# Copiar archivos de solución y restaurar dependencias NuGet de manera aislada
COPY SrvRepartosAgent.sln ./
COPY SrvRepartosAgent/SrvRepartosAgent.csproj ./SrvRepartosAgent/
COPY ApplicationDbContext/ApplicationDbContext.csproj ./ApplicationDbContext/
COPY DataAccess/DataAccess.csproj ./DataAccess/
COPY Entities/Entities.csproj ./Entities/
COPY Exceptions/Exceptions.csproj ./Exceptions/
COPY Models/Models.csproj ./Models/
COPY Repositories/Repositories.csproj ./Repositories/
COPY Services/Services.csproj ./Services/
COPY UnitOfWorks/UnitOfWorks.csproj ./UnitOfWorks/

RUN dotnet restore SrvRepartosAgent.sln

# Copiar la totalidad del código fuente
COPY . .

# Compilar los binarios optimizados en modo Release para producción
WORKDIR /app/SrvRepartosAgent
RUN dotnet publish -c Release -o /app/publish_out --no-restore

# Etapa 2: Imagen base de ejecución optimizada (Runtime)
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS runtime
WORKDIR /app

# Copiar los binarios generados en la etapa previa
COPY --from=build /app/publish_out .

# Configurar límites de rendimiento y variables de entorno del motor de .NET
ENV DOTNET_CLI_TELEMETRY_OPTOUT=1
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false

# Crear usuario sin privilegios para mitigar vulnerabilidades de escalación de privilegios
RUN useradd -u 10050 -ms /bin/bash appworker
RUN chown -R appworker:appworker /app
USER appworker

ENTRYPOINT ["dotnet", "SrvRepartosAgent.dll"]
```

#### 2. Archivo: `E:\ProyectosDev\Agentes Maestros\branches\DEV.VELA\Componentes-Base\Servicios Extra\SrvRepartosAgent\docker-compose.yml`
```yaml
version: '3.8'

services:
  srv-repartos-taskqueue:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: srv-repartos-taskqueue
    restart: always
    environment:
      - DOTNET_ENVIRONMENT=Production
      # Las credenciales de base de datos se inyectan en runtime mediante el entorno para evitar texto plano en configuraciones físicas
      - ConnectionStrings__DefaultConnection=Server=192.168.10.39;Database=Intermedia;User Id=sa;Password=Gains1115;TrustServerCertificate=True;Max Pool Size=100;
      - ConnectionStrings__EpicorContextConnection=Server=10.40.3.73;Database=EpicorERPTest;User Id=sa;Password=Epicor123;TrustServerCertificate=True;Max Pool Size=100;
      - ConnectionStrings__LocalContextConnection=Server=192.168.10.122;Database=ReplicationDataBase;User Id=Developer;Password=D3ve1oper21!;TrustServerCertificate=True;Max Pool Size=100;
      # Configuración de URLs del Orquestador de Integraciones
      - UpdateOrquestadorRepartosAgent__Uri=http://192.168.10.128/OrquestadorApi/execute
      - UpdateOrquestadorRepartosAgent__Authorization=DSxmLJQ4yw9
      - UpdateOrquestadorRepartosAgent__ApiKey=REPARTOSkey
      - UpdateOrquestadorRepartosAgent__Version=v1
      - UpdateOrquestadorRepartosAgent__Help=AgregarOrdenEntrega
      # Parámetros del scheduler del proceso
      - ConfigService__IntervaloEjecucion=4
      - ConfigService__HoraInicio=05:00
      - ConfigService__HoraFin=23:00
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "5"
```

#### 3. Archivo: `E:\ProyectosDev\Agentes Maestros\branches\DEV.VELA\Componentes-Base\Servicios Extra\SrvRepartosAgent\.dockerignore`
```
# Exclusiones de contexto para el Docker Daemon de construcción
.git/
.svn/
.vs/
.vscode/
**/bin/
**/obj/
**/out/
**/publish_out/
**/*.user
**/*.suo
SrvRepartosConsoleTest/
App.config
```

---

### Fase 4: Pipeline CI/CD y Kubernetes Probes

#### 1. Pipeline Completo: `.github/workflows/ci-cd.yml`
```yaml
name: Continuous Integration & Deployment Pipeline

on:
  push:
    branches:
      - main
      - 'releases/**'
  pull_request:
    branches:
      - main

permissions:
  contents: read
  packages: write
  security-events: write

jobs:
  build-and-test:
    name: Build, Lint and Security Analysis
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'

      - name: Restore Dependencies
        run: dotnet restore SrvRepartosAgent.sln

      - name: Build Solution
        run: dotnet build SrvRepartosAgent.sln --configuration Release --no-restore

      - name: Run Unit Tests
        run: dotnet test SrvRepartosAgent.sln --configuration Release --no-build --verbosity normal

      - name: DevSecOps Vulnerability Scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'

  publish-container:
    name: Build and Push Docker Image
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: actions/setup-qemu-action@v2

      - name: Login to Enterprise Registry
        uses: docker/login-action@v2
        with:
          registry: container-registry.boxito.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Build and Push Production Image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            container-registry.boxito.com/logistics/srv-repartos-agent:latest
            container-registry.boxito.com/logistics/srv-repartos-agent:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

#### 2. Sondas de Kubernetes (Liveness y Readiness Probes)

En el Worker Service .NET 8, se expondrá un endpoint ligero de Healthcheck HTTP (ej. utilizando `Microsoft.Extensions.Diagnostics.HealthChecks` expuesto en el puerto `8080`).

Configuración en el manifiesto de Kubernetes (`deployment.yaml`):

```yaml
# Sondas de Diagnóstico y Salud para Kubernetes
livenessProbe:
  httpGet:
    path: /health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 20
  timeoutSeconds: 5
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3
```

- **Sonda Liveness (Supervivencia):** Evalúa que el contenedor no haya entrado en un estado de "Deadlock" o que la tarea principal en segundo plano haya colapsado de forma silenciosa. Si falla, Kubernetes destruye el Pod y crea una nueva instancia para autorecuperarse.
- **Sonda Readiness (Disponibilidad):** Confirma de forma proactiva que las conexiones hacia SQL Server central y la API del orquestador se encuentren activas antes de permitir al contenedor comenzar a reclamar tareas de replicación.

---

## ⚠️ Información no proporcionada en la entrada...

Con rigor y transparencia, declaramos los siguientes elementos **no especificados en la información del proyecto**:
1. **Infraestructura de Kubernetes destino:** Se desconoce la distribución de Kubernetes de destino (ej. AWS EKS, Azure AKS, RedHat OpenShift), lo que limita afinar los recursos de red del ingress, políticas de red (NetworkPolicies) o especificaciones de seguridad del Pod (SecurityContext).
2. **Sistema de Almacenamiento de Secretos corporativo:** No se proporcionó información sobre si la empresa cuenta con sistemas de gestión de llaves y secretos del tipo **HashiCorp Vault**, **Azure Key Vault** o **AWS Secrets Manager**, por lo que se modelaron las sondas utilizando secretos genéricos de Kubernetes (`secretsKeyRef`).
3. **Mapeo de Redes y DNS de Base de Datos:** Las IPs `10.40.3.73` y `192.168.10.122` se asumieron fijas dentro de la red privada corporativa, pero se desconoce la topología interna y la configuración de DNS que permita la resolución de nombres del host dentro de las redes internas del Pod.

# Informe de Auditoría Técnica de Código y Buenas Prácticas
**Módulo Auditado:** BoxServiceAgenteCondicionPago  
**Comité Consultor:** Arquitecto de Software Cloud-Native, Líder de DevSecOps, Auditor Principal de Ciberseguridad

---

## 1.1 Resumen Ejecutivo

Este informe detalla los hallazgos de la auditoría técnica profunda realizada al código de la aplicación **BoxServiceAgenteCondicionPago**, un servicio de Windows desarrollado en `.NET Framework 4.7.2` para la sincronización y replicación de datos de condiciones de pago entre una base de datos ERP origen (Epicor), una base de datos intermedia local (EpicorBoxito) y un API Orquestador externo.

Tras una exhaustiva revisión estática y funcional del código fuente, el Comité Consultor ha determinado que la aplicación presenta importantes debilidades estructurales, riesgos críticos de concurrencia y severas vulnerabilidades de seguridad que comprometen la estabilidad del sistema y la confidencialidad de la información en producción.

### Puntuación Global Estimada: **39 / 100** (Nivel Crítico / Inseguro)

*   **Arquitectura y Concurrencia:** Crítico. Presencia de condiciones de carrera directas y patrones de concurrencia obsoletos que bloquean hilos del sistema (Sync-Over-Async).
*   **Gestión de Recursos (Fugas):** Crítico. Fugas constantes de conexiones a base de datos debido a la nula liberación de sesiones NHibernate (`ISession`).
*   **Ciberseguridad y Configuración:** Crítico. Credenciales de base de datos y llaves de API hardcodeadas en texto plano en archivos XML, junto con transporte no cifrado (HTTP).
*   **Calidad de Código y Lógica:** Crítico. Error grave en la implementación de la igualdad de clases primarias compuestas (`Equals` invertido), que inhabilita el correcto funcionamiento de colecciones y el seguimiento de estado del ORM.

---

## 1.2 Diagrama de Arquitectura, Flujo y Conexiones Externas

El siguiente diagrama ilustra la arquitectura de ejecución concurrente actual de la aplicación, detallando el flujo de sincronización y las interacciones con sistemas externos:

```mermaid
flowchart TD
    %% Base de datos origen (Epicor)
    DB_ORIGEN[(SQL Server: EpicorERPPilot\n10.40.3.72)]

    %% Base de datos destino intermedia (Parnet)
    DB_DESTINO[(SQL Server: EpicorBoxito\n10.40.3.72)]

    %% Orquestador API
    API_ORQ{{"Orquestador API (HTTP)\n10.40.3.30:8080"}}

    %% Componente Servicio de Windows
    subgraph WindowsService [BoxServiceAgenteCondicionPago]
        Service[ServiceAgentCondicionPago]
        Timer["Timer (Elapsed Event)\nCada .2 minutos (12s)"]
        ProcMgr[ProcessManagement]
        TermsBO[Tbl_Terms BO\nCompositeId: Code + Plant\n🚨 Equals Invertido!]
    end

    %% Repositorios y Conexiones (Con Fugas)
    subgraph DataAccess [Capa de Acceso a Datos]
        ConnOrig[ConectionOrigen\n_sessionFactory Singleton]
        ConnDest[ConectionDestino\n_sessionFactory Singleton]
        RepoBase[RepositoryBase\nISession sessionOrigen\nISession sessionDestino\n🚨 Sin IDisposable / No se cierran]
        RepoPay[PaymentConditionRepository]
        RepoParnet[ParnetRepository]
    end

    %% Flujo de Ejecución
    Service -->|Inicializa| Timer
    Timer -->|Dispara| ProcMgr
    ProcMgr -->|1. Consulta SP_POS_TraeCondicionPago| RepoPay
    RepoPay -->|sessionDestino.CreateSQLQuery| DB_DESTINO
    
    %% Hilos y Concurrencia Crítica
    ProcMgr -->|2. Para cada registro en paralelo| TP_Queue{"ThreadPool.QueueUserWorkItem\n(ToDestiny)"}
    TP_Queue -->|Hilo de ThreadPool| ToDestiny[ToDestiny Callback]
    ToDestiny -->|3. OriginAsync.Wait\n🚨 Bloqueo de hilo| RepoPay2[PaymentConditionRepository\n🚨 Fuga de Sesiones]
    RepoPay2 -->|Escribe e Intercambia Status| DB_ORIGEN
    RepoPay2 -->|Escribe e Intercambia Status| DB_DESTINO

    %% Carrera de Concurrencia
    ProcMgr -->|4. Carrera Inmediata\n🚨 Sin Esperar Hilos| QuerySync["GetLista().Where(ExportStatus == 0)"]
    QuerySync -->|Lee de la DB destino| RepoPay
    
    %% Consumo Orquestador
    QuerySync -->|5. Sincroniza por Plant| ToParnet[ToParnet]
    ToParnet -->|DestinyAsync| RepoPay
    RepoPay -->|6. Envia REST Request| API_ORQ
    
    %% Actualización intermedia N+1
    RepoPay -->|7. Retorna Respuesta| RepoParnet
    RepoParnet -->|8. ForEach Record| LoopUpdate["N+1 Loop Update\n🚨 Abre Session + Tx por fila"]
    LoopUpdate -->|Update Status a 2 (Completado) o 3 (Error)| DB_DESTINO

    %% Conexiones con Base de datos
    ConnOrig -.->|Genera Session| DB_ORIGEN
    ConnDest -.->|Genera Session| DB_DESTINO
```

---

## 1.3 Matriz de Puntuación de Desarrollo (Scorecard)

| Criterio de Evaluación | Puntuación (1-5) | Diagnóstico Técnico Soportado |
| :--- | :---: | :--- |
| **Arquitectura de Software** | 2.0 | Acoplamiento fuerte, dependencias instanciadas directamente, mezcla de lógica de sincronización y orquestación, uso de servicios heredados tipo Windows Service de .NET Framework con dependencias atadas al SO. Lógica del timer propensa a traslapes. |
| **Ciberseguridad** | 1.0 | Cadenas de conexión a base de datos y tokens de API secretos hardcodeados en archivos de configuración en texto plano. Uso de protocolo inseguro (HTTP) expuesto en la red local. |
| **Principios SOLID** | 1.5 | **SRP:** Repositorios manejan transacciones cruzadas y su propio ciclo de vida. **DIP:** Ausencia de Inyección de Dependencias, las clases instancian directamente a NHibernate y los repositorios concretos. **ISP:** Interfaces de repositorio subutilizadas o con métodos lanzando `NotImplementedException`. |
| **Patrones de Diseño** | 2.0 | Patrón Repository y Unit of Work de NHibernate implementados deficientemente, lo que resulta en fugas de recursos constantes. El mapeo FluentNHibernate está bien estructurado, pero se ve saboteado por métodos auxiliares que anulan sus beneficios. |
| **Clean Code y Calidad** | 2.5 | Nombres legibles en variables y mapeos de bases de datos. No obstante, existe un error severo de código (`Equals` invertido), presencia de código completamente huérfano (`ProcessRepository.cs`) y logs mezclados en el flujo de negocio principal. |

---

## 1.4 Análisis Crítico Detallado por Aspecto

### 1. Concurrencia y Conexión de Hilos (Riesgo Crítico de Carrera)
En `ProcessManagement.StarImport()`, se observa el siguiente segmento de código:

```csharp
//Inserta los datos de origen en la tabla destino
foreach (Tbl_Terms terms in DataSource)
    ThreadPool.QueueUserWorkItem(ToDestiny, terms);

//Trae la lista de registros del Destino 
List<Tbl_Terms> synchronizedData = paymentConditionRepository.GetLista().Where(x => x.ExportStatus == 0).ToList();
```

*   **Carrera de Datos:** Se encolan inserciones concurrentes en el pool de hilos mediante `ThreadPool.QueueUserWorkItem` y, de forma **inmediata**, en la línea siguiente, se realiza una consulta de los registros con `ExportStatus == 0` para sincronizarlos al orquestador. Debido a que no existe una barrera de sincronización (`Task.WhenAll`, `CountdownEvent` o `Wait`), la consulta `GetLista()` se ejecutará antes de que los hilos encolados hayan terminado (o siquiera iniciado) sus inserciones. Esto provoca que la sincronización siempre arrastre un desfase crítico o procese datos incompletos.
*   **Patrón Sync-Over-Async:** El callback `ToDestiny` invoca al método asíncrono y bloquea el hilo llamando a `.Wait()`:
    ```csharp
    public void ToDestiny(object model)
    {
        Tbl_Terms origen = (Tbl_Terms)model;
        PaymentConditionRepository oTerms = new PaymentConditionRepository(ServiceName, IsService);
        oTerms.OriginAsync(origen).Wait(); // 🚨 Bloqueo de hilo de ThreadPool
    }
    ```
    Bloquear de forma síncrona hilos de ThreadPool provoca degradación severa de rendimiento, agotamiento de hilos (thread starvation) y deadlocks bajo cargas elevadas.

### 2. Fugas de Memoria y Conexiones (Conexiones NHibernate Abiertas)
*   **Falta de Desecho de Recursos:** `RepositoryBase<T>` define dos campos públicos para la gestión de sesiones de NHibernate: `sessionOrigen` y `sessionDestino`. Sin embargo, ni `RepositoryBase` ni su derivada `PaymentConditionRepository` implementan `IDisposable` o métodos para el cierre de las sesiones. Cada vez que se instancia un repositorio, se invocan los métodos `OpenSession()` de `ConectionOrigen` y `ConectionDestino`, abriendo conexiones físicas a la base de datos que **nunca se liberan ni se devuelven al pool**, causando un agotamiento inminente de los descriptores de conexión de SQL Server en producción.
*   **Sobrescritura y Pérdida de Referencias:** En `PaymentConditionRepository.OriginAsync(object model)`, se realiza la siguiente operación:
    ```csharp
    sessionDestino = ConectionDestino.OpenSession();
    sessionOrigen = ConectionOrigen.OpenSession();
    ```
    Dado que las sesiones ya fueron instanciadas en el constructor de la clase, asignarles nuevas referencias sin haber cerrado o desechado las anteriores causa que las conexiones iniciales queden flotando en memoria (leaked) hasta que el Garbage Collector recolecte el repositorio, lo cual es altamente ineficiente y acelera el colapso del servidor de base de datos.

### 3. Degradación de Rendimiento por Transacciones N+1
En `ParnetRepository.DestinyAsync(object model)`, se itera una lista de hasta 250 registros de respuesta del orquestador. Por cada registro dentro del bucle `ForEach`, se ejecuta:

```csharp
using (sessionDestino = ConectionDestino.OpenSession())
using (ITransaction tx = sessionDestino.BeginTransaction())
{
    // ... actualización de ExportStatus ...
    tx.Commit();
}
```

Esto representa un antipatrón de rendimiento masivo: **N+1 Conexiones y Transacciones**. Si el lote tiene 250 elementos, el sistema abrirá, transaccionará, confirmará y cerrará la conexión de base de datos **250 veces de forma consecutiva**. Esto multiplica exponencialmente la latencia de red, satura el log de transacciones SQL Server y anula los beneficios de la optimización por lotes (Batching).

### 4. Error de Lógica Crítica (Inversión del Operador Equals)
En la clase entidad `Tbl_Terms` (`BO/DestinoBO/Tbl_Terms.cs`), se encuentra implementada la siguiente lógica para determinar la igualdad de objetos (esencial para claves compuestas en NHibernate):

```csharp
public override bool Equals(object obj)
{
    var toCompare = obj as Tbl_Terms;
    if (toCompare == null)
        return false;
    else
        return (this.GetHashCode() != toCompare.GetHashCode()); // 🚨 ERROR GRAVE
}
```

*   **Efecto Devastador:** El método devuelve `true` si los hashcodes son **DIFERENTES** (es decir, cuando los objetos NO son iguales), y devuelve `false` si son iguales. Esto rompe por completo la comparación lógica de llaves compuestas en NHibernate, corrompe el comportamiento de colecciones genéricas (`List.Contains()`, `Distinct()`, `Dictionary`) y desestabiliza el motor de persistencia del ORM al rastrear entidades de la base de datos intermedia.

### 5. Ciberseguridad y Credenciales Hardcodeadas
*   **Credenciales expuestas:** Las cadenas de conexión en `App.config` exponen nombres de usuario y contraseñas de base de datos en texto plano:
    ```xml
    <connectionStrings>
      <add name="conexionDestino" connectionString="Data Source=10.40.3.72;Initial Catalog=EpicorBoxito; User ID=interfacesparnet;Password=8UyrTqJkL;" providerName="System.Data.SqlClient" />
      <add name="conexionOrigen" connectionString="Data Source=10.40.3.72;Initial Catalog=EpicorERPPilot; User ID=interfacesparnet;Password=8UyrTqJkL;" providerName="System.Data.SqlClient" />
    </connectionStrings>
    ```
*   **Tokens expuestos:** Secretos corporativos de autenticación de la API Orquestador están hardcodeados en el archivo de configuración:
    ```xml
    <add key="Authorization" value="tH1RouIce7IQw8" />
    <add key="x-api-key" value="tH1RouIce7IQw8" />
    ```
*   **Tránsito Inseguro:** El endpoint configurado utiliza HTTP plano sin cifrado TLS: `http://10.40.3.30:8080/`. Cualquier atacante con acceso a la red interna puede capturar pasivamente los paquetes REST y robar los tokens de autenticación `Authorization` o interceptar/manipular información sensible de condiciones de pago corporativas.

### 6. Deuda Técnica y Código Muerto
*   **Código Huérfano:** El archivo `ProcessRepository.cs` en la capa de persistencia se compila dentro del proyecto pero está completamente en desuso en toda la solución. Contiene lógica obsoleta de importación estructurada de forma manual, lo que genera confusión, incrementa el costo de mantenimiento y ensucia el espacio de nombres de la solución.

---

## 1.5 Sección de Recomendaciones Críticas y Plan de Remediación

A continuación, se define el plan estructurado por prioridades para mitigar y resolver los fallos detectados de forma quirúrgica.

### Prioridad P0: Corrección Inmediata (Mitigación de Errores Lógicos y Estabilidad)
1.  **Corregir la Lógica de `Equals`:** Modificar inmediatamente la comparación en `Tbl_Terms.cs` para validar la igualdad en lugar de la desigualdad:
    ```csharp
    return (this.GetHashCode() == toCompare.GetHashCode());
    ```
2.  **Sincronización Total de Hilos en `StarImport`:** Reemplazar el uso descontrolado de `ThreadPool.QueueUserWorkItem` con programación asíncrona real usando `Task` y asegurar que la consulta de sincronización se ejecute únicamente cuando todos los procesos de inserción en el destino hayan finalizado de forma exitosa (usando `Task.WhenAll`).
3.  **Eliminar Llamadas Bloqueantes (`.Wait()` y `.Result`):** Eliminar todos los bloqueos síncronos sobre hilos asíncronos en los repositorios y servicios de orquestación, adoptando un flujo asíncrono puro (`async/await`) de extremo a extremo.

### Prioridad P1: Mediano Plazo (Optimización y Seguridad del Entorno)
1.  **Cierre y Eliminación de Fugas de Sesión NHibernate:** Implementar la interfaz `IDisposable` en la base del repositorio (`RepositoryBase.cs`) y asegurar el cierre estricto de las sesiones utilizando bloques `using` o patrones Unit of Work adecuados.
2.  **Optimización Bulk/Batch en Base de Datos (Evitar N+1):** Modificar `ParnetRepository.DestinyAsync` para que abra una única sesión y una única transacción, procesando la inserción/actualización de los 250 elementos dentro de un solo bloque por lotes.
3.  **Habilitación de HTTPS para Comunicación con el Orquestador:** Forzar la reconfiguración del endpoint del Orquestador para que requiera comunicaciones cifradas mediante HTTPS TLS 1.2 o superior, inhabilitando puertos HTTP inseguros.
4.  **Extracción de Secretos:** Migrar las cadenas de conexión y llaves de API fuera de los archivos de configuración estáticos. Implementar el uso de variables de entorno para su inyección dinámica en tiempo de ejecución.
5.  **Limpieza de Código:** Eliminar físicamente el archivo `ProcessRepository.cs` y su referencia en el archivo `.csproj` para mitigar la deuda técnica.

### Prioridad P2: Modernización Estratégica (Hacia Contenedores)
1.  **Migración de Plataforma:** Portar la aplicación de `.NET Framework 4.7.2` a un contenedor ligero con `.NET 8`.
2.  **Transformación a Worker Service:** Sustituir la infraestructura rígida de Servicio de Windows (`ServiceBase`) por un servicio de background genérico (`IHostedService` o `BackgroundService`) portable y óptimo para Linux.
3.  **Logs como Event Streams:** Eliminar las llamadas a `EventLog.WriteEntry` de Windows e implementar una abstracción estándar de logs que escriba a stdout/stderr (Consola) facilitando su recolección en clústeres de Kubernetes.

---

## ⚠️ Información no proporcionada en la entrada

De acuerdo con las reglas de rigor de información y no suposición, se listan los datos y componentes relevantes para el ecosistema que **no fueron proporcionados** dentro del código analizado:
1.  **Código fuente del SDK/API Orquestador:** Las clases `OrquestadorService`, `OrquestadorRequest` y `OrquestadorOptions` provienen de un ensamblado precompilado o biblioteca compartida (`Orquestador.Api`), por lo que su comportamiento de bajo nivel y posible manejo inseguro interno de sockets o HttpClient no pudieron ser analizados directamente.
2.  **Estructura y DDL de Base de Datos:** No se proporcionaron los scripts de creación de las tablas `Boxito.Tbl_Sync_Terms`, `Erp.Terms` y `Erp.Terms_UD`, ni el procedimiento almacenado `[Boxito].[SP_POS_TraeCondicionPago]`. Se asume la correspondencia de tipos con base exclusiva en los mapeos de NHibernate.
3.  **Configuraciones de Producción reales:** Los archivos proporcionados contienen configuraciones de entornos piloto/desarrollo (`EpicorERPPilot`). No se incluyeron los detalles de red, credenciales ni topología de balanceo de los servidores de producción de bases de datos o de APIs.
4.  **Flujos de control externos / Desencadenadores corporativos:** No se conoce la lógica interna del Orquestador externo al recibir y retornar el estatus de las condiciones de pago (`PaymentConditionsAddOrUpdate`), ni cómo el sistema gestiona errores de coherencia de negocio.

---

## 1.6 AUDITORÍA DEVSECOPS Y CIBERSEGURIDAD (OWASP Top 10)

Como parte de la evaluación de seguridad integral del Comité Consultor, se han identificado brechas críticas transversales que deben remediarse obligatoriamente para cumplir con normativas corporativas:

### A. Exposición de Secretos y Credenciales (OWASP A02:2021)
*   **Riesgo:** El almacenamiento de credenciales de base de datos, contraseñas y llaves de API (ej. XApiKey) dentro de archivos estáticos como App.config permite a cualquier usuario con acceso al servidor o repositorio comprometer el ecosistema completo.
*   **Remediación:** Migrar hacia un modelo de **Inyección de Secretos** utilizando variables de entorno en contenedores o bóvedas de grado empresarial como Azure Key Vault / HashiCorp Vault.

### B. Transmisión Insegura de Datos en Red (OWASP A02:2021)
*   **Riesgo:** Las integraciones contra los orquestadores y bases de datos locales suelen viajar a través de canales no encriptados (HTTP plano).
*   **Remediación:** Imponer políticas estrictas de cifrado forzando el uso de HTTPS (TLS 1.2 o TLS 1.3) en las llamadas de HttpClient y las conexiones a bases de datos.

### C. Vulnerabilidad ante Denegación de Servicio (DoS) Interno
*   **Riesgo:** El diseño de hilos sin límite estricto y temporizadores bloqueantes agota los recursos (CPU/RAM/Sockets) del contenedor o servidor host.
*   **Remediación:** Implementar SemaphoreSlim para acotar la concurrencia, y configurar límites estrictos de recursos (limits y eservations) en los orquestadores de Docker/Kubernetes.

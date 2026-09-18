# Informe de Auditoría Técnica de Código y Buenas Prácticas
**Aplicación:** BoxServiceAgenteCustomerGroup (Grupo de Clientes)  
**Clasificación:** Confidencial / Uso Corporativo  
**Comité Consultor:** Arquitecto de Software Cloud-Native, Líder de DevSecOps, Auditor Principal de Ciberseguridad

---

## 1.1 Resumen Ejecutivo

### Puntuación Global Estimada: **35 / 100** (Nivel de Riesgo: CRÍTICO)

Tras realizar un análisis estático de código exhaustivo de la base del agente **BoxServiceAgenteCustomerGroup**, el comité consultor ha clasificado el estado actual del componente como de **Riesgo Crítico**. Aunque la lógica funcional de recolección de datos y sincronización básica está presente, el software contiene múltiples defectos graves en el manejo de concurrencia, gestión de recursos de bases de datos, seguridad de secretos y lógica booleana básica.

#### Principales Hallazgos Críticos:
1. **Defecto de Lógica Catastrófico en la Entidad Principal (`Customer.cs`)**: El método sobreescrito `Equals` contiene una inversión de lógica que provoca que dos objetos se consideren iguales si sus hashcodes son distintos, y diferentes si sus hashcodes son idénticos. Esto inutiliza cualquier validación de igualdad en colecciones (`HashSet`, `Dictionary`), cambios de estado en NHibernate o filtros LINQ.
2. **Fuga Crítica de Conexiones de Base de Datos (Connection Leak)**: Las sesiones de NHibernate (`ISession`) se abren de forma indefinida en cada iteración del temporizador (cada 4 minutos) y en la ejecución paralela por sucursales. No existen bloques `using` ni llamadas explícitas a `.Close()` o `.Dispose()`. Bajo condiciones de producción, esto provocará la saturación de los pools de conexiones tanto en la base de datos de Epicor ERP como en la base de datos local en cuestión de horas.
3. **Peligro de Concurrencia e Inseguridad de Hilos (Thread Safety Violations)**: Se comparte de manera simultánea una instancia única de `CustomerGroupRespository` y sus variables de instancia `sessionOrigen` y `sessionDestino` entre hilos del `ThreadPool` que procesan las sucursales concurrentemente. La clase `ISession` de NHibernate **no es segura para subprocesos (Thread-Safe)**. Su acceso concurrente corromperá el estado interno del ORM, provocando bloqueos, lecturas sucias e interrupciones aleatorias de ejecución.
4. **Patrones Anticuados y "Fire and Forget" Incontrolados**: La invocación de hilos a través de `ThreadPool.QueueUserWorkItem` sin esperar su finalización (`Task` no esperados) evita la captura de excepciones y puede silenciar fallos de sincronización graves, además de impedir una finalización ordenada (graceful shutdown) del servicio.
5. **Fuga de Secretos en Código Fuente**: El archivo de configuración `App.config` expone de forma directa contraseñas de producción para bases de datos SQL Server y API keys de acceso para el Orquestador API.

---

## 1.2 Diagrama de Arquitectura, Flujo y Conexiones Externas

El siguiente diagrama ilustra el flujo de ejecución actual del agente, exponiendo los puntos críticos de comunicación de datos y las vulnerabilidades identificadas de fuga de conexiones e inseguridad de hilos.

```mermaid
flowchart TD
    %% Componentes Principales
    subgraph WindowsService [Windows Service - .NET Framework 4.7.2]
        Service[ServiceAgenteCustomerGroup]
        Timer[System.Timers.Timer]
        Procesos[ProcesosAgente]
    end

    subgraph LogicLayer [Capa de Negocio y Coordinación]
        Manager[CustomerGroupManagements]
    end

    subgraph RepositoryLayer [Capa de Repositorio]
        Repo[CustomerGroupRespository]
        BaseRepo[RespositoryBase &lt;Customer&gt;]
        ParnetRepo[ParnetRepository]
        ConfigRepo[ConfigurationRepository]
    end

    subgraph DataAccessLayer [ORM & Comunicaciones]
        NH_Source[NHibernate: EpicorErpTest]
        NH_Dest[NHibernate: EpicorBoxito]
    end

    subgraph ExternalBackends [Sistemas Externos]
        SourceDB[(Base Origen: EpicorERPPilot)]
        DestDB[(Base Destino: EpicorBoxito)]
        Orchestrator[Orquestador API Service]
    end

    %% Flujo de ejecución
    Service -->|1. Inicia| Timer
    Timer -->|2. Dispara cada X minutos| Procesos
    Procesos -->|3. Instancia y llama a ProcesoAsync| Manager

    %% Flujo de datos origen (GetBySP)
    Manager -->|4. GetBySP SP_POS_Trae_CustomerGroups| Repo
    Repo -->|Hereda| BaseRepo
    BaseRepo -->|SQL Query| NH_Dest
    NH_Dest -->|Conexión sin liberar| DestDB

    %% Sincronización intermedia (ToDestiny)
    Manager -->|5. Recorre origen: ToDestiny| Repo
    Repo -->|Inserta registro intermedio| BaseRepo
    BaseRepo -->|Transacción dual secuencial| NH_Source
    BaseRepo -->|Transacción dual secuencial| NH_Dest
    NH_Source -->|Actualiza ExportStatus en origen| SourceDB
    NH_Dest -->|Guarda registro intermedio| DestDB

    %% Envío concurrente a sucursales (ThreadPool)
    Manager -->|6. Obtiene pendientes ExportStatus=0| Repo
    Manager -->|7. Agrupa por Sucursal (Plant)| Manager
    Manager -->|8. Despacha concurrente| ThreadPool[ThreadPool.QueueUserWorkItem]
    
    %% Inseguridad de hilos (Riesgo Crítico)
    ThreadPool -.->|9. Concurrencia sin control sobre la misma Session| Repo
    Repo -->|10. OrigenAsync| ConfigRepo
    ConfigRepo -->|11. Serializa JSON y envía HTTP| Orchestrator

    %% Actualización posterior
    Repo -->|12. Llama a Execute| ParnetRepo
    ParnetRepo -->|13. Destino: Actualiza estado intermedio| NH_Dest
    NH_Dest -->|Conexión filtrada por hilo| DestDB

    %% Estilos de alerta
    classDef criticalRisk fill:#ffcccc,stroke:#ff3333,stroke-width:2px;
    classDef leakRisk fill:#ffe5cc,stroke:#ff8000,stroke-width:2px;
    class ThreadPool,ParnetRepo criticalRisk;
    class NH_Source,NH_Dest leakRisk;
```

---

## 1.3 Matriz de Puntuación de Desarrollo (Scorecard)

A continuación, se detalla la evaluación cuantitativa de la aplicación basada en las mejores prácticas de ingeniería de software corporativo (Puntuación de 1 a 5, donde 1 es Deficiente y 5 es Sobresaliente).

| Dimensión | Puntuación | Justificación del Comité |
| :--- | :---: | :--- |
| **Arquitectura** | **2 / 5** | Diseño acoplado a un servicio de Windows monolítico clásico. Ausencia total de Inyección de Dependencias (DI) o de inversión de control (IoC). La lógica de orquestación, concurrencia y persistencia se encuentra mezclada en el repositorio y la capa de gestión, violando la separación de responsabilidades. |
| **Seguridad** | **1 / 5** | Almacenamiento de secretos (contraseñas y llaves de API) en texto plano dentro de archivos de configuración (`App.config`). Ausencia de rotación de credenciales o cifrado en reposo. Uso de cuentas de base de datos con privilegios amplios (`interfacesparnet`). |
| **SOLID** | **2 / 5** | **SRP (Single Responsibility)**: El repositorio maneja lógica de negocio, control de hilos y acceso a datos simultáneamente. <br>**OCP (Open/Closed)**: Incapacidad de extender el comportamiento de transporte sin reescribir código. <br>**DIP (Dependency Inversion)**: Se depende directamente de implementaciones concretas y estáticas (`EpicorErpTest`, `EpicorBoxito`, `ConfigurationRepository`). |
| **Patrones de Diseño** | **2 / 5** | Abuso de concurrencia de bajo nivel (`ThreadPool.QueueUserWorkItem`) y "Fire and Forget" en lugar del patrón *Producer-Consumer* o la biblioteca de flujos TPL (`Task.WhenAll`). Uso incorrecto y no atómico del patrón *Double Transaction Commit* en bases de datos separadas. |
| **Clean Code** | **2 / 5** | Presencia de código duplicado no utilizado (`CustumerGroupRespository.cs` e `IRepository.cs` redundantes). Errores tipográficos severos en nombres de clases y archivos ("Custumer", "Respository"). Bug gravísimo en la sobreescritura de `Equals` y posible excepción de puntero nulo en `GetHashCode`. |

---

## 1.4 Análisis Crítico Detallado por Aspecto

### 1.4.1 Defecto Catastrófico en la Lógica de Igualdad (`Customer.cs`)
En el archivo `Customer.cs`, la sobreescritura de los métodos `Equals` y `GetHashCode` presenta fallos de diseño inaceptables:
* **Inversión lógica en `Equals`**:
  ```csharp
  public override bool Equals(object obj)
  {
      var toCompare = obj as Customer;
      if (toCompare == null)
          return false;
      else
          return (this.GetHashCode() != toCompare.GetHashCode());
  }
  ```
  La instrucción `this.GetHashCode() != toCompare.GetHashCode()` causará que dos instancias de clientes se marquen como **iguales** cuando tengan códigos de hash **diferentes**, y se consideren **distintos** cuando tengan el **mismo** hash. Esto rompe cualquier lógica de de-duplicación, actualización de caché, búsquedas en listas indexadas o seguimiento de cambios por NHibernate.
* **Vulnerabilidad de Excepción por Referencia Nula (`NullReferenceException`) en `GetHashCode`**:
  ```csharp
  public override int GetHashCode()
  {
      int hashCode = 0;
      hashCode = hashCode ^ Company.GetHashCode() ^ Plant.GetHashCode() ^ Code.GetHashCode();
      return hashCode;
  }
  ```
  Si cualquiera de las propiedades virtuales `Company`, `Plant` o `Code` es nula (escenario común antes de poblar la entidad o durante la creación de proxies por NHibernate), el método lanzará un error fatal que detendrá el proceso de sincronización.

### 1.4.2 Fuga Masiva de Conexiones de Base de Datos y Falta de Liberación de Recursos
* **Falta del Patrón Disposable en Sesiones**: Las conexiones en `CustomerGroupRespository` y `ParnetRepository` se abren indefinidamente mediante `EpicorErpTest.OpenSession()` y `EpicorBoxito.OpenSession()`. La falta de un método `.Dispose()` o un bloque `using` evita que las sesiones se cierren. La conexión física con SQL Server permanece abierta e inutilizable en el pool de conexiones hasta el desbordamiento.
* **Fuga de Conexiones en `ParnetRepository.Destino`**:
  Cada llamada al método `Destino` crea una sesión fresca pero nunca la cierra:
  ```csharp
  sessionDestino = EpicorBoxito.OpenSession(); // Se abre una sesión por cada registro a procesar
  ITransaction txDestino = sessionDestino.BeginTransaction();
  // ... (Falta un using o sessionDestino.Dispose() en el finally)
  ```
  Si la sincronización procesa 200 registros de clientes, se crearán 200 sesiones NHibernate independientes de forma secuencial sin cerrar, agotando instantáneamente el pool estándar de conexiones (que típicamente soporta un máximo de 100).

### 1.4.3 Inseguridad de Hilos (Thread Safety) y Riesgo de Corrupción de Datos
* **Acceso Multihilo No Sincronizado a NHibernate `ISession`**:
  En `CustomerGroupManagements.ProcesoAsync`, se utiliza `ThreadPool.QueueUserWorkItem(ToOrigin, oModel)` para procesar múltiples sucursales en hilos separados de forma simultánea. Sin embargo, todas las tareas apuntan a la misma instancia compartida del repositorio (`custumerGroupRespository`), el cual contiene las variables de instancia `sessionOrigen` y `sessionDestino`.
  **La documentación de NHibernate establece explícitamente que `ISession` no es seguro para hilos**. El uso compartido de una sesión de NHibernate a través de múltiples hilos concurrently causará un comportamiento errático, aserciones internas fallidas del ORM, corrupción de transacciones y estados de sesión inconsistentes.

### 1.4.4 Lógica de Transacción Dual No Atómica
En `RespositoryBase<T>.Insert`, el código intenta realizar confirmaciones en dos sistemas de base de datos diferentes:
```csharp
ITransaction txDestino = sessionDestino.BeginTransaction();
ITransaction txOrigen = sessionOrigen.BeginTransaction();
// ...
txDestino.Commit();
txOrigen.Commit();
```
Esto representa una transacción distribuida simulada (Chained Transactions) que carece de soporte de confirmación en dos fases (2PC) o protocolos de consenso de consistencia eventual (como Sagas o Compensación). Si `txDestino.Commit()` se completa con éxito, pero ocurre un fallo de red o error de servidor justo antes de `txOrigen.Commit()`, las bases de datos quedarán desincronizadas (el destino tendrá los datos, pero el origen seguirá mostrando el estado de pendiente/antiguo).

### 1.4.5 Modificación Peligrosa del Comportamiento del Hilo
En `CustomerGroupRespository.Execute`, se realiza una reconfiguración injustificada de las propiedades de ejecución de hilos del ThreadPool:
```csharp
Thread.CurrentThread.IsBackground = false;
```
Por definición, los hilos de `ThreadPool` administrados por el runtime de .NET deben operar como hilos de fondo (`IsBackground = true`). Cambiar esta propiedad a `false` de forma manual forzará al sistema operativo a mantener el proceso completo del agente con vida, impidiendo el apagado normal del Windows Service al presionar detener, forzando a los operadores de infraestructura a finalizar el proceso mediante `taskkill` o el Administrador de Tareas.

### 1.4.6 Código Muerto, Redundancias y Desorden de Estructura (Deuda Técnica)
* **Archivo Duplicado Inútil**: `CustumerGroupRespository.cs` es una copia parcial mal escrita con un error tipográfico ("Custumer") que no tiene uso activo pero consume espacio de análisis y compilación.
* **Mal Mapeo Semántico**: `DataSourceMap.cs` mapea el modelo `DataSource` a la tabla `dbo.PriceLst`. Esto denota una copia ciega del Agente de Listas de Precios. En un sistema de producción real, esto confunde a los desarrolladores y puede provocar alteraciones de datos imprevistas si otro componente interactúa con la misma tabla bajo asunciones erróneas.
* **Control Inestable de Logs**: El sistema implementa un registro de logs personalizado en `EventsLog.cs` que abre y escribe archivos de texto sin bloques de exclusión mutua (`lock`). Al dispararse múltiples hilos concurrentes para escribir logs, el sistema colisionará provocando excepciones `IOException` por bloqueo de archivos.

---

## 1.5 Sección de Recomendaciones Críticas y Plan de Remediación

Para elevar la salud técnica del agente y asegurar su viabilidad en producción, se definen tres niveles de prioridad de remediación:

### Prioridad P0: Acciones Inmediatas (Remediación de Emergencia)
1. **Corregir la Lógica de `Equals` en `Customer.cs`**:
   Implementar una comparación basada en el valor real del identificador compuesto de clave primaria y eliminar la susceptibilidad a valores nulos.
2. **Refactorizar el Manejo de Sesiones NHibernate**:
   Asegurar que todas las llamadas de base de datos abran la sesión de forma local, realicen el trabajo y la liberen de inmediato a través de sentencias `using`.
3. **Eliminar el Acceso Compartido de Sesiones en Multihilo**:
   Instanciar repositorios y sesiones independientes por cada hilo que procese una sucursal en lugar de compartir una única instancia global de sesión.
4. **Eliminar `Thread.CurrentThread.IsBackground = false`**:
   Permitir que el motor de .NET administre los hilos del pool correctamente para posibilitar paradas controladas del servicio.

### Prioridad P1: Mejoras a Corto Plazo (Refactorización de Código)
1. **Migración Completa a la Biblioteca de Tareas en Paralelo (TPL)**:
   Reemplazar `ThreadPool.QueueUserWorkItem` por `Task.Run()` y utilizar `Task.WhenAll` para orquestar la concurrencia de procesamiento por sucursal con captura estructurada de excepciones.
2. **Remoción de Credenciales Hardcodeadas**:
   Mover las cadenas de conexión y llaves de API a variables de entorno del sistema o a un proveedor de configuración protegido (por ejemplo, `Secrets Manager` o `Vault`).
3. **Limpieza de Código Muerto**:
   Eliminar definitivamente el archivo redundante `CustumerGroupRespository.cs` y reestructurar los mapeos de base de datos a nombres consistentes con la entidad de Customer Group.

### Prioridad P2: Evolución de Arquitectura
1. **Migración a .NET 8 (Worker Service)**:
   Abandonar el modelo obsoleto de Windows Services clásico (`ServiceBase`) y transformar la solución en un Worker Service multiplataforma moderno compatible con contenedores Docker Linux nativos.
2. **Inyección de Dependencias (DI)**:
   Utilizar el contenedor oficial `Microsoft.Extensions.DependencyInjection` para gestionar de forma limpia el ciclo de vida de los repositorios, servicios de persistencia y configuraciones.

---

## ⚠️ Información no proporcionada en la entrada
Con el objetivo de cumplir estrictamente con la directiva de no suposición de datos, el comité consultor declara que los siguientes datos de contexto técnico **NO** se proporcionaron en el código fuente analizado:
1. **Estructura Detallada de las Tablas de Base de Datos**: No se cuenta con el código de creación DDL de las tablas `Boxito.Tbl_Sync_CustomerGroup` ni `dbo.PriceLst`.
2. **Procedimiento Almacenado `SP_POS_Trae_CustomerGroups`**: No se proporcionó la definición interna ni la lógica de extracción de datos del procedimiento almacenado alojado en el SQL Server origen.
3. **Lógica Interna del Orquestador API**: No se incluye el código fuente del framework de comunicación `Orquestador.Api.Services.OrquestadorService`, por lo que se asume que su llamada `ExecuteAsync` es correcta pero opera como una "caja negra" con límite de timeout de 20 segundos establecido en código.
4. **Infraestructura de Red**: No se dispone de detalles sobre proxies, certificados SSL o mecanismos de autenticación de red entre el servidor de base de datos local `10.40.3.72` y el orquestador `10.40.3.30`.

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

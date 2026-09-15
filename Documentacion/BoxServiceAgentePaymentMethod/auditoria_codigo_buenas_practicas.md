# Informe de Auditoría Técnica de Código y Buenas Prácticas
**Aplicación:** BoxServiceAgentePaymentMethod
**Comité Consultor:** Arquitecto de Software Cloud-Native, Líder de DevSecOps, Auditor Principal de Ciberseguridad

## 1.1 RESUMEN EJECUTIVO
**Puntuación Global Estimada:** 35 / 100 (Crítico)

El aplicativo es un Servicio de Windows desarrollado en .NET Framework, orquestado mediante `System.Timers.Timer` y diseñado para sincronizar formas de pago con una base de datos SQL Server y una API orquestadora externa. 

La arquitectura actual presenta riesgos críticos de seguridad y estabilidad. Las vulnerabilidades de inyección SQL (SQLi) evidentes en el repositorio, acopladas a antipatrones de concurrencia como *async-over-sync* (`.Wait()` y `.Result`), comprometen gravemente tanto la confidencialidad de la base de datos como la resiliencia del hilo de ejecución, propenso a *deadlocks*. Adicionalmente, la alta dependencia de servicios exclusivos de Windows (EventLog y ServiceBase) lo convierte en un sistema rígido. Se requiere una intervención inmediata antes de escalar o migrar.

## 1.2 DIAGRAMA DE ARQUITECTURA, FLUJO Y CONEXIONES EXTERNAS

```mermaid
flowchart TD
    subgraph BoxServiceAgentePaymentMethod [Windows Service (.NET)]
        Program[Program.cs] -->|Inicializa| Svc[ServiceAgentePaymentMethod.cs]
        Svc -->|Timer Interval| Mgt[PaymentManagement.cs]
        Mgt -->|Task.Run| Flow1(Obtiene Formas de Pago DB)
        Mgt -->|Task.Run| Flow2(Sincroniza Datos a Orquestador)
        
        Flow1 --> Repo1[PaymentRepository.cs]
        Flow2 --> Repo1
        
        Flow2 -->|ThreadPool.QueueUserWorkItem| Proc[IProcess - Run]
        Proc --> RepoSync[TblSyncRepository.cs]
    end

    subgraph Data Layer [SQL Server DB]
        DB[(Base de Datos)]
        SP1[SP_POST_TraeFormaPago]
        SP2[SP_POST_TraeFormaPagoU]
        Tbl[Tbl_Sync_FormaPago]
    end

    subgraph External
        OrquestadorAPI[API Orquestador Parnet]
    end
    
    subgraph OS
        WinEventLog[Windows Event Log]
    end

    Repo1 -->|NHibernate / SP| SP1
    Repo1 -->|NHibernate / SP| SP2
    RepoSync -->|Raw SQL UPDATE| Tbl
    Repo1 -->|HTTP/REST ConfigRepository| OrquestadorAPI
    
    Svc -.-> WinEventLog
    Mgt -.-> WinEventLog
    Repo1 -.-> WinEventLog
```

## 1.3 MATRIZ DE PUNTUACIÓN DE DESARROLLO (SCORECARD 1-5)

| Dimensión | Puntuación | Justificación |
| :--- | :---: | :--- |
| **Arquitectura** | 2 | Fuerte acoplamiento a SO subyacente (Servicio Windows). Inyección de dependencias inexistente. |
| **Seguridad** | 1 | **CRÍTICO:** Vulnerabilidad de Inyección SQL detectada en `TblSyncRepository`. Manejo inseguro de secretos en `app.config`. |
| **Principios SOLID** | 2 | Violación flagrante de SRP: `PaymentManagement` orquesta hilos, ejecuta lógicas de negocio, e interactúa con datos. |
| **Patrones de Diseño** | 2 | Uso de patrón Repository pero mezclado con consultas crudas y llamadas directas a SPs (rompiendo la abstracción del ORM NHibernate). |
| **Clean Code** | 2 | Antipatrones asíncronos (`.Wait()`, `.Result`), código comentado residual, manejo de excepciones genéricas tragadas y control de flujo errático. |

## 1.4 ANÁLISIS CRÍTICO DETALLADO POR ASPECTO

1. **Seguridad (Ciberseguridad):**
   * **Inyección SQL Directa:** En `TblSyncRepository.cs` la línea `string query = "UPDATE [SAS1005].[dbo].[Tbl_Sync_FormaPago] set ExportEstatus = "+ staus + " WHERE Sucursal = "+ payment.Sucursal + " AND Clave = '"+payment.Clave+"' ";` es extremadamente vulnerable. 
   * **Hardcoding de Secretos:** La API de orquestación utiliza `x-api-key` y `Authorization` provenientes de `ConfigurationManager`. Estos secretos a menudo no están encriptados y representan un riesgo si el código fuente es comprometido.
2. **Estabilidad y Concurrencia (Arquitectura):**
   * **Deadlocks / Async-Over-Sync:** En `PaymentRepository.cs` se efectúan llamadas como `status.Wait()` y `status.Result` para consumir un método asíncrono `SendAsyncOrquestador`. Esto bloquea el hilo subyacente y puede causar agotamiento en el *Thread Pool*.
   * **Orquestación de Hilos Manual:** El uso de `ThreadPool.QueueUserWorkItem(Execute, proc)` en vez del uso idiomático y gestionado de `Task` de TPL (.NET Task Parallel Library) reduce la trazabilidad de excepciones; si un proceso falla, se traga el error en el hilo y genera un log aislado.
3. **Mantenibilidad (DevOps / Clean Code):**
   * **Locking Log:** El acoplamiento a `EventLog.WriteEntry` esparcido por todas las capas impide la recolección centralizada de logs para monitoreo o alertas eficientes.

## 1.5 SECCIÓN DE RECOMENDACIONES CRÍTICAS Y PLAN DE REMEDIACIÓN

*   **[P0 - CRÍTICA] Remediación de SQL Injection:** Parametrizar la consulta cruda en `TblSyncRepository` usando NHibernate nativo o `CreateSQLQuery().SetParameter()`.
*   **[P1 - ALTA] Erradicar Async-Over-Sync:** Cambiar todas las firmas de la interfaz `IProcess` y los repositorios a retornar `Task` y usar `await` a todo lo largo de la pila de llamadas (desde el Timer hacia abajo).
*   **[P2 - MEDIA] Inyección de Dependencias (DI) e ILogger:** Reestructurar la instanciación acoplada (los repositorios se instancian con `new PaymentRepository()` dentro de un loop) para usar un contenedor de IoC y cambiar `EventLog` por el estándar genérico `ILogger`.
*   **[P3 - MEDIA] Configuración Resiliente:** Encriptar `app.config` o extraer los secretos a Variables de Entorno, preparando el camino hacia 12-Factor App.
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

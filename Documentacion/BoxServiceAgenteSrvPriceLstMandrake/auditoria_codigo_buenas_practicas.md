# Informe de Auditoría Técnica de Código: BoxServiceAgenteSrvPriceLstMandrake

## 1.1 RESUMEN EJECUTIVO
**Puntuación Global Estimada:** 42/100 (Crítico - Riesgo de Estabilidad y Rendimiento)

El componente `BoxServiceAgenteSrvPriceLstMandrake` es un Servicio de Windows desarrollado en .NET Framework 4.7.2. Su objetivo es extraer precios de una tabla origen, calcular impuestos (IVA) iterando por sucursal, y enviar dichos datos a un orquestador mediante HTTP. La arquitectura actual presenta **fallas críticas de concurrencia**, **fugas masivas de conexiones a base de datos** y múltiples **anti-patrones de diseño**, destacando el uso de `Thread.Sleep` para coordinar hilos y cálculos erróneos en los intervalos del servicio. Requiere refactorización urgente antes de cualquier modernización hacia contenedores.

## 1.2 DIAGRAMA DE ARQUITECTURA, FLUJO Y CONEXIONES EXTERNAS

```mermaid
flowchart TD
    subgraph Windows Service
        Timer[Timer 4.3s - Comentado como 1 Hora]
        Entry[Service.cs - ProcesosAgente]
        Entry -->|Llama| Mgt[PriceLstManagement]
        
        subgraph Hilos (Anti-patrón ThreadPool)
            Mgt -.-> |QueueUserWorkItem| T1(ProcesoEnvio - Chunk 1)
            Mgt -.-> |QueueUserWorkItem| T2(ProcesoEnvio - Chunk 2)
            Mgt -.-> |QueueUserWorkItem| Tn(ProcesoEnvio - Chunk N)
        end
    end

    subgraph Data Access (NHibernate & DbContext)
        DB_Main[(EpicorBoxito DB\nSQL Server 10.40.3.72)]
        DB_Branch[(Sucursal DB\nSASConexion)]
        
        Mgt -->|SP_POS_TraePriceToMandrakote| DB_Main
        Mgt -->|N+1 Conexiones sin cierre| DB_Branch
        T1 -->|Update ExportStatus| DB_Main
    end

    subgraph External Services
        Orq[API Orquestador\n10.40.3.30]
        T1 -->|HTTP POST /AddPriceLst| Orq
    end
```

## 1.3 MATRIZ DE PUNTUACIÓN DE DESARROLLO (SCORECARD 1-5)

| Categoría | Puntuación (1-5) | Observaciones |
| :--- | :---: | :--- |
| **Arquitectura** | 2 | Sistema frágil. Mezcla repositorios genéricos con DbContext y FluentNHibernate. Dependencia estática a Windows. |
| **Seguridad** | 2 | Credenciales en texto claro en `App.config` (`Password=FdSaTrQZ`, `Authorization=SN...`). Logs exponen errores al visor de eventos. |
| **SOLID** | 1 | Violación de Responsabilidad Única (`PriceLstManagement` gestiona Hilos, Lógica, Accesos a DB y Logging). |
| **Patrones** | 2 | Implementación deficiente de UnitOfWork; abre transacción DB, ejecuta HTTP bloqueante y luego hace commit. |
| **Clean Code** | 1 | Bloques masivos de código viejo comentado, hardcoding severo (`Thread.Sleep(16000)`), y errores aritméticos simples. |

## 1.4 ANÁLISIS CRÍTICO DETALLADO POR ASPECTO

1. **Defecto Crítico de Temporizador (Timer):**
   En `Service.cs`, se define `timer.Interval = 0.5 * 87 * 100; // 1 Hora`. Esto resulta matemáticamente en **4,350 milisegundos (4.3 segundos)**, NO en 1 hora (que sería 3,600,000 ms). Esto causa que el servicio se dispare descontroladamente saturando la base de datos y la red.
2. **Gestión de Concurrencia Kamikaze (Anti-patrón Threading):**
   En `PriceLstManagement.cs`, se mezclan Tasks y ThreadPool: `Task.Run(() => { ... ThreadPool.QueueUserWorkItem ... })` sin seguimiento. Para "esperar" a que terminen, usa un mágico e inestable `Thread.Sleep(16000);`. Si el proceso dura 17 segundos, el hilo principal avanza creando condiciones de carrera y corrompiendo memoria.
3. **Fugas de Conexión y Rendimiento (N+1 Sessions):**
   En el método `CalculaIVA`, se abre una conexión por cada sucursal distinta usando `SASConexionDB.OpenSession(company, sucursal)`. Esta sesión nunca se cierra explícitamente ni utiliza un bloque `using`, garantizando un desbordamiento del Pool de Conexiones.
4. **Acoplamiento de Entorno y 12-Factor (Logs y Configuración):**
   Total dependencia a `EventLog.WriteEntry` y al archivo estático `App.config`. Un contenedor en producción no tiene un Visor de Eventos de Windows útil y requiere que los logs salgan por `stdout/stderr`.
5. **Transaccionalidad Distribuida Fallida:**
   En `EpicorBoxitoUOW.cs` abre transacción en BD, luego realiza un llamado de red externo HTTP. Si la red se degrada, la conexión a la BD permanece bloqueando tablas indefinidamente hasta hacer timeout.

## 1.5 SECCIÓN DE RECOMENDACIONES CRÍTICAS Y PLAN DE REMEDIACIÓN

*   **P0 (Crítico e Inmediato):** 
    - Corregir de inmediato el cálculo de `timer.Interval` (utilizar `TimeSpan.FromHours(1).TotalMilliseconds`).
    - Eliminar `Thread.Sleep(16000)` y reemplazar `ThreadPool` por asincronía basada en TPL (`Task.WhenAll()`).
*   **P1 (Seguridad y Fugas de Memoria):** 
    - Refactorizar el cálculo de IVA: Cachear el IVA por sucursal al iniciar el ciclo en lugar de abrir la base de datos por cada grupo. Implementar siempre `using` en las transacciones/sesiones de NHibernate.
    - Asegurar que la transacción de BD (`BeginTransaction`) no envuelva llamadas HTTP.
*   **P2 (Deuda Técnica y Escalabilidad):** 
    - Desacoplar las responsabilidades: Mover la lógica de cálculo a servicios dedicados (`ITaxCalculator`) y remover variables duras (`http://10.40.3.30/`).
    - Migrar de .NET Framework a un entorno moderno como .NET Core 8/Worker Services para futura contenerización.

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

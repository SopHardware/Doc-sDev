# Informe de Auditoría Técnica de Código y Buenas Prácticas
**Aplicación:** BoxServiceAgenteConversionPart  
**Fecha:** Octubre 2023 (o actual)  
**Comité Evaluador:** Arquitecto Cloud-Native, Líder DevSecOps, Auditor Principal de Ciberseguridad

---

## 1.1 RESUMEN EJECUTIVO

El presente informe detalla los hallazgos de la auditoría técnica realizada sobre el código fuente del componente `BoxServiceAgenteConversionPart`. Se trata de un servicio de Windows desarrollado en `.NET Framework 4.7.2` responsable de sincronizar y procesar información de "Conversión de Partes" entre bases de datos Epicor (Origen y Destino) e integrarse con un API REST (Orquestador).

**Puntuación Global Estimada: 42 / 100**

El sistema actual cumple su función de sincronización básica, sin embargo, adolece de deudas técnicas severas, acoplamiento profundo al sistema operativo Windows, violaciones críticas a principios SOLID y exposición de credenciales. La dependencia de transacciones distribuidas simuladas manualmente (`txOrigen` y `txDestino`) y el uso de `ThreadPool.QueueUserWorkItem` sin control de concurrencia adecuado representan un riesgo inminente de corrupción de datos y fuga de memoria.

---

## 1.2 DIAGRAMA DE ARQUITECTURA, FLUJO Y CONEXIONES EXTERNAS

```mermaid
flowchart TD
    subgraph "Windows Server (On-Premise / IaaS)"
        WS[Windows Service\nBoxServiceAgenteConversionPart]
        Timer[System.Timers.Timer\nInterval: 4 min]
        CL[clManagement\nOrquestador Lógico]
        TP[ThreadPool & Task.WaitAll\nProcesamiento Concurrente]
        
        WS -->|OnStart| Timer
        Timer -->|Trigger| CL
        CL --> TP
    end

    subgraph "Persistencia (SQL Server)"
        DB_Orig[(EpicorERPPilot\nBase de Datos Origen)]
        DB_Dest[(EpicorBoxito\nBase de Datos Destino)]
    end

    subgraph "Servicios Externos"
        API[Orquestador API\nHTTP Port 8080]
    end

    TP -->|Lectura/Escritura (NHibernate)| DB_Orig
    TP -->|Lectura/Escritura (NHibernate)| DB_Dest
    TP -->|POST (JSON) auth: x-api-key| API

    %% Transacciones
    note1[Riesgo: Transacción Manual\nFalta de 2PC o Saga Pattern] -.-> TP
```

---

## 1.3 MATRIZ DE PUNTUACIÓN DE DESARROLLO (SCORECARD)

| Dominio | Puntuación (1-5) | Observación Principal |
| :--- | :---: | :--- |
| **Arquitectura** | 2 | Monolito acoplado a Windows Service. Configuración estática en `App.config`. |
| **Seguridad** | 1 | Credenciales hardcodeadas en texto plano en `App.config` (BD y API-Keys). |
| **Principios SOLID** | 2 | Violación flagrante de SRP y DIP. Ausencia total de Inyección de Dependencias. |
| **Patrones de Diseño** | 2 | Ausencia de patrones resilientes (Retry, Circuit Breaker). Transacciones manuales riesgosas. |
| **Clean Code** | 3 | Nombramiento aceptable, pero bloques `try-catch` capturan `Exception` genéricas e ignoran rollbacks parciales. |

---

## 1.4 ANÁLISIS CRÍTICO DETALLADO POR ASPECTO

### A. Diseño y Arquitectura (Acoplamiento a SO)
- **Bloqueo a Windows:** El uso de `ServiceBase` obliga a la ejecución exclusiva en Windows, limitando severamente la adopción de ecosistemas Cloud-Native (Linux/Kubernetes).
- **Control de Ciclos de Vida:** El `System.Timers.Timer` puede disparar ejecuciones concurrentes si el bloqueo de `TaskGral.IsCompleted` falla. Un enfoque moderno utilizaría `BackgroundService` de .NET Core con `IHostedService`.

### B. Seguridad (Hardcoded Secrets)
- **Credenciales Expuestas:** En `App.config`, las cadenas de conexión (`User ID=interfacesparnet;Password=8UyrTqJkL;`) y los tokens del API (`x-api-key`, `Authorization`) están en texto claro. 
- **Ausencia de TLS:** La comunicación hacia el orquestador (`http://10.40.3.30:8080/`) viaja sin cifrado.

### C. Calidad del Código y SOLID
- **Inyección de Dependencias (DIP):** Las clases instancian directamente a sus dependencias (`new ConversionPartRepository()`, `ConexionBD.OpenSession()`). Esto imposibilita las pruebas unitarias (Unit Testing).
- **Responsabilidad Única (SRP):** El `ConversionPartRepository` no sólo maneja datos, sino que formatea payloads (`ConvRequest`), envía peticiones HTTP al Orquestador y escribe en logs. Esto es un antipatrón masivo.

### D. Transaccionalidad y Resiliencia
- **Doble Transacción Peligrosa:** En `OriginAsync()`, se abren transacciones en `sessionOrigen` y `sessionDestino`. Si el commit de `sessionDestino` falla, el de `sessionOrigen` no se hace rollback adecuadamente, dejando los sistemas inconsistentes.
- **Manejo de Errores:** En `DestinyAsync`, los errores HTTP (Status == 3) lanzan un `new Exception()` vacío, perdiendo el stack trace original y el contexto.

---

## 1.5 SECCIÓN DE RECOMENDACIONES CRÍTICAS Y PLAN DE REMEDIACIÓN

### Prioridad 0 (P0) - Riesgos Críticos Inmediatos
1. **Remover Secretos del Código:** Migrar las contraseñas y API Keys a variables de entorno o un gestor de secretos (ej. Azure Key Vault, HashiCorp Vault) inmediatamente.
2. **Refactorizar el Manejo Transaccional:** Implementar un patrón **Outbox** o en su defecto un bloque `TransactionScope` global si se soporta MSDTC, para evitar estados inconsistentes entre la Base de Datos Origen y Destino.

### Prioridad 1 (P1) - Estabilidad y Arquitectura
1. **Extraer Lógica de Dominio y HTTP:** Remover las llamadas REST del repositorio (`ConversionPartRepository`) y moverlas a un servicio cliente tipado (ej. `IOrchestratorClient`).
2. **Inyección de Dependencias:** Incorporar un contenedor IoC (ej. `Microsoft.Extensions.DependencyInjection`) para desacoplar NHibernate y los servicios lógicos.

### Prioridad 2 (P2) - Modernización
1. **Migración de Framework:** Migrar el proyecto de `.NET Framework 4.7.2` a `.NET 8` como un `Worker Service` multiplataforma.
2. **Resiliencia HTTP:** Usar `HttpClientFactory` en combinación con `Polly` para agregar reintentos automatizados y Circuit Breaker al comunicarse con el Orquestador.
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

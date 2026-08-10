# EstadoCuentaProveedores

[![.NET Framework](https://img.shields.io/badge/.NET_Framework-4.5.2%20%7C%204.7.2-blue.svg)](#)
[![.NET 8](https://img.shields.io/badge/.NET-8.0-purple.svg)](#)
[![SQL Server](https://img.shields.io/badge/SQL_Server-Epicor%2011%20%7C%20Operativa-CC2927.svg)](#)

Repositorio central del ecosistema **EstadoCuentaProveedores**. Esta plataforma es responsable de sincronizar, consolidar y exponer la información financiera de los proveedores, actuando como puente entre el ERP Epicor 11 y el portal de autogestión de proveedores.

## Arquitectura y Componentes

El ecosistema actual opera bajo un modelo híbrido que integra aplicaciones legadas (WebForms, ASMX, WinForms) con procesos de modernización (.NET 8 Worker Services). Comprende cinco componentes principales:

1.  **EstadoCuentaNet**: Portal web (ASP.NET WebForms) consumido por los proveedores.
2.  **WSEstadoCuenta**: API heredada (ASMX) que encapsula la lógica de lectura hacia Epicor.
3.  **BoxEstadoCuentasService**: Worker Service (.NET 8) encargado de la orquestación y sincronización periódica de catálogos y transacciones.
4.  **AgenteEstadoCuenta (SOAP/REST)**: Procesos de escritorio legacy (WinForms) diseñados para ingesta de datos con estrategias de *full reload*.

> ⚠️ **Estado del repositorio (`branch: DEV.VELA`)**: Se ha completado una auditoría arquitectónica profunda. Se identificaron vulnerabilidades críticas (OWASP Top 10), deuda técnica significativa y deriva de contratos (drift) en las integraciones SOAP. Los equipos de desarrollo deben consultar la documentación adjunta antes de planificar evolutivos o despliegues.

## Documentación Técnica

La documentación ha sido estructurada en tres vectores principales para facilitar el *onboarding* y la remediación técnica. 

### 1. [Auditoría Técnica](AUDITORIA_TECNICA.md)
Análisis de código fuente, configuraciones y despliegues. Contiene el blueprint arquitectónico y el roadmap de remediación.

*   [Resumen ejecutivo y hallazgos críticos](AUDITORIA_TECNICA.md#1-resumen-ejecutivo)
*   [Diagrama de topología y dependencias](AUDITORIA_TECNICA.md#2-inventario-de-componentes)
*   [Vector de remediación (SQLi, Credenciales hardcodeadas, Drift WCF)](AUDITORIA_TECNICA.md#3-hallazgos-detallados)
*   [Roadmap priorizado (30-90 días)](AUDITORIA_TECNICA.md#5-recomendaciones-priorizadas)

### 2. [Auditoría de Base de Datos](AUDITORIA_BASE_DATOS.md)
Diccionario de datos, mapeo de dependencias y análisis estático de las persistencias SQL Server y SQLite.

*   [Inventario de conexiones y ambientes](AUDITORIA_BASE_DATOS.md#3-inventario-de-conexiones)
*   [Procedimientos Almacenados (Epicor/Operativa)](AUDITORIA_BASE_DATOS.md#4-procedimientos-almacenados)
*   [Catálogo de tablas y vistas](AUDITORIA_BASE_DATOS.md#5-tablas-bd-estadocuenta)
*   [Estrategia de persistencia en cachés SQLite](AUDITORIA_BASE_DATOS.md#8-bases-sqlite-locales)
*   [Riesgos y planes de mitigación de esquemas](AUDITORIA_BASE_DATOS.md#9-riesgos-y-recomendaciones-sobre-datos)

### 3. [Casos de Uso](CASOS_DE_USO.md)
Especificación funcional extraída por ingeniería inversa desde la capa de Controladores/Code-Behind hacia la Base de Datos (35 flujos documentados).

*   [Portal de Autogestión (16 flujos)](CASOS_DE_USO.md#sitio-web-estadocuentanet--portal-webforms)
*   [Integraciones vía WS ASMX (3 flujos)](CASOS_DE_USO.md#servicio-web-wseestadocuenta--asmx)
*   [Worker Service .NET 8 (3 flujos)](CASOS_DE_USO.md#servicio-windows-boxestadocuentasservice)
*   [Procesos de ingesta / Escritorio (8 flujos)](CASOS_DE_USO.md#escritorio-agenteestadocuenta)
*   [Flujos transversales (Autenticación, Paginación, Exportación)](CASOS_DE_USO.md#casos-de-uso-transversales--entre-aplicaciones)
*   [Matriz de flujos rotos/incompletos](CASOS_DE_USO.md#casos-de-uso-rotos--incompletos)

---
*Documentación generada tras revisión estática del código base. Mantener actualizada conforme se resuelva la deuda técnica.*
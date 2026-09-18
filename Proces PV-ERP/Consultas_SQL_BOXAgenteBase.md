# Diccionario de Datos y Consultas SQL - BOXAgenteBase

Este documento recopila la estructura y las consultas SQL extraídas directamente del código fuente (`ApplicationDbContext` y repositorios bajo `DataAccess`) del BOXAgenteBase. A diferencia de lo esperado con `DbContext`, el proyecto no utiliza Entity Framework para el mapeo, sino que implementa un envoltorio (*wrapper*) ADO.NET asíncrono puro (vía `SqlConnection` y `SqlCommand`).

## 1. Base de Datos de Replicación (Local DB)

Esta base de datos almacena el estado de cada transacción que viaja hacia Epicor, permitiendo tolerancia a fallos, reintentos y encolamiento.

### Tablas Principales

**Tabla: `ReplicateData`**
Almacena la trama principal de los registros que van de la sucursal a Epicor.
*   `ReplicationDataGuid` (uniqueidentifier) - Llave Primaria (Id).
*   `DataId` (varchar) - Identificador único proveniente del origen (FolioÚnico).
*   `ReplicationDataStatus` (int) - Estado del registro (Ej. 0=NEW, 1=INPROCESS).
*   `CreateAt` (datetime) - Fecha de creación.
*   `ModifiedAt` (datetime) - Fecha de última modificación.
*   `ContentData` (varchar/json) - Objeto JSON transaccional (ej. VentaContado).
*   `DocumentVersion` (varchar) - Versión de la trama (Ej. "v1", "v3").
*   `DocumentName` (varchar) - Tipo de documento (Ej. 'CashSale', 'Payment').
*   `Attempts` (int) - Número de reintentos de interfaz hacia Epicor.
*   `IsError` (bit) - Bandera de error al procesar en Epicor.
*   `ProcessDescription` (varchar) - Respuesta/Descripción del Orquestador de Epicor.
*   `InitialRequest` (varchar) - Petición enviada originalmente.
*   `Source` (varchar) - Origen de la trama.

**Tabla: `ReplicateDataHistory`**
*   `ReplicationDataGuid` (uniqueidentifier) - FK hacia ReplicateData.
*   `Attempts` (int) - Número de intento.
*   `ProcessDescription` (varchar) - Error/Detalle del intento fallido.
*   `InitialRequest` (varchar)
*   `CreateAt` (datetime)

**Tabla: `PriorityDocuments`**
Permite orquestar el orden en que las tramas se empujan a Epicor.
*   `DataId` (varchar)
*   `DocumentPriority` (int)
*   `Procesed` (int)
*   `GroupingFolio` (varchar)

### Ejemplos de Consultas (DML) de Replicación

**Inserción en ReplicateData:**
```sql
INSERT INTO [dbo].[ReplicateData]
    ([DataId], [ReplicationDataStatus], [CreateAt], [ContentData], [DocumentVersion], [DocumentName], [Source])
VALUES
    (@DataId, @ReplicationDataStatus, @CreateAt, @ContentData, @DocumentVersion, @DocumentName, @Source)
```

**Obtención de datos encolados con Reintentos (`GetStrCommandReplicateData`):**
```sql
SELECT 
    RD.ReplicationDataGuid, RD.DataId, RD.ReplicationDataStatus, RD.CreateAt, 
    RD.ModifiedAt, RD.ContentData, RD.DocumentName, RD.DocumentVersion, RD.Attempts, 
    RD.IsError, RD.ProcessDescription, RD.InitialRequest, TMP.DocumentPriority
FROM ReplicateData RD WITH(NOLOCK)
INNER JOIN @TMPTable TMP ON RD.ReplicationDataGuid = TMP.ReplicationDataGuid 
ORDER BY ISNULL(TMP.DocumentPriority,100), ModifiedAt
```
*(Nota: Para obtener tramas pendientes, se filtran estados IN (0,1) y se revisan las prioridades enlazando contra `PriorityDocuments`).*

---

## 2. Base de Datos Sucursal (Parnet DB)

El Agente Base realiza *Pooling* constante a las bases de datos transaccionales (Parnet) de cada sucursal para extraer Ventas, Cobros y Notas de Crédito. Esto se hace construyendo strings SQL dinámicas e inyectándolas vía ADO.NET.

### Tablas Operativas Consultadas
*   **Ventas:** `FacturaEncabezado`, `FacturaDetalle`, `salidaencabezado`, `salidadetalle`, `OrdEntradaEncabezado`, `OrdVtaEncabezado`.
*   **Cobranza (Multicobro):** `cobroencabezado`, `cobrodetalle`, `cajaingreso`, `cajaingresofactura`, `cajaingresodetalle`.
*   **Notas de Crédito y Anticipos:** `CxCNCreditoEncabezado`, `CxCNCreditoDetalle`, `DevolucionFacturaSaldo`.
*   **Catálogos Maestros:** `Cliente`, `Articulo`, `ArticuloKit`, `Motivo`.

### Ejemplos de Consultas (DML) de Extracción

**Extracción de Ventas Contado (`GetStrCommandVentasFEncabezado`):**
Obtiene los encabezados de facturas (Tickets o Contado) que no se han extraído (`ExportStatus = 0`), que están activas (`Estatus = 'A'`) y valida que no sean ventas por traspaso.
```sql
SELECT TOP (n)
    FE.Empresa AS [FEEmpresa], FE.Folio AS [FEFolio], FE.ExportStatus AS [FEExportStatus],
    FE.FolioUnico AS [FEFolioUnico], SAL.Importe, SAL.ImporteAplicado,
    NULL, NULL, NULL, SE.Observacion8 ObservacionTraspaso
FROM FacturaEncabezado FE  
LEFT JOIN ClienteSaldoDocumento SAL ON FE.Folio = SAL.Folio 
                                    AND FE.Empresa = SAL.Empresa
                                    AND SAL.Operacion = 'FACTURA'
LEFT JOIN OrdEntradaEncabezado OE ON FE.Empresa = OE.Empresa
                                    AND FE.Folio = OE.Observacion4
LEFT JOIN salidaencabezado SE ON FE.Empresa = SE.Empresa
                                    AND FE.Folio = SE.Folio
WHERE FE.ExportStatus = 0 
  AND FE.Estatus = 'A' 
  AND FE.CondicionPago = 'CONT' 
  AND FE.Anticipo IS NULL
  AND OE.Folio IS NULL 
  AND DATEDIFF(SECOND,FE.FechaCaptura,GETDATE()) >= @saleselapsedtimesecond
ORDER BY FE.Fecha
```

**Extracción de Detalles de Venta (`GetStrCommandVentasFDetalles`):**
Esta consulta cruza `FacturaDetalle` con las salidas de inventario (`salidadetalle`), desempaquetando *Kits* si existen, apoyándose en la tabla `ArticuloKit`.
```sql
SELECT 
    SD.Empresa FDEmpresa, SD.Folio FDFolio, SD.Partida FDPartida, 
    SD.Cantidad CantidadFD, SD.Articulo ArticuloFD, ART.Clave SkuBox, 
    ART.SkuBox FDArticulo, SD.PrecioUnitario FDPrecio, SD.TotalImporte FDTotalImporte, 
    SD.TotalDescuento FDTotalDescuento, SD.TotalImpuesto FDTotalImpuesto, 
    SD.Total FDTotal, SD.DescripcionArticulo FDDescripcionArticulo, 
    SD.Cantidad FDCantidad, SD.UMedPartida FDUMedPartida, SD.CantidadUMedInv FDCantidadUMedInv
FROM salidadetalle SD 
INNER JOIN Articulo ART ON ART.Clave = SD.Articulo
WHERE SD.Empresa = @Empresa AND SD.Folio = @Folio AND Art.Kit != 'S'
ORDER BY SD.Partida
```

**Extracción de Multicobro (`GetStrCommandReplicatePaymentData`):**
Parnet soporta que un Cobro cubra varias facturas y use distintas formas de pago. La consulta arma una tabla temporal en memoria con la lógica de cruce de la tabla recaudadora principal (`cajaingreso`).
```sql
SELECT 
    CE.Empresa, CE.Folio, fac.folio [FolioFactura], 
    CASE WHEN CIF.foliofactura IS NULL THEN 0 ELSE 1 END Multicobro
FROM cobroencabezado CE
INNER JOIN cobrodetalle det on CE.empresa = det.empresa and CE.folio = det.folio
INNER JOIN facturaencabezado fac on det.empresa = fac.empresa and det.foliodocumento = fac.folio
LEFT JOIN cajaingresofactura CIF ON det.Folio = CIF.FolioCobro and det.FolioDocumento = FolioFactura 
WHERE (1= CASE 
            WHEN CIF.foliofactura IS NULL AND CE.ExportStatus IN (0) THEN  1 
            WHEN CIF.foliofactura IS NOT NULL AND ISNULL(CIF.ExportStatus,0) in (0) THEN 1 
            ELSE 0
        END)			
AND CE.Estatus = 'A' AND ISNULL(CE.FolioUnico,'') != '' 
GROUP BY CE.Empresa, CE.Folio, fac.folio, CIF.foliofactura
```

### Marcado de Extracción (Evitar duplicidad)

Cuando una trama sube a la base local de réplica sin errores de dependencias (NC o Anticipos), se dispara un `UPDATE` en la tabla origen de Parnet para marcar el registro como "Sincronizado" o "En Proceso" dependiendo su `ExportStatus` (donde 1 suele significar *Completado/Visto*).

Ejemplo de marcado al completar una Venta de Crédito (en `DataAccess/Replication/ReplicationDataUpdateDA` y `GetValidateEpoch`):
```sql
UPDATE [dbo].[FacturaEncabezado]
SET [ExportStatus] = 1, -- (O el estatus derivado del State de réplica)
    [CreateAt] = @CreateAt
WHERE Folio = @Folio 
  AND Empresa = @Empresa 
  AND [FolioUnico] = @prevDataId
  AND [ExportStatus] = 0 -- O estatus previo (0=Nuevo)
```
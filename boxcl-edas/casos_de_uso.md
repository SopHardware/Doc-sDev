# Casos de Uso y Reglas de Negocio Ingeridas

A partir del análisis del código fuente (específicamente `frmLogin.cs` y `frmEmbarque.cs`), se han extraído los siguientes flujos operativos y reglas de negocio embebidas en la interfaz.

## 1. Actores del Sistema
*   **Operador de Sucursal/Almacén:** Realiza las lecturas de los folios, valida rutas e ingresa credenciales.

## 2. Casos de Uso Principales

### CU01 - Autenticación y Validación de Configuración (`frmLogin.cs`)
*   **Flujo Principal:** El operador ingresa su Usuario y Contraseña. Al presionar Entrar, el sistema cambia el cursor (`WaitCursor`), lee la URL del servicio desde el archivo de configuración a través de la clase `Utility` e instancia el Web Service `EDA_BoxCajas`. Llama al método `ValidaUsuarioNew(sucursal, user, pass)`.
*   **Procesamiento de Respuesta:** El servidor responde con una cadena delimitada por pipes (ej. `OK|v1.2|...`). El sistema hace un `.Split('|')` para verificar si la versión del cliente coincide con la requerida en el índice `[1]` de la respuesta.
*   **Manejo de Excepciones:** Si el servicio ASMX no responde, se captura un `Exception` genérico y se muestra el mensaje real del error de red a través de un `MessageBox`, interrumpiendo el acceso.

### CU02 - Embarque y Validación de Rutas (`frmEmbarque.cs`)
*   **Flujo Principal:** El operador ingresa o escanea una ruta. Dependiendo de un `RadioButton` (`rdTras`), el sistema consulta los destinos de la ruta llamando a `ObtenerSucursalesDestinoDeRuta` u `ObtenerFacturasDestinoDeRuta`. Si la respuesta es exitosa, se puebla un `ListBox` (`lstDestino`) con los datos.
*   **Regla de Negocio (Tipificación de Folio):** Al escanear el folio de una caja/tarima, el sistema valida internamente mediante manipulación de strings (`folio.Substring(0,2)`) para determinar el tipo. Si comienza con `EP` o `EL`, se asume un flujo especial, de lo contrario cae por defecto a `SC`.
*   **Flujos Alternativos:** Si el web service devuelve una cadena que contiene la palabra `"Error"` o un guion `"-"`, el sistema asume que la ruta no existe o es inválida, y alerta al usuario mediante un `MessageBox`.

### CU03 - Gestión Offline (Inferido de `/DAO/`)
*   Aunque algunos formularios interactúan síncronamente con el ASMX, la existencia de `dsOffline` y `BoxCajasDB.sdf` indica que flujos específicos de recolección (como Inventario Cíclico o Segregación) guardan la información localmente en DataSets offline cuando no hay red y se procesan posteriormente.
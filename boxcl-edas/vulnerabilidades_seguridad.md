# Auditoría de Seguridad y Vulnerabilidades

El análisis del código fuente de `BoxCajas` revela múltiples brechas de seguridad críticas típicas de aplicaciones .NET Compact Framework desarrolladas hace más de una década.

## 1. Comunicaciones Inseguras (Transmisión en Texto Plano)
*   **Nivel de Riesgo:** **Crítico**
*   **Hallazgo:** Las referencias web (Web References) en el `.csproj` y el comportamiento del código apuntan a endpoints HTTP (ej. `http://192.168.20.19/BoxCajasWS_prueba/EDA_BoxCajas.asmx`). Las credenciales (`txtPass.Text`) se envían sin cifrado o hash previo desde el `frmLogin.cs` directo por la red corporativa.
*   **Consecuencia:** Cualquier atacante o malware dentro de la misma subred puede interceptar la comunicación (Man-in-the-Middle) con herramientas como Wireshark y obtener usuarios, contraseñas y datos del negocio de las cajas.
*   **Remediación Tecnológica (Migración):** La futura aplicación (MAUI o Flutter) debe requerir TLS 1.2+ (HTTPS) obligatoriamente en todos los consumos REST, y no enviar jamás contraseñas en plano, sino utilizar OAuth2 o JWT Tokens.

## 2. Information Disclosure en Excepciones
*   **Nivel de Riesgo:** **Medio-Alto**
*   **Hallazgo:** A lo largo de la UI (ej. `frmLogin.cs` línea 83), ante una falla de conexión, se utiliza `MessageBox.Show("Error al intentar conectar: " + ex.Message)`.
*   **Consecuencia:** El `ex.Message` en el ecosistema .NET puede incluir nombres de servidores internos, cadenas de conexión SQL truncadas, o rutas físicas del backend (ej. `EndpointNotFoundException`). Esto otorga a un usuario malintencionado inteligencia sobre la topología interna.
*   **Remediación:** Implementar un middleware global de captura de errores y mostrar al usuario únicamente mensajes sanitizados (ej. "Ha ocurrido un error de conexión con el servidor. Intente más tarde"). Registrar el error real en un log local cifrado.

## 3. Protocolos de API Frágiles
*   **Nivel de Riesgo:** **Alto**
*   **Hallazgo:** Los servicios devuelven "Resultados" (strings) que la aplicación divide (`.Split('|')`).
*   **Consecuencia:** Es susceptible a Inyección o desbordamientos lógicos. Si un dato devuelto por el servidor incluye un `|` (pipe) por error de captura, la aplicación puede generar un `IndexOutOfRangeException` y cerrarse súbitamente (Denial of Service local). No hay validación de integridad (checksums) ni estructura rígida.
*   **Remediación:** Usar JSON para toda transferencia de datos, aprovechando serializadores seguros que escapan caracteres automáticamente.

## 4. Almacenamiento Local (BoxCajasDB.sdf)
*   **Nivel de Riesgo:** **Alto**
*   **Hallazgo:** Se distribuye el archivo físico `BoxCajasDB.sdf`. A falta de inspeccionar el motor en ejecución, este tipo de bases de datos suelen no requerir contraseña o usar contraseñas débiles en estas implementaciones.
*   **Remediación:** En la versión nueva, usar SQLite cifrado (SQLCipher).
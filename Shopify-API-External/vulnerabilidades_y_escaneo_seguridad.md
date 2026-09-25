# Vulnerabilidades y Escaneo de Seguridad - BoxShopifyApiExternal

## 1. Problemas Críticos Encontrados

### 1.1 Bypass de Validación SSL (ServerCertificateCustomValidationCallback)
- **Ubicación:** `Program.cs` en la configuración de `ShopifyClient`.
- **Descripción:** Se ha introducido un callback que retorna `true` indiscriminadamente para el host `localhost`. Aunque esté comentado como uso local, esto es un riesgo grave. Si el código se despliega en un entorno de producción o pruebas donde un DNS local apunte a endpoints maliciosos, el sistema será vulnerable a ataques Man-in-the-Middle (MitM).
- **Recomendación:** Envolver esta configuración en una directiva de preprocesador `#if DEBUG` o con una evaluación estricta de entorno `app.Environment.IsDevelopment()` para garantizar que jamás se ejecute en los despliegues de Producción.

### 1.2 Credenciales y API Keys Hardcodeadas en el Código
- **Ubicación:** `Security/SecurityAuthFilter_.cs`
- **Descripción:** El filtro de seguridad contiene credenciales directamente escritas en duro (`ExpectedApiKey = "3E3C9AF9-4B4C-4F6F-94A8-A8DB639F354C"`, `ExpectedBasicUser = "SYSUSER"`, `ExpectedBasicPass = "box2021"`). Hardcodear credenciales es una de las vulnerabilidades más críticas en el OWASP Top 10 (Exposure of Sensitive Information).
- **Recomendación:** Mover todas las API Keys, usuarios y contraseñas al archivo `appsettings.json`, inyectarlas vía patrón Options (IConfiguration) y, preferiblemente, manejar los valores en Producción a través de un gestor de secretos (Azure Key Vault, AWS Secrets Manager) o Variables de Entorno seguras.

### 1.3 Paso de Credenciales a través de Query String
- **Ubicación:** `ShopifyLoginController` (Endpoint `v1/Login`, método que recibe `[FromQuery] ShopifyLoginDto`).
- **Descripción:** Enviar credenciales (usuario/contraseña o tokens de login) a través del Query String en peticiones GET o POST provoca que estas credenciales queden expuestas directamente en texto plano en los logs de los servidores de aplicaciones (IIS, Nginx), balanceadores de carga, firewalls y en el historial del navegador.
- **Recomendación:** Modificar el parámetro para que se reciba desde el cuerpo de la petición utilizando `[FromBody] ShopifyLoginDto` sobre el verbo HTTP POST, asegurando además que toda la comunicación ocurra estrictamente sobre HTTPS.

### 1.4 Verificación de Tokens Implementada Manualmente
- **Ubicación:** `ShopifyController` (Endpoints `/v1/products` y `/v1/list`) y `Security/SecurityAuthFilter.cs`.
- **Descripción:** La verificación de la existencia del token Bearer se está realizando manualmente extrayendo la cadena del header `Authorization`. Esta implementación personalizada es propensa a errores, no valida correctamente la firma criptográfica (Signature) del JWT, ni su caducidad (Expiration), lo que expone a la API a tokens falsificados o expirados.
- **Recomendación:** Configurar e implementar `app.UseAuthentication()` y el esquema estándar de validación `AddJwtBearer(...)` en `Program.cs`. Sustituir la lógica manual de los controladores por el decorador nativo `[Authorize]`.

## 2. Puntos Fuertes Identificados
- **Manejo Centralizado de Errores:** La implementación de `UseExceptionHandler` evita fugas de stack traces o información sensible en caso de excepciones no controladas.
- **Cierre de Conexiones de Logging:** Uso adecuado de `Log.CloseAndFlush()` en el bloque finally del `Program.cs` asegura que los eventos de auditoría no se pierdan al cerrarse la aplicación abruptamente.

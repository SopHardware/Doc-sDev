# Análisis Profundo de Ciberseguridad - BoxShopifyApiExternal

**Contexto:** Este análisis evalúa la API desde una perspectiva de ciberseguridad para un paso a Producción, asumiendo la existencia de un WAF (Web Application Firewall). 

**Premisa Fundamental:** Un WAF protege el perímetro (Capa 7 OSI), mitigando ataques genéricos (inyecciones SQL, XSS, DDoS), pero **NO puede corregir fallos arquitectónicos o vulnerabilidades en la lógica de negocio de la aplicación**. El propio WAF puede convertirse en un vector de fuga de datos si la aplicación está mal diseñada.

---

## 🚨 1. Requerimientos Básicos (Bloqueantes para Producción)
*Si estos puntos no se corrigen, el WAF no podrá proteger la API; de hecho, registrará información confidencial.*

### 1.1 Credenciales en Query String (Fuga de Datos Crítica)
- **Problema:** El endpoint `v1/Login` del `ShopifyLoginController` recibe el `ShopifyLoginDto` a través de `[FromQuery]`. Esto provoca que las credenciales viajen directamente en la URL.
- **Impacto con WAF:** El WAF, los balanceadores de carga, proxys inversos y el propio servidor web (IIS/Kestrel) **registran las URLs completas en sus logs de acceso**. Se estarán guardando credenciales en texto plano en los logs de toda la infraestructura, violando normativas como GDPR o PCI-DSS.
- **Solución:** Cambiar el decorador a `[FromBody]` e invocar el endpoint exclusivamente mediante un verbo `POST`.

### 1.2 Validación Manual y Débil del Token JWT
- **Problema:** Tanto el `ShopifyController` como el `SecurityAuthFilter.cs` extraen el token verificando únicamente si la cadena de texto empieza con "Bearer ". **No se está validando la firma criptográfica (Signature), el emisor (Issuer), la audiencia (Audience) ni la fecha de caducidad (Expiration)** del JSON Web Token.
- **Impacto:** Cualquier atacante puede fabricar un token con estructura válida o reutilizar un token expirado, y la API lo aceptará como legítimo. Un WAF no puede validar la firma interna de un JWT si la aplicación no lo hace.
- **Solución:** Eliminar la validación manual por cadenas. Utilizar el paquete nativo `Microsoft.AspNetCore.Authentication.JwtBearer`. Configurar `builder.Services.AddAuthentication().AddJwtBearer(...)` en `Program.cs` y usar el decorador `[Authorize]` en los controladores.

### 1.3 Remoción del Bypass de SSL (`ServerCertificateCustomValidationCallback`)
- **Problema:** En `Program.cs`, la configuración del `ShopifyClient` incluye un callback que acepta ciegamente los certificados SSL si el host es `localhost`.
- **Impacto:** Si la configuración de producción apunta erróneamente a una URL local o existe una vulnerabilidad de envenenamiento DNS (DNS Spoofing), se facilita un ataque de intermediario (*Man-in-the-Middle*).
- **Solución:** Encapsular estrictamente ese bloque de código dentro de una directiva de preprocesador `#if DEBUG` o evaluar el entorno en tiempo de ejecución: `if (builder.Environment.IsDevelopment())`.

---

## 🛡️ 2. Requerimientos Medios (Defensa en Profundidad)
*Prácticas necesarias para robustecer la API asumiendo que un atacante ya logró evadir o hacer bypass del WAF.*

### 2.1 Restricción de Hostnames (`AllowedHosts`)
- **Problema:** El archivo `appsettings.json` define `"AllowedHosts": "*"`, lo que permite que la API responda a cualquier cabecera `Host` HTTP.
- **Sinergia con WAF:** El WAF normalmente filtra la cabecera Host, pero si un atacante descubre la IP directa del servidor (haciendo un bypass completo del WAF), la API le responderá sin reparos.
- **Solución:** Restringir el valor al dominio o dominios reales de la aplicación (ej. `"api.boxshopify.tuempresa.com"`).

### 2.2 Limpieza de Credenciales Hardcodeadas (Código Muerto)
- **Problema:** El archivo `SecurityAuthFilter_.cs` (versión con guion bajo) contiene claves de API y credenciales básicas escritas directamente en el código fuente (`"3E3C9AF9..."`, `"SYSUSER"`, `"box2021"`).
- **Impacto:** Aunque el código no esté activo, si el repositorio fuente se filtra o un desarrollador habilita la clase por error, se exponen credenciales con altos privilegios.
- **Solución:** Eliminar este archivo completamente o borrar las credenciales. Los secretos deben residir de forma segura (ej. Azure Key Vault, AWS Secrets Manager) e inyectarse en el CI/CD mediante Variables de Entorno.

### 2.3 Habilitar HSTS y Redirección HTTPS
- **Problema:** El pipeline en `Program.cs` no incluye `app.UseHttpsRedirection()` ni `app.UseHsts()`.
- **Sinergia con WAF:** Aunque el WAF imponga HTTPS hacia el cliente final (SSL Offloading), el tráfico en la red interna entre el WAF y el servidor de la API podría estar viajando en texto plano (HTTP).
- **Solución:** Asegurar que la comunicación interna también esté encriptada y habilitar los middlewares de seguridad de .NET para forzar túneles seguros.

---

## 💎 3. Requerimientos Ideales (Nivel Enterprise)
*Configuraciones avanzadas para máxima resiliencia.*

### 3.1 Rate Limiting a Nivel de Lógica de Negocio (L7)
- **Contexto:** El WAF es excelente deteniendo ataques volumétricos (DDoS a nivel de red). Sin embargo, si un atacante realiza intentos de login a un ritmo moderado (ej. 50 por minuto), el WAF podría dejarlo pasar considerándolo tráfico "legítimo".
- **Solución:** Implementar el middleware nativo `Microsoft.AspNetCore.RateLimiting` (introducido en .NET 7/8). Se deben crear políticas de segmentación, por ejemplo: máximo 5 intentos de autenticación por IP por minuto, o un máximo de llamadas al endpoint de productos vinculado al ID del *Claim* del usuario autenticado.

### 3.2 Validación Estricta de DTOs (Input Validation Avanzada)
- **Contexto:** El WAF filtra patrones conocidos como `<script>` o `UNION SELECT`, pero no conoce las reglas del negocio de tu API (ej. si `limit` no debe superar 100).
- **Solución:** Aprovechar las bibliotecas como `FluentValidation` o usar rigurosamente los atributos `DataAnnotations` en los DTOs (por ejemplo, agregar `[Range(1, 100)]` en `limit`). Esto previene que un atacante envíe peticiones masivas (ej. `limit=999999`) provocando que la aplicación agote la memoria o los recursos de CPU (Ataque de Denegación de Servicio a nivel aplicativo).

### 3.3 Arquitectura Zero Trust y Rotación de Secretos
- **Contexto:** Esta API funciona como un intermediario hacia Shopify y Epicor. Si el servidor de `BoxShopifyApiExternal` es comprometido, el atacante tendría la llave a los sistemas core de la compañía.
- **Solución:** Las llaves (API Keys) utilizadas para consumir Shopify y Epicor deben seguir el Principio de Mínimo Privilegio (PoLP). Además, no deben ser estáticas de por vida; se debe diseñar una estrategia donde un Key Vault rote estas credenciales automáticamente sin requerir nuevos despliegues de código de la API.
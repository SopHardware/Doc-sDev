# Buenas Prácticas y Deuda Técnica - BoxShopifyApiExternal

## 1. Buenas Prácticas Implementadas

- **Patrón CQRS y Mediator:** La arquitectura utiliza `MediatR` de forma muy clara para separar la intención (Queries/Commands) de los manejadores (Handlers). Esto promueve la alta cohesión, el bajo acoplamiento y facilita la testabilidad del sistema.
- **Filtros Personalizados Controlados:** El registro de `SecurityAuthFilter` y `ApiKeyAuthAttribute` demuestran un intento sólido de centralizar la autorización usando `IAsyncAuthorizationFilter`, manteniendo limpios los controladores.
- **Manejo de Respuestas Homogéneas:** La aplicación hace uso del middleware `ProblemDetails` y un manejador de excepciones global (`GlobalExceptionHandler`), lo que garantiza que los clientes de la API siempre reciban estructuras de error consistentes (estándar RFC 7807).
- **Observabilidad (Logging):** Existe una configuración madura de `Serilog` inyectada en el Host de .NET, lo cual es fundamental y excelente para un microservicio de integración que necesita monitorear fallos hacia APIs de terceros.

## 2. Deuda Técnica a Resolver

### 2.1 Código Comentado y "Muerto"
- **Descripción:** Archivos como `Program.cs` y `SecurityAuthFilter.cs` contienen bloques enteros de código comentado (ej. clientes HTTP antiguos, validaciones de BD comentadas en el filtro, métodos de autorización enteros en `SecurityAuthFilter_.cs`).
- **Impacto:** Reduce drásticamente la legibilidad, aumenta el "ruido" al navegar por el código y genera confusión sobre la lógica actualmente activa.
- **Plan de Acción:** Eliminar todo el código fuente comentado. El control de versiones (Git) es la herramienta adecuada para recuperar implementaciones anteriores si alguna vez son requeridas.

### 2.2 Reinvención de la Rueda en Autenticación
- **Descripción:** Se ha creado una lógica customizada para validar la presencia de headers `Authorization` tanto en el controlador (`ShopifyController`) como en los filtros personalizados, en lugar de apoyarse en las políticas de autorización de `Microsoft.AspNetCore.Authorization`.
- **Impacto:** Aumenta la superficie de ataque y el esfuerzo de mantenimiento. El código framework estándar está mucho más probado contra vulnerabilidades que implementaciones "in-house".
- **Plan de Acción:** Migrar la seguridad al middleware nativo de ASP.NET Core (`AddAuthentication`, `AddJwtBearer`, y políticas de Claims).

### 2.3 Registro de Servicios Manual vs Automático
- **Descripción:** En `Program.cs` existe un bloque complejo (casi 15 líneas) utilizando Reflection (`assembly.GetTypes()`) para buscar e instanciar handlers específicos en una lista blanca de strings quemados en el código (`handlersPermitidos`).
- **Impacto:** Acoplamiento fuerte a nombres de clases mágicos (strings). Si se refactoriza o renombra un handler, la inyección de dependencias fallará en tiempo de ejecución de manera silenciosa.
- **Plan de Acción:** Si se requiere un registro muy estricto, registrar cada Handler explícitamente (`services.AddTransient(...)`). Si no, confiar en la característica nativa de MediatR para que descubra todos los handlers del assembly.

### 2.4 Tipado Débil en Respuestas de Error
- **Descripción:** En algunos filtros y controladores se retornan tipos anónimos u objetos sueltos (ej. `return Unauthorized(new { status = 401, error = "..." })`).
- **Impacto:** Los consumidores de la API no tienen un contrato estricto del error y la documentación Swagger no podrá mapear correctamente este modelo.
- **Plan de Acción:** Retornar siempre objetos fuertemente tipados (clases BaseResponse, ErrorResponse) o usar de manera nativa los resultados estándar basados en `ProblemDetails`.

# BoxShopifyApiExternal

## Descripción
BoxShopifyApiExternal es un microservicio Web API (Gateway) desarrollado en ASP.NET Core 8 (.NET 8.0). Su propósito principal es actuar como capa intermedia y de abstracción para integraciones entre el sistema interno (Epicor) y la plataforma de e-commerce Shopify.

## Tecnologías Principales
- **.NET 8.0**
- **MediatR** (Patrón CQRS)
- **Serilog** (Logging estructurado)
- **Swagger / OpenAPI** (Documentación y testing de endpoints)

## Estructura de Endpoints Principales
- POST /api/ShopifyLogin/v1/Login : Autenticación inicial y obtención del token Bearer.
- GET /api/Shopify/v1/products : Obtener el catálogo paginado de productos.
- GET /api/Shopify/v1/list : Obtener las listas de precios.

## Configuración de Entorno
El proyecto requiere configuraciones base en el ppsettings.json para:
- **ShopifyInternal:BaseUrl**
- **Epicor:BaseUrl**

## Uso
1. Clonar el repositorio y abrir en Visual Studio 2022 o VS Code.
2. Compilar dependencias. Asegurar que los proyectos Application.Services y Common.Builders estén accesibles.
3. Ejecutar y acceder al portal Swagger en /swagger para probar los endpoints interactuando con el Token Bearer proporcionado por la ruta de Login.

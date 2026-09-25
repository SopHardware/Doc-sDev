# Casos de Uso - BoxShopifyApiExternal

## 1. Login (Autenticación)
- **Endpoint:** POST /api/ShopifyLogin/v1/Login
- **Descripción:** Permite a un cliente autenticarse enviando credenciales (ShopifyLoginDto) por query string. Devuelve un AuthenticatedUserDto (con token).
- **Controlador:** ShopifyLoginController
- **Handler:** LoginShopifyExternalQuery

## 2. Obtener Productos
- **Endpoint:** GET /api/Shopify/v1/products
- **Descripción:** Permite obtener una lista de productos paginada, especificando días (days), límite (limit) y desplazamiento (offset).
- **Seguridad:** Requiere token Bearer (header Authorization) y [ApiKeyAuth].
- **Controlador:** ShopifyController
- **Handler:** ProductsShopifyExternalQuery

## 3. Obtener Lista de Precios
- **Endpoint:** GET /api/Shopify/v1/list
- **Descripción:** Permite obtener la lista de precios de Shopify.
- **Seguridad:** Requiere token Bearer (header Authorization) y [ApiKeyAuth].
- **Controlador:** ShopifyController
- **Handler:** ShopifyPriceListExternalQuery

# Estructura de Base de Datos - BoxShopifyApiExternal

## 1. Análisis de Contexto
El proyecto BoxShopifyApiExternal es una API de integración (Gateway/Proxy) que no expone una capa de persistencia directa (Entity Framework Core o Dapper) en este código fuente.

## 2. Dependencias de Datos
El proyecto no tiene cadenas de conexión de base de datos directas ni configuraciones de DbContext locales. La persistencia e integridad se maneja a través de las APIs externas:
- **Shopify API**: Integración con el e-commerce.
- **Epicor API**: ERP donde seguramente residan las entidades de base de datos finales.

## 3. Modelos de Datos en Tránsito (DTOs)
- ShopifyLoginDto
- AuthenticatedUserDto
- MainProductsShopifyDto
- MainSearchShopifyPriceListDto

Estas estructuras de datos se manipulan a través de comandos y consultas manejadas por MediatR y retransmitidas por HTTP Clients (como ShopifyClient y EpicorClient).

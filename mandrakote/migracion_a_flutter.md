# Hoja de Ruta Estratégica: Migración de Android Java Legacy a Flutter (Enfoque Tablets)

Este documento establece la estrategia arquitectónica y técnica para reescribir nuestra aplicación legacy de Android (Java) a Flutter, con un enfoque principal en la experiencia de usuario para Tablets (Android e iPadOS).

## 1. Diseño Adaptativo UI para Tablets

El paso de móviles a tablets requiere aprovechar el espacio adicional de la pantalla. En Flutter, utilizaremos los siguientes patrones y widgets:

*   **`LayoutBuilder` y `MediaQuery`**: Para tomar decisiones de diseño basadas en el espacio disponible y los puntos de interrupción (breakpoints). Definiremos breakpoints claros (ej. > 600dp para tablets).
*   **Patrón Master-Detail (Split View)**: En lugar de navegar a una nueva pantalla para ver detalles, usaremos una vista dividida donde la lista (Master) está a la izquierda y el detalle (Detail) a la derecha.
*   **`NavigationRail`**: Para la navegación principal en tablets (orientación horizontal), reemplazando el `BottomNavigationBar` típico de móviles, maximizando el espacio vertical.
*   **Widgets Flexibles**: Uso intensivo de `Expanded`, `Flexible`, `GridView` y `Wrap` para contenido que debe refluir según el tamaño de la pantalla.

## 2. Mapeo Tecnológico: Android Java a Dart/Flutter

Para asegurar una transición fluida, mapearemos las tecnologías legacy a sus equivalentes modernos en el ecosistema Flutter:

| Capa / Concepto | Android Java Legacy | Flutter / Dart Moderno |
| :--- | :--- | :--- |
| **UI Declarativa** | XML Layouts + Activities/Fragments | Árbol de Widgets (Stateless/Stateful) |
| **Gestión de Estado** | MVP / MVC / LiveData | **Riverpod** (recomendado) o **BLoC** |
| **Base de Datos Local** | SQLite / Room | **Drift** (tipado fuerte) o **sqflite** |
| **Preferencias** | SharedPreferences | **flutter_secure_storage** / shared_preferences |
| **Red / API REST** | Retrofit + OkHttp | **Dio** (interceptores, caché) + json_serializable |
| **Inyección de Dependencias**| Dagger / Hilt | **get_it** + **injectable** (generación de código) |
| **Navegación** | Intents / Navigation Component | **go_router** (soporte profundo para web/deeplinks) |

## 3. Integración Nativa y Hardware

Dado que la aplicación legacy puede tener dependencias de hardware o SDKs específicos, la estrategia de integración será:

*   **Plugins Oficiales (pub.dev)**: Priorizar el uso de plugins mantenidos por la comunidad o Google (ej. `camera`, `geolocator`, `permission_handler`) para acceso a hardware y permisos.
*   **Platform Channels (`MethodChannel`)**: Para lógica de negocio legacy compleja, SDKs de terceros sin soporte en Flutter (ej. periféricos específicos de hardware), o algoritmos propietarios en Java/C++, crearemos canales de comunicación bidireccionales.
*   **EventChannels**: Para flujos de datos continuos desde el hardware nativo (ej. sensores, lectores de códigos de barras) hacia Flutter.

## 4. Plan de Migración por Fases

Adoptaremos un enfoque iterativo para mitigar riesgos, evitando el patrón "Big Bang" en la medida de lo posible.

### Fase 1: Preparación y Arquitectura Base (Semanas 1-2)
*   Definición de la arquitectura base (Riverpod, go_router, get_it).
*   Creación del Design System en Flutter (Tokens de color, tipografía, componentes base adaptativos).
*   Configuración de CI/CD para Flutter (Android e iOS).

### Fase 2: Capa de Datos y Dominio (Semanas 3-5)
*   Migración de modelos de datos y lógica de negocio pura a Dart.
*   Implementación de clientes de red (Dio) y repositorios locales (Drift).
*   Pruebas unitarias de la capa de dominio.

### Fase 3: UI Core y Navegación Adaptativa (Semanas 6-8)
*   Implementación del esqueleto de la app (`NavigationRail`, Layouts base).
*   Desarrollo de las pantallas principales usando el patrón Master-Detail.
*   Integración con la capa de estado (Providers/BLoCs).

### Fase 4: Integración Nativa y Features Complejas (Semanas 9-11)
*   Implementación de `MethodChannels` para funcionalidades nativas retenidas.
*   Migración de flujos complejos (ej. procesamiento en segundo plano, hardware específico).

### Fase 5: QA, Pulido y Lanzamiento (Semanas 12-14)
*   Pruebas exhaustivas en dispositivos físicos (Tablets Android y iPads).
*   Optimización de rendimiento (DevTools, reducción de rebuilds).
*   Despliegue beta y posterior rollout gradual a producción.

## Mockup de Interfaz para Tablets (UI Wireframe)

A continuación se ilustra la arquitectura de pantalla adaptativa de **Mandrakote Tablet App** en Flutter, implementando un layout con **NavigationRail** para el módulo de **Órdenes de Compra**.

### Mapeo de Pantallas Legacy a Flutter
- **Navegación Global (`NavigationRail`):** Centraliza el acceso del sistema legacy (`ActivityMenuPrincipal`), manteniendo siempre accesible la barra lateral para alternar entre Tablero (`ActivityTableroMes`), Clientes (`ActivityDetalleCliente`), Cotizaciones (`ActivityCatalogoCotizacion`), Órdenes (`ActivityCatalogoOrden`), Visitas (`ActivityCrearVisita`), Perfil (`ActivityPerfilUsuario`) y Configuración.
- **Panel Maestro (`ActivityCatalogoOrden` -> Master):** Panel intermedio de lista con buscador reactivo (`Q Buscar...`), mostrando folios de órdenes de compra (`> OC-9982`), clientes asociados y estado de surtido.
- **Panel Detalle (`ActivityDetalleOrdenCompra` -> Detail):** Panel derecho (`Expanded`) que despliega la orden seleccionada: cliente, datos de envío (`ActivityDatosEnvio`), integración directa de geolocalización con botón de mapa (`MapsActivity` vía `[ 📍 Ver Mapa ]`), listado de artículos solicitados con cantidades y acción de edición (`[ 📝 Editar ]`).

```text
+---------------------------------------------------------------------------------+
|   | ÓRDENES DE COMPRA (Catálogo)| DETALLE ORDEN DE COMPRA #OC-9982            |
| 📊|                             |                                             |
| 👥| Q Buscar...                 | 🏢 Cliente: Constructora Alfa S.A.          |
| 📝|                             | 🚚 Datos de Envío: Calle 42 #123, Centro    |
| 🛒| > OC-9982                   | 📅 Fecha Entrega: 25/09/2026                |
| 🗺️|   Constructora Alfa S.A.    |                                             |
|   |   Pendiente de surtir       | ------------------------------------------- |
|   | --------------------------- | 📦 Artículos Solicitados:                   |
|   | > OC-9981                   | 1. Ladrillo Rojo (Millar)      x 5          |
|   |   Grupo Constructor Sur     | 2. Varilla 3/8" (Tonelada)     x 2          |
|   |   Entregada                 |                                             |
| 👤|                             |                                             |
| ⚙️|                             |       [ 📍 Ver Mapa ]  [ 📝 Editar ]          |
+---------------------------------------------------------------------------------+
```

### Estructura de Widgets en Flutter

```dart
Row(
  children: [
    // 1. Navegación global lateral persistente (Reemplazo de ActivityMenuPrincipal)
    NavigationRail(
      selectedIndex: _selectedIndex,
      onDestinationSelected: (int index) => setState(() => _selectedIndex = index),
      labelType: NavigationRailLabelType.none,
      leading: const Padding(
        padding: EdgeInsets.symmetric(vertical: 8.0),
        child: Icon(Icons.menu),
      ),
      destinations: const [
        NavigationRailDestination(icon: Icon(Icons.bar_chart), label: Text('Tablero')),
        NavigationRailDestination(icon: Icon(Icons.people), label: Text('Clientes')),
        NavigationRailDestination(icon: Icon(Icons.description), label: Text('Cotizaciones')),
        NavigationRailDestination(icon: Icon(Icons.shopping_cart), label: Text('Órdenes')),
        NavigationRailDestination(icon: Icon(Icons.map), label: Text('Visitas')),
      ],
      trailing: Expanded(
        child: Align(
          alignment: Alignment.bottomCenter,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: const [
              Icon(Icons.account_circle),
              SizedBox(height: 12),
              Icon(Icons.settings),
              SizedBox(height: 12),
            ],
          ),
        ),
      ),
    ),
    const VerticalDivider(thickness: 1, width: 1),

    // 2. Panel Maestro: Catálogo de Órdenes (ActivityCatalogoOrden)
    SizedBox(
      width: 300,
      child: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: TextField(
              decoration: const InputDecoration(
                prefixIcon: Icon(Icons.search),
                hintText: 'Q Buscar...',
                border: OutlineInputBorder(),
              ),
              onChanged: (val) => ref.read(ordenFilterProvider.notifier).state = val,
            ),
          ),
          Expanded(
            child: ListView.builder(
              itemCount: ordenes.length,
              itemBuilder: (context, index) {
                final orden = ordenes[index];
                return ListTile(
                  title: Text('> ${orden.folio}'),
                  subtitle: Text('${orden.cliente}\\n${orden.estado}'),
                  selected: orden.id == selectedOrdenId,
                  onTap: () => setState(() => selectedOrdenId = orden.id),
                );
              },
            ),
          ),
        ],
      ),
    ),
    const VerticalDivider(thickness: 1, width: 1),

    // 3. Panel Detalle: Detalle de Orden de Compra (ActivityDetalleOrdenCompra)
    Expanded(
      child: OrderDetailView(
        ordenId: selectedOrdenId,
        onVerMapa: () {
          // Invocación adaptada de MapsActivity
        },
        onEditar: () {
          // Edición de la orden
        },
      ),
    ),
  ],
)
```

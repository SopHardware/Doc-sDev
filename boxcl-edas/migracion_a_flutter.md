# Estrategia de Migración a Flutter (Tablets Android e iPadOS)

## 1. Diseño Adaptativo de UI y Ruptura de Paradigma
Migrar la UI WinForms (`frmEmbarque`, `frmMenu`) a Flutter implica pasar de programación Imperativa a **Declarativa**.
*   **Patrones Multi-Panel para Tablets:** Flutter permite un manejo excepcional del espacio. Usando `LayoutBuilder`, la aplicación detectará si corre en una Tablet. 
    *   El `MenuOperacion` se puede convertir en un `NavigationRail` permanentemente anclado a la izquierda.
    *   Los listados (que usaban `Resco.SmartGrid`) se convertirán en `ListView.builder` o `PaginatedDataTable` altamente performantes y con scroll suave a 60fps.

## 2. Mapeo Tecnológico (C# -> Dart)
*   **Red de Datos (Reemplazo ASMX):** Los Web Services ASMX devolverán JSON tras ser modernizados del lado del servidor. En Flutter, se utilizará el paquete **`dio`**. 
    *   *Ventaja inmediata:* `dio` maneja interceptores. En lugar de instanciar el servicio y configurar el timeout/URL en cada formulario (como se hacía con la clase `Utility`), se configura un único cliente HTTP global que inyecta automáticamente tokens de seguridad (JWT) y gestiona reintentos en zonas de almacén con poca señal WiFi.
*   **Gestión de Estado (Reemplazo del Smart UI):** Se usará **Riverpod**. Las reglas de validación (como el análisis de folio "EP" / "EL") residirán en `Notifiers` de Riverpod. La interfaz (Widgets) simplemente "escuchará" el estado (ej. `AsyncLoading`, `AsyncData`, `AsyncError`), eliminando la necesidad de cambiar el cursor manualmente a `WaitCursor` y bloqueando la UI con un spinner moderno (CircularProgressIndicator).
*   **Bases de Datos Locales (Reemplazo SQL CE):** Se sustituirá `BoxCajasDB.sdf` (y los arcaicos `dsOffline` y `dsTemp`) por el motor **Drift** (antes Moor) en Dart. Drift provee persistencia en SQLite con tipado fuerte y consultas SQL generadas en código, ideal para soportar el módulo de Segregación y Entradas Pendientes de forma offline.

## 3. Integración Nativa (Hardware y Seguridad)
*   **Lectura de Cajas (Scanners):** Se integrará el plugin oficial `mobile_scanner` para acceder a la cámara en iPads/Androids genéricos. Si el cliente usa hardware Rugged (Scanner Láser empotrado), se crearán **Platform Channels** (Method Channels) para escuchar los eventos nativos del lector sin interactuar físicamente con un teclado.
*   **Almacenamiento Seguro:** Las cadenas de conexión y configuraciones inseguras presentes en la vieja clase `Utility` se migrarán a **`flutter_secure_storage`** para estar cifradas a nivel de Sistema Operativo (Keychain/Keystore).

## 4. Fases de Migración Recomendadas
1.  **Modernización del Backend:** Esencial reemplazar `BoxCajasWS_prueba` por API REST (Node.js, .NET Core o similar).
2.  **Core Domain en Dart:** Replicar las reglas de negocio (Clases y Validadores) usando el paquete `freezed` para clases inmutables y de respuesta.
3.  **Capa de Persistencia:** Mapear el `dsOffline.xsd` hacia tablas de Drift en SQLite.
4.  **UI Adaptativa:** Construcción del Layout general (NavigationRail y SplitViews) y migración pantalla por pantalla (Login -> Menú -> Embarques -> Almacén).
5.  **Despliegue Multiplataforma:** Build nativo tanto para Android (`.aab`/`.apk`) como para iPadOS (`.ipa`).
## 5. Mockup de Interfaz (Dart) - Diseño Adaptativo Multi-Panel
A continuación se presenta un concepto de cómo luciría la pantalla principal migrada a Flutter, implementando un `NavigationRail` para la Tablet y una vista dividida (Split-View) para el escaneo de embarques (reemplazo de `frmEmbarque.cs`).

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class EmbarqueTabletPage extends ConsumerWidget {
  const EmbarqueTabletPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Escucha el estado de la UI (Cargando, Datos, Error)
    final embarqueState = ref.watch(embarqueProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Control de Embarques'),
        backgroundColor: Theme.of(context).colorScheme.primary,
        foregroundColor: Colors.white,
      ),
      body: Row(
        children: [
          // MENÚ LATERAL (NavigationRail)
          NavigationRail(
            selectedIndex: 1,
            onDestinationSelected: (int index) { /* Navegación */ },
            labelType: NavigationRailLabelType.all,
            destinations: const [
              NavigationRailDestination(icon: Icon(Icons.inventory), label: Text('Almacén')),
              NavigationRailDestination(icon: Icon(Icons.local_shipping), label: Text('Embarque')),
              NavigationRailDestination(icon: Icon(Icons.sync), label: Text('Sincronizar')),
            ],
          ),
          const VerticalDivider(thickness: 1, width: 1),
          
          // CONTENIDO PRINCIPAL (Split View)
          Expanded(
            child: Row(
              children: [
                // PANEL IZQUIERDO: Formulario de Escaneo
                Container(
                  width: 350,
                  padding: const EdgeInsets.all(24.0),
                  color: Colors.grey.shade50,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      const Text('Captura de Ruta / Folio', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                      const SizedBox(height: 20),
                      TextField(
                        decoration: const InputDecoration(
                          labelText: 'Ingrese Ruta',
                          border: OutlineInputBorder(),
                          prefixIcon: Icon(Icons.map),
                        ),
                        // Envía evento de validación de ruta al ViewModel
                        onSubmitted: (value) => ref.read(embarqueProvider.notifier).validarRuta(value),
                      ),
                      const SizedBox(height: 20),
                      TextField(
                        decoration: const InputDecoration(
                          labelText: 'Escanee Folio',
                          border: OutlineInputBorder(),
                          prefixIcon: Icon(Icons.qr_code_scanner),
                        ),
                        // Envía evento del scanner
                        onSubmitted: (value) => ref.read(embarqueProvider.notifier).escanearFolio(value),
                      ),
                      const Spacer(),
                      // Indicador de estado (Carga/Espera)
                      if (embarqueState.isLoading) 
                        const Center(child: CircularProgressIndicator()),
                    ],
                  ),
                ),
                
                // PANEL DERECHO: Visualización de Datos (Reemplazo de Resco.SmartGrid)
                Expanded(
                  child: Padding(
                    padding: const EdgeInsets.all(24.0),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const Text('Destinos / Cajas Embarcadas', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                        const SizedBox(height: 16),
                        Expanded(
                          child: ListView.builder(
                            itemCount: embarqueState.cajas.length,
                            itemBuilder: (context, index) {
                              final caja = embarqueState.cajas[index];
                              return Card(
                                child: ListTile(
                                  leading: const Icon(Icons.widgets),
                                  title: Text(caja.folio, style: const TextStyle(fontWeight: FontWeight.bold)),
                                  subtitle: Text(caja.descripcion),
                                  trailing: Chip(
                                    label: Text(caja.estatus),
                                    backgroundColor: caja.estatus == 'Sincronizado' ? Colors.green.shade100 : Colors.orange.shade100,
                                  ),
                                ),
                              );
                            },
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

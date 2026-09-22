# Estrategia de Migración a .NET MAUI

Este documento define la estrategia arquitectónica paso a paso para migrar la aplicación Android Java legacy (Mandrakote Tableta) hacia **.NET MAUI** (Multi-platform App UI), con un enfoque específico en la optimización de la experiencia de usuario (UX/UI) para tablets.

## 1. Estrategia Arquitectónica

La migración no debe ser un "lift-and-shift" (copiar y pegar código), sino una reingeniería basada en **Clean Architecture** y el patrón **MVVM (Model-View-ViewModel)**.

### 1.1. Mapeo de Tecnologías (Stack Tecnológico)

| Componente Legacy (Android Java) | Componente Moderno (.NET MAUI) | Justificación |
|----------------------------------|--------------------------------|---------------|
| **Activities / Fragments**       | **ContentPage / ContentView**  | MAUI utiliza páginas y vistas reutilizables basadas en XAML o C# Markup. |
| **Roboguice (DI)**               | **MAUI DI (Microsoft.Extensions.DependencyInjection)** | Integrado nativamente en el `MauiProgram.cs`, estándar de la industria en .NET. |
| **OrmLite (Base de datos)**      | **EF Core SQLite** o **sqlite-net-pcl** | EF Core ofrece un ORM robusto con migraciones; `sqlite-net` es ideal si se busca máximo rendimiento y ligereza. |
| **Retrofit (Networking)**        | **Refit** | Refit es el equivalente directo de Retrofit en .NET, permitiendo definir APIs REST mediante interfaces. |
| **AsyncTask**                    | **Task (async / await)**       | El modelo asíncrono nativo de C# elimina los memory leaks y simplifica el código en background. |
| **Gson (Serialización)**         | **System.Text.Json**           | Nativo, altamente optimizado y de bajo consumo de memoria en .NET. |

## 2. Recomendaciones de UI para Tablets

Las tablets ofrecen un espacio de pantalla significativamente mayor que los teléfonos. La migración debe aprovechar este espacio para evitar interfaces estiradas o vacías.

- **Uso de `FlyoutPage`:** Reemplazar los menús de hamburguesa simples por un `FlyoutPage` (anteriormente MasterDetailPage) que puede mantenerse fijo (pinned) en modo horizontal, permitiendo navegación rápida sin ocultar el menú.
- **Patrón Split-View con `Grid`:** Para pantallas complejas como `ActivityDetalleCotizacion` o `ActivityDetalleOrdenCompra`, utilizar un `Grid` para dividir la pantalla. Por ejemplo:
  - *Columna Izquierda (30%):* Lista de ítems o resumen.
  - *Columna Derecha (70%):* Detalles del ítem seleccionado, formularios de edición y acciones.
- **Componentes Responsivos:** Utilizar `VisualStateManager` y `OnIdiom` para ajustar márgenes, tamaños de fuente y disposición de elementos específicamente cuando la app se ejecuta en una tablet (`Device.Idiom == TargetIdiom.Tablet`).

## 3. Soporte Multiplataforma

Aunque el objetivo principal son las tablets Android, .NET MAUI compila desde un único código base para múltiples plataformas.
- **Android & iOS:** Compartirán el 95%+ del código UI y de negocio.
- **Windows (WinUI 3):** Al estar diseñada para tablets, la interfaz se adaptará naturalmente a pantallas de escritorio, abriendo la posibilidad de uso en PCs de escritorio o tablets Surface sin esfuerzo adicional.
- **Código Específico de Plataforma:** Si se requiere acceso a hardware específico (ej. escáneres de códigos de barras industriales), se utilizarán las carpetas `Platforms/Android` o la inyección de dependencias mediante interfaces (ej. `IBarcodeScanner`).

## 4. Plan de Migración por Fases

Para mitigar riesgos, se propone una migración iterativa:

### Fase 1: Setup y Arquitectura Base (Semanas 1-2)
- Creación de la solución .NET MAUI.
- Configuración del contenedor de inyección de dependencias (`MauiProgram.cs`).
- Definición de la estructura de carpetas (Views, ViewModels, Models, Services, Data).
- Implementación de la capa de red con **Refit** y autenticación.

### Fase 2: Migración de la Capa de Datos (Semanas 3-4)
- Diseño de la base de datos local con **EF Core SQLite** o **sqlite-net**.
- Migración de los modelos de datos (Entities).
- Implementación del patrón Repositorio para abstraer el acceso a datos.

### Fase 3: Refactorización de "Clases Dios" y Lógica de Negocio (Semanas 5-8)
- Desacoplar la lógica de negocio de las antiguas Activities (ej. `ActivityDetalleCotizacion`).
- Escribir los **ViewModels** correspondientes en C#, implementando `INotifyPropertyChanged` (o usando el Toolkit de MVVM `CommunityToolkit.Mvvm` para generar código boilerplate).
- Implementar Unit Tests para los ViewModels y Servicios.

### Fase 4: Implementación de UI para Tablets (Semanas 9-12)
- Creación de las pantallas en XAML utilizando `Grid` y `FlyoutPage`.
- Binding de datos entre las Views y los ViewModels.
- Pruebas de usabilidad en emuladores y dispositivos físicos (Tablets de 8" y 10").

### Fase 5: Pruebas, QA y Despliegue (Semanas 13-14)
- Pruebas de integración y pruebas de estrés (manejo de memoria).
- Distribución de versiones Beta a través de App Center o Google Play Console (Internal Testing).
- Despliegue final y apagado progresivo de la app legacy.

## Mockup de Interfaz para Tablets (UI Wireframe)

A continuación se presenta el wireframe adaptativo para tabletas de la aplicación **Mandrakote**, aplicando el patrón **Maestro-Detalle** para el módulo de **Cotizaciones**.

### Mapeo de Pantallas Legacy a .NET MAUI
- **Navegación Global (Barra Superior y Lateral):** Sustituye la navegación de `ActivityMenuPrincipal`. La barra superior aloja el perfil (`ActivityPerfilUsuario`) y configuración, mientras que el menú lateral persistente ofrece accesos rápidos a Tablero (`ActivityTableroMes`), Clientes (`ActivityDetalleCliente`), Cotizaciones (`ActivityCatalogoCotizacion`), Órdenes (`ActivityCatalogoOrden`) y Visitas (`ActivityCrearVisita`).
- **Panel Maestro (`ActivityCatalogoCotizacion` -> Master):** Columna izquierda intermedia (`CollectionView`) con barra de búsqueda rápida y lista de cotizaciones activas (folio, cliente y monto total).
- **Panel Detalle (`ActivityDetalleCotizacion` -> Detail):** Columna derecha de detalle (`ContentView` / `ScrollView`) que expone los metadatos del cliente, fechas, estado, la tabla de partidas cotizadas (producto, cantidad, subtotal) y los botones de acción (`Modificar`, `Aprobar`, `Enviar PDF`) en un solo plano visual continuo sin cambios de Activity.

```text
+---------------------------------------------------------------------------------+
| ☰ Mandrakote Tablet App                              [ 👤 Perfil ] [ ⚙️ Ajustes] |
+---------+--------------------------+--------------------------------------------+
| 📊 Tabl | CATÁLOGO COTIZACIONES    | DETALLE DE COTIZACIÓN #45092               |
| 👥 Clie |                          |                                            |
| 📝 Coti | [ Buscar cotización... ] | Cliente: Ferretería El Sol                 |
| 🛒 Orde |                          | Fecha: 21/09/2026      Estado: Pendiente   |
| 🗺️ Visi | 📄 Cotización #45092     |                                            |
|         |    Ferretería El Sol     | +----------------------------------------+ |
|         |    $ 12,450.00           | | Producto         Cant     Subtotal     | |
|         |                          | | Tubo PVC 2"       50      $ 2,500.00   | |
|         | 📄 Cotización #45091     | | Cemento Cruz Azul 20      $ 3,200.00   | |
|         |    Construrama Norte     | +----------------------------------------+ |
|         |    $ 8,300.00            |                                            |
|         |                          | [ Modificar ] [ Aprobar ] [ Enviar PDF ]   |
+---------+--------------------------+--------------------------------------------+
```

### Estructura de Contenedores MAUI

```xml
<Grid ColumnDefinitions="80, 280, *" RowDefinitions="Auto, *">
    <!-- Fila 0: Barra superior de la aplicación -->
    <Border Grid.Row="0" Grid.ColumnSpan="3" Style="{StaticResource HeaderBar}">
        <Grid ColumnDefinitions="*, Auto">
            <Label Text="☰ Mandrakote Tablet App" VerticalOptions="Center" />
            <HorizontalStackLayout Grid.Column="1" Spacing="8">
                <Button Text="👤 Perfil" Command="{Binding NavigatePerfilCommand}" />
                <Button Text="⚙️ Ajustes" Command="{Binding NavigateSettingsCommand}" />
            </HorizontalStackLayout>
        </Grid>
    </Border>

    <!-- Fila 1, Columna 0: Barra de navegación lateral (ActivityMenuPrincipal) -->
    <VerticalStackLayout Grid.Row="1" Grid.Column="0" Spacing="6" Padding="6">
        <Button Text="📊 Tabl" Command="{Binding NavigateTableroCommand}" />
        <Button Text="👥 Clie" Command="{Binding NavigateClientesCommand}" />
        <Button Text="📝 Coti" BackgroundColor="{StaticResource AccentColor}" />
        <Button Text="🛒 Orde" Command="{Binding NavigateOrdenesCommand}" />
        <Button Text="🗺️ Visi" Command="{Binding NavigateVisitasCommand}" />
    </VerticalStackLayout>

    <!-- Fila 1, Columna 1: Panel Maestro - Catálogo de Cotizaciones (ActivityCatalogoCotizacion) -->
    <VerticalStackLayout Grid.Row="1" Grid.Column="1" Spacing="8" Padding="10">
        <Label Text="CATÁLOGO COTIZACIONES" FontAttributes="Bold" />
        <SearchBar Placeholder="Buscar cotización..." Text="{Binding SearchTerm}" />
        <CollectionView ItemsSource="{Binding Cotizaciones}"
                        SelectedItem="{Binding SelectedCotizacion}"
                        SelectionMode="Single">
            <CollectionView.ItemTemplate>
                <DataTemplate>
                    <Border Padding="8" Margin="0,4">
                        <VerticalStackLayout>
                            <Label Text="{Binding Folio, StringFormat='📄 Cotización #{0}'}" FontAttributes="Bold" />
                            <Label Text="{Binding ClienteNombre}" />
                            <Label Text="{Binding Total, StringFormat='{0:C}'}" TextColor="{StaticResource PrimaryColor}" />
                        </VerticalStackLayout>
                    </Border>
                </DataTemplate>
            </CollectionView.ItemTemplate>
        </CollectionView>
    </VerticalStackLayout>

    <!-- Fila 1, Columna 2: Panel Detalle - Detalle de Cotización (ActivityDetalleCotizacion) -->
    <ScrollView Grid.Row="1" Grid.Column="2" Padding="15">
        <VerticalStackLayout Spacing="12">
            <Label Text="{Binding SelectedCotizacion.Folio, StringFormat='DETALLE DE COTIZACIÓN #{0}'}" 
                   FontSize="18" FontAttributes="Bold" />
            <Label Text="{Binding SelectedCotizacion.ClienteNombre, StringFormat='Cliente: {0}'}" />
            <Grid ColumnDefinitions="*, *">
                <Label Text="{Binding SelectedCotizacion.Fecha, StringFormat='Fecha: {0:dd/MM/yyyy}'}" />
                <Label Grid.Column="1" Text="{Binding SelectedCotizacion.Estado, StringFormat='Estado: {0}'}" />
            </Grid>
            
            <!-- Tabla de partidas de la cotización -->
            <Border Padding="10">
                <CollectionView ItemsSource="{Binding SelectedCotizacion.Partidas}">
                    <!-- Columnas: Producto | Cant | Subtotal -->
                </CollectionView>
            </Border>

            <!-- Acciones de negocio -->
            <HorizontalStackLayout Spacing="10" HorizontalOptions="End">
                <Button Text="Modificar" Command="{Binding ModificarCommand}" />
                <Button Text="Aprobar" Command="{Binding AprobarCommand}" />
                <Button Text="Enviar PDF" Command="{Binding EnviarPdfCommand}" />
            </HorizontalStackLayout>
        </VerticalStackLayout>
    </ScrollView>
</Grid>
```

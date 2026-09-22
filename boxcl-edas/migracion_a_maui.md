# Estrategia de Migración a .NET MAUI (Tablets Android e iPadOS)

## 1. Rediseño de UI para Tablets y Reemplazo de *Resco Toolkit*
La dependencia actual de **Resco MobileForms Toolkit v5.8** (grids avanzados, controles de imagen y listas) no tiene un camino de actualización directo en MAUI.
*   **Transición Visual:** Se debe abandonar el esquema visual WinForms (controles absolutos o apilados estáticos) en favor del sistema de diseño declarativo de **XAML** en .NET MAUI (`Grid`, `VerticalStackLayout`, `FlexLayout`).
*   **Tablets (Diseño Multi-panel):** Aprovechando el mayor tamaño de pantalla, las múltiples pestañas o ventanas secuenciales de `frmEmbarque` o `frmSurtido` deben unificarse. Se puede usar un `FlyoutPage` o vistas de columnas (Master-Detail) donde el escáner se active a la izquierda y la lista de cajas (el antiguo `Resco.SmartGrid`) se muestre dinámicamente usando un `CollectionView` en el lado derecho.

## 2. Mapeo y Modernización Tecnológica
*   **Backend ASMX a REST API:** 
    *   *Antiguo:* `EDA_BoxCajas.asmx` y respuestas `.Split('|')`.
    *   *Nuevo:* Reescribir los ASMX en un microservicio moderno (ej. **ASP.NET Core Web API**). El cliente MAUI consumirá esto usando `HttpClient` de forma 100% asíncrona (`async`/`await`) serializando/deserializando **JSON** vía `System.Text.Json`, lo cual elimina la fragilidad del parser manual.
*   **Gestión de Estado (MVVM):** Implementar el **MVVM Toolkit** (`CommunityToolkit.Mvvm`). Las reglas de negocio (ej. validación del prefijo "EP" o "EL") vivirán en el `EmbarqueViewModel` y no en el code-behind.
*   **Persistencia (Local DB):**
    *   *Antiguo:* `BoxCajasDB.sdf` (SQL CE) y DataSets (`dsOffline.xsd`).
    *   *Nuevo:* **Entity Framework Core (SQLite)**. Se mapearán las tablas locales a modelos C# modernos (POCOs) para la gestión del modo offline (inventarios cíclicos).

## 3. Integración Nativa de Escáner y Hardware
*   **Lector Código de Barras:** Para dispositivos de consumo (iPad, Galaxy Tab) se puede usar `ZXing.Net.MAUI` integrado con la cámara de la tablet. Si se migra a Tablets industriales (Honeywell, Zebra), se implementará un servicio que intercepte los *Broadcast Intents* del teclado virtual (DataWedge) usando código de plataforma específico (`#if ANDROID`).
*   **Seguridad:** Uso de `SecureStorage` para reemplazar el archivo de configuración en texto plano `App.config` o el instanciamiento inseguro de credenciales.

## 4. Fases de Migración Recomendadas
1.  **Refactor del Backend:** Transformar los servicios `BoxCajasWS_prueba` de ASMX a REST JSON.
2.  **Arquitectura Core MAUI:** Set-up de Inyección de Dependencias, capa SQLite EF Core y configuración de red.
3.  **Desarrollo UI/UX:** Creación de los Layouts en XAML pensados para pantallas horizontales (Landscape) de Tablets de 10 pulgadas.
4.  **Integración de Reglas de Negocio:** Migrar lógica contenida en `frmLogin`, `frmEmbarque`, `frmRecepConso` hacia sus respectivos ViewModels.
5.  **Pruebas en Terreno:** Despliegue mediante Intune o Mobile Device Management (MDM).
## 5. Mockup de Interfaz (XAML) - Diseño Split-View para Tablet
A continuación se presenta un concepto de cómo luciría la pantalla de Embarque (`frmEmbarque.cs`) refactorizada para una Tablet usando `Grid` y `CollectionView` en MAUI:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:BoxCaja.MAUI.ViewModels"
             x:Class="BoxCaja.MAUI.Views.EmbarqueView"
             Title="Control de Embarques">

    <ContentPage.BindingContext>
        <vm:EmbarqueViewModel />
    </ContentPage.BindingContext>

    <!-- Layout adaptativo para Tablet (Landscape) -->
    <Grid ColumnDefinitions="350, *" RowDefinitions="Auto, *">
        
        <!-- Header Global -->
        <Border Grid.Row="0" Grid.ColumnSpan="2" BackgroundColor="{StaticResource Primary}" Padding="15">
            <Label Text="Módulo de Embarques" TextColor="White" FontSize="24" FontAttributes="Bold" />
        </Border>

        <!-- PANEL IZQUIERDO: Controles de Captura -->
        <VerticalStackLayout Grid.Row="1" Grid.Column="0" Padding="20" Spacing="15" BackgroundColor="{StaticResource Gray100}">
            <Label Text="Captura de Ruta / Folio" FontSize="18" FontAttributes="Bold" />
            
            <Entry Placeholder="Ingrese Ruta" Text="{Binding Ruta}" ReturnCommand="{Binding ValidarRutaCommand}" />
            
            <HorizontalStackLayout Spacing="10">
                <RadioButton Content="Traslado" IsChecked="{Binding EsTraslado}" />
                <RadioButton Content="Factura" IsChecked="{Binding EsFactura}" />
            </HorizontalStackLayout>
            
            <Button Text="Validar Ruta" Command="{Binding ValidarRutaCommand}" />
            
            <BoxView HeightRequest="1" Color="LightGray" Margin="0,10" />
            
            <Entry Placeholder="Escanee Folio (EP, EL, SC)" Text="{Binding FolioEscaneado}" ReturnCommand="{Binding EscanearFolioCommand}" />
            
            <!-- Indicador de Carga -->
            <ActivityIndicator IsRunning="{Binding IsBusy}" IsVisible="{Binding IsBusy}" />
        </VerticalStackLayout>

        <!-- PANEL DERECHO: Visualización de Datos (Sustituto de Resco.SmartGrid) -->
        <Grid Grid.Row="1" Grid.Column="1" Padding="20" RowDefinitions="Auto, *">
            <Label Grid.Row="0" Text="Destinos / Cajas Embarcadas" FontSize="18" FontAttributes="Bold" Margin="0,0,0,10" />
            
            <CollectionView Grid.Row="1" ItemsSource="{Binding Destinos}">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <Frame Margin="0,0,0,10" Padding="15" BorderColor="LightGray" HasShadow="True">
                            <Grid ColumnDefinitions="*, Auto">
                                <VerticalStackLayout Grid.Column="0">
                                    <Label Text="{Binding Folio}" FontSize="16" FontAttributes="Bold" />
                                    <Label Text="{Binding Descripcion}" TextColor="Gray" />
                                </VerticalStackLayout>
                                <Label Grid.Column="1" Text="{Binding Estatus}" TextColor="{Binding EstatusColor}" FontAttributes="Bold" VerticalOptions="Center" />
                            </Grid>
                        </Frame>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>
        </Grid>
    </Grid>
</ContentPage>
```

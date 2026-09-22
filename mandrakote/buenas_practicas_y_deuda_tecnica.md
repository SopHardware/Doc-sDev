# Evaluación de Arquitectura y Deuda Técnica

Este documento presenta un diagnóstico detallado de la arquitectura actual de la aplicación Android Java legacy (Mandrakote Tableta) y la deuda técnica acumulada, así como propuestas de patrones de diseño para su refactorización.

## 1. Diagnóstico de la Arquitectura Actual

Tras analizar el código fuente y la estructura del proyecto, se han identificado múltiples anti-patrones y problemas críticos de diseño que dificultan el mantenimiento, la escalabilidad y la estabilidad de la aplicación.

### 1.1. Clases Dios (God Classes) y Código Espagueti
El proyecto sufre de una severa falta de separación de responsabilidades (Single Responsibility Principle). Las `Activities` actúan como "Clases Dios", concentrando la lógica de presentación, reglas de negocio, llamadas a red y acceso a base de datos en un solo lugar.
- **Ejemplos críticos:** 
  - `ActivityDetalleCotizacion.java` (más de 3,100 líneas de código).
  - `ActivityDetalleOrdenCompra.java` (más de 1,800 líneas de código).
- **Impacto:** El código es altamente acoplado (código espagueti), imposible de someter a pruebas unitarias (Unit Testing) y propenso a regresiones cada vez que se introduce un cambio.

### 1.2. Uso de Patrones y Librerías Deprecadas
El ecosistema de Android ha evolucionado significativamente, pero el proyecto se ha quedado estancado en tecnologías obsoletas:
- **AsyncTasks:** Se encontraron más de 45 implementaciones de `AsyncTask` anidadas dentro de las Activities. Este patrón está oficialmente deprecado por Google debido a que provoca fugas de memoria (Memory Leaks) al retener el contexto de la Activity si esta es destruida (por ejemplo, al rotar la pantalla) mientras el hilo en background sigue ejecutándose.
- **Roboguice:** Se utiliza `org.roboguice:roboguice:3.0.1` para la inyección de dependencias. Esta librería está abandonada y no es compatible con las versiones modernas de Android.
- **Android Support Library:** El proyecto sigue utilizando `com.android.support` (v23.3.0) en lugar de haber migrado a **AndroidX**, lo cual impide el uso de componentes modernos de Jetpack.
- **OrmLite:** Se emplea `ormlite-android:4.48` para la persistencia de datos, un ORM antiguo que carece de las optimizaciones y la seguridad de tipos que ofrecen soluciones modernas como Room.

### 1.3. Acoplamiento Fuerte y Retención de Contextos
La inyección de dependencias con Roboguice no se está utilizando para desacoplar la lógica de negocio. Las instancias de red (Retrofit) y base de datos (OrmLite) se instancian o se inyectan directamente en las vistas. Además, el paso de `Context` a clases de negocio y tareas asíncronas genera un alto riesgo de `OutOfMemoryError`.

---

## 2. Propuestas de Patrones de Diseño para Refactorización

Para mitigar esta deuda técnica y preparar el terreno para una arquitectura limpia (ya sea refactorizando en Android nativo o migrando a .NET MAUI), se deben adoptar los siguientes patrones:

### 2.1. Clean Architecture (Arquitectura Limpia)
Separar el proyecto en capas bien definidas con reglas de dependencia estrictas (de afuera hacia adentro):
- **Capa de Presentación (UI):** Solo contiene lógica visual.
- **Capa de Dominio (Casos de Uso / Interactors):** Contiene las reglas de negocio puras, sin dependencias del framework.
- **Capa de Datos (Repositorios):** Maneja la obtención de datos, decidiendo si vienen de la red (API) o de la base de datos local.

### 2.2. Patrón MVVM (Model-View-ViewModel)
Reemplazar el patrón MVC implícito (donde la Activity es el Controlador y la Vista al mismo tiempo) por MVVM:
- **View (Activity/Fragment/Page):** Observa los cambios de estado.
- **ViewModel:** Mantiene el estado de la vista y expone flujos de datos reactivos. Sobrevive a los cambios de configuración (rotación de pantalla), eliminando el problema de los `AsyncTasks`.

### 2.3. Patrón Repository (Repositorio)
Crear una abstracción sobre las fuentes de datos. La capa de dominio o el ViewModel no deben saber si los datos provienen de Retrofit o de OrmLite; solo deben interactuar con una interfaz `IRepository`.

### 2.4. Inyección de Dependencias (DI) Moderna
Reemplazar Roboguice por un contenedor de inyección de dependencias moderno y con soporte activo. Esto permitirá inyectar repositorios en los ViewModels y casos de uso, facilitando el testing mediante *Mocks*.

### 2.5. Programación Asíncrona Moderna
Eliminar por completo los `AsyncTasks`. La lógica asíncrona debe manejarse mediante corrutinas (Coroutines en Kotlin) o `async/await` (en C#/.NET), garantizando que los procesos en segundo plano se cancelen adecuadamente cuando la vista sea destruida, evitando fugas de memoria.
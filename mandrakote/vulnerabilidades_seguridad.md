# Vulnerabilidades de Seguridad y Auditoría (Migración a MAUI)

## Auditoría de Seguridad (Basado en OWASP Mobile Top 10)

Esta auditoría estática se basa en el código y configuración de la aplicación legacy Android Java, enfocándose en vulnerabilidades comunes según el OWASP Mobile Top 10, que deben ser mitigadas durante la migración a .NET MAUI.

### 1. Comunicaciones Inseguras (Uso de HTTP en Producción)
*   **Nivel de Riesgo:** **Crítico**
*   **Ubicación:** `app/src/main/AndroidManifest.xml` (Línea 23)
*   **Descripción:** La aplicación tiene explícitamente configurado `android:usesCleartextTraffic="true"` en el bloque `<application>`, lo que permite enviar y recibir tráfico en texto plano (HTTP). Esto expone los datos a ataques de *Man-In-The-Middle* (MITM).
*   **Recomendación en MAUI:**
    *   No incluir configuraciones que permitan *cleartext traffic* en los `Platforms/Android/AndroidManifest.xml` ni en `Platforms/iOS/Info.plist` (NSAppTransportSecurity).
    *   Usar siempre `HTTPS` en `HttpClient` o `Refit`.

### 2. Almacenamiento de Datos Inseguro (Shared Preferences en Texto Plano)
*   **Nivel de Riesgo:** **Alto**
*   **Ubicación:** `app/src/main/java/com/boxito/boxm/MayoreoApplication.java` (Líneas 94 y 103) y `PreferencesController.java`.
*   **Descripción:** Se utiliza `SharedPreferences` con `MODE_PRIVATE` sin ningún tipo de cifrado. En dispositivos *rooteados*, cualquier aplicación maliciosa puede leer este archivo XML y extraer tokens de sesión, credenciales o datos sensibles.
    ```java
    SharedPreferences sp = getContext().getSharedPreferences("com.boxito.boxm.boxito_mayoreo", MODE_PRIVATE);
    ```
*   **Recomendación en MAUI:**
    *   Utilizar **`Microsoft.Maui.Storage.SecureStorage`** para guardar tokens y credenciales. `SecureStorage` utiliza el *Keystore* de Android y el *Keychain* de iOS automáticamente.

### 3. Exposición de Claves Criptográficas (Keystore en el Repositorio)
*   **Nivel de Riesgo:** **Crítico**
*   **Ubicación:** `certificado/ventunmay.jks`
*   **Descripción:** El archivo Java KeyStore (`.jks`) utilizado para firmar la aplicación está versionado en el código fuente. Si las contraseñas del keystore o el alias están hardcodeadas en un `build.gradle` o se ven comprometidas, un atacante podría publicar actualizaciones maliciosas haciéndose pasar por el desarrollador legítimo.
*   **Recomendación en MAUI:**
    *   Eliminar de inmediato los keystores del repositorio y agregarlos a `.gitignore`.
    *   Manejar la firma de la aplicación mediante un sistema CI/CD seguro (ej. Azure Pipelines o GitHub Actions) utilizando *Secure Files* o inyectando los certificados en tiempo de compilación.

### 4. Dependencias Obsoletas con Vulnerabilidades Conocidas (CVEs)
*   **Nivel de Riesgo:** **Alto**
*   **Ubicación:** `app/build.gradle`
*   **Descripción:** El proyecto utiliza versiones extremadamente viejas de librerías, las cuales tienen múltiples vulnerabilidades de seguridad documentadas:
    *   `com.squareup.okhttp:okhttp:2.3.0` (Soporte TLS obsoleto)
    *   `com.google.android.gms:play-services:9.0.2` (Componentes antiguos con fallos conocidos)
    *   `org.apache.http.legacy` (Deprecado y propenso a errores de seguridad)
*   **Recomendación en MAUI:**
    *   Al cambiar de ecosistema (Java a .NET), este riesgo se mitiga al abandonar estas librerías.
    *   En MAUI, asegurarse de mantener los paquetes NuGet actualizados (dependabot) y preferir `HttpClient` nativo (con `AndroidMessageHandler` u `NSUrlSessionHandler`) para aprovechar las actualizaciones de seguridad del SO.

### 5. Configuración de API Keys en Recursos
*   **Nivel de Riesgo:** **Medio**
*   **Ubicación:** `app/src/main/AndroidManifest.xml`
*   **Descripción:** El `google_maps_key` se lee desde un recurso estático de strings (`@string/google_maps_key`). Es fácil realizar ingeniería inversa en el APK y extraer esta clave.
*   **Recomendación en MAUI:**
    *   Si bien las claves de Google Maps deben estar en el manifiesto, es importante **restringir** la API Key en la consola de Google Cloud para que solo pueda ser usada por el *Package Name* y el hash SHA-1 de firma de la app, previniendo abusos.

---
*Este reporte ha sido autogenerado basándose en el análisis estático de las configuraciones y el código del repositorio legado.*
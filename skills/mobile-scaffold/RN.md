# mobile-scaffold — REACT NATIVE (Phases 3-4)

> Sub-archivo de `SKILL.md`. Cargado via PHASE ROUTER para Phases 3 y 4 cuando `framework = react-native`.
> NUNCA ejecutar directamente — siempre invocado desde `skills/mobile-scaffold/SKILL.md`.

---

## Phase 3: React Native Scaffold

### GATE IN
- [ ] Phase 2 completada sin errores bloqueantes
- [ ] `framework = react-native` confirmado
- [ ] `workflow` determinado: `expo` (default) | `bare`
- [ ] `name`, `navigation`, `state`, `bridges` capturados y normalizados

### MUST DO

> Todas las acciones son OBLIGATORIAS

**Paso previo — Resolver versión de Expo:**

1. [ ] Consultar Context7 para resolver la versión actual de `expo` en npm (latest stable):
   - Si Context7 disponible: `resolve-library-id("expo")` → `query-docs(id, "latest stable SDK version")`
   - Si Context7 no disponible o no responde en ~30 seg: usar **versión baseline: Expo SDK 52 / RN 0.76.x** — continuar sin bloquear
   - Registrar la versión resuelta para usarla en `app.json` y `package.json`

---

#### Sección A — Expo managed workflow (cuando `workflow = expo`, DEFAULT)

2. [ ] **Generar estructura de directorios y archivos** mediante instrucciones al LLM:

```
{nombre}/
  app.json
  package.json
  App.tsx
  src/
    navigation/
      RootNavigator.tsx
    screens/
      HomeScreen.tsx
    store/                    ← Solo si state = zustand o redux
      useAppStore.ts          ← Solo si state = zustand
      store.ts                ← Solo si state = redux
    components/
      SafeArea.tsx
  assets/
    icon.png                  ← placeholder — reemplazar con ícono real antes de publicar
    splash.png                ← placeholder — reemplazar con splash screen real antes de publicar
  eas.json
  tsconfig.json
  .gitignore
```

3. [ ] **Generar `app.json`** con:
   - `sdkVersion`: versión resuelta en Paso 1 (ej: `"52.0.0"`)
   - `newArchEnabled: true`
   - `ios.bundleIdentifier`: `"com.example.{nombre}"`
   - `android.package`: `"com.example.{nombre}"`
   - `plugins`: lista de plugins según `--bridges`:
     - cámara → `"expo-camera"`
     - push → `"expo-notifications"`
     - storage → `"expo-secure-store"`
     - Si `bridges = []` → `plugins: []`

4. [ ] **Generar `package.json`** con dependencias según configuración:

   Dependencias SIEMPRE incluidas (versiones actuales o baseline):
   ```json
   "@react-navigation/native": "^7.x",
   "@react-navigation/native-stack": "^7.x",
   "react-native-screens": "^4.x",
   "react-native-safe-area-context": "^4.x"
   ```

   Dependencias condicionales por `--navigation`:
   - `tabs` → agregar `"@react-navigation/bottom-tabs": "^7.x"`
   - `drawer` → agregar `"@react-navigation/drawer": "^7.x"`

   > Compatibilidad framework+state validada en DISCOVERY.md Phase 1 — ver tabla de compatibilidad.

   Dependencias condicionales por `--state`:
   - `zustand` → agregar `"zustand": "^5.x"`
   - `redux` → agregar `"@reduxjs/toolkit": "^2.x"`, `"react-redux": "^9.x"`
   - `jotai` → agregar `"jotai": "^2.x"`
   - `context` → sin dependencias adicionales de state

   Dependencias condicionales por `--bridges`:
   - cámara → agregar `"expo-camera": "~16.x"`
   - push → agregar `"expo-notifications": "~0.29.x"`
   - storage → agregar `"expo-secure-store": "~14.x"`

5. [ ] **Generar `App.tsx`**:
   ```tsx
   import { NavigationContainer } from '@react-navigation/native';
   import { RootNavigator } from './src/navigation/RootNavigator';

   export default function App() {
     return (
       <NavigationContainer>
         <RootNavigator />
       </NavigationContainer>
     );
   }
   ```

6. [ ] **Generar `src/navigation/RootNavigator.tsx`** según `--navigation`:
   - `stack` → `createNativeStackNavigator` con `HomeScreen` como screen inicial
   - `tabs` → `createBottomTabNavigator` con `HomeScreen` como tab inicial
   - `drawer` → `createDrawerNavigator` con `HomeScreen` como drawer item inicial

7. [ ] **Generar `src/screens/HomeScreen.tsx`** como placeholder funcional con:
   - Uso explícito del navigator correspondiente (tipado TypeScript)
   - Comentario: `// TODO: Reemplazar con la pantalla real`

8. [ ] **Generar `src/store/`** solo si `state` es `zustand` o `redux`:
   - `zustand` → `useAppStore.ts` con store base tipado:
     ```ts
     import { create } from 'zustand';

     interface AppState {
       // TODO: Definir el estado de la app
     }

     export const useAppStore = create<AppState>()(() => ({
       // TODO: Valores iniciales
     }));
     ```
   - `redux` → `store.ts` con `configureStore` de `@reduxjs/toolkit` y un slice de ejemplo

9. [ ] **Generar `src/components/SafeArea.tsx`**:
   ```tsx
   import { SafeAreaView, StyleSheet, ViewProps } from 'react-native';

   export function SafeArea({ children, style }: ViewProps) {
     return (
       <SafeAreaView style={[styles.container, style]}>
         {children}
       </SafeAreaView>
     );
   }

   const styles = StyleSheet.create({
     container: { flex: 1 },
   });
   ```

10. [ ] **Generar `eas.json`** con 3 profiles:
    ```json
    {
      "cli": { "version": ">= 10.0.0" },
      "build": {
        "development": {
          "developmentClient": true,
          "distribution": "internal"
        },
        "preview": {
          "distribution": "internal"
        },
        "production": {
          "autoIncrement": true
        }
      }
    }
    ```

11. [ ] **Generar `tsconfig.json`**:
    ```json
    {
      "extends": "expo/tsconfig.base",
      "compilerOptions": {
        "strict": true
      }
    }
    ```

12. [ ] **Generar `.gitignore`** para Expo managed:
    ```
    node_modules/
    .expo/
    ios/
    android/
    dist/
    .env
    *.p8
    ```
    Nota: `ios/` y `android/` se excluyen porque en Expo managed los nativos no se versionan en el repo.

---

#### Sección B — Bare workflow (cuando `workflow = bare`)

> Todo lo de Sección A EXCEPTO: NO generar `eas.json`. En su lugar, agregar los archivos nativos:

13. [ ] **Generar `ios/Podfile`**:
    ```ruby
    require_relative '../node_modules/react-native/scripts/react_native_pods'
    require_relative '../node_modules/@react-native-community/cli-platform-ios/native_modules'

    platform :ios, '15.1'
    prepare_react_native_project!

    target '{NombreApp}' do
      config = use_native_modules!
      use_react_native!(
        :path => config[:reactNativePath],
        :hermes_enabled => true,
        :fabric_enabled => true
      )
    end
    ```
    Donde `{NombreApp}` = PascalCase del nombre del proyecto (ej: `my-app` → `MyApp`)

14. [ ] **Generar `ios/{NombreApp}/Info.plist`**:
    - Entradas base de permisos según `--bridges`:
      - cámara → ver Phase 4 (se completa ahí)
      - push → ver Phase 4
      - storage → no requiere entradas de Info.plist adicionales
    - Generar con entradas base si no hay bridges: solo `CFBundleDisplayName`, `CFBundleIdentifier`, `NSAppTransportSecurity`

15. [ ] **Generar `android/build.gradle`** (nivel raíz):
    ```groovy
    buildscript {
        ext {
            buildToolsVersion = "35.0.0"
            minSdkVersion = 24
            compileSdkVersion = 35
            targetSdkVersion = 35
            ndkVersion = "26.1.10909125"
        }
        repositories { google(); mavenCentral() }
        dependencies {
            classpath("com.android.tools.build:gradle:8.1.1")
            classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion")
        }
    }
    ```

16. [ ] **Generar `android/app/build.gradle`** con:
    - `applicationId "com.example.{nombre}"`
    - `minSdkVersion`, `compileSdkVersion`, `targetSdkVersion` referenciando `rootProject.ext`
    - Dependencias nativas según bridges (ver bridges en `package.json` de Sección B)

17. [ ] **Generar `android/app/src/main/AndroidManifest.xml`**:
    - Permisos base + permisos por bridges (ver Phase 4)

18. [ ] **Generar `android/app/src/main/MainApplication.kt`**:
    ```kotlin
    package com.example.{nombre}

    import android.app.Application
    import com.facebook.react.PackageList
    import com.facebook.react.ReactApplication
    import com.facebook.react.ReactHost
    import com.facebook.react.ReactNativeHost
    import com.facebook.react.ReactPackage
    import com.facebook.react.defaults.DefaultNewArchitectureEntryPoint.load
    import com.facebook.react.defaults.DefaultReactNativeHost

    class MainApplication : Application(), ReactApplication {
      override val reactNativeHost: ReactNativeHost =
        DefaultReactNativeHost(this) {
          override fun getPackages(): List<ReactPackage> =
            PackageList(this).packages
          override fun getJSMainModuleName(): String = "index"
          override fun getUseDeveloperSupport(): Boolean = BuildConfig.DEBUG
          override val isNewArchEnabled: Boolean = BuildConfig.IS_NEW_ARCHITECTURE_ENABLED
          override val isHermesEnabled: Boolean = BuildConfig.IS_HERMES_ENABLED
        }

      override fun onCreate() {
        super.onCreate()
        if (BuildConfig.IS_NEW_ARCHITECTURE_ENABLED) { load() }
      }
    }
    ```

19. [ ] **Generar `android/gradle.properties`**:
    ```properties
    android.useAndroidX=true
    android.enableJetifier=true
    newArchEnabled=true
    hermesEnabled=true
    ```

20. [ ] **Generar `metro.config.js`** para Nueva Arquitectura:
    ```js
    const { getDefaultConfig } = require('expo/metro-config');
    // Si bare sin Expo: usar require('@react-native/metro-config').getDefaultConfig(__dirname)
    const config = getDefaultConfig(__dirname);
    module.exports = config;
    ```

21. [ ] **`package.json` de Bare** — cambiar bridges a versiones bare:
    - cámara → `"react-native-vision-camera": "^4.x"` (en lugar de `expo-camera`)
    - push → `"@notifee/react-native": "^9.x"` (en lugar de `expo-notifications`)
    - storage → `"react-native-keychain": "^9.x"` (en lugar de `expo-secure-store`)

22. [ ] **`.gitignore` de Bare** — incluir nativos en repo pero excluir builds:
    ```
    node_modules/
    ios/build/
    ios/Pods/
    android/build/
    android/app/build/
    android/.gradle/
    .env
    *.p8
    *.keystore
    google-services.json
    GoogleService-Info.plist
    ```

### CHECKPOINT Phase 3
- [ ] `app.json` válido con `newArchEnabled: true` y `sdkVersion` correcto (Expo managed)
- [ ] `package.json` con todas las dependencias para `navigation` + `state` + `bridges` seleccionados
- [ ] `eas.json` generado con 3 profiles: `development`, `preview`, `production` (solo Expo managed)
- [ ] `App.tsx` con `NavigationContainer` + `RootNavigator`
- [ ] `RootNavigator.tsx` con tipo de navigator correcto según `--navigation`
- [ ] `src/store/` generado con patrón correcto (no stubs vacíos) si `state = zustand | redux`
- [ ] Para bare: `Podfile` con `hermes_enabled: true` y `fabric_enabled: true`
- [ ] Para bare: `build.gradle` raíz con `targetSdkVersion 35` y `minSdkVersion 24`
- [ ] Para bare: `gradle.properties` con `newArchEnabled=true` y `hermesEnabled=true`
- [ ] Para bare: bridges reemplazados por versiones bare (`vision-camera`, `notifee`, `keychain`)

### IF FAILS
```
Context7 no responde al resolver versión de Expo:
  → Usar versión baseline: Expo SDK 52 / RN 0.76.x
  → Documentar en README: "Versión baseline usada — verificar última versión en https://expo.dev"
  → No bloquear la ejecución

Versión de dependencia no encontrada en Context7:
  → Usar versión baseline del spec (^7.x, ^5.x, etc.)
  → Continuar sin bloquear

Nombre del proyecto produce PascalCase ambiguo (ej: "my-app-2" → "MyApp2"):
  → Usar el resultado mecánico sin modificar
  → Notificar al usuario el nombre PascalCase usado para ios/ y MainApplication
```

---

## Phase 4: Native Bridges Configuration

### GATE IN
- [ ] Phase 3 completada — estructura de proyecto generada
- [ ] Al menos un bridge seleccionado (`bridges` es lista no vacía)
- [ ] Si `bridges = []` → **SKIP esta phase completa**, continuar a Phase 5

### MUST DO

> Ejecutar solo para bridges seleccionados. Omitir secciones de bridges no seleccionados.

---

#### Bridge: Cámara

**Expo managed:**
- El plugin `"expo-camera"` en `app.json` gestiona los permisos automáticamente via EAS Build
- No requiere modificaciones manuales a `Info.plist` ni `AndroidManifest.xml`
- Agregar comentario en `app.json` junto al plugin:
  ```
  // expo-camera: los permisos se inyectan automáticamente via EAS Build
  ```

**Bare — iOS:**
1. [ ] Agregar a `ios/{NombreApp}/Info.plist`:
   ```xml
   <key>NSCameraUsageDescription</key>
   <string>This app requires camera access to take photos.</string>
   <key>NSPhotoLibraryUsageDescription</key>
   <string>This app requires photo library access.</string>
   <key>NSPhotoLibraryAddUsageDescription</key>
   <string>This app saves photos to your library.</string>
   ```

**Bare — Android:**
2. [ ] Agregar a `android/app/src/main/AndroidManifest.xml`:
   ```xml
   <uses-permission android:name="android.permission.CAMERA" />
   <uses-feature android:name="android.hardware.camera" android:required="false" />
   ```

---

#### Bridge: Push Notifications

**Expo managed:**
- ADVERTENCIA — incluir prominentemente en el output generado:

  > **IMPORTANTE:** `expo-notifications` requiere **EAS Build** para funcionar en producción.
  > NO funciona en Expo Go. Asegurate de usar `eas build` para distribuir tu app.
  > Referencia: https://docs.expo.dev/push-notifications/push-notifications-setup/

**Bare — iOS:**
1. [ ] Indicar explícitamente al usuario:
   ```
   ACCIÓN MANUAL REQUERIDA (iOS):
   Push Notifications en bare workflow requiere habilitar la capability
   "Push Notifications" manualmente en Xcode:
     1. Abrir ios/{NombreApp}.xcworkspace en Xcode
     2. Seleccionar el target → Signing & Capabilities
     3. Hacer clic en "+ Capability" → agregar "Push Notifications"
   Esta configuración NO puede automatizarse desde CLI.
   ```

**Bare — Android:**
2. [ ] Agregar a `android/app/src/main/AndroidManifest.xml`:
   ```xml
   <!-- Requerido para Android 13+ (API 33+). En API < 33 se muestra automáticamente. -->
   <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
   ```

---

#### Bridge: Secure Storage

**Expo managed:**
- `expo-secure-store` usa iOS Keychain y Android Keystore automáticamente
- No requiere configuración adicional de permisos

**Bare:**
1. [ ] `react-native-keychain` — en Android, configurar `encryptedSharedPreferences`:
   ```ts
   import * as Keychain from 'react-native-keychain';

   // Ejemplo de uso seguro en Android
   await Keychain.setGenericPassword('username', 'password', {
     storage: Keychain.STORAGE_TYPE.AES_GCM,  // encryptedSharedPreferences API ≥ 23
   });
   ```
   Agregar comentario en el archivo de ejemplo:
   ```
   // NOTA: minSdkVersion = 24 garantiza soporte de hardware Keystore en Android.
   // encryptedSharedPreferences está disponible desde API 23 — cubierto por el minSdkVersion del proyecto.
   ```
2. [ ] No requiere entradas en `AndroidManifest.xml` ni `Info.plist` para storage básico

---

### CHECKPOINT Phase 4
- [ ] **Cámara:**
  - Expo: plugin `"expo-camera"` presente en `app.json`
  - Bare iOS: `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription` en `Info.plist`
  - Bare Android: `CAMERA` permission y `camera` feature en `AndroidManifest.xml`
- [ ] **Push Notifications:**
  - Expo: ADVERTENCIA de Expo Go prominente incluida en el output
  - Bare iOS: instrucción de Xcode manual incluida en el output
  - Bare Android: `POST_NOTIFICATIONS` permission en `AndroidManifest.xml`
- [ ] **Secure Storage:**
  - Expo: sin configuración adicional (documentado)
  - Bare: ejemplo de `encryptedSharedPreferences` incluido con comentario de minSdkVersion
- [ ] Todos los bridges seleccionados procesados; bridges no seleccionados omitidos

### IF FAILS
```
bridge = push, workflow = expo, usuario no leyó la advertencia:
  → Repetir la advertencia de Expo Go en el resumen final (no solo en el archivo)
  → No bloquear — es una advertencia, no un error

bridge = push, workflow = bare, iOS:
  → Incluir instrucción de Xcode SIEMPRE — no hay alternativa automatizable
  → Documentar también en README.md bajo "Native Setup Required"

Info.plist no generado en Phase 3 (bare, sin bridges iniciales pero con bridge nuevo):
  → Generar Info.plist ahora con las entradas requeridas
  → No bloquear Phase 4 por ausencia de Info.plist previo
```

---

## Phase 5: README + .gitignore

### GATE IN
- Phases 3-4 completadas — proyecto generado en el directorio de destino

### MUST DO

1. Generar `README.md` en la raíz del proyecto con este contenido:

   ```markdown
   # {nombre}

   Proyecto mobile generado por King Framework `/mobile-scaffold`.

   ## Getting Started

   ### Prerequisitos
   [Expo managed]: Node.js ≥ 18, expo-cli (`npm install -g expo`)
   [Bare workflow]: Node.js ≥ 18, Xcode (iOS), Android Studio (Android)

   ### Instalación
   [Expo managed]: `npm install`
   [Bare workflow]: `npm install && cd ios && pod install`

   ### Ejecutar en simulador/emulador
   [Expo managed]: `npx expo start`
   [Bare workflow iOS]: `npx react-native run-ios`
   [Bare workflow Android]: `npx react-native run-android`

   ## Native Bridges configurados
   [Listar solo los bridges seleccionados con instrucción básica de uso]
   - Cámara: [indicar paquete usado y referencia al componente generado]
   - Push Notifications: [indicar paquete y advertencia de setup Firebase si aplica]
   - Secure Storage: [indicar paquete y referencia al ejemplo generado]

   ## Deployment
   Para configurar CI/CD y submission a App Store / Play Store:
   `/mobile-deploy --target both`
   ```

2. Generar `.gitignore` en la raíz del proyecto.

   Para React Native **Expo managed workflow**:
   ```gitignore
   # Dependencies
   node_modules/
   .npm

   # Expo
   .expo/
   dist/
   web-build/

   # Native (managed — no están en el repo, pero por si se hace eject)
   ios/
   android/

   # Mobile credentials — NUNCA commitear
   *.p8
   *.p12
   *.keystore
   *.jks
   google-services.json
   GoogleService-Info.plist
   AuthKey_*.p8

   # Environment
   .env
   .env.local
   .env.*.local

   # IDE
   .idea/
   .vscode/
   *.iml
   ```

   Para React Native **bare workflow**:
   ```gitignore
   # Dependencies
   node_modules/
   .npm

   # Native builds
   ios/build/
   android/app/build/
   android/.gradle/

   # Mobile credentials — NUNCA commitear
   *.p8
   *.p12
   *.keystore
   *.jks
   google-services.json
   GoogleService-Info.plist
   AuthKey_*.p8
   keystore.properties

   # Environment
   .env
   .env.local
   .env.*.local

   # IDE
   .idea/
   .vscode/
   *.iml
   ```

### CHECKPOINT
- [ ] README.md generado con sección "Getting Started" y comandos ejecutables
- [ ] README documenta cada bridge configurado con instrucción básica de uso
- [ ] .gitignore contiene las 7 entradas mobile-específicas (*.p8, *.p12, *.keystore, *.jks, google-services.json, GoogleService-Info.plist, AuthKey_*.p8)
- [ ] .gitignore contiene entradas de builds/deps (node_modules/ para Expo; ios/build/, android/app/build/ para bare)

### IF FAILS
- README.md no generado: es obligatorio — crearlo con el template mínimo (Getting Started + comandos)
- .gitignore ausente: es obligatorio — al menos las 7 entradas de credenciales mobile deben estar presentes
- Bridges no documentados en README: agregar sección minimal con nombre del paquete usado

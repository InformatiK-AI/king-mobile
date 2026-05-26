# mobile-scaffold — FLUTTER (Phases 3-4)

> Sub-archivo de `SKILL.md`. Cargado via PHASE ROUTER para Phases 3 y 4 cuando `framework = flutter`.
> NUNCA ejecutar directamente — siempre invocado desde `skills/mobile-scaffold/SKILL.md`.

---

## Phase 3: Flutter Scaffold

### GATE IN
- [ ] Phase 2 completada sin errores bloqueantes
- [ ] `framework = flutter` confirmado en configuración capturada
- [ ] Nombre, path, navigation, state y bridges resueltos

### MUST DO
> Todas las acciones son OBLIGATORIAS

1. [ ] **Resolver versiones actuales via Context7**:
   - Consultar Context7: versión latest stable de `flutter` y `dart`
   - Si Context7 no está disponible: usar baseline `Flutter 3.24.x` / `Dart 3.5.x` y notificar al usuario:
     ```
     Context7 no disponible. Usando versiones baseline: Flutter 3.24.x, Dart 3.5.x.
     Verificá las versiones actuales en https://flutter.dev/docs/development/tools/sdk/releases
     ```

2. [ ] **Generar `pubspec.yaml`**:

   Estructura base (siempre presente):
   ```yaml
   name: {nombre-kebab}
   description: A new Flutter application.
   publish_to: 'none'
   version: 0.1.0+1

   environment:
     sdk: ">=3.5.0 <4.0.0"

   dependencies:
     flutter:
       sdk: flutter
     go_router: ^14.0.0

   dev_dependencies:
     flutter:
       sdk: flutter
     flutter_test:
       sdk: flutter
     flutter_lints: ^4.0.0

   flutter:
     uses-material-design: true
   ```

   > Compatibilidad framework+state validada en DISCOVERY.md Phase 1 — ver tabla de compatibilidad.

   Agregar según `--state`:

   | `--state` | `dependencies` | `dev_dependencies` |
   |-----------|---------------|-------------------|
   | `riverpod` (default) | flutter_riverpod ^2.6.0, riverpod_annotation ^2.6.0 | build_runner ^2.4.0, riverpod_generator ^2.6.0 |
   | `provider` | provider ^6.1.0 | — |
   | `bloc` | flutter_bloc ^8.1.0, equatable ^2.0.0 | — |
   | `getx` | get ^4.6.0 | — |

   Agregar según `--bridges`:

   | `--bridges` | `dependencies` |
   |-------------|----------------|
   | cámara | image_picker ^1.1.0 |
   | push | firebase_messaging ^15.1.0, flutter_local_notifications ^17.2.0 |
   | storage | flutter_secure_storage ^9.2.0 |

3. [ ] **Generar estructura de directorios y archivos**:

   ```
   {nombre}/
   ├── pubspec.yaml
   ├── analysis_options.yaml
   ├── lib/
   │   ├── main.dart
   │   ├── app/
   │   │   ├── app.dart
   │   │   └── router.dart
   │   ├── features/
   │   │   └── home/
   │   │       └── presentation/
   │   │           └── home_screen.dart
   │   └── core/
   │       └── providers/
   │           └── app_providers.dart
   ├── test/
   │   └── widget_test.dart
   └── integration_test/
       └── app_test.dart
   ```

   Contenido de cada archivo:

   **`lib/main.dart`** — ProviderScope solo si `state = riverpod`; sin wrapper para otros:
   ```dart
   // Si state = riverpod:
   import 'package:flutter/material.dart';
   import 'package:flutter_riverpod/flutter_riverpod.dart';
   import 'app/app.dart';

   void main() {
     runApp(const ProviderScope(child: MyApp()));
   }

   // Si state != riverpod:
   import 'package:flutter/material.dart';
   import 'app/app.dart';

   void main() {
     runApp(const MyApp());
   }
   ```

   **`lib/app/app.dart`** — MaterialApp.router con go_router:
   ```dart
   import 'package:flutter/material.dart';
   import 'router.dart';

   class MyApp extends StatelessWidget {
     const MyApp({super.key});

     @override
     Widget build(BuildContext context) {
       return MaterialApp.router(
         title: '{nombre-kebab}',
         theme: ThemeData(
           colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
           useMaterial3: true,
         ),
         routerConfig: appRouter,
       );
     }
   }
   ```

   **`lib/app/router.dart`** — GoRouter con ruta home declarativa:
   ```dart
   import 'package:go_router/go_router.dart';
   import '../features/home/presentation/home_screen.dart';

   final appRouter = GoRouter(
     initialLocation: '/',
     routes: [
       GoRoute(
         path: '/',
         builder: (context, state) => const HomeScreen(),
       ),
     ],
   );
   ```

   **`lib/features/home/presentation/home_screen.dart`** — Widget básico:
   ```dart
   import 'package:flutter/material.dart';

   class HomeScreen extends StatelessWidget {
     const HomeScreen({super.key});

     @override
     Widget build(BuildContext context) {
       return Scaffold(
         appBar: AppBar(
           title: const Text('Home'),
         ),
         body: const Center(
           child: Text('Welcome to {nombre-kebab}'),
         ),
       );
     }
   }
   ```

   **`lib/core/providers/app_providers.dart`** — Según `--state`:

   Si `state = riverpod`:
   ```dart
   // Run: flutter pub run build_runner build (o watch para desarrollo)
   import 'package:flutter_riverpod/flutter_riverpod.dart';

   // Ejemplo: StateProvider simple
   final counterProvider = StateProvider<int>((ref) => 0);

   // Ejemplo: NotifierProvider para lógica más compleja
   class CounterNotifier extends Notifier<int> {
     @override
     int build() => 0;

     void increment() => state++;
   }

   final counterNotifierProvider = NotifierProvider<CounterNotifier, int>(
     CounterNotifier.new,
   );
   ```

   Si `state = provider`:
   ```dart
   import 'package:flutter/foundation.dart';

   class CounterModel extends ChangeNotifier {
     int _count = 0;
     int get count => _count;

     void increment() {
       _count++;
       notifyListeners();
     }
   }
   ```

   Si `state = bloc`:
   ```dart
   import 'package:flutter_bloc/flutter_bloc.dart';
   import 'package:equatable/equatable.dart';

   // Events
   abstract class CounterEvent extends Equatable {
     const CounterEvent();
     @override
     List<Object?> get props => [];
   }

   class IncrementCounter extends CounterEvent {}

   // State
   class CounterState extends Equatable {
     final int count;
     const CounterState({this.count = 0});
     @override
     List<Object?> get props => [count];
   }

   // Bloc
   class CounterBloc extends Bloc<CounterEvent, CounterState> {
     CounterBloc() : super(const CounterState()) {
       on<IncrementCounter>((event, emit) {
         emit(CounterState(count: state.count + 1));
       });
     }
   }
   ```

   Si `state = getx`:
   ```dart
   import 'package:get/get.dart';

   class CounterController extends GetxController {
     final count = 0.obs;

     void increment() => count++;
   }
   ```

   Si `bridges` incluye `flutter_secure_storage`, agregar al final del archivo independientemente del state:
   ```dart
   import 'package:flutter_secure_storage/flutter_secure_storage.dart';

   final secureStorage = FlutterSecureStorage(
     aOptions: AndroidOptions(encryptedSharedPreferences: true),
   );
   ```

   **`test/widget_test.dart`** — Test básico del widget home:
   ```dart
   import 'package:flutter/material.dart';
   import 'package:flutter_test/flutter_test.dart';
   import 'package:{nombre-kebab}/main.dart';

   void main() {
     testWidgets('HomeScreen renders correctly', (WidgetTester tester) async {
       await tester.pumpWidget(const MyApp());
       expect(find.text('Home'), findsOneWidget);
     });
   }
   ```

   **`integration_test/app_test.dart`** — Test de integración placeholder:
   ```dart
   import 'package:flutter_test/flutter_test.dart';
   import 'package:integration_test/integration_test.dart';
   import 'package:{nombre-kebab}/main.dart' as app;

   void main() {
     IntegrationTestWidgetsFlutterBinding.ensureInitialized();

     testWidgets('App launches successfully', (WidgetTester tester) async {
       app.main();
       await tester.pumpAndSettle();
       expect(find.byType(MaterialApp), findsOneWidget);
     });
   }
   ```

   **`analysis_options.yaml`**:
   ```yaml
   include: package:flutter_lints/flutter.yaml

   linter:
     rules:
       avoid_print: true
       prefer_const_constructors: true
   ```

### CHECKPOINT Phase 3
- [ ] `pubspec.yaml` generado con `sdk: ">=3.5.0 <4.0.0"`
- [ ] `go_router` presente en `dependencies`
- [ ] State management correcto según `--state` en `dependencies`
- [ ] `build_runner` y `riverpod_generator` en `dev_dependencies` solo si `state = riverpod`
- [ ] Bridges seleccionados agregados a `dependencies`
- [ ] Estructura de directorios generada: `lib/app/`, `lib/features/home/presentation/`, `lib/core/providers/`
- [ ] `main.dart` usa `ProviderScope` solo si `state = riverpod`
- [ ] `analysis_options.yaml` generado con `include: package:flutter_lints/flutter.yaml`

### IF FAILS
```
Context7 no disponible:
  → Usar versiones baseline: Flutter 3.24.x, Dart 3.5.x
  → Notificar al usuario con URL de referencia: https://flutter.dev/docs/development/tools/sdk/releases
  → Continuar con el scaffold — no es un error bloqueante

Versión de dependencia incierta:
  → Usar las versiones especificadas en esta spec como baseline
  → Documentar en README.md: "Verificar versiones actuales en pub.dev antes de ejecutar flutter pub get"

State no reconocido:
  → Usar riverpod como fallback
  → Notificar al usuario: "State `{state}` no reconocido. Usando riverpod por defecto."
```

---

## Phase 4: Native Bridges Configuration

### GATE IN
- [ ] Phase 3 completada
- [ ] Al menos un bridge seleccionado en `bridges`
- [ ] Si `bridges = []`: saltar Phase 4 completamente — no hay configuración nativa que hacer

### MUST DO
> Ejecutar solo para cada bridge seleccionado

#### Bridge: Cámara (`image_picker`)

**iOS** — Agregar a `ios/Runner/Info.plist`:
```xml
<key>NSCameraUsageDescription</key>
<string>Required to take photos.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Required to access photos.</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Required to save photos to your library.</string>
```

**Android** — `android/app/src/main/AndroidManifest.xml`:
- `image_picker` básico no requiere permiso explícito en AndroidManifest para READ_EXTERNAL_STORAGE en Android 13+
- Si se usa acceso directo a la cámara (no image_picker): agregar dentro de `<manifest>`:
  ```xml
  <uses-permission android:name="android.permission.CAMERA" />
  ```
- Documentar en README.md: "Para Android 12 o inferior, agregar READ_EXTERNAL_STORAGE si se accede a galería."

---

#### Bridge: Push Notifications (`firebase_messaging`)

> REQUIERE configuración de Firebase. Generar placeholders con instrucciones claras.

1. [ ] Generar `android/app/google-services.json.placeholder` con contenido:
   ```
   PLACEHOLDER — Reemplazar con el archivo google-services.json real descargado desde Firebase Console.
   Pasos:
     1. Ir a https://console.firebase.google.com
     2. Crear o seleccionar tu proyecto
     3. Agregar app Android con package name: com.example.{nombre-kebab} (ajustar según tu app)
     4. Descargar google-services.json y reemplazar este archivo
     5. NUNCA commitear google-services.json — verificar que esté en .gitignore
   ```

2. [ ] Generar `ios/Runner/GoogleService-Info.plist.placeholder` con contenido:
   ```
   PLACEHOLDER — Reemplazar con el archivo GoogleService-Info.plist real descargado desde Firebase Console.
   Pasos:
     1. Ir a https://console.firebase.google.com
     2. Crear o seleccionar tu proyecto
     3. Agregar app iOS con bundle ID de tu app (ver ios/Runner.xcodeproj)
     4. Descargar GoogleService-Info.plist y reemplazar este archivo
     5. NUNCA commitear GoogleService-Info.plist — verificar que esté en .gitignore
   ```

3. [ ] Agregar a `android/app/build.gradle` (al final del archivo):
   ```groovy
   apply plugin: 'com.google.gms.google-services'
   ```

4. [ ] Agregar a `android/build.gradle` (en `dependencies` del buildscript):
   ```groovy
   classpath 'com.google.gms:google-services:4.4.0'
   ```

5. [ ] Instrucción para iOS (documentar en README.md):
   ```
   iOS Push Notifications:
     1. Abrir ios/Runner.xcworkspace en Xcode
     2. Seleccionar el target Runner
     3. En "Signing & Capabilities" → "+ Capability" → agregar "Push Notifications"
     4. Agregar también "Background Modes" → activar "Remote notifications"
   ```

6. [ ] Emitir ADVERTENCIA explícita al usuario:
   ```
   ADVERTENCIA: google-services.json y GoogleService-Info.plist contienen credenciales de Firebase.
   Estos archivos NUNCA deben commitearse. Verificar que estén incluidos en .gitignore.
   Los archivos .placeholder generados son seguros — reemplazarlos con los reales antes de compilar.
   ```

---

#### Bridge: Secure Storage (`flutter_secure_storage`)

**Android** — Verificar `minSdkVersion` en `android/app/build.gradle`:
- `flutter_secure_storage` con `encryptedSharedPreferences: true` requiere `minSdkVersion >= 23`
- Si el proyecto fue generado con valores recomendados (minSdk 24): ya compatible, no requiere cambio
- Si `minSdkVersion < 23`: actualizar a 23 y notificar al usuario:
  ```
  flutter_secure_storage requiere minSdkVersion >= 23 para encryptedSharedPreferences.
  Actualizado android/app/build.gradle: minSdkVersion 23
  ```

**iOS** — Usa iOS Keychain automáticamente. No requiere configuración adicional en Info.plist ni Podfile.

**Uso de ejemplo** — Ya generado en Phase 3 dentro de `lib/core/providers/app_providers.dart`:
```dart
final secureStorage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
);
```

---

### CHECKPOINT Phase 4
- [ ] Cámara seleccionada → permisos en `ios/Runner/Info.plist` (NSCameraUsageDescription, NSPhotoLibraryUsageDescription)
- [ ] Cámara seleccionada → nota en README sobre Android y `image_picker`
- [ ] Push seleccionado → `android/app/google-services.json.placeholder` generado con instrucciones
- [ ] Push seleccionado → `ios/Runner/GoogleService-Info.plist.placeholder` generado con instrucciones
- [ ] Push seleccionado → `apply plugin: 'com.google.gms.google-services'` en `android/app/build.gradle`
- [ ] Push seleccionado → ADVERTENCIA de `.gitignore` emitida al usuario
- [ ] Secure Storage seleccionado → `minSdkVersion >= 23` verificado/actualizado en `android/app/build.gradle`
- [ ] Secure Storage seleccionado → ejemplo de uso presente en `lib/core/providers/app_providers.dart`

### IF FAILS
```
Bridge push seleccionado sin cuenta de Firebase:
  → Generar placeholders igualmente (son seguros — no contienen credenciales reales)
  → Guiar al usuario a Firebase Console: https://console.firebase.google.com
  → No bloquear el scaffold — el proyecto compila sin los archivos reales hasta que se prueba push

Permisos de iOS no determinables (Info.plist no generado por flutter create aún):
  → Listar los permisos necesarios con instrucción de agregar manualmente:
    ```
    Agregar manualmente a ios/Runner/Info.plist después de ejecutar `flutter create`:
    NSCameraUsageDescription → "Required to take photos."
    NSPhotoLibraryUsageDescription → "Required to access photos."
    ```
  → Documentar en README.md

android/app/build.gradle no accesible:
  → Documentar el cambio requerido en README.md con instrucción explícita:
    ```
    En android/app/build.gradle, verificar que minSdkVersion >= 23 (flutter_secure_storage).
    ```
  → Continuar — no es bloqueante para la generación del scaffold
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
   Flutter SDK ≥ 3.24, Dart ≥ 3.5

   ### Instalación
   `flutter pub get`
   [Solo si state=riverpod]: `flutter pub run build_runner build`

   ### Ejecutar en simulador/emulador
   iOS: `flutter run -d ios`
   Android: `flutter run -d android`

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

   Para **Flutter**:
   ```gitignore
   # Flutter
   .dart_tool/
   .flutter-plugins
   .flutter-plugins-dependencies
   build/
   *.g.dart
   *.freezed.dart

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

   # IDE
   .idea/
   .vscode/
   ```

### CHECKPOINT
- [ ] README.md generado con sección "Getting Started" y comandos ejecutables
- [ ] README documenta cada bridge configurado con instrucción básica de uso
- [ ] .gitignore contiene las 7 entradas mobile-específicas (*.p8, *.p12, *.keystore, *.jks, google-services.json, GoogleService-Info.plist, AuthKey_*.p8)
- [ ] .gitignore contiene entradas de builds/deps (build/, .dart_tool/ para Flutter)

### IF FAILS
- README.md no generado: es obligatorio — crearlo con el template mínimo (Getting Started + comandos)
- .gitignore ausente: es obligatorio — al menos las 7 entradas de credenciales mobile deben estar presentes
- Bridges no documentados en README: agregar sección minimal con nombre del paquete usado

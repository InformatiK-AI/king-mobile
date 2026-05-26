# mobile-scaffold — DISCOVERY (Phases 1-2)

> Sub-archivo de `SKILL.md`. Cargado via PHASE ROUTER para Phases 1 y 2.
> NUNCA ejecutar directamente — siempre invocado desde `skills/mobile-scaffold/SKILL.md`.

---

## Phase 1: Validate Environment

### GATE IN
- [ ] Parámetros `--framework`, `--navigation`, `--state`, `--bridges` recibidos O modo interactivo disponible
- [ ] Directorio de destino (`--name` o pregunta pendiente) identificado

### MUST DO
> Todas las acciones son OBLIGATORIAS

1. [ ] **Detectar CLI del framework solicitado**:
   ```
   --framework react-native (Expo managed, default):
     → verificar: npx expo --version
     → versión mínima requerida: Expo SDK 51+

   --framework react-native --workflow bare:
     → verificar: npx react-native --version
     → versión mínima requerida: RN 0.73+

   --framework flutter:
     → verificar: flutter --version
     → Dart ≥ 3.5 requerido (incluido en Flutter SDK)
     → parsear línea "Dart version X.Y.Z" del output
   ```

2. [ ] **Detectar Node.js ≥ 18** (requerido para React Native):
   - Ejecutar: `node --version`
   - Parsear versión major; si < 18 → BLOQUEANTE
   - Omitir este check si `--framework flutter`

3. [ ] **Verificar directorio de destino**:
   - Si no existe → OK, continuar
   - Si existe y está vacío → OK, continuar
   - Si existe y tiene contenido → BLOQUEANTE condicional:
     ```
     Preguntar: "El directorio `{nombre}` ya existe y no está vacío. ¿Sobreescribir? (s/N)"
     - Usuario confirma (s) → proceder con advertencia documentada
     - Usuario no confirma (N o Enter) → DETENER con mensaje claro (no es error)
     ```

4. [ ] **Validar combinación `--framework` + `--state`** usando la tabla de compatibilidad:

   | Framework | States válidos | States INCOMPATIBLES |
   |-----------|---------------|----------------------|
   | react-native (expo + bare) | zustand, redux, jotai, context | riverpod, provider, bloc, getx |
   | flutter | riverpod, provider, bloc, getx | zustand, redux, jotai, context |

   ```
   Si --state fue especificado como argumento CLI:
     → Validar compatibilidad ahora (tabla inline arriba)
     → Si incompatible: BLOQUEANTE con mensaje explicativo (ver abajo)
   Si --state NO fue especificado:
     → Omitir validación aquí
     → Phase 2 asignará default (zustand para RN, riverpod para Flutter)
     → El default siempre es compatible — no requiere re-validación
   ```
   - Si `--state` especificado y es incompatible → BLOQUEANTE con mensaje explicativo:
     ```
     Error: `{state}` es incompatible con `{framework}`.
     {state} es una librería {JavaScript|Dart} — no puede usarse con {framework}.
     Equivalente correcto: {sugerencia}

     Sugerencias de equivalencia:
       zustand/jotai en Flutter → riverpod, provider, bloc o getx
       riverpod/provider en React Native → zustand, redux, jotai o context
       redux en Flutter → riverpod, provider, bloc o getx
       bloc en React Native → zustand, redux, jotai o context
     ```

### CHECKPOINT
> Verificar antes de continuar a Phase 2

- [ ] CLI del framework detectada y versión compatible
- [ ] Node.js ≥ 18 confirmado (solo para React Native; omitido para Flutter)
- [ ] Directorio destino disponible (no existe, vacío, o confirmación de overwrite obtenida)
- [ ] Combinación framework+state válida (si `--state` fue especificado)

### IF FAILS
```
CLI no encontrada:
  → Error explícito con nombre de herramienta y URL de instalación:
    - Expo:    instalar con `npm install -g expo-cli` o usar `npx create-expo-app`
               URL: https://docs.expo.dev/get-started/installation/
    - Flutter: URL: https://flutter.dev/docs/get-started/install
  → No continuar — la CLI es un prerequisito no negociable

Node.js < 18 (solo React Native):
  → Informar versión detectada vs. requerida
  → Sugerir: `nvm install 18` o descargar desde https://nodejs.org
  → No continuar

Combinación framework+state incompatible:
  → Error con tabla de equivalencia (ver MUST DO paso 4)
  → Sugerir el equivalente correcto según el framework elegido
  → No continuar hasta que el usuario corrija el parámetro

Directorio existente sin confirmación de overwrite:
  → DETENER silenciosamente (no es un error, es una decisión del usuario)
  → Mensaje: "Operación cancelada. Elegí un nombre diferente con --name o confirmá el overwrite."
```

---

## Phase 2: Framework Selection + Configuration

### GATE IN
- [ ] Phase 1 completada sin errores bloqueantes

### MUST DO
> Todas las acciones son OBLIGATORIAS

1. [ ] **Resolver `--framework`** si no fue especificado:
   - Usar AskUserQuestion con opciones:
     ```
     ¿Qué framework querés usar?
     1. React Native con Expo (recomendado para proyectos nuevos)
     2. React Native bare workflow (para módulos nativos custom)
     3. Flutter
     ```

2. [ ] **Resolver `--workflow`** para React Native si no fue especificado:
   - Default = `expo` (más simple para proyectos nuevos, sin configuración nativa inicial)
   - Solo aplicable si `--framework react-native`; omitir para Flutter

3. [ ] **Resolver `--navigation`** si no fue especificado:
   - Default = `stack`
   - React Native: Stack Navigator via React Navigation
   - Flutter: go_router con rutas stack

4. [ ] **Resolver `--state`** si no fue especificado:
   - Default = `zustand` para React Native
   - Default = `riverpod` para Flutter
   - Si el usuario lo especifica ahora: validar compatibilidad inline usando la tabla de Phase 1 paso 4

5. [ ] **Resolver `--bridges`** si no fue especificado:
   - Usar AskUserQuestion con multiSelect:
     ```
     ¿Qué bridges nativos necesita el proyecto? (selección múltiple)
     - Cámara              (expo-camera / react-native-vision-camera / image_picker)
     - Push Notifications  (expo-notifications / @notifee/react-native / firebase_messaging)
     - Secure Storage      (expo-secure-store / react-native-keychain / flutter_secure_storage)
     - Ninguno             (proyecto base sin bridges)
     ```
   - Si el usuario selecciona "Ninguno": registrar `bridges: []`

6. [ ] **Capturar y normalizar `--name`**:
   - Si no fue especificado: preguntar "¿Cuál es el nombre del proyecto?"
   - Normalizar a kebab-case:
     ```
     "My App"     → "my-app"
     "MiApp2025"  → "mi-app-2025"
     "mi app!"    → "mi-app"
     ```
   - Si se normalizó: notificar al usuario: "Nombre normalizado a: `{nombre-kebab}`"
   - Si el nombre resultante está vacío (solo caracteres especiales): solicitar un nombre válido

7. [ ] **Determinar path de destino**:
   - Path = `{directorio-actual-del-usuario}/{nombre-kebab}`
   - Documentar el path absoluto resuelto en la configuración

8. [ ] **Emitir resumen de configuración confirmada** antes de proceder:
   ```
   Configuracion del proyecto:
   - Nombre:     {nombre-kebab}
   - Path:       {path-absoluto}
   - Framework:  {framework} ({workflow si aplica})
   - Navigation: {navigation}
   - State:      {state}
   - Bridges:    {lista o "ninguno"}
   ```

### CHECKPOINT
> Verificar antes de continuar a Phase 3

- [ ] `framework` capturado y resuelto
- [ ] `workflow` capturado (React Native) o marcado como N/A (Flutter)
- [ ] `navigation` capturado con default aplicado si necesario
- [ ] `state` capturado con default aplicado si necesario
- [ ] `bridges` capturado (puede ser lista vacía)
- [ ] `name` capturado, normalizado y validado
- [ ] `path` de destino absoluto calculado
- [ ] Resumen de configuración mostrado al usuario
- [ ] Bifurcación determinada: react-native → cargar RN.md | flutter → cargar FLUTTER.md

### IF FAILS
```
Usuario no responde a pregunta de framework:
  → Usar default: react-native con Expo
  → Notificar: "Usando React Native con Expo por defecto."

Usuario no responde a pregunta de bridges:
  → Usar default: bridges: [] (proyecto base sin bridges)
  → Notificar: "Proyecto base sin bridges nativos. Podés agregarlos después."

Nombre del proyecto no normalizable (solo caracteres especiales, ej: "!@#$%"):
  → No usar default ni inventar nombre
  → Solicitar explícitamente: "El nombre `{entrada}` no puede normalizarse.
     Ingresá un nombre válido (letras, números, guiones)."
  → No continuar hasta obtener un nombre válido

State incompatible detectado en esta fase (usuario lo especificó en el diálogo):
  → Validar compatibilidad inline usando la tabla de Phase 1 paso 4
  → Aplicar mismo mensaje de error que Phase 1 paso 4 (IF FAILS)
```

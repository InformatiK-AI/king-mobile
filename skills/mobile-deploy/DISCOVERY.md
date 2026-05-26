# mobile-deploy — DISCOVERY (Phases 1-2)

> Sub-archivo de `SKILL.md`. Cargado via PHASE ROUTER para Phases 1 y 2.
> NUNCA ejecutar directamente — siempre invocado desde `skills/mobile-deploy/SKILL.md`.

---

## Phase 1: Project Detection

### GATE IN
- [ ] Ejecutarse desde el directorio raíz del proyecto mobile del usuario

### MUST DO
> Todas las acciones son OBLIGATORIAS

1. [ ] **Detectar tipo de proyecto**:
   ```
   Si existe package.json con dependencia "expo":
     → tipo = expo-managed

   Si existe package.json con "react-native" pero SIN "expo":
     → tipo = rn-bare

   Si existe pubspec.yaml:
     → tipo = flutter

   Si ninguno de los anteriores:
     → BLOQUEANTE: "No se detectó un proyecto mobile. Ejecutá `/mobile-scaffold` primero
        o navegá al directorio raíz del proyecto."
   ```

2. [ ] **Detectar bundle ID / package name según tipo**:
   ```
   expo-managed:
     → leer app.json → expo.ios.bundleIdentifier (iOS)
     → leer app.json → expo.android.package (Android)

   rn-bare:
     → leer android/app/build.gradle → applicationId (Android)
     → leer ios/*.xcodeproj o ios/*/Info.plist → CFBundleIdentifier (iOS)

   flutter:
     → leer pubspec.yaml → name (nombre del paquete)
     → leer android/app/build.gradle → applicationId (Android)
   ```
   - Si no se encuentra en ninguno de los paths anteriores: solicitar al usuario con `--bundle-id`
   - NUNCA asumir un bundle ID — solicitar explícitamente antes de continuar

3. [ ] **Resolver `--target`** si no fue especificado:
   - Usar AskUserQuestion con opciones:
     ```
     ¿Qué plataforma querés configurar para CI/CD?
     1. iOS (App Store)
     2. Android (Google Play)
     3. Ambas plataformas (recomendado)
     ```

4. [ ] **Verificar remote de GitHub**:
   - Ejecutar: `git remote -v`
   - Debe existir al menos un remote con `github.com` en la URL
   - Si no existe → BLOQUEANTE:
     ```
     El repositorio necesita un remote de GitHub para configurar GitHub Actions.
     Agregalo con:
       git remote add origin https://github.com/{usuario}/{repo}.git
     ```

5. [ ] **Verificar disponibilidad de `.github/workflows/`**:
   - Si el directorio no existe: crearlo
   - Si existe: verificar que se puede escribir en él

### CHECKPOINT
> Verificar antes de continuar a Phase 2

- [ ] Tipo de proyecto detectado: `expo-managed` | `rn-bare` | `flutter`
- [ ] Bundle ID / package name detectados o solicitados explícitamente al usuario
- [ ] `--target` confirmado: `ios` | `android` | `both`
- [ ] Remote de GitHub verificado y presente
- [ ] Directorio `.github/workflows/` disponible (creado si no existía)

### IF FAILS
```
Proyecto no detectado (no package.json con deps mobile ni pubspec.yaml):
  → Instrucción: "No se detectó un proyecto mobile en el directorio actual.
     Opciones:
       1. Ejecutá `/mobile-scaffold` para crear un proyecto nuevo.
       2. Navegá al directorio raíz del proyecto con `cd {ruta-del-proyecto}`."
  → No continuar

Bundle ID no encontrado en los paths esperados:
  → Solicitar explícitamente: "No se encontró el bundle ID en los archivos del proyecto.
     Especificalo con el flag --bundle-id:
       expo-managed: actualizá app.json → expo.ios.bundleIdentifier / expo.android.package
       rn-bare: verificá android/app/build.gradle → applicationId
       flutter: verificá android/app/build.gradle → applicationId"
  → No asumir ni inventar un bundle ID

Remote GitHub ausente:
  → Instrucción exacta:
     git remote add origin https://github.com/{usuario}/{repo}.git
  → Reemplazar {usuario}/{repo} con los valores reales del proyecto
  → No continuar hasta que el remote esté configurado
```

---

## Phase 2: Secrets Validation

### GATE IN
- [ ] Phase 1 completada — tipo, bundle ID, target y remote confirmados

### MUST DO
> Todas las acciones son OBLIGATORIAS

1. [ ] **Construir lista de secrets requeridos** según `--target` y tipo de proyecto:

   **Para iOS (`--target ios` | `--target both`) — todos los tipos:**
   | Secret | Descripción |
   |--------|-------------|
   | `APP_STORE_CONNECT_KEY_ID` | Key ID de App Store Connect API (10 caracteres) |
   | `APP_STORE_CONNECT_ISSUER_ID` | Issuer ID (UUID) |
   | `APP_STORE_CONNECT_KEY_CONTENT` | Contenido del .p8 en base64 |

   **Para iOS — solo `rn-bare` y `flutter` (Fastlane signing):**
   | Secret | Descripción |
   |--------|-------------|
   | `DISTRIBUTION_CERT` | Certificado .p12 en base64 |
   | `DIST_CERT_PASSWORD` | Password del certificado |
   | `PROVISIONING_PROFILE` | Provisioning profile en base64 |
   | `KEYCHAIN_PASSWORD` | Password para el keychain de build en macOS (no dejar vacío) |

   **Para Android (`--target android` | `--target both`) — todos los tipos:**
   | Secret | Descripción |
   |--------|-------------|
   | `KEYSTORE_BASE64` | Keystore en base64 |
   | `KEYSTORE_PASSWORD` | Password del keystore |
   | `KEY_ALIAS` | Alias de la key |
   | `KEY_PASSWORD` | Password de la key |
   | `GOOGLE_PLAY_JSON_KEY` | Service account JSON completo |

   **Solo para `expo-managed`:**
   | Secret | Descripción |
   |--------|-------------|
   | `EXPO_TOKEN` | Token de autenticación de Expo (eas.io) |

2. [ ] **Verificar estado de cada secret** con `gh secret list --repo {remote}`:
   - Para cada secret en la lista construida en paso 1: verificar si aparece en el output de `gh`
   - Si `gh` no puede autenticarse (`gh auth status` falla):
     → Advertir al usuario y mostrar la lista de secrets a configurar manualmente
     → No bloquear el skill — continuar con estado "no verificable"
   - Si un secret falta → BLOQUEANTE: mostrar tabla de secrets faltantes con instrucción:
     ```bash
     # Para configurar cada secret faltante:
     gh secret set NOMBRE_DEL_SECRET --repo {usuario}/{repo}
     # o via GitHub UI: Settings → Secrets and variables → Actions → New repository secret
     ```

3. [ ] **Verificar fecha de expiración del certificado iOS** (solo si `DISTRIBUTION_CERT` está configurado y `--target ios` | `--target both`):
   - Si `openssl` está disponible:
     ```bash
     # Decodificar base64 y extraer fecha de expiración:
     echo "$DISTRIBUTION_CERT" | base64 -d | openssl x509 -noout -enddate
     ```
   - Si vence en < 30 días → WARNING:
     ```
     ADVERTENCIA: El certificado de distribución vence en {N} días ({fecha}).
     Renovalo en https://developer.apple.com/account/resources/certificates/list
     antes de que expire para evitar interrupciones en el CI.
     ```
   - Si ya venció (fecha < hoy) → BLOQUEANTE:
     ```
     BLOQUEANTE: El certificado de distribución iOS venció el {fecha}.
     Pasos para renovar:
       1. Ir a https://developer.apple.com/account/resources/certificates/list
       2. Revocar el certificado vencido
       3. Crear un nuevo Distribution Certificate
       4. Exportar como .p12 y convertir a base64:
            base64 -i Certificates.p12 | tr -d '\n'
       5. Actualizar el secret DISTRIBUTION_CERT en el repositorio GitHub
     No se puede continuar sin un certificado válido.
     ```

4. [ ] **Informar estado de todos los secrets**:
   ```
   Secrets requeridos para {target} ({tipo}):

   ✓ APP_STORE_CONNECT_KEY_ID — configurado
   ✓ APP_STORE_CONNECT_ISSUER_ID — configurado
   ✗ APP_STORE_CONNECT_KEY_CONTENT — FALTANTE
   ...
   ```
   - Mostrar la tabla completa aunque todos estén configurados (confirma el estado al usuario)

### CHECKPOINT
> Verificar antes de continuar a Phase 3 o 4

- [ ] Lista de secrets requeridos construida según `--target` y tipo de proyecto
- [ ] Estado de cada secret verificado (configurado / faltante / no verificable)
- [ ] Todos los secrets requeridos están configurados OR lista de faltantes emitida con instrucciones exactas
- [ ] Certificado iOS no vencido (si `--target ios` | `--target both`)

### IF FAILS
```
gh CLI no autenticado:
  → Mostrar la lista de secrets requeridos para configuración manual
  → Advertir: "No se pudo verificar el estado de los secrets (gh CLI no autenticado).
     Verificá y configurá manualmente antes de continuar:
       gh auth login
     o configurá los secrets desde GitHub UI."
  → No bloquear — continuar con advertencia documentada

Secret faltante:
  → Mostrar instrucción exacta para ese secret:
      gh secret set {NOMBRE_SECRET} --repo {usuario}/{repo}
  → Esperar que el usuario configure el secret antes de continuar
  → No asumir que el secret se configurará después — es BLOQUEANTE

Certificado iOS vencido:
  → BLOQUEANTE — no hay workaround seguro
  → Proveer instrucciones completas de renovación (ver MUST DO paso 3)
  → No continuar hasta que el certificado sea renovado y el secret actualizado

--target no respondido por el usuario:
  → Usar default `both` y notificar: "Configurando deployment para iOS y Android (default). Usá `--target ios` o `--target android` para cambiar."

Directorio `.github/workflows/` no escribible:
  → BLOQUEANTE: verificar permisos del repositorio antes de continuar
```

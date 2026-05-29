---
name: mobile-app-store-submit
version: 1.0
api_version: 1.0.0
description: "Automatiza la submisión a App Store (iOS) y Play Store (Android) con EAS Submit: privacy manifest iOS 17+, ASO metadata, screenshots guide, y rejection-reason checker pre-submisión que bloquea credenciales de test y permisos sin NSUsage strings. Usar cuando: subir app a tiendas, automatizar submit, generar privacy manifest, preparar ASO metadata, configurar CI de store submit."
---

# /mobile-app-store-submit — Submisión Automática a App Store y Play Store

Automatiza la submisión a App Store (iOS) y Play Store (Android) con EAS Submit para proyectos configurados con `/mobile-deploy`. Incluye rejection-reason checker pre-submisión, generación de privacy manifest iOS 17+, ASO metadata por plataforma, y CI workflow de submit en merge a `main`.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Framework mobile del proyecto | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones, permisos, bundle IDs | Yes | framework |
| `knowledge/_inject/mobile-production-patterns.md` | Patrones de CI/CD mobile, EAS | Optional | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- `eas.json` no existe en la raíz del proyecto — requiere `/mobile-deploy` configurado previamente
- EAS CLI no está instalado (`eas --version` falla) y no se puede instalar en contexto actual
- No existe `app.json` o `app.config.js` con `bundleIdentifier` (iOS) y `package` (Android) configurados
- El proyecto no tiene builds disponibles en EAS (sin `--latest` accesible)

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA ejecutar `eas submit` sin pasar primero el rejection checker completo
- NUNCA hardcodear Apple ID, contraseñas, o App Store Connect API keys en los archivos generados
- NUNCA generar `ios/PrivacyInfo.xcprivacy` con `NSPrivacyAccessedAPITypes` vacío si hay dependencias que lo requieren
- NUNCA omitir el registro en `.king/store/submit-history.json` tras cada submisión
- NUNCA incluir test credentials ("test@", "password123") ni sandbox keys en builds de producción

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- Privacy manifest: `ios/PrivacyInfo.xcprivacy`
- ASO metadata iOS: `store/ios/metadata.json`
- ASO metadata Android: `store/android/metadata.json`
- Screenshots guide: `store/screenshots/README.md`
- CI workflow: `.github/workflows/store-submit.yml`
- Pre-submit checker: `scripts/pre-submit-check.ts`
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
PHASE 0           PHASE 1              PHASE 2           PHASE 3         PHASE 4         PHASE 5
SESSION        →  VERIFY EAS        →  REJECTION      →  PRIVACY      →  ASO          →  CI
(registry)        CONFIGURED           CHECKER           MANIFEST         METADATA         WORKFLOW
                  (eas.json existe,     (6 checks:        (iOS 17+,        (ios +           (.github/
                  app.json válido)      4 bloqueantes     NSPrivacy        android,         workflows/
                                        + 2 avisos)       por deps)        limits)          store-submit)
                                                                                              ↓
                                                                                           PHASE 5
                                                                                           → SUBMIT
                                                                                           (eas submit
                                                                                            ambas plat.)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Validación de constraints de plataforma (privacy manifest, permisos iOS/Android)
- **@devops** → CI workflow de submit automático (`.github/workflows/store-submit.yml`)
- **@security** → Escalación si se detectan credenciales de test o sandbox keys en código (veto total)

## CASTLE ACTIVO: C·_·S·T·_·E

> **Nota**: §2 del plan declara `C·S·E`. La tabla §8 (autoritativa) declara `C·_·S·T·_·E` añadiendo **T (Testing / pre-submit checker ejecutable)**. Se usa §8.

- **C (Contracts)**: Privacy manifest presente con `NSPrivacyAccessedAPITypes` según dependencias declaradas; ASO metadata dentro de límites de caracteres de cada plataforma
- **S (Security)**: Sin test credentials en código de producción; sin API keys en metadata ni en el binario; App Store Connect keys solo en secrets de CI
- **T (Testing)**: `scripts/pre-submit-check.ts` ejecutable localmente — reporta cada check con PASS/FAIL/WARN antes de cualquier submit
- **E (Environment)**: `APPLE_ID`, `ASC_API_KEY`, `GOOGLE_PLAY_KEY_FILE` solo en `.env` del servidor CI / secrets de GitHub Actions, nunca en código

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## Phase 1: Verify EAS Configured

Verificar que el proyecto tiene EAS Build configurado antes de continuar.

**Verificaciones** (en orden, BLOQUEANTE si falla alguna):

1. `eas.json` existe en la raíz → si no existe: BLOQUEANTE — mostrar mensaje:
   ```
   BLOQUEANTE: eas.json no encontrado.
   Este skill requiere /mobile-deploy configurado previamente.
   Ejecuta /mobile-deploy para configurar EAS Build antes de continuar.
   ```

2. `app.json` o `app.config.js` tiene `expo.ios.bundleIdentifier` y `expo.android.package` → si no: BLOQUEANTE

3. EAS CLI disponible (`eas --version`) → si no: instalar con `npm install -g eas-cli` y verificar de nuevo

**Config a recolectar** (interactivo si no está en `app.json`):
- Apple Team ID (`TEAM_ID`) — necesario para privacy manifest y AASA
- Bundle ID iOS y package name Android
- Versión de build disponible en EAS (confirmar con `eas build:list --limit 1`)

---

## Phase 2: Rejection Checker

Ejecutar el conjunto de 6 checks ANTES de cualquier submit. Generar `scripts/pre-submit-check.ts` que los automatiza.

**Tabla de checks** (ejecutados en este orden):

| # | Check | Verifica | Resultado si falla |
|---|-------|----------|--------------------|
| 1 | Privacy manifest | `ios/PrivacyInfo.xcprivacy` existe con `NSPrivacyAccessedAPITypes` no vacío | **BLOQUEANTE** — iOS 17+ requiere privacy manifest; skill lo genera en Phase 3 |
| 2 | NSUsage strings | Cada permiso declarado en `app.json` bajo `expo.ios.infoPlist` tiene su `NS*UsageDescription` correspondiente | **BLOQUEANTE** — App Store rechaza sin descripción de uso |
| 3 | Icon sizes | `assets/icon.png` es 1024×1024px; `assets/icon-notification.png` es 120×120px | **BLOQUEANTE** — tamaños incorrectos causan rechazo automático |
| 4 | No test credentials | Código fuente no contiene `"test@"`, `"password123"`, ni sandbox keys (`pk_test_`, `sk_test_`) | **BLOQUEANTE** — credenciales de test en producción causan rechazo |
| 5 | Content rating | `eas.json` tiene `contentRating` configurado para iOS y Android | **ADVERTENCIA** — no bloquea, pero es requerido en review manual |
| 6 | Privacy policy URL | `app.json` tiene `expo.extra.privacyPolicyUrl` o equivalente | **ADVERTENCIA** — App Store puede solicitar durante review |

**Comportamiento al encontrar BLOQUEANTE**:
- Detener la ejecución del checker en ese check
- Mostrar mensaje claro con el problema y la acción requerida
- Si el check es "Privacy manifest" (check 1): continuar a Phase 3 para generarlo automáticamente, luego re-ejecutar el checker

**Formato de salida del checker**:
```
[PASS] Privacy manifest: ios/PrivacyInfo.xcprivacy presente con 3 tipos declarados
[PASS] NSUsage strings: NSCameraUsageDescription, NSMicrophoneUsageDescription presentes
[FAIL] Icon sizes: assets/icon.png es 512×512 — se requiere 1024×1024
→ BLOQUEANTE: corregir icon.png antes de continuar
```

---

## Phase 3: Privacy Manifest (iOS 17+)

Generar `ios/PrivacyInfo.xcprivacy` con los `NSPrivacyAccessedAPITypes` correctos según las dependencias detectadas en `package.json`.

**Detección de dependencias y tipos correspondientes**:

| Dependencia en package.json | NSPrivacyAccessedAPIType |
|-----------------------------|--------------------------|
| `expo-tracking-transparency` | `NSPrivacyAccessedAPICategoryUserDefaults` |
| `expo-notifications` | `NSPrivacyAccessedAPICategoryUserDefaults` |
| `expo-contacts` | `NSPrivacyAccessedAPICategoryContacts` |
| `expo-camera` | `NSPrivacyAccessedAPICategoryCamera` |
| `expo-location` | `NSPrivacyAccessedAPICategoryLocation` |
| `expo-media-library` | `NSPrivacyAccessedAPICategoryPhotoLibrary` |
| `@react-native-async-storage/async-storage` | `NSPrivacyAccessedAPICategoryUserDefaults` |

**Archivo generado** (`ios/PrivacyInfo.xcprivacy`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <!-- Generado por /mobile-app-store-submit según dependencias detectadas -->
    <!-- Agregar o quitar entradas según las dependencias reales del proyecto -->
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>CA92.1</string>
      </array>
    </dict>
  </array>
  <key>NSPrivacyCollectedDataTypes</key>
  <array/>
  <key>NSPrivacyTracking</key>
  <false/>
</dict>
</plist>
```

Agregar una entrada `<dict>` por cada tipo detectado. Documentar en comentario qué dependencia originó cada tipo.

---

## Phase 4: ASO Metadata

Generar los archivos de metadata para App Store Connect y Google Play Console.

**`store/ios/metadata.json`** (límites estrictos):

```json
{
  "title": "Nombre de la app (máx 30 chars)",
  "subtitle": "Tagline corto (máx 30 chars)",
  "description": "Descripción completa de la app. Incluir propuesta de valor, características principales y call to action. (máx 4000 chars)",
  "keywords": "keyword1,keyword2,keyword3 (máx 100 chars total, separados por coma)",
  "support_url": "https://tudominio.com/support",
  "marketing_url": "https://tudominio.com",
  "privacy_url": "https://tudominio.com/privacy"
}
```

**`store/android/metadata.json`** (límites estrictos):

```json
{
  "title": "Nombre de la app (máx 50 chars)",
  "short_description": "Descripción corta visible en la ficha de Play Store (máx 80 chars)",
  "full_description": "Descripción completa con características y beneficios. (máx 4000 chars)",
  "category": "PRODUCTIVITY",
  "content_rating": "EVERYONE"
}
```

**`store/screenshots/README.md`**: guía con dimensiones requeridas por plataforma:

- iOS iPhone 6.7" (1290×2796): mínimo 3, máximo 10 screenshots
- iOS iPad 12.9" (2048×2732): requerido si la app soporta iPad
- Android Phone (1080×1920 o superior): mínimo 2, máximo 8 screenshots
- Android 10" tablet (1920×1200): requerido si soporta tablets

La guía incluye sugerencias de contenido por screenshot (onboarding, feature principal, pantalla de perfil, etc.). No genera imágenes — indica dimensiones y contenido sugerido.

---

## Phase 5: CI Workflow + Submit

### CI Workflow (`.github/workflows/store-submit.yml`)

Workflow de GitHub Actions que se activa en merge a `main`:

```yaml
name: Store Submit
on:
  push:
    branches: [main]
jobs:
  pre-submit-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - name: Run pre-submit checks
        run: npx ts-node scripts/pre-submit-check.ts
  submit:
    needs: pre-submit-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install -g eas-cli
      - name: Submit iOS
        run: eas submit --platform ios --latest --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
      - name: Submit Android
        run: eas submit --platform android --latest --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
```

**Secrets requeridos** (nunca en código, solo en GitHub repo settings):
- `EXPO_TOKEN` — token de autenticación EAS
- Configurar en: Settings → Secrets and variables → Actions

### Submisión (tras pasar todos los checks)

```bash
eas submit --platform ios --latest
eas submit --platform android --latest
```

**Registro de historial** (`.king/store/submit-history.json`): append-only, una entrada por submit:

```json
{
  "submissions": [
    {
      "timestamp": "2026-05-28T00:00:00.000Z",
      "platform": "ios",
      "buildId": "eas-build-id",
      "status": "submitted",
      "version": "1.0.0",
      "buildNumber": "1"
    }
  ]
}
```

---

## CONFIGURACIÓN MANUAL REQUERIDA

### Apple Developer Account (no automatizable)

Ver `docs/apple-developer-setup.md` para los pasos que requieren intervención manual:

1. Crear App ID en Apple Developer Portal (https://developer.apple.com)
2. Crear la app en App Store Connect (https://appstoreconnect.apple.com)
3. Generar App Store Connect API Key (Roles → Integrations → App Store Connect API)
4. Configurar el contenido de privacidad en App Store Connect (Data types collected)
5. Subir screenshots y metadata de forma manual o via `fastlane deliver` la primera vez

Estos pasos son bloqueantes para la primera submisión y no pueden automatizarse con EAS Submit. El skill genera `docs/apple-developer-setup.md` con instrucciones paso a paso.

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] `eas.json` existe y tiene builds configurados para iOS y Android
- [ ] Rejection checker pasó todos los checks BLOQUEANTES con PASS
- [ ] `ios/PrivacyInfo.xcprivacy` generado con tipos correctos según dependencias de `package.json`
- [ ] NSUsage strings presentes para todos los permisos declarados en `app.json`
- [ ] `store/ios/metadata.json` con todos los campos dentro de límites de caracteres
- [ ] `store/android/metadata.json` con todos los campos dentro de límites de caracteres
- [ ] `store/screenshots/README.md` con guía de dimensiones por plataforma
- [ ] `.github/workflows/store-submit.yml` con pre-submit-check como prerequisito del submit job
- [ ] `scripts/pre-submit-check.ts` ejecutable y reporta PASS/FAIL/WARN por check
- [ ] `docs/apple-developer-setup.md` con pasos manuales de Apple Developer Account
- [ ] `.king/store/submit-history.json` creado/actualizado tras cada submit
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment — C·S·T·E, usando §8)_ |
| Artifacts | _(privacy manifest, ASO metadata, screenshots guide, CI workflow, pre-submit-check, submit history)_ |
| Next Recommended | `/mobile-deep-linking` (rutas Universal Links + App Links) |
| Risks | _(riesgos activos o "None")_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| App en stores y se quiere deep linking | `/mobile-deep-linking` |
| Se quiere medir conversión desde stores | `/mobile-analytics` |

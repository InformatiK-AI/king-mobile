---
name: mobile-biometric-auth
version: 1.0
api_version: 1.0.0
description: "Implementa autenticación biométrica (FaceID/TouchID iOS, Fingerprint/Face Android) con almacenamiento seguro de tokens en Keychain (iOS) y Keystore (Android). Detecta el framework (Expo, bare RN, Flutter), verifica si hay auth existente de /auth-in-one-command, y genera: servicio biométrico con enroll/authenticate/revoke, wrapper de Keychain/Keystore tipado, pantalla de enrollment opt-in post-login (silencia 7 días si rechazada), hook useBiometric, e integración con el flujo de auth existente. BLOQUEA si detecta AsyncStorage usado para session tokens y guía la migración."
---

# /mobile-biometric-auth — Autenticación Biométrica con Keychain/Keystore

Añade autenticación biométrica a una app mobile existente: FaceID / TouchID en iOS, Fingerprint / Face Authentication en Android. Almacena los access tokens exclusivamente en Keychain (iOS) o Keystore (Android) — nunca en AsyncStorage. Soporta Expo, bare React Native y Flutter.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Framework mobile del proyecto | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones, native bridges, permisos biométricos | Yes | framework |
| `knowledge/_inject/mobile-production-patterns.md` | Patrones de secure storage, fallback auth | Optional | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- No existe proyecto mobile scaffoldeado (sin `package.json`/`pubspec.yaml` ni estructura mobile reconocible)
- Framework no detectable y modo no-interactivo activo
- Se detecta AsyncStorage (u otro almacenamiento plaintext) usado para session tokens — BLOQUEAR con mensaje de migración (ver nota en ABSOLUTE RESTRICTIONS)
- El dispositivo/simulador objetivo no soporta biometría Y no se ha confirmado flujo de fallback

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA almacenar el access token en AsyncStorage, MMKV sin cifrado, SecureRandom sin Keychain, ni ningún storage plaintext — solo Keychain (iOS) o Keystore (Android)
- NUNCA continuar si se detecta `AsyncStorage.setItem` / `AsyncStorage.getItem` con tokens de sesión: emitir error bloqueante con mensaje `"[BLOCKED] AsyncStorage detectado para session token. Migrar a expo-secure-store (Expo), react-native-keychain (bare RN) o flutter_secure_storage (Flutter). Ver instrucciones en Phase 2."` y detener
- NUNCA hardcodear la key de Keychain/Keystore — usar siempre `{bundleId}.access_token` inyectada desde config
- NUNCA loguear el access token, refresh token ni biometric challenge en consola, Sentry ni snapshots de UI tests
- NUNCA saltarse el enrollment opt-in: la pantalla `BiometricSetupScreen` DEBE aparecer post-login exitoso, nunca en el splash ni antes del primer login

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- `src/services/biometric-auth.ts` — servicio con `enroll()`, `authenticate()`, `revoke()` (RN) o `lib/services/biometric_auth_service.dart` (Flutter)
- `src/services/secure-storage.ts` — wrapper tipado de Keychain/Keystore (RN) o `lib/services/secure_storage_service.dart` (Flutter)
- `src/screens/BiometricSetupScreen.tsx` — pantalla de enrollment opt-in post-login (RN) o `lib/screens/biometric_setup_screen.dart` (Flutter)
- `src/hooks/useBiometric.ts` — hook que expone estado y métodos (RN) — no aplica en Flutter
- Integración con `AuthScreen` existente (si `/auth-in-one-command` está instalado) o generación de auth screen básico con biometric
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
PHASE 0        PHASE 1          PHASE 2          PHASE 3           PHASE 4          PHASE N+1
SESSION     →  DETECT        →  BIOMETRIC     →  SECURE         →  ENROLLMENT    →  SESSION
(registry)     FRAMEWORK        SERVICE          STORAGE           SCREEN +          (escribir
               + DETECT         (enroll /        (Keychain /       integrate-        session)
               EXISTING         authenticate /   Keystore          auth-flow)
               AUTH             revoke)          wrapper)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Validación de permisos biométricos por plataforma (NSFaceIDUsageDescription, USE_BIOMETRIC), comportamiento en background y lock screen
- **@security** → Escalación si se detecta token en AsyncStorage o logging de credentials (veto total); valida key naming convention en Keychain/Keystore
- **@developer** → Integración con el flujo de auth existente (AuthScreen, contexto de sesión)

## CASTLE ACTIVO: _·_·S·T·_·E

- **S (Security)**: Tokens de sesión exclusivamente en Keychain/Keystore; skill BLOQUEA si detecta AsyncStorage para tokens; key de storage = `{bundleId}.access_token`
- **T (Testing)**: Enrollment opt-in es testeable de forma aislada; `BiometricSetupScreen` acepta prop `onEnroll` / `onSkip` para tests unitarios; fallback a password tras 3 intentos biométricos verificable con mocks
- **E (Environment)**: Cero credentials, tokens ni biometric challenges en logs, consola ni snapshots de UI; `bundleId` viene de config, no hardcodeado

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## Phase 1: Detect Framework + Detect Existing Auth

Detectar el framework mobile y si existe auth configurada previamente.

**Detección de framework** (de `package.json` / `pubspec.yaml` / `.king/knowledge/stack.md`):

| Framework | Señal | Librería biométrica | Storage seguro |
|-----------|-------|---------------------|----------------|
| React Native Expo | `expo` en dependencies | `expo-local-authentication` | `expo-secure-store` |
| React Native bare | `react-native` sin `expo` | `react-native-biometrics` | `react-native-keychain` |
| Flutter | `pubspec.yaml` presente | `local_auth` | `flutter_secure_storage` |

Si el framework no se detecta y el modo es interactivo, preguntar. Si es no-interactivo → BLOCKING CONDITION.

**Detección de AsyncStorage (OBLIGATORIA)**:

Buscar en el proyecto `AsyncStorage.setItem` o `AsyncStorage.getItem` con patrones que indiquen tokens de sesión (`token`, `accessToken`, `authToken`, `session`). Si se encuentra → BLOCKING CONDITION: emitir mensaje de migración y detener.

**Detección de auth existente** (`/auth-in-one-command`):

| Señal | Acción |
|-------|--------|
| `src/screens/AuthScreen.tsx` o `src/auth/` existe | Modo integración: modificar AuthScreen existente para añadir prompt biométrico post-login |
| No existe auth previa | Modo generación: crear auth screen básico con soporte biométrico desde cero |

---

## Phase 2: Biometric Service

Generar el servicio biométrico según framework.

| Framework | Archivo generado | Métodos |
|-----------|-----------------|---------|
| Expo | `src/services/biometric-auth.ts` | `enroll()`, `authenticate()`, `revoke()` vía `expo-local-authentication` |
| bare RN | `src/services/biometric-auth.ts` | `enroll()`, `authenticate()`, `revoke()` vía `react-native-biometrics` |
| Flutter | `lib/services/biometric_auth_service.dart` | `enroll()`, `authenticate()`, `revoke()` vía `local_auth` |

El servicio **MUST**:
- `enroll()`: verificar disponibilidad biométrica en el dispositivo; si no disponible, retornar `{ enrolled: false, reason: "NOT_AVAILABLE" }`; si disponible, guardar flag de enrollment en Keychain/Keystore key `{bundleId}.biometric_enrolled`
- `authenticate()`: ejecutar prompt biométrico con mensaje localizable (`"Iniciar sesión con FaceID / Huella"`); contar intentos fallidos con `maxAttempts: 3`; tras 3 fallos retornar `{ success: false, fallback: true }` para activar fallback a password
- `revoke()`: eliminar `{bundleId}.access_token` y `{bundleId}.biometric_enrolled` del Keychain/Keystore; llamar en logout
- NUNCA loguear el resultado del challenge ni el token recuperado

**Flujo de autenticación**:

```
[App Open]
  ↓
[Check: {bundleId}.biometric_enrolled en Keychain/Keystore?]
  ├── No → skip biometric, continuar con auth normal (password/PIN)
  └── Sí →
       ↓
[Prompt biométrico — "Iniciar sesión con FaceID / Huella"]
  ├── Success → leer {bundleId}.access_token de Keychain/Keystore → sesión activa
  └── Fail (acumulado = 3) → fallback: mostrar AuthScreen con password
       (intentos fallidos se registran localmente con timestamp en {bundleId}.bio_fail_log)
```

---

## Phase 3: Secure Storage

Generar el wrapper tipado de Keychain/Keystore.

| Framework | Archivo generado | Librería subyacente |
|-----------|-----------------|---------------------|
| Expo | `src/services/secure-storage.ts` | `expo-secure-store` |
| bare RN | `src/services/secure-storage.ts` | `react-native-keychain` |
| Flutter | `lib/services/secure_storage_service.dart` | `flutter_secure_storage` |

El wrapper **MUST** exponer:

```typescript
// React Native (Expo / bare)
interface SecureStorage {
  setToken(bundleId: string, token: string): Promise<void>;
  getToken(bundleId: string): Promise<string | null>;
  deleteToken(bundleId: string): Promise<void>;
  setFlag(key: string, value: string): Promise<void>;
  getFlag(key: string): Promise<string | null>;
  deleteFlag(key: string): Promise<void>;
}
```

Keys de Keychain/Keystore usadas por este skill:

| Key | Valor | Descripción |
|-----|-------|-------------|
| `{bundleId}.access_token` | JWT / opaque token | Access token de sesión — NUNCA en AsyncStorage |
| `{bundleId}.biometric_enrolled` | `"true"` | Flag de enrollment activo |
| `{bundleId}.bio_enroll_declined_at` | ISO timestamp | Timestamp de rechazo del opt-in (7 días de silencio) |
| `{bundleId}.bio_fail_log` | JSON serializado | Log local de intentos fallidos con timestamp |

**ABSOLUTE**: el wrapper NEVER usa AsyncStorage como fallback — si Keychain/Keystore no está disponible, lanzar error explícito: `"SecureStorageUnavailableError"`.

---

## Phase 4: Enrollment Screen + Integrate Auth Flow

**4a — BiometricSetupScreen**

Generar `src/screens/BiometricSetupScreen.tsx` (RN) o `lib/screens/biometric_setup_screen.dart` (Flutter):

- Aparece **post-login exitoso** (nunca en splash, nunca antes del primer login)
- Mostrar: ícono de biométrica + mensaje `"¿Activar FaceID / Huella para iniciar sesión más rápido?"`
- Botón **"Activar"**: llama `biometricAuth.enroll()` → si exitoso, guarda token en Keychain/Keystore (`secureStorage.setToken(bundleId, token)`) → navegar a home
- Botón **"Ahora no"**: guardar `{bundleId}.bio_enroll_declined_at = new Date().toISOString()` → navegar a home sin volver a preguntar en 7 días
- Lógica de silencio de 7 días: en AuthContext / auth guard, verificar `bio_enroll_declined_at`; si han pasado menos de 7 días, no mostrar `BiometricSetupScreen`
- Props para testabilidad: `onEnroll?: () => void`, `onSkip?: () => void`

**4b — Integración con auth flow**

| Escenario | Acción |
|-----------|--------|
| `/auth-in-one-command` presente | Modificar `AuthScreen.tsx` existente: añadir `useBiometric()` hook; si `biometric.enrolled`, mostrar botón "Entrar con FaceID / Huella" y ejecutar `biometricAuth.authenticate()` antes del submit de password |
| No hay auth previa | Generar `src/screens/AuthScreen.tsx` básico con campo email+password + sección biométrica condicional |

**Hook `useBiometric.ts`** (React Native):

```typescript
interface UseBiometricReturn {
  isEnrolled: boolean;
  isAvailable: boolean;
  authenticate: () => Promise<{ success: boolean; fallback: boolean }>;
  enroll: () => Promise<{ enrolled: boolean; reason?: string }>;
  revoke: () => Promise<void>;
}
```

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] Servicio biométrico generado con `enroll()`, `authenticate()`, `revoke()` para el framework detectado
- [ ] `authenticate()` cuenta intentos y activa fallback tras 3 fallos
- [ ] Wrapper `secure-storage` generado con keys `{bundleId}.access_token` y `{bundleId}.biometric_enrolled`
- [ ] Ningún token ni credential presente en logs, consola ni snapshots
- [ ] `BiometricSetupScreen` aparece post-login exitoso (no en splash)
- [ ] Botón "Ahora no" persiste `bio_enroll_declined_at` y silencia re-pregunta por 7 días
- [ ] Lógica de 7 días implementada en AuthContext o guard correspondiente
- [ ] `revoke()` cableado al logout (elimina token + flag de enrollment)
- [ ] Si se detectó AsyncStorage para tokens → skill BLOQUEÓ con mensaje de migración (no se continuó)
- [ ] Integración con AuthScreen existente (`/auth-in-one-command`) o generación de auth básico
- [ ] Hook `useBiometric.ts` expone `{ isEnrolled, isAvailable, authenticate, enroll, revoke }`
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment — S·T·E)_ |
| Artifacts | _(biometric-auth service, secure-storage wrapper, BiometricSetupScreen, useBiometric hook, AuthScreen integration)_ |
| Next Recommended | `/mobile-offline-sync` (persistencia segura offline) o `/mobile-analytics` (eventos biometric_enrolled, biometric_fallback) |
| Risks | _(riesgos activos o "None")_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Auth biométrica configurada y se quiere persistencia segura offline | `/mobile-offline-sync` |
| Se quiere medir tasa de enrollment y fallbacks biométricos | `/mobile-analytics` |
| Se quiere preparar submisión a stores (declarar uso de FaceID) | `/mobile-app-store-submit` |

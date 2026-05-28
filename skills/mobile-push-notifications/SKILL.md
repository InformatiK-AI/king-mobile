---
name: mobile-push-notifications
version: 1.0
api_version: 1.0.0
description: "Configura push notifications con FCM (Android) + APNs (iOS), token registry server-side con revocación, segmentación por cohort y opt-in WCAG-compliant que no interrumpe el onboarding. Soporta Expo, bare React Native y Flutter."
---

# /mobile-push-notifications — Notificaciones Push con Token Registry

Configura notificaciones push para una app mobile existente (generada por `/mobile-scaffold`): FCM para Android, APNs para iOS, registro de tokens server-side con revocación, segmentación por cohort y un opt-in respetuoso del onboarding.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Framework mobile del proyecto | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones, native bridges, permisos | Yes | framework |
| `knowledge/_inject/mobile-production-patterns.md` | Patrones de push, token TTL, cohorts | Optional | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- No existe proyecto mobile scaffoldeado (sin `package.json`/`pubspec.yaml` ni estructura mobile reconocible)
- Framework no detectable y modo no-interactivo activo
- El usuario no tiene acceso a un proyecto Firebase ni a un Apple Push key (.p8) y no puede crearlos
- El proyecto ya tiene una integración de push conflictiva sin posibilidad de migración

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA hardcodear push tokens, `FCM_SERVER_KEY`, `.p8` keys ni API keys en el código cliente
- NUNCA generar un token registry sin mecanismo de revocación (`revokedAt`) — es bloqueante de seguridad
- NUNCA mostrar el prompt de permisos en el splash screen ni en los 2 primeros pasos del onboarding
- NUNCA enviar notificaciones a tokens revocados o de usuarios eliminados

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- Cliente de notificaciones: `src/services/notifications.ts` (RN) o `lib/services/notification_service.dart` (Flutter)
- Hook de opt-in: `src/hooks/useNotificationPermission.ts` (prompt post-onboarding)
- Token registry server-side con revocación: `server/notifications/token-registry.ts`
- Wrapper de envío con segmentación: `server/notifications/send.ts` + `server/notifications/send-to-cohort.ts`
- `.env.example` actualizado con `FCM_SERVER_KEY`, `APNS_KEY_ID`, `APNS_TEAM_ID`
- `docs/push-setup.md` — configuración de Firebase + Apple Push Services
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
PHASE 0      PHASE 1        PHASE 2        PHASE 3         PHASE 4         PHASE 5       PHASE N+1
SESSION   →  DETECT      →  CLIENT      →  TOKEN        →  OPT-IN       →  TEST       →  SESSION
(registry)   FRAMEWORK      SETUP          REGISTRY        SCREEN          SEND          (escribir
             + config       (handler)      (revocación)    (post-onb.)     (sandbox)     session)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Validación de constraints de plataforma (permisos iOS/Android, comportamiento background)
- **@developer** → Token registry server-side (endpoints de registro/revocación, persistencia)
- **@security** → Escalación si se detecta token o key hardcodeado (veto total)

## CASTLE ACTIVO: C·_·S·_·_·E

- **C (Contracts)**: Token registry cumple el schema `PushToken`; endpoints de registro/revocación definidos
- **S (Security)**: Sin tokens ni keys hardcodeados; revocación obligatoria en logout; tokens revocados no reciben envíos
- **E (Environment)**: `FCM_SERVER_KEY`, `APNS_KEY_ID`, `APNS_TEAM_ID` solo en `.env`, nunca en código ni en el binario

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## Phase 1: Detect Framework + Collect Config

Detectar el framework mobile y recolectar la configuración necesaria.

**Detección** (de `package.json` / `pubspec.yaml` / `.king/knowledge/stack.md`):

| Framework | Señal | Librería de push |
|-----------|-------|------------------|
| React Native Expo | `expo` en dependencies | `expo-notifications` |
| React Native bare | `react-native` sin `expo` | `@notifee/react-native` + `@react-native-firebase/messaging` |
| Flutter | `pubspec.yaml` presente | `firebase_messaging` |

Si el framework no se detecta y el modo es interactivo, preguntar. Si es no-interactivo → BLOCKING CONDITION.

**Config a recolectar**: `projectId` de Firebase (Android), `bundleId` iOS, Apple Team ID. Estos valores van a `.env` / `app.config.js` — NUNCA al código fuente.

---

## Phase 2: Client Setup (handler de notificaciones)

Generar el cliente de notificaciones según framework:

| Framework | Archivo generado | Contenido |
|-----------|------------------|-----------|
| Expo | `src/services/notifications.ts` | Config de `expo-notifications`, handler foreground/background, registro de device token |
| bare RN | `src/services/notifications.ts` | `@notifee/react-native` display + `@react-native-firebase/messaging` token |
| Flutter | `lib/services/notification_service.dart` | `firebase_messaging` con handlers `onMessage` / `onBackgroundMessage` |

El handler **MUST**:
- Registrar el device token y enviarlo al token registry (Phase 3) — nunca persistirlo en cliente en plaintext sin revocación
- Manejar notificación en foreground (mostrar in-app) y en background (sistema)
- Exponer el payload `route` para integración con deep linking (si `/mobile-deep-linking` está instalado)

**Config generada por framework**: Expo → `app.json`/`app.config.js` + `eas.json`; bare RN → `google-services.json` ref + APNs entitlements; Flutter → `AndroidManifest.xml` + `firebase_messaging` setup.

---

## Phase 3: Token Registry (server-side, con revocación)

Generar `server/notifications/token-registry.ts` con el schema y endpoints. Schema obligatorio:

```typescript
interface PushToken {
  userId: string;
  token: string;
  platform: "ios" | "android";
  createdAt: Date;
  lastUsedAt: Date;
  revokedAt?: Date; // TTL: revocar en logout + 30 días de inactividad
}
```

Endpoints **MUST**:
- `POST /notifications/tokens` — registra/actualiza el token del usuario autenticado
- `DELETE /notifications/tokens/{userId}` — revoca (setea `revokedAt`), llamado en logout
- Job/TTL que revoca tokens con `lastUsedAt` > 30 días

**ABSOLUTE**: el registry **MUST NOT** persistir el `FCM_SERVER_KEY` ni el `.p8` — esos viven en `.env` del servidor.

---

## Phase 4: Opt-in Screen (WCAG, post-onboarding)

Generar `src/hooks/useNotificationPermission.ts` + una pantalla "enable notifications" que respeta el onboarding:

- El prompt del sistema **MUST** aparecer DESPUÉS del onboarding principal, nunca en splash ni en los primeros 2 pasos
- Mostrar beneficio claro ANTES del prompt del sistema
- Botón "Activar ahora" + botón "Más tarde"
- Si el usuario eligió "Más tarde": no volver a preguntar en 7 días (persistir timestamp)

El hook expone: `{ status: "undetermined" | "granted" | "denied", requestPermission(), canAskAgain }`.

---

## Phase 5: Test Send + Segmentación

Generar el wrapper de envío y el helper de cohorts:

- `server/notifications/send.ts` — envío con segmentación por user property; agrupa por platform para batch FCM v1 + APNs HTTP/2
- `server/notifications/send-to-cohort.ts` — acepta array de userIds, filtra tokens activos (no revocados) del registry, agrupa por platform
- `docs/push-setup.md` — pasos para configurar Firebase project + Apple Push key (.p8)

**Test de envío**: documentar el comando para enviar una notificación de prueba a un token de sandbox y verificar entrega en iOS Simulator + Android Emulator.

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] Cliente de notificaciones generado para el framework detectado, con handler foreground+background
- [ ] Token registry con schema `PushToken` y endpoint de revocación (`DELETE`) funcional
- [ ] Revocación cableada al logout de la app
- [ ] Opt-in NO aparece en splash ni en los 2 primeros pasos del onboarding
- [ ] "Más tarde" suprime el prompt por 7 días
- [ ] `send-to-cohort` filtra tokens revocados y agrupa por platform
- [ ] `.env.example` con `FCM_SERVER_KEY`, `APNS_KEY_ID`, `APNS_TEAM_ID`; ningún secreto en código
- [ ] `docs/push-setup.md` con pasos de Firebase + Apple Push
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment — C·S·E)_ |
| Artifacts | _(notifications client, token-registry, send/send-to-cohort, opt-in hook, docs)_ |
| Next Recommended | `/mobile-deep-linking` (rutas desde push) o `/mobile-analytics` (eventos push_opted_in) |
| Risks | _(riesgos activos o "None")_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Push configurado y se quiere abrir pantallas desde la notificación | `/mobile-deep-linking` |
| Se quiere medir opt-in y engagement de push | `/mobile-analytics` |
| Se quiere preparar submisión a stores | `/mobile-app-store-submit` |

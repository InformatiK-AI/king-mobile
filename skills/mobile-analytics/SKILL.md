---
name: mobile-analytics
version: 1.0
api_version: 1.0.0
description: "Configura analytics mobile con eventos predefinidos por tipo de producto (SaaS / Consumer), screen tracking automático del navegador (React Navigation / Expo Router / go_router), funnels, session replay opt-in con PII masking, y cohort analysis. Genera el catálogo tipado de eventos, singleton analytics, capa de privacidad GDPR con consentimiento explícito en onboarding, y wrapper de navegador. Providers: Posthog (open source, session replay), Amplitude (funnels/cohorts), Mixpanel (industry standard). BLOQUEA si tracking inicia antes del consentimiento. Usar cuando: agregar analytics, eventos tipados mobile, screen tracking automático, session replay react native, PII masking analytics, GDPR analytics mobile, posthog react native, amplitude react native, mixpanel react native, funnel analysis mobile, cohort analysis mobile."
---

# /mobile-analytics — Analytics Mobile con Eventos Tipados y GDPR Compliance

Configura analytics mobile sobre una app existente (generada por `/mobile-scaffold`): catálogo de eventos tipados por tipo de producto, screen tracking automático del navegador detectado, singleton de tracking con `track()` / `identify()` / `screen()`, capa de privacidad con PII masking y consentimiento GDPR, y session replay opt-in (Posthog). Soporta Expo, bare React Native y Flutter.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Framework mobile + tipo de producto (saas/consumer) | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones, navegador activo, permisos | Yes | framework |
| `knowledge/_inject/mobile-production-patterns.md` | Patrones de privacy, consentimiento, replay | Optional | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- No existe proyecto mobile scaffoldeado (sin `package.json`/`pubspec.yaml` ni estructura mobile reconocible)
- Framework o navegador no detectable y modo no-interactivo activo
- Se detecta envío de eventos de analytics ANTES de que exista un gate de consentimiento — BLOQUEAR con mensaje de migración
- El provider seleccionado requiere `ANALYTICS_API_KEY` y esta no está en `.env` ni puede ser creada

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA enviar eventos de analytics antes del consentimiento explícito del usuario — el gate de privacidad es obligatorio antes del primer evento de negocio
- NUNCA hardcodear `ANALYTICS_API_KEY` en el código fuente — solo en `.env` / `.env.example`
- NUNCA capturar PII (email, número de tarjeta, teléfono) en eventos o replays sin aplicar el masking definido en `privacy.ts`
- NUNCA activar session replay por defecto — es opt-in explícito; el default es `sessionReplay: false`
- NUNCA loguear el contenido de eventos con PII en consola, Sentry ni snapshots de UI tests
- NUNCA aplicar masking parcial — si un campo contiene PII, el valor completo se reemplaza por `"***"`, no se trunca

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- `src/analytics/events.ts` — catálogo tipado de eventos con propiedades (SaaS + Consumer)
- `src/analytics/analytics.ts` — singleton con `track()`, `identify()`, `screen()`
- `src/analytics/privacy.ts` — PII masking + opt-out + gate de consentimiento
- `src/navigation/analytics-wrapper.tsx` — wrapper del navegador detectado para screen tracking automático
- `.env.example` actualizado con `ANALYTICS_API_KEY`
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
PHASE 0         PHASE 1          PHASE 2          PHASE 3           PHASE 4           PHASE 5           PHASE N+1
SESSION      →  DETECT        →  SETUP         →  DEFINE         →  SCREEN         →  PRIVACY        →  SESSION
(registry)      FRAMEWORK        SDK              EVENTS            TRACKING           CONSENT +          (escribir
                + SELECT         (provider        (catálogo         (auto              OPTIONAL           session)
                PROVIDER         init,            tipado            wrapper del        SESSION
                                 API key)         SaaS/Consumer)    navegador)         REPLAY)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Validación de integración con el navegador (React Navigation / Expo Router / go_router), comportamiento de screen tracking en background y hot reload
- **@qa** → Verificación del GDPR consent flow: eventos NO enviados antes del consentimiento; PII masking comprobado en el payload de red (no solo en UI)
- **@security** → Escalación si se detecta `ANALYTICS_API_KEY` hardcodeada en código o si session replay captura PII sin masking (veto total)

## CASTLE ACTIVO: C·_·S·T·_·E

> **Nota**: §2 del plan declara `C·S·E`. La tabla §8 (autoritativa) declara `C·_·S·T·_·E` añadiendo **T (Testing)**. Se usa §8.

- **C (Contracts)**: Catálogo de eventos tipado en `events.ts`; `track()` solo acepta eventos del union type definido — error de compilación si se pasa un evento no declarado
- **S (Security)**: PII masking obligatorio activo en `privacy.ts`; `ANALYTICS_API_KEY` solo en `.env`; session replay OFF por defecto; ningún valor PII real llega al servidor de analytics
- **T (Testing)**: `analytics.ts` singleton inyectable con mock; `AnalyticsWrapper` acepta prop `onScreenChange` para tests unitarios; consent gate verificable con test de integración que comprueba cero eventos antes del consentimiento
- **E (Environment)**: `ANALYTICS_API_KEY` solo en `.env` / `.env.example`; cero credentials en logs, consola ni snapshots; provider init falla explícitamente si la key no está presente

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## Phase 1: Detect Framework + Select Provider

Detectar el framework mobile, el navegador activo y el tipo de producto, luego seleccionar el provider de analytics.

**Detección de framework** (de `package.json` / `pubspec.yaml` / `.king/knowledge/stack.md`):

| Framework | Señal | Navegador detectado |
|-----------|-------|---------------------|
| React Native Expo | `expo` en dependencies | Expo Router (`expo-router`) o React Navigation (`@react-navigation/native`) |
| React Native bare | `react-native` sin `expo` | React Navigation (`@react-navigation/native`) |
| Flutter | `pubspec.yaml` presente | go_router (`go_router` en pubspec) o Navigator 2.0 |

Si el framework no se detecta y el modo es interactivo, preguntar. Si es no-interactivo → BLOCKING CONDITION.

**Selección de provider**:

| Provider | SDK | Ventaja clave | Cuándo usar |
|----------|-----|---------------|-------------|
| Posthog | `posthog-react-native` | Open source, self-hosteable, session replay | Default si session replay es requerido |
| Amplitude | `@amplitude/analytics-react-native` | Funnels + cohorts maduros | Si funnels/cohorts son prioridad y no se necesita session replay |
| Mixpanel | `mixpanel-react-native` | Industry standard, retrocompatible | Si hay integración previa con Mixpanel web |

Si no hay preferencia en `.king/knowledge/stack.md` y el modo es interactivo → preguntar. Si es no-interactivo → usar Posthog como default.

**Detección de tipo de producto** (de `stack.md` o interactivo):

| Tipo | Catálogo de eventos |
|------|---------------------|
| `saas` | `SAAS_EVENTS` (ver Phase 3) |
| `consumer` | `CONSUMER_EVENTS` (ver Phase 3) |
| Ambos / no detectado | Generar ambos catálogos en `events.ts` |

**Config a recolectar**: `ANALYTICS_API_KEY` del provider → va a `.env` y `.env.example`. NUNCA al código fuente.

---

## Phase 2: Setup SDK

Inicializar el provider seleccionado según framework.

| Provider + Framework | Instalación | Archivo de init |
|----------------------|-------------|-----------------|
| Posthog + RN (Expo/bare) | `npx expo install posthog-react-native` o `npm install posthog-react-native` | `src/analytics/analytics.ts` |
| Amplitude + RN | `npm install @amplitude/analytics-react-native` | `src/analytics/analytics.ts` |
| Mixpanel + RN | `npm install mixpanel-react-native` | `src/analytics/analytics.ts` |
| Posthog + Flutter | `flutter pub add posthog_flutter` | `lib/analytics/analytics_service.dart` |

El init del SDK **MUST**:
- Leer `ANALYTICS_API_KEY` exclusivamente desde `process.env.ANALYTICS_API_KEY` (RN) o `String.fromEnvironment('ANALYTICS_API_KEY')` (Flutter)
- Lanzar error explícito `"AnalyticsInitError: ANALYTICS_API_KEY not set"` si la key no está presente
- No inicializar el provider hasta que el gate de consentimiento retorne `true` (ver Phase 5)
- Exponer un flag `isInitialized: boolean` para que el consent gate lo controle

**Estructura del singleton `analytics.ts`** (React Native):

```typescript
interface AnalyticsService {
  initialize(apiKey: string): Promise<void>;
  track<T extends EventName>(event: T, properties: EventProperties[T]): void;
  identify(userId: string, traits?: UserTraits): void;
  screen(screenName: string, properties?: ScreenProperties): void;
  optOut(): void;
  isInitialized: boolean;
}
```

El singleton verifica `isInitialized` antes de cada `track()` — si no está inicializado (consentimiento no dado), el evento se descarta silenciosamente y se registra en debug log local (nunca enviado).

---

## Phase 3: Define Events (Catálogo Tipado)

Generar `src/analytics/events.ts` con el catálogo completo de eventos tipados.

### Eventos SaaS

```typescript
// SaaS mobile events
export interface SaaSEvents {
  app_opened: {
    cold_start: boolean;
    session_id: string;
  };
  onboarding_completed: {
    steps_completed: number;
    duration_ms: number;
  };
  first_feature_used: {
    feature_name: string;
  };
  plan_upgrade_viewed: {
    current_plan: string;
    target_plan: string;
  };
  plan_upgraded: {
    from_plan: string;
    to_plan: string;
    revenue: number;
  };
}
```

### Eventos Consumer

```typescript
// Consumer / fitness / social events
export interface ConsumerEvents {
  app_opened: {
    cold_start: boolean;
    push_notification_id?: string;
  };
  content_viewed: {
    content_id: string;
    content_type: string;
    duration_ms: number;
  };
  social_action: {
    action: string;
    target_id: string;
  };
  push_opted_in: {
    prompt_context: string;
  };
  session_ended: {
    duration_ms: number;
    screens_visited: number;
  };
}
```

### Evento universal de screen tracking

```typescript
export interface ScreenViewedEvent {
  screen_viewed: {
    screen_name: string;
    timestamp: number; // Unix ms
    previous_screen?: string;
  };
}
```

### Union type para el singleton

```typescript
export type AppEvents = SaaSEvents & ConsumerEvents & ScreenViewedEvent;
export type EventName = keyof AppEvents;
export type EventProperties = AppEvents;
```

El contrato garantiza error de compilación TypeScript si se llama `track()` con un evento no declarado o con propiedades incorrectas — este es el gate **C (Contracts)** del CASTLE.

---

## Phase 4: Screen Tracking (Automático)

Generar `src/navigation/analytics-wrapper.tsx` que instrumenta el navegador detectado para disparar `screen_viewed` en cada cambio de pantalla sin configuración adicional.

| Navegador | Mecanismo de tracking | Señal en el árbol de navegación |
|-----------|----------------------|---------------------------------|
| React Navigation | `NavigationContainer` `onStateChange` callback | `navigationRef.current?.getCurrentRoute()?.name` |
| Expo Router | `usePathname()` hook + `useEffect` en el layout raíz | Pathname del segmento activo |
| go_router (Flutter) | `GoRouter` `observers` con `NavigatorObserver` | `didPush` / `didPop` / `didReplace` |

**AnalyticsWrapper para React Navigation / Expo Router**:

```typescript
// src/navigation/analytics-wrapper.tsx
interface AnalyticsWrapperProps {
  children: React.ReactNode;
  onScreenChange?: (screenName: string) => void; // para tests
}
```

El wrapper **MUST**:
- Disparar `analytics.screen(screenName, { timestamp: Date.now() })` en cada cambio de pantalla
- Derivar el nombre de pantalla del estado del navegador — nunca de un prop manual
- Respetar el gate de consentimiento: si `analytics.isInitialized === false`, no registrar el evento (el consentimiento no fue dado aún)
- Exponer `onScreenChange` como prop para testabilidad (Phase CASTLE T)

**Integración en el árbol de la app**:

| Framework | Punto de integración |
|-----------|---------------------|
| React Navigation | Envolver `<NavigationContainer>` con `<AnalyticsWrapper>` en `App.tsx` |
| Expo Router | Añadir lógica en `app/_layout.tsx` raíz usando `usePathname()` |
| Flutter | Registrar `AnalyticsNavigatorObserver` en `GoRouter` / `MaterialApp.router` |

---

## Phase 5: Privacy, Consent + Optional Session Replay

### 5a — Consent Gate (GDPR, obligatorio)

Generar `src/analytics/privacy.ts` con el gate de consentimiento y PII masking.

El tracking **MUST NOT** iniciar hasta que el usuario acepte explícitamente. El consent flow se integra en el onboarding de la app **antes del primer evento de negocio**:

```
[Onboarding Step N-1: ConsentScreen]
  ↓
[Usuario acepta analytics]  →  analytics.initialize(apiKey)  →  [tracking activo]
[Usuario rechaza]           →  analytics permanece no-inicializado  →  [tracking nunca activo]
```

**`privacy.ts` MUST exponer**:

```typescript
interface PrivacyService {
  hasConsent(): boolean;
  grantConsent(): Promise<void>;   // persiste en AsyncStorage "analytics_consent" = "granted"
  revokeConsent(): Promise<void>;  // persiste "denied", llama analytics.optOut()
  maskPII(value: string): string;  // retorna "***" para email, tarjeta, teléfono; valor original si no hay PII
}
```

**Reglas de PII masking** (aplicar en `privacy.ts`):

| Tipo de PII | Patrón de detección | Valor enmascarado |
|-------------|--------------------|--------------------|
| Email | `/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/` | `"***"` |
| Número de tarjeta | `/\b\d{13,19}\b/` | `"***"` |
| Teléfono | `/(\+?\d[\s\-.]?){7,14}\d/` | `"***"` |

El masking se aplica **antes** de enviar al provider — no es decorativo en la UI.

### 5b — Opt-out

El usuario puede revocar el consentimiento desde configuración de la app:
- `privacy.revokeConsent()` → llama `analytics.optOut()` del provider → todos los envíos futuros cesan
- El estado de opt-out se persiste para sobrevivir reinicios de la app
- En el próximo arranque: `hasConsent() === false` → SDK no se inicializa

### 5c — Session Replay (opt-in, OFF por defecto)

**Solo disponible si el provider es Posthog.** Si el usuario activa explícitamente session replay (desde configuración de privacidad de la app):

Config de Posthog con replay activo:

```typescript
PostHog.initAsync(apiKey, {
  sessionReplay: {
    maskAllTextInputs: true,           // enmascara todos los TextInput por defecto
    maskAllImages: false,
    maskTextContent: true,             // aplica maskPII() antes de enviar contenido de texto
    captureLogs: false,                // NUNCA capturar logs en replay
  }
});
```

**Masking adicional en session replay**:
- Campos con `secureTextEntry: true` (RN) → enmascarados automáticamente por `maskAllTextInputs`
- Valores de email, tarjeta, teléfono en cualquier campo → `maskPII()` aplicado al texto capturado
- Si el usuario revoca el consentimiento → `posthog.optOutCapturing()` inmediatamente; replay se detiene

**El session replay NUNCA se activa** sin que el usuario haya:
1. Dado consentimiento general de analytics (Phase 5a)
2. Activado específicamente la opción de session replay en configuración

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] Framework y navegador detectados correctamente; provider seleccionado y documentado
- [ ] `ANALYTICS_API_KEY` solo en `.env` / `.env.example`; cero hardcode en código fuente
- [ ] `events.ts` define catálogo tipado completo: SaaS (5 eventos) + Consumer (5 eventos) + `screen_viewed`
- [ ] `track()` rechaza en compilación eventos no declarados (contrato TypeScript activo — CASTLE C)
- [ ] `analytics.ts` singleton no inicializa el SDK hasta que `hasConsent() === true`
- [ ] `analytics-wrapper.tsx` dispara `screen_viewed` automáticamente en cada cambio de pantalla con nombre + timestamp
- [ ] `analytics-wrapper.tsx` expone `onScreenChange` prop para tests (CASTLE T)
- [ ] `privacy.ts` expone `hasConsent()`, `grantConsent()`, `revokeConsent()`, `maskPII()`
- [ ] `maskPII()` enmascara email, tarjeta y teléfono antes del envío al provider (CASTLE S)
- [ ] Consent flow está integrado en el onboarding ANTES del primer evento de negocio
- [ ] NO se envían eventos si el usuario rechaza o aún no ha visto el consent flow
- [ ] Session replay: OFF por defecto; solo activable por opt-in explícito + consentimiento previo
- [ ] Session replay (si activo): `maskAllTextInputs: true`, `captureLogs: false`, PII masking en texto capturado
- [ ] `analytics.ts` singleton inyectable con mock para tests unitarios (CASTLE T)
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment — C·S·T·E)_ |
| Artifacts | _(events.ts, analytics.ts, privacy.ts, analytics-wrapper.tsx, .env.example)_ |
| Next Recommended | `/mobile-app-store-submit` (submisión a stores con privacy manifest) o `/mobile-offline-sync` (persistencia segura offline) |
| Risks | _(riesgos activos o "None")_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Analytics configurado y se quiere subir a stores con privacy manifest correcto | `/mobile-app-store-submit` |
| Se quiere persistencia de datos analíticos offline (sin conexión) | `/mobile-offline-sync` |
| Se quiere añadir deep linking desde notificaciones con tracking de fuente | `/mobile-deep-linking` |

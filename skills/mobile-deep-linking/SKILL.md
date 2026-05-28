---
name: mobile-deep-linking
version: 1.0
api_version: 1.0.0
description: "Configura Universal Links (iOS) y App Links (Android) con AASA y assetlinks.json, fallback web con banner de store, y manejo de rutas desde notificaciones push. Soporta React Navigation y Expo Router. Usar cuando: deep links, universal links, app links, AASA, assetlinks, fallback web, abrir pantalla desde notificación, rutas desde push."
---

# /mobile-deep-linking — Universal Links + App Links con Fallback Web

Configura deep linking completo para una app mobile existente: Universal Links iOS via `apple-app-site-association` (AASA), App Links Android via `assetlinks.json`, fallback web con banner de store cuando la app no está instalada, y conexión con el handler de push notifications de `/mobile-push-notifications`.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Framework mobile del proyecto | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones, bundle IDs, configuración de navegación | Yes | framework |
| `knowledge/_inject/mobile-production-patterns.md` | Patrones de routing, deep link handlers | Optional | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- No existe proyecto mobile scaffoldeado (sin `package.json`/`pubspec.yaml` ni estructura mobile reconocible)
- No se puede determinar el dominio del producto (requerido para AASA y assetlinks)
- No se proporcionan rutas a linkear (mínimo una ruta es requerida)
- Framework de navegación no detectable y modo no-interactivo activo

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA generar AASA con `appIDs` sin el TEAM_ID real del proyecto — usar placeholder `TEAM_ID` documentado si no se conoce
- NUNCA hardcodear `sha256_cert_fingerprints` en `assetlinks.json` — usar placeholder `${SHA256_FINGERPRINT}` con instrucción explícita de completar
- NUNCA omitir el fallback web — es parte del contrato de calidad S (Security/UX) de este skill
- NUNCA conectar la integración push sin verificar que `/mobile-push-notifications` está instalado

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- AASA: `public/.well-known/apple-app-site-association`
- App Links: `public/.well-known/assetlinks.json`
- Configuración de rutas: `src/navigation/deep-link-config.ts`
- Handler de deep links: `src/services/deep-link-handler.ts`
- Docs de testing: `docs/deep-link-testing.md`
- Script de verificación: `scripts/verify-deep-links.ts`
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
PHASE 0          PHASE 1              PHASE 2                PHASE 3          PHASE 4            PHASE 5
SESSION       →  COLLECT           →  GENERATE           →  APP ROUTER     →  FALLBACK       →  PUSH
(registry)       DOMAIN +             AASA +                 CONFIG            WEB               INTEGRATION
                 ROUTES               ASSETLINKS             (React Nav /      (middleware:       (conectar con
                 (inputs del          (public/.well-         Expo Router       UA detection +     push-notif
                 usuario)             known/)                deep-link-        store banner)      handler si
                                                             config.ts)                           instalado)
                                                                                                    ↓
                                                                                                 PHASE 5
                                                                                                 → TEST CMDS
                                                                                                 (docs/
                                                                                                  deep-link-
                                                                                                  testing.md)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Configuración de Universal Links / App Links en el proyecto nativo (entitlements iOS, intent filters Android)
- **@developer** → Middleware de fallback web server-side (detección de user agent, banner de store)
- **@qa** → Comandos de prueba con `xcrun simctl` y `adb shell am start`; verificación en device matrix

## CASTLE ACTIVO: C·_·S·_·_·_

> §2 y §8 del plan coinciden en `C·_·S·_·_·_` para este skill.

- **C (Contracts)**: Las rutas declaradas en AASA (`components`) coinciden exactamente con las rutas del router de la app (`deep-link-config.ts`); cualquier ruta en AASA sin handler en el router es un bug de contrato
- **S (Security / UX)**: Fallback web presente y funcional — un deep link NUNCA debe resultar en página en blanco o error 404; el banner "Descargá la app" con link al store debe estar visible cuando la app no está instalada

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## Phase 1: Collect Domain + Routes

Recolectar los inputs del usuario necesarios para generar AASA y assetlinks.

**Inputs requeridos** (preguntar si no están en `.king/knowledge/`):

| Input | Ejemplo | Dónde se usa |
|-------|---------|--------------|
| Dominio del producto | `app.miproducto.com` | AASA, assetlinks, fallback web |
| Rutas a linkear | `/product/:id`, `/invite/:code` | AASA components, router config |
| TEAM_ID Apple | `ABC123DEF4` | AASA appIDs campo |
| Bundle ID iOS | `com.miproducto.app` | AASA appIDs campo |
| Package name Android | `com.miproducto.app` | assetlinks target |
| SHA256 fingerprint | (del keystore de producción) | assetlinks — puede dejarse como placeholder |
| Framework de navegación | React Navigation / Expo Router | deep-link-config.ts |

**Detección automática** desde `app.json`:
- `expo.ios.bundleIdentifier` → Bundle ID iOS
- `expo.android.package` → Package name Android
- `expo.owner` + proyecto → TEAM_ID (si no disponible, usar placeholder documentado)

**Transformación de rutas** (`:param` → wildcard para AASA):
- `/product/:id` → `"/product/*"` en AASA components
- `/invite/:code` → `"/invite/*"` en AASA components
- `/reset-password/:token` → `"/reset-password/*"` en AASA components

---

## Phase 2: Generate AASA + assetlinks

Generar los archivos de verificación de dominio.

### `public/.well-known/apple-app-site-association`

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAM_ID.com.miproducto.app"],
        "components": [
          { "/": "/product/*", "comment": "Pantalla de producto" },
          { "/": "/invite/*", "comment": "Invitación de usuario" },
          { "/": "/reset-password/*", "comment": "Reset de contraseña" }
        ]
      }
    ]
  }
}
```

**Reglas obligatorias**:
- El archivo DEBE servirse con `Content-Type: application/json` y sin redirecciones desde `https://DOMINIO/.well-known/apple-app-site-association`
- NO puede estar detrás de autenticación
- Incluir un `comment` por componente describiendo la pantalla de destino
- Si TEAM_ID no está disponible: usar `"TEAM_ID.com.miproducto.app"` como placeholder y documentar claramente:
  ```
  // ACCIÓN REQUERIDA: reemplazar TEAM_ID con tu Apple Team ID real
  // Obtenerlo en: https://developer.apple.com/account → Membership
  ```

### `public/.well-known/assetlinks.json`

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.miproducto.app",
      "sha256_cert_fingerprints": ["${SHA256_FINGERPRINT}"]
    }
  }
]
```

**Instrucción de completar** (siempre incluir como comentario en `docs/deep-link-testing.md`):
```bash
# Obtener SHA256 fingerprint del keystore de producción:
keytool -list -v -keystore release.keystore -alias release -storepass PASSWORD \
  | grep "SHA256" | awk '{print $2}'
```

El archivo se sirve desde `https://DOMINIO/.well-known/assetlinks.json` con `Content-Type: application/json`.

---

## Phase 3: App Router Config

Generar la configuración de rutas y el handler de deep links según el framework de navegación detectado.

### `src/navigation/deep-link-config.ts`

**Para Expo Router** (basado en file-system routing):

```typescript
// deep-link-config.ts — generado por /mobile-deep-linking
export const DEEP_LINK_DOMAIN = 'https://app.miproducto.com';

export const deepLinkConfig = {
  prefixes: [DEEP_LINK_DOMAIN, 'miproducto://'],
  config: {
    screens: {
      // Mapeo de rutas web → pantallas de la app
      // CONTRATO: estas rutas DEBEN coincidir con los components del AASA
      ProductScreen: 'product/:id',
      InviteScreen: 'invite/:code',
      ResetPasswordScreen: 'reset-password/:token',
    },
  },
};
```

**Para React Navigation**:

```typescript
export const deepLinkConfig = {
  prefixes: ['https://app.miproducto.com', 'miproducto://'],
  config: {
    screens: {
      Main: {
        screens: {
          ProductScreen: 'product/:id',
          InviteScreen: 'invite/:code',
          ResetPasswordScreen: 'reset-password/:token',
        },
      },
    },
  },
};
```

### `src/services/deep-link-handler.ts`

Handler que parsea la URL entrante y navega a la pantalla correcta:

```typescript
import { Linking } from 'react-native';

interface DeepLinkRoute {
  screen: string;
  params: Record<string, string>;
}

export function parseDeepLink(url: string): DeepLinkRoute | null {
  // Parsear ruta y extraer params
  // Retorna null si la ruta no está declarada en deepLinkConfig
}

export function handleDeepLink(url: string, navigation: NavigationProp<any>): void {
  const route = parseDeepLink(url);
  if (!route) {
    console.warn(`[DeepLink] Ruta no reconocida: ${url}`);
    return;
  }
  navigation.navigate(route.screen, route.params);
}

// Listener para app en foreground
export function initDeepLinkListener(navigation: NavigationProp<any>): () => void {
  const subscription = Linking.addEventListener('url', ({ url }) => {
    handleDeepLink(url, navigation);
  });
  return () => subscription.remove();
}

// Handler para app abierta desde deep link (cold start)
export async function handleInitialDeepLink(navigation: NavigationProp<any>): Promise<void> {
  const url = await Linking.getInitialURL();
  if (url) handleDeepLink(url, navigation);
}
```

**Integración en el navigator raíz**:

```typescript
// En App.tsx o _layout.tsx (Expo Router)
useEffect(() => {
  handleInitialDeepLink(navigation);
  const unsubscribe = initDeepLinkListener(navigation);
  return unsubscribe;
}, []);
```

---

## Phase 4: Fallback Web

Generar el middleware server-side que detecta el user agent y sirve el fallback correcto.

**Comportamiento requerido**:

1. Si el user agent es iOS o Android y la app no está instalada → redirigir a la URL web con un banner "Descargá la app"
2. Si el user agent es iOS o Android y la app está instalada → el OS intercepta antes (AASA/assetlinks); el middleware no interviene
3. Si el user agent es desktop browser → servir la página web normalmente

**Middleware (Express/Next.js)** (`server/middleware/deep-link-fallback.ts`):

```typescript
export function deepLinkFallbackMiddleware(req, res, next) {
  const ua = req.headers['user-agent'] || '';
  const isDeepLinkPath = DEEP_LINK_PATHS.some(path => req.path.match(path));

  if (!isDeepLinkPath) return next();

  const isMobile = /iPhone|iPad|Android/i.test(ua);
  if (!isMobile) return next();

  // Extraer parámetros de la URL para pasar al frontend
  const originalUrl = `${process.env.APP_DOMAIN}${req.path}`;
  const iosStoreUrl = 'https://apps.apple.com/app/APPLE_APP_ID';
  const androidStoreUrl = 'https://play.google.com/store/apps/details?id=com.miproducto.app';

  // Redirigir a página web con metadatos para el banner
  res.redirect(`/app-redirect?next=${encodeURIComponent(originalUrl)}&ios=${encodeURIComponent(iosStoreUrl)}&android=${encodeURIComponent(androidStoreUrl)}`);
}
```

**Banner "Descargá la app"** (`src/web/AppStoreBanner.tsx` — componente web):

```tsx
// Banner non-intrusivo: top de la página, con botón al store correcto
// Detecta iOS (App Store) vs Android (Play Store) para mostrar el link correcto
// Botón "Continuar en la web" para cerrar el banner sin ir al store
```

**`DEEP_LINK_PATHS`**: array de regex derivado de las rutas declaradas en Phase 1.

---

## Phase 5: Push Integration + Test Commands

### Integración con Push Notifications

Si `/mobile-push-notifications` está instalado, conectar el deep link handler con el notification handler.

**Verificar** que `src/services/notifications.ts` expone el payload `route` (parte del contrato de `/mobile-push-notifications`):

```typescript
// En src/services/notifications.ts (generado por /mobile-push-notifications)
// El handler ya expone el campo route — solo conectar:
notifications.setNotificationResponseReceivedListener((response) => {
  const route = response.notification.request.content.data?.route;
  if (route && navigation) {
    handleDeepLink(route, navigation);
  }
});
```

**Payload de notificación** que activa la navegación:
```json
{
  "title": "Nuevo producto disponible",
  "body": "El producto que seguías ya está en stock",
  "data": {
    "route": "/product/456"
  }
}
```

Al tap en la notificación, la app navega a `ProductScreen` con `id=456` via el deep link handler.

Si `/mobile-push-notifications` NO está instalado: documentar la integración en `docs/deep-link-testing.md` para implementarla cuando se configure push.

### Test Commands (`docs/deep-link-testing.md`)

**iOS Simulator**:
```bash
# Abrir un product deep link
xcrun simctl openurl booted "https://app.miproducto.com/product/123"

# Abrir un invite deep link
xcrun simctl openurl booted "https://app.miproducto.com/invite/abc123"

# Verificar que AASA es accesible desde Apple CDN
curl -I "https://app.miproducto.com/.well-known/apple-app-site-association"
# Debe retornar 200 con Content-Type: application/json
```

**Android Emulator**:
```bash
# Abrir un product deep link
adb shell am start -a android.intent.action.VIEW \
  -d "https://app.miproducto.com/product/123"

# Abrir un invite deep link
adb shell am start -a android.intent.action.VIEW \
  -d "https://app.miproducto.com/invite/abc123"

# Verificar que assetlinks es accesible
curl "https://app.miproducto.com/.well-known/assetlinks.json"
```

**Script de verificación** (`scripts/verify-deep-links.ts`): verifica que AASA y assetlinks son accesibles en producción y retornan JSON válido. Usado en CI pre-submit.

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] Dominio y rutas recolectados; rutas transformadas a wildcards para AASA
- [ ] `public/.well-known/apple-app-site-association` generado con TEAM_ID + routes correctas
- [ ] `public/.well-known/assetlinks.json` generado con package_name + sha256 placeholder documentado
- [ ] `src/navigation/deep-link-config.ts` con rutas que coinciden EXACTAMENTE con AASA components (contrato C)
- [ ] `src/services/deep-link-handler.ts` con handlers foreground + cold start
- [ ] Fallback web generado — ningún deep link resulta en 404 o página en blanco (contrato S)
- [ ] Banner "Descargá la app" con links correctos a App Store y Play Store
- [ ] Integración push documentada (o implementada si `/mobile-push-notifications` está instalado)
- [ ] `docs/deep-link-testing.md` con comandos `xcrun simctl openurl` y `adb shell am start`
- [ ] `scripts/verify-deep-links.ts` verifica AASA y assetlinks en producción
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment — C·S)_ |
| Artifacts | _(AASA, assetlinks, deep-link-config, deep-link-handler, fallback web, docs/testing, verify script)_ |
| Next Recommended | `/mobile-app-store-submit` (si no completado) o cierre del arco mobile |
| Risks | _(SHA256 fingerprint pendiente de completar; TEAM_ID si no disponible)_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Deep linking configurado y se quiere subir a stores | `/mobile-app-store-submit` |
| Se quiere medir clicks en deep links | `/mobile-analytics` |
| Push notifications no configuradas aún | `/mobile-push-notifications` |

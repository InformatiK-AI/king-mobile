# /mobile-deep-linking

Configura Universal Links (iOS) y App Links (Android) con `apple-app-site-association` (AASA) y `assetlinks.json`, fallback web con banner de store, y manejo de rutas desde notificaciones push.

## Uso

```
/mobile-deep-linking
```

El skill solicita dominio, rutas, y configuración del proyecto de forma interactiva (o los detecta desde `app.json` / `.king/knowledge/`).

## Qué hace

1. **Recolecta dominio + rutas** — inputs requeridos: dominio (`app.miproducto.com`), rutas (`/product/:id`, `/invite/:code`), TEAM_ID Apple, bundle ID iOS, package name Android
2. **Genera AASA + assetlinks** — archivos de verificación de dominio en `public/.well-known/`; SHA256 fingerprint como placeholder documentado
3. **Configura app router** — `deep-link-config.ts` con rutas que coinciden con AASA; `deep-link-handler.ts` con handlers foreground + cold start
4. **Fallback web** — middleware que detecta user agent → banner "Descargá la app" con link al store correcto cuando la app no está instalada
5. **Integración push** — conecta el deep link handler con el notification handler de `/mobile-push-notifications` (si instalado); payload `{ "route": "/product/123" }` abre la pantalla correcta al tap
6. **Comandos de test** — `docs/deep-link-testing.md` con `xcrun simctl openurl` y `adb shell am start`

## Outputs

| Archivo | Descripción |
|---------|-------------|
| `public/.well-known/apple-app-site-association` | AASA con applinks, TEAM_ID, y rutas como wildcards |
| `public/.well-known/assetlinks.json` | App Links Android con package_name y sha256 placeholder |
| `src/navigation/deep-link-config.ts` | Configuración de rutas para React Navigation o Expo Router |
| `src/services/deep-link-handler.ts` | Handler que parsea URL y navega a pantalla correcta |
| `docs/deep-link-testing.md` | Comandos para probar deep links en iOS Simulator y Android Emulator |
| `scripts/verify-deep-links.ts` | Script que verifica AASA y assetlinks accesibles en producción |

## Condiciones de bloqueo

- Sin dominio configurado → preguntar al usuario
- Sin rutas a linkear → preguntar al usuario (mínimo una ruta)
- Framework de navegación no detectable en modo no-interactivo

## Contrato CASTLE activo: C·S

- **C**: rutas en AASA coinciden exactamente con rutas en `deep-link-config.ts`
- **S**: fallback web presente — ningún deep link resulta en 404 o página en blanco

## Notas

- `apple-app-site-association` debe servirse sin extensión `.json`, con `Content-Type: application/json`, y SIN redirecciones desde el dominio
- `assetlinks.json` requiere el SHA256 fingerprint del keystore de producción — ver instrucciones en `docs/deep-link-testing.md`
- La integración push requiere `/mobile-push-notifications` instalado previamente; si no está, el skill documenta cómo conectarlo cuando se configure
- Probar siempre en dispositivo real o emulador — los simuladores pueden comportarse diferente en la resolución de AASA/assetlinks

## Skill

> `skills/mobile-deep-linking/SKILL.md`

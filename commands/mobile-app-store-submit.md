# /mobile-app-store-submit

Automatiza la submisión a App Store (iOS) y Play Store (Android) con EAS Submit. Incluye rejection-reason checker pre-submisión, privacy manifest iOS 17+, ASO metadata, screenshots guide, y CI workflow de submit en merge a `main`.

**Requiere**: `/mobile-deploy` configurado previamente (`eas.json` debe existir).

## Uso

```
/mobile-app-store-submit
```

Sin argumentos — el skill detecta la configuración del proyecto desde `eas.json`, `app.json`, y `package.json`.

## Qué hace

1. **Verifica EAS configurado** — confirma `eas.json`, bundle IDs, y builds disponibles. BLOQUEA si no existe.
2. **Rejection checker** — 6 checks (4 bloqueantes, 2 advertencias) antes de cualquier submit:
   - Privacy manifest `ios/PrivacyInfo.xcprivacy` (BLOQUEANTE)
   - NSUsage strings por permiso en `app.json` (BLOQUEANTE)
   - Icon sizes 1024×1024 (BLOQUEANTE)
   - Sin test credentials en código (BLOQUEANTE)
   - Content rating en `eas.json` (ADVERTENCIA)
   - Privacy policy URL en `app.json` (ADVERTENCIA)
3. **Privacy manifest** — genera `ios/PrivacyInfo.xcprivacy` con `NSPrivacyAccessedAPITypes` según dependencias de `package.json`
4. **ASO metadata** — genera `store/ios/metadata.json`, `store/android/metadata.json`, `store/screenshots/README.md`
5. **CI workflow** — genera `.github/workflows/store-submit.yml` (submit automático en merge a `main`)
6. **Submit** — ejecuta `eas submit --platform ios --latest` y `--platform android --latest`; registra en `.king/store/submit-history.json`

## Outputs

| Archivo | Descripción |
|---------|-------------|
| `ios/PrivacyInfo.xcprivacy` | Privacy manifest iOS 17+ con tipos por dependencia |
| `store/ios/metadata.json` | ASO metadata App Store (title 30, subtitle 30, description 4000, keywords 100) |
| `store/android/metadata.json` | ASO metadata Play Store (title 50, short 80, full 4000) |
| `store/screenshots/README.md` | Guía de screenshots requeridos por plataforma (no genera imágenes) |
| `.github/workflows/store-submit.yml` | CI workflow: pre-submit-check → eas submit en merge a main |
| `scripts/pre-submit-check.ts` | Checker ejecutable localmente: reporta PASS/FAIL/WARN por check |
| `docs/apple-developer-setup.md` | Pasos manuales de Apple Developer Account (no automatizables) |
| `.king/store/submit-history.json` | Historial de submisiones append-only |

## Condiciones de bloqueo

- `eas.json` no existe → ejecutar `/mobile-deploy` primero
- Privacy manifest ausente → generado automáticamente en Phase 3
- NSUsage string faltante → mostrar qué permisos necesitan descripción
- Icon 1024×1024 ausente → indicar cómo corregir en `app.json`
- Test credentials detectadas → indicar línea y archivo donde aparecen

## Notas

- La primera submisión a App Store requiere pasos manuales en Apple Developer Portal (ver `docs/apple-developer-setup.md`)
- EAS Submit gestiona la autenticación con App Store Connect y Google Play API via `EXPO_TOKEN`
- El skill NO sube screenshots automáticamente — genera la guía con dimensiones y contenido sugerido
- `APPLE_ID`, `ASC_API_KEY`, `GOOGLE_PLAY_KEY_FILE` nunca en código — solo en GitHub Secrets

## Skill

> `skills/mobile-app-store-submit/SKILL.md`

# mobile-saas-starter — Spec del Template V8

> Spec del template `mobile-saas-starter` para V8. Esta es la fuente de verdad para su implementación.
> El código del template se implementa en V8 DESPUÉS de que los 6 skills de M10 estén completos.

---

## Stack

| Capa | Librería / Servicio | Versión baseline |
|------|--------------------|--------------------|
| Framework | React Native + Expo SDK 53 + Expo Router | SDK 53 |
| Auth + DB + Storage | Supabase | latest |
| Subscriptions + Paywalls | RevenueCat | latest |
| Analytics + Session Replay | Posthog | latest |
| Crash + Performance | Sentry | latest |
| Build + Submit | EAS Build + EAS Submit | latest |
| Push Notifications | Expo Notifications (FCM + APNs) | `expo-notifications` ~0.29.x |
| Offline Sync | WatermelonDB | latest |

---

## Features Out-of-the-Box

| Feature | Skill que la provee | Config incluida |
|---------|--------------------|--------------------|
| Push notifications | `/mobile-push-notifications` | FCM + APNs + token registry + opt-in WCAG |
| Offline sync | `/mobile-offline-sync` | WatermelonDB + outbox pattern + conflict LWW |
| Biometric auth | `/mobile-biometric-auth` | Keychain/Keystore + FaceID/Fingerprint + fallback |
| Analytics | `/mobile-analytics` | Posthog + eventos predefinidos consumer + PII masking |
| Store submission | `/mobile-app-store-submit` | EAS Submit + privacy manifest iOS 17+ + ASO metadata |
| Deep linking | `/mobile-deep-linking` | AASA + assetlinks.json + Expo Router config + fallback web |
| Navigation | Expo Router | Tabs + modal + stack structure preconfigurados |
| Subscriptions | RevenueCat | Paywall Builder + A/B testing + sandbox SKU |
| Dark mode + i18n | Tokens + i18next | inglés + español precargados |
| CI/CD | EAS pipelines | Channels: `preview` → `internal` → `production` |
| Store-ready | — | Privacy manifest iOS 17+ preconfigurado + ASO templates |

---

## Criterios de Done del Template

> Criterios de aceptación cuando el template se implemente en V8:

- [ ] `expo run:ios` + `expo run:android` pasan en CI desde clone limpio (sin configuración adicional)
- [ ] RevenueCat paywall renderiza en sandbox con SKU real configurado
- [ ] Push notification de prueba se entrega en iOS y Android (Expo Notifications + token registry activo)
- [ ] 5 operaciones offline → sync sin pérdida en test automatizado (WatermelonDB + outbox)
- [ ] README.md con setup completo en ≤ 5 pasos

---

## Dependencia y Secuencia

El template `mobile-saas-starter` **no se implementa en M10**. M10 entrega los 6 skills de infraestructura; el template se construye sobre ellos en V8.

Secuencia requerida antes de implementar el template:

1. `/mobile-push-notifications` — completo (M10 Sprint 4-5)
2. `/mobile-offline-sync` — completo (M10 Sprint 4-5)
3. `/mobile-biometric-auth` — completo (M10 Sprint 5-6)
4. `/mobile-analytics` — completo (M10 Sprint 5-6)
5. `/mobile-app-store-submit` — completo (M10 Sprint 6-7)
6. `/mobile-deep-linking` — completo (M10 Sprint 6-7)
7. **V8**: scaffold del template integrando todos los skills anteriores

Esta spec es la fuente de verdad. No modificar sin actualizar el plan de M10.

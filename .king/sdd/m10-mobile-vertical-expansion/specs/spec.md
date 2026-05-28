# Delta Specs — M10 Mobile Vertical Expansion

> Requirements en RFC 2119. Los escenarios Given/When/Then completos viven en
> `mejora/planes-detallados/M10-mobile-vertical-expansion.md` §7 (Gherkin) — esta spec referencia
> esos escenarios como criterio de aceptación y captura solo los requirements normativos por capability.

## REQ-1 — mobile-push-notifications

- El skill **MUST** generar cliente FCM (Android) + APNs (iOS) según framework detectado (Expo / bare RN / Flutter).
- El token registry **MUST** soportar revocación (`revokedAt`) en logout; **SHALL** tener TTL.
- El skill **MUST NOT** generar tokens hardcodeados ni API keys en código cliente (solo `.env`).
- El opt-in **MUST** mostrarse después del onboarding principal, nunca en splash ni en los 2 primeros pasos.
- Acceptance: M10 §7 / M-85.1 (4 escenarios). CASTLE: C·S·E.

## REQ-2 — mobile-biometric-auth

- El skill **MUST** almacenar access tokens solo en Keychain (iOS) / Keystore (Android), **MUST NOT** en AsyncStorage.
- El skill **MUST** BLOQUEAR si detecta AsyncStorage usado para session tokens, con mensaje de migración.
- **MUST** ofrecer fallback a password tras 3 intentos biométricos fallidos.
- El enrollment **MUST** ser opt-in post-login; si se rechaza, no re-preguntar en 7 días.
- Acceptance: M10 §7 / M-85.3 (4 escenarios). CASTLE: S·T·E.

## REQ-3 — mobile-analytics

- El skill **MUST** generar eventos tipados por tipo de producto (SaaS / consumer) con propiedades.
- **MUST** instrumentar screen tracking automático según el router detectado.
- Session replay **MUST** estar off por defecto; si se activa, PII masking **MUST** estar activo.
- El tracking **MUST NOT** iniciar sin consentimiento explícito (GDPR consent flow en onboarding).
- Acceptance: M10 §7 / M-85.4 (4 escenarios). CASTLE: C·S·E.

## REQ-4 — mobile-offline-sync

- El skill **MUST** implementar DB local (WatermelonDB RN / Drift Flutter) + outbox pattern.
- Las mutaciones offline **MUST** persistir con `status: pending` y reflejarse en UI inmediatamente.
- El sync **MUST** ser FIFO al reconectar; conflict resolution **SHALL** ser last-write-wins con audit log.
- **MUST** generar test automatizado: 5 ops offline → sync → 0 pérdidas, 0 conflictos en base.
- El indicador de sync **MUST** ser no intrusivo (visible solo con pendientes).
- Acceptance: M10 §7 / M-85.2 (4 escenarios). CASTLE: C·S·T·L.

## REQ-5 — mobile-app-store-submit

- El skill **MUST** BLOQUEAR si `eas.json` no existe (prerequisito `/mobile-deploy`).
- El rejection checker **MUST** ejecutarse ANTES de submit y BLOQUEAR sin privacy manifest iOS17+ o NSUsage strings faltantes.
- **MUST** generar `ios/PrivacyInfo.xcprivacy` con los tipos detectados de las dependencias.
- **MUST NOT** permitir submit con test credentials en código.
- Acceptance: M10 §7 / M-85.5 (4 escenarios). CASTLE: C·S·T·E.

## REQ-6 — mobile-deep-linking

- El skill **MUST** generar AASA + `assetlinks.json` con las rutas declaradas por el usuario.
- Las rutas en AASA/assetlinks **MUST** coincidir con el router de la app (contrato C).
- **MUST** proveer fallback web cuando la app no está instalada.
- **SHOULD** conectar con el handler de push de `/mobile-push-notifications` si está instalado.
- Acceptance: M10 §7 / M-85.6 (4 escenarios). CASTLE: C·S.

## REQ-7 — Agente mobile con contratos bilaterales

- **MUST** existir los 5 contratos en `agents/_common/contracts/`: `mobile-developer`, `mobile-frontend`,
  `mobile-performance`, `mobile-security`, `mobile-qa`.
- Cada contrato **MUST** tener: Propósito, Escenarios de Interacción, ≥1 Request Format, ≥1 Response Format
  (mismo schema que `devops-mobile.md`).
- `agents/mobile.md` **MUST** referenciar los 5 contratos en una sección "Contratos bilaterales activos".
- Acceptance: M10 §7 / "Contratos bilaterales mobile.md" (2 escenarios).

## REQ-8 — Knowledge + Template spec

- **MUST** existir `knowledge/_inject/mobile-production-patterns.md` (≤120 líneas), consolidando los
  patrones de los 6 skills.
- **MUST** existir `knowledge/_inject/mobile-saas-starter-spec.md` con stack, features y criterios de done del template V8.

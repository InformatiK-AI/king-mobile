# Mobile Production Patterns (para inyección)

> Reference doc slim para agente @mobile. Cubre los 6 dominios M10. Referencia a skills en `skills/`.

## Push Notifications — Token Registry + Opt-in

**Patrón**: Token registry server-side con revocación explícita.

```typescript
interface PushToken {
  userId: string; token: string; platform: "ios" | "android";
  createdAt: Date; lastUsedAt: Date; revokedAt?: Date;
}
// TTL: revocar en logout + 30 días de inactividad
```

- Agrupar envíos por platform → FCM v1 batch + APNs HTTP/2.
- Opt-in post-onboarding: nunca en splash ni primeros 2 pasos. Botón "Más tarde" silencia 7 días.
- `send-to-cohort` filtra `revokedAt != null` antes de despachar.

**Anti-patrón**: enviar a tokens revocados; hardcodear `FCM_SERVER_KEY` en cliente.

---

## Offline-First — Outbox + Conflict LWW + Audit Log

**Patrón**: toda mutación offline escribe en outbox ANTES de intentar sync.

```typescript
interface OutboxEntry { id: string; table: string; op: "create"|"update"|"delete";
  payload: object; updated_at: string; retries: number; }
// Conflict resolution: Last-Write-Wins por updated_at (timestamp del servidor)
```

- Cola outbox en FIFO estricto; retry exponencial con tope 5 intentos.
- Conflictos registrados en `.king/sync/conflict-log.json` — nunca silenciosos.
- `SyncStatusBanner` visible solo con pendientes; oculto en estado normal.

**Anti-patrón**: resolver conflictos sin audit log; perder una mutación offline.

---

## Biometric Auth — Keychain/Keystore, nunca AsyncStorage

**Patrón**: tokens de sesión exclusivamente en Keychain (iOS) / Keystore (Android).

```
Keys: {bundleId}.access_token | {bundleId}.biometric_enrolled
      {bundleId}.bio_enroll_declined_at | {bundleId}.bio_fail_log
```

- `authenticate()`: máximo 3 intentos biométricos → fallback a password.
- `revoke()` cableado al logout — elimina token + flag de enrollment.
- `BiometricSetupScreen` aparece post-login exitoso; "Ahora no" silencia 7 días.

**Anti-patrón**: `AsyncStorage.setItem` con session token → BLOCKING CONDITION.

---

## Analytics — Eventos Tipados + PII Masking + Consent GDPR

**Patrón**: gate de consentimiento obligatorio antes del primer evento de negocio.

```typescript
// events.ts: union type — track() falla en compilación si el evento no existe
type TrackEvent = "onboarding_completed" | "subscription_started" | ...;
// privacy.ts: PII field → reemplazar con "***" completo, nunca truncar
```

- Session replay OFF por defecto; opt-in explícito.
- `ANALYTICS_API_KEY` solo en `.env`; provider init falla si no está presente.
- Screen tracking automático via wrapper del navegador (Expo Router / React Navigation).

**Anti-patrón**: enviar eventos antes del consentimiento; masking parcial de PII.

---

## Store Submission — Privacy Manifest iOS 17+ + Rejection Checker

**Patrón**: ejecutar rejection checker ANTES de `eas submit` — siempre.

Checks bloqueantes (4 de 6):
1. `NSPrivacyAccessedAPITypes` vacío con deps que lo requieren → block.
2. Credenciales de test (`test@`, `password123`) en código → block.
3. Permisos sin NSUsageDescription string → block.
4. Builds de producción con sandbox/test keys → block.

- `ios/PrivacyInfo.xcprivacy` generado según deps del `package.json`.
- Registro obligatorio en `.king/store/submit-history.json` tras cada submit.

**Anti-patrón**: `eas submit` sin pasar el checker; `NSPrivacyAccessedAPITypes` vacío.

---

## Deep Linking — AASA + assetlinks + Fallback Web

**Patrón**: rutas en AASA y `deep-link-config.ts` deben coincidir exactamente — cualquier ruta en AASA sin handler es un bug de contrato.

```json
// AASA: usar placeholder TEAM_ID si no se conoce — nunca inventar el valor
// assetlinks.json: usar placeholder ${SHA256_FINGERPRINT} — instrucción explícita
```

- Fallback web obligatorio: detección de user agent → banner de store si app no instalada.
- Integración con push: payload `route` en notificación → handler de deep link.
- Fase 0 recolecta dominio + rutas antes de generar AASA/assetlinks.

**Anti-patrón**: omitir fallback web; `appIDs` con TEAM_ID inventado en AASA.

---

## Señales de Alerta Cross-Cutting

| Señal | Dominio | Acción |
|-------|---------|--------|
| `AsyncStorage` + `token`/`session` | biometric | BLOQUEAR — migrar a Keychain/Keystore |
| Evento analytics antes del consent gate | analytics | BLOQUEAR — gate es obligatorio |
| `eas submit` sin rejection checker | store | BLOQUEAR — ejecutar checker primero |
| Token en outbox sin `updated_at` | offline | BLOQUEAR — campo obligatorio para LWW |
| Push token sin `revokedAt` en schema | push | BLOQUEAR — revocación es requisito |

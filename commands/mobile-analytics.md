---
name: mobile-analytics
description: "Configurar analytics mobile con eventos tipados y GDPR compliance. Usar cuando se necesite: agregar analytics, eventos tipados mobile, screen tracking automático, session replay react native, PII masking analytics, GDPR analytics mobile, posthog react native, amplitude react native, mixpanel react native, funnel analysis mobile, cohort analysis mobile, consent flow analytics, opt-out analytics, catálogo de eventos SaaS, catálogo de eventos consumer."
argument-hint: "[--provider=posthog|amplitude|mixpanel] [--product=saas|consumer] [--session-replay]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - Agent
  - AskUserQuestion
---

# /mobile-analytics

Configura analytics mobile con catálogo de eventos tipados por tipo de producto (SaaS / Consumer), screen tracking automático del navegador detectado, singleton `track()` / `identify()` / `screen()`, capa de privacidad GDPR con consentimiento explícito en onboarding, y session replay opt-in con PII masking (Posthog). Soporta Expo, bare React Native y Flutter.

## Instrucciones

1. Invocar el skill usando la herramienta Skill con `skill: king-mobile:mobile-analytics`
2. El skill detecta automáticamente el framework, el navegador y el tipo de producto si no se especifican
3. Seguir todas las fases del skill en orden:
   - Phase 0 (Load Context) → Phase 1 (detect framework + select provider) → Phase 2 (setup SDK) → Phase 3 (define events) → Phase 4 (screen tracking) → Phase 5 (privacy + consent + optional session replay) → Phase N+1
4. Argumentos opcionales:
   - `--provider=posthog|amplitude|mixpanel` → omite la selección interactiva de provider
   - `--product=saas|consumer` → omite la detección de tipo de producto
   - `--session-replay` → activa la configuración de session replay (solo Posthog; opt-in, OFF por defecto)
5. Prerequisito: proyecto mobile scaffoldeado (`/mobile-scaffold`)

## Output esperado

- `src/analytics/events.ts` — catálogo tipado de eventos con propiedades (SaaS: `app_opened`, `onboarding_completed`, `first_feature_used`, `plan_upgrade_viewed`, `plan_upgraded`; Consumer: `app_opened`, `content_viewed`, `social_action`, `push_opted_in`, `session_ended`; universal: `screen_viewed`)
- `src/analytics/analytics.ts` — singleton con `track<T>()`, `identify()`, `screen()`; no inicializa hasta consentimiento; inyectable con mock
- `src/analytics/privacy.ts` — `hasConsent()`, `grantConsent()`, `revokeConsent()`, `maskPII()`; masking de email, tarjeta y teléfono antes del envío
- `src/navigation/analytics-wrapper.tsx` — wrapper del navegador detectado; dispara `screen_viewed` automáticamente con nombre de pantalla + timestamp en cada cambio
- `.env.example` actualizado con `ANALYTICS_API_KEY`

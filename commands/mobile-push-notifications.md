---
name: mobile-push-notifications
description: "Configurar push notifications. Usar cuando se necesite: agregar notificaciones push, configurar FCM, configurar APNs, push en React Native, push en Flutter, notificaciones mobile, token registry de push, segmentación de notificaciones, opt-in de notificaciones."
argument-hint: "[--framework=expo|rn|flutter]"
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

# /mobile-push-notifications

Configura notificaciones push (FCM + APNs) con token registry server-side, revocación, segmentación por cohort y opt-in WCAG-compliant sobre una app mobile existente.

## Instrucciones

1. Invocar el skill usando la herramienta Skill con `skill: king-mobile:mobile-push-notifications`
2. El skill detecta automáticamente el framework (Expo / bare RN / Flutter) si no se especifica
3. Seguir todas las fases del skill en orden:
   - Phase 0 (Load Context) → Phase 1 (detect + config) → Phase 2 (client) → Phase 3 (token registry) → Phase 4 (opt-in) → Phase 5 (test send) → Phase N+1
4. El argumento `--framework` omite la detección interactiva
5. Prerequisito: proyecto mobile scaffoldeado (`/mobile-scaffold`)

## Output esperado

- Cliente de notificaciones con handler foreground/background
- Token registry server-side con schema `PushToken` y revocación en logout
- Helper de segmentación por cohort (`send-to-cohort`)
- Opt-in que no interrumpe el onboarding (post-onboarding, "más tarde" = 7 días)
- `.env.example` con claves de push y `docs/push-setup.md`

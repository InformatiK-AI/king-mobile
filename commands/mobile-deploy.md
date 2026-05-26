---
name: mobile-deploy
description: "Configurar CI/CD para App Store y Google Play. Usar cuando se necesite: configurar deploy mobile, setup CI/CD app, publicar en App Store, publicar en Google Play, configurar GitHub Actions mobile, automatizar submission iOS, automatizar submission Android, configurar App Store Connect, configurar Google Play Service Account, signing artifacts, fastlane mobile, pipeline mobile."
argument-hint: "[--platform=ios|android|both]"
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

# /mobile-deploy

Configura CI/CD de submission automatizada a iOS App Store y Google Play Store via GitHub Actions, con gestión activa de credenciales y validación de signing artifacts.

## Instrucciones

1. Invocar el skill usando la herramienta Skill con `skill: king-mobile:mobile-deploy`
2. Requiere proyecto mobile existente (ejecutar `/mobile-scaffold` primero si no existe)
3. Seguir todas las fases del skill en orden:
   - Phase 0 (Load Context) → DISCOVERY.md (Fases 1-2) → IOS.md (Fase 3) → ANDROID.md (Fase 4) → CI.md (Fases 5-7) → Phase N+1
4. El argumento `--platform` limita la configuración a una sola plataforma

## Output esperado

- `.github/workflows/ios-deploy.yml` con pipeline completo de submission a App Store
- `.github/workflows/android-deploy.yml` con pipeline completo a Google Play
- Instrucciones para configurar secrets en GitHub (NEVER en el chat)
- Validación de signing certificates y provisioning profiles

## Seguridad

- NUNCA solicitar App Store Connect API Key ni Google Play Service Account JSON en el chat
- Credenciales se configuran exclusivamente via GitHub Secrets
- Phase 6 ejecuta gate CASTLE S (validación de seguridad obligatoria)

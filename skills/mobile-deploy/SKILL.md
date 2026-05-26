---
name: mobile-deploy
version: 1.0
description: "Configura CI/CD para submission automatizada a iOS App Store y Google Play Store via GitHub Actions, con gestión activa de credenciales (App Store Connect API + Google Play Service Account)."
---

# Mobile Deploy — CI/CD para App Store y Google Play

Skill para configurar CI/CD de submission automatizada a iOS App Store y Google Play Store via GitHub Actions, con gestión activa de credenciales y validación de signing artifacts.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Stack del proyecto | Yes | project |
| `.king/knowledge/environments.md` | Configuración de ambientes | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | EAS, Fastlane, bridges nativos | Yes | framework |
| `knowledge/_inject/devops-essentials.md` | CI/CD patterns, GitHub Actions | Yes | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS

- Proyecto mobile no detectado (no package.json con deps RN ni pubspec.yaml Flutter)
- `--target` no especificado y modo no-interactivo
- Secrets requeridos no configurados en el repositorio GitHub
- Certificado iOS vencido (expiración < 0 días)
- Keystore Android no encontrado en el path especificado
- Repositorio no tiene remote de GitHub configurado (`git remote -v` vacío)

### ABSOLUTE RESTRICTIONS

- NUNCA generar steps CI con `env`, `printenv`, `echo ${{ secrets.* }}`, `cat *.p8`, `cat *.json`, `set -x` en scope de credenciales
- NUNCA hardcodear credenciales en workflows — siempre `${{ secrets.X }}`
- NUNCA usar `--verbose` en lanes de Fastlane en jobs de producción
- NUNCA hacer `cat` de `AuthKey_*.p8` ni de ningún archivo de certificado en CI
- NUNCA saltar el gate de @security en Phase 6 (CI generation)
- NUNCA continuar si el CASTLE Assessment retorna BREACHED

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- `.github/workflows/ios-deploy.yml` (si --target ios | both)
- `.github/workflows/android-deploy.yml` (si --target android | both)
- `fastlane/Fastfile` + `fastlane/Appfile` (si proyecto = RN bare o Flutter)
- `CREDENTIALS-SETUP.md` con instrucciones de setup y fechas de expiración
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
Phase 0 (Load Context) → [DISCOVERY.md] Phase 1 (Project Detection) → Phase 2 (Secrets Validation)
→ [IOS.md] Phase 3 (iOS Credentials) → [ANDROID.md] Phase 4 (Android Credentials)
→ [CI.md] Phase 5 (GitHub Actions) → Phase 6 (CASTLE S Gate) → Phase 7 (Report)
→ Phase N+1 (Write Session)
```

> Phase 3 solo si --target = ios | both
> Phase 4 solo si --target = android | both

## CASTLE: C·A·S·T·L·E

- **C (Contracts)**: Workflows generados cumplen schema YAML de GitHub Actions
- **A (Architecture)**: Skill sigue convenciones King v2.0 (phases, gates, session, PHASE ROUTER)
- **S (Security)**: Sin credenciales hardcodeadas; gate @security obligatorio en Phase 6
- **T (Testing)**: CHECKPOINT de validación de secrets y signing antes de generar CI
- **L (Logging)**: `CREDENTIALS-SETUP.md` generado + session document
- **E (Environment)**: Workflows parametrizados por ambiente (staging / production)

## Agentes

- **@mobile** → validación de App Store / Play Store requirements y signing
- **@devops** → revisión de GitHub Actions workflows generados
- **@security** → gate OBLIGATORIO en Phase 6 antes de finalizar

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

## PHASE ROUTER

> Cargar sub-archivo según fase activa:
> - **Phase 1-2**: Leer `skills/mobile-deploy/DISCOVERY.md`
> - **Phase 3**: Leer `skills/mobile-deploy/IOS.md` (solo si --target ios | both)
> - **Phase 4**: Leer `skills/mobile-deploy/ANDROID.md` (solo si --target android | both)
> - **Phase 5-7**: Leer `skills/mobile-deploy/CI.md`

## FINAL CHECKPOINT

- [ ] Workflows GitHub Actions generados y validados sintácticamente (YAML válido)
- [ ] grep en workflows: 0 matches de credenciales hardcodeadas o logging de secrets
- [ ] Cada job tiene `permissions: {}` explícito
- [ ] Cada workflow tiene step con `if: failure()` para notificación de errores
- [ ] `CREDENTIALS-SETUP.md` generado con fechas de expiración y guía de rotación
- [ ] @security gate completado en Phase 6
- [ ] Backup de Android Keystore documentado y confirmado por el usuario

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment)_ |
| Artifacts | _(paths de workflows generados)_ |
| Next Recommended | `/review` — code review de los workflows generados |
| Risks | _(riesgos activos o "None")_ |

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Deploy configurado exitosamente | `/review` — review de workflows CI/CD |
| Secrets faltantes detectados | Configurar secrets con `gh secret set`, luego reintentar |
| Certificado iOS vencido | Renovar en developer.apple.com, luego reintentar |

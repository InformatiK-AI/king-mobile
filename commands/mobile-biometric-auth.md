---
name: mobile-biometric-auth
description: "Implementar autenticación biométrica. Usar cuando se necesite: agregar FaceID, agregar TouchID, agregar huella digital, biometric login, autenticación biométrica mobile, Keychain iOS tokens, Keystore Android tokens, secure storage para sesión, biometric enrollment, reemplazar AsyncStorage para tokens, migrar tokens a Keychain, migrar tokens a Keystore."
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

# /mobile-biometric-auth

Implementa autenticación biométrica (FaceID / TouchID en iOS, Fingerprint / Face Authentication en Android) sobre una app mobile existente. Almacena los access tokens exclusivamente en Keychain (iOS) o Keystore (Android). Detecta automáticamente el framework y si existe auth previa de `/auth-in-one-command`. BLOQUEA si detecta AsyncStorage usado para session tokens y guía la migración.

## Instrucciones

1. Invocar el skill usando la herramienta Skill con `skill: king-mobile:mobile-biometric-auth`
2. El skill detecta automáticamente el framework (Expo / bare RN / Flutter) si no se especifica
3. Seguir todas las fases del skill en orden:
   - Phase 0 (Load Context) → Phase 1 (detect framework + existing auth) → Phase 2 (biometric service) → Phase 3 (secure storage) → Phase 4 (enrollment screen + integrate auth flow) → Phase N+1
4. El argumento `--framework` omite la detección interactiva
5. Prerequisito: proyecto mobile scaffoldeado (`/mobile-scaffold`)

## Output esperado

- Servicio biométrico `biometric-auth.ts` con `enroll()`, `authenticate()` (fallback tras 3 intentos) y `revoke()`
- Wrapper `secure-storage.ts` con key `{bundleId}.access_token` en Keychain/Keystore — cero AsyncStorage
- `BiometricSetupScreen` opt-in post-login (rechazar = silencio 7 días)
- Hook `useBiometric.ts` con `{ isEnrolled, isAvailable, authenticate, enroll, revoke }`
- Integración con `AuthScreen` existente o generación de auth básico si no hay auth previa

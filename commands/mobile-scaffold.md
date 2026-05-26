---
name: mobile-scaffold
description: "Generar proyecto mobile completo. Usar cuando se necesite: crear app mobile, generar proyecto React Native, generar proyecto Flutter, scaffold mobile, nueva app iOS Android, configurar Expo, setup mobile con navegación, configurar state management mobile, inicializar app mobile, crear proyecto Expo managed, crear proyecto Expo bare."
argument-hint: "[--framework=rn|flutter] [--expo=managed|bare]"
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

# /mobile-scaffold

Genera proyectos React Native (Expo managed o bare) y Flutter con navegación, state management y native bridges preconfigurados, listos para desarrollo inmediato.

## Instrucciones

1. Invocar el skill usando la herramienta Skill con `skill: king-mobile:mobile-scaffold`
2. El skill detecta automáticamente el framework si no se especifica
3. Seguir todas las fases del skill en orden:
   - Phase 0 (Load Context) → DISCOVERY.md (Fases 1-2) → RN.md o FLUTTER.md (Fases 3-5) → Phase N+1
4. El argumento `--framework` omite la detección interactiva
5. El argumento `--expo` selecciona el workflow de Expo (solo para React Native)

## Output esperado

- Proyecto mobile completo con estructura de carpetas estándar
- Navegación pre-configurada (React Navigation o Go Router)
- State management integrado (Zustand/Redux o Riverpod/Bloc)
- Native bridges para cámara, storage y permisos
- `.env.example` y `.gitignore` configurados

---
name: mobile-scaffold
version: 1.0
description: "Genera proyectos React Native (Expo managed o bare) y Flutter con navegación, state management y native bridges preconfigurados."
---

# /mobile-scaffold — Generación de Proyectos Mobile

Skill standalone para generar proyectos mobile completos (React Native o Flutter) con navegación, state management y native bridges preconfigurados, listos para desarrollo inmediato.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Stack del proyecto | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones, frameworks, native bridges | Yes | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- Framework no especificado y modo no-interactivo activo
- Directorio destino existe y no está vacío (sin confirmación explícita del usuario)
- Combinación framework+state incompatible (ej: flutter+zustand, react-native+riverpod)
- CLI del framework no instalada en el entorno

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA generar archivos con credenciales o API keys reales
- NUNCA sobreescribir directorio existente sin confirmación explícita del usuario
- NUNCA saltar el CHECKPOINT de validación de entorno (Phase 1)
- NUNCA asumir un framework por defecto — preguntar si no se especifica

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- Proyecto mobile generado en el directorio del usuario
- README.md con comandos de setup ejecutables (iOS + Android)
- .gitignore con entradas mobile-específicas (*.p8, *.keystore, google-services.json, etc.)
- Session document en .king/sessions/

### PHASES OVERVIEW

```
PHASE 0     PHASE 1      PHASE 2       PHASE 3       PHASE 4        PHASE 5        PHASE N+1
SESSION  →  ENV        →  FRAMEWORK  →  SCAFFOLD   →  NATIVE      →  README +    →  SESSION
(registry)  (validate     (select +     (genera       (bridges       .gitignore)    (escribir
            CLI + env)    configure)    proyecto)     + permisos)                   session)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Validación de constraints de plataforma
- **@architect** → Validación de estructura del proyecto generado

## CASTLE ACTIVO: C·A·S·T·L·_

- **C (Contracts)**: Proyecto generado cumple estructura esperada del framework elegido
- **A (Architecture)**: Skill sigue convenciones King v2.0 (phases, gates, session, PHASE ROUTER)
- **S (Security)**: Sin credenciales ni API keys reales en archivos generados
- **T (Testing)**: CHECKPOINT de validación de entorno obligatorio antes de scaffold
- **L (Logging)**: README.md con comandos ejecutables y session document

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## PHASE ROUTER

> Cargar sub-archivo según fase activa:
> - **Phase 1-2**: Leer `skills/mobile-scaffold/DISCOVERY.md`
> - **Phase 3-5** (React Native): Leer `skills/mobile-scaffold/RN.md`
> - **Phase 3-5** (Flutter): Leer `skills/mobile-scaffold/FLUTTER.md`

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] Proyecto generado en directorio del usuario
- [ ] README.md existe con sección "Getting Started" y comandos ejecutables
- [ ] .gitignore contiene entradas mobile-específicas (*.p8, *.keystore, google-services.json)
- [ ] Navegación configurada con archivos de config correctos e imports válidos
- [ ] State management generado con patrón correcto (no stubs vacíos)
- [ ] Permisos en AndroidManifest.xml e Info.plist para cada bridge seleccionado
- [ ] Combinación framework+state válida verificada
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment)_ |
| Artifacts | _(directorio generado, README.md)_ |
| Next Recommended | `/mobile-deploy` — configurar CI/CD para el proyecto generado |
| Risks | _(riesgos activos o "None")_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Scaffold generado exitosamente | `/mobile-deploy` — configurar deployment |
| Usuario quiere ajustar stack | Repetir `/mobile-scaffold` con parámetros distintos |

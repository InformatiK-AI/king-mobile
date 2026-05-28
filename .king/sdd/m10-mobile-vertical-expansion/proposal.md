# Proposal — M10 Mobile Vertical Expansion

> Fuente de verdad: `mejora/planes-detallados/M10-mobile-vertical-expansion.md`
> Plugin destino: `king-mobile` · Rama base: `develop` · Persistencia: openspec

## Intent

Completar `king-mobile` más allá de scaffold+deploy (P0 ya hechos) con los 6 skills de producción
que un founder mobile necesita para lanzar en 2026, y deshuérfanar el agente `mobile.md` con sus
5 contratos bilaterales. Hoy `mobile-scaffold` + `mobile-deploy` cubren ~20% del trabajo real; M10
cubre el 80% restante (push, offline, biometría, analytics, store submission, deep linking).

## Scope

**Incluye:**
- 6 skills nuevos: `mobile-push-notifications`, `mobile-biometric-auth`, `mobile-analytics`,
  `mobile-offline-sync`, `mobile-app-store-submit`, `mobile-deep-linking` (SKILL.md + command c/u).
- 5 contratos bilaterales en `agents/_common/contracts/`: `mobile-developer`, `mobile-frontend`,
  `mobile-performance`, `mobile-security`, `mobile-qa`.
- Modificación de `agents/mobile.md` (sección "Contratos bilaterales activos").
- 2 knowledge inject: `mobile-production-patterns.md` (≤120 líneas), `mobile-saas-starter-spec.md`.

**Excluye (no tocar):** `skills/mobile-scaffold`, `skills/mobile-deploy` (P0 completos).
**Difiere:** implementación del código del template `mobile-saas-starter` (es V8; aquí solo la spec).

## Capabilities afectadas

| Capability | Acción |
|------------|--------|
| `mobile-push` | NUEVA — FCM+APNs, token registry con revocación, opt-in WCAG |
| `mobile-biometric` | NUEVA — Keychain/Keystore, bloqueo de AsyncStorage para tokens |
| `mobile-analytics` | NUEVA — eventos tipados, screen tracking, PII masking, GDPR consent |
| `mobile-offline` | NUEVA — WatermelonDB/Drift, outbox, conflict resolution, test offline |
| `mobile-store-submit` | NUEVA — rejection checker, privacy manifest iOS17+, ASO, EAS Submit |
| `mobile-deep-linking` | NUEVA — AASA + assetlinks, fallback web, integración push |
| `agent:mobile` | EXTENDIDA — 5 contratos bilaterales operativos |

## Approach

Ejecución **sprint por sprint** (M10 §6): cada sprint en un worktree desde `develop`, implementación
vía `sdd-apply`, luego `/review` → `/fix` (si aplica) → `/sdd-verify` (contra Gherkin §7 + CASTLE §8)
→ `/merge` a develop. 7 sprints. Cada SKILL.md clona la anatomía King v2.0 de `mobile-scaffold`.

`/refactor` y `/optimize` se omiten: los entregables son prosa estructurada (Markdown/YAML), no
código con comportamiento runtime ni complejidad algorítmica.

## Risks

| Riesgo | Mitigación |
|--------|------------|
| Inconsistencia de formato entre skills | Clonar frontmatter+anatomía de `mobile-scaffold` (api_version:1.0.0, sin BOM) |
| Contexto se llena en 7 sprints | Delegar cada sprint a sub-agente con contexto fresco + referencias a M10/patrón |
| Worktree sin contexto SDD | `.king/sdd/` commiteado en develop → worktrees lo heredan vía git |
| develop divergía de master | RESUELTO — backmerge FF master→develop ya aplicado |

## Rollback

Cada sprint es un merge atómico a develop. Rollback = `git revert` del merge del sprint, o reset de
develop al SHA previo al sprint. Los artefactos SDD en `.king/sdd/` no afectan runtime.

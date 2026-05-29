# Design — M10 Mobile Vertical Expansion

> El diseño técnico detallado por skill (fases, framework support, outputs verificables, schemas)
> vive en `mejora/planes-detallados/M10-mobile-vertical-expansion.md` §2. Este documento captura solo
> las **decisiones de arquitectura transversales** y el contrato de formato — no duplica §2.

## D1 — Anatomía de cada skill (King v2.0)

Cada SKILL.md clona la estructura de `skills/mobile-scaffold/SKILL.md`:

- **Frontmatter**: `name`, `version: 1.0`, `api_version: 1.0.0`, `description`. UTF-8 sin BOM.
- **Secciones**: título + descripción · Knowledge Injection · QUICK REFERENCE (BLOCKING CONDITIONS,
  ABSOLUTE RESTRICTIONS, REQUIRED OUTPUTS, PHASES OVERVIEW) · AGENTES INVOLUCRADOS · CASTLE ACTIVO ·
  detalle de fases · session-management (Phase N+1).
- **Command** asociado en `commands/<skill>.md` (wrapper delgado que invoca el skill).

## D2 — Mapa de fases por skill (de M10 §2)

| Skill | Fases |
|-------|-------|
| push-notifications | 0 detect-framework → 1 collect-config → 2 client-setup → 3 token-registry → 4 opt-in-screen → 5 test-send |
| biometric-auth | 0 detect-framework+existing-auth → 1 biometric-service → 2 secure-storage → 3 enrollment-screen → 4 integrate-auth |
| analytics | 0 detect+select-provider → 1 setup-sdk → 2 define-events → 3 screen-tracking → 4 privacy-consent → 5 session-replay |
| offline-sync | 0 detect+schema → 1 local-db → 2 outbox → 3 sync-engine → 4 conflict-resolution → 5 ui-indicator → 6 test-offline |
| app-store-submit | 0 verify-eas → 1 rejection-checker → 2 privacy-manifest → 3 aso-metadata → 4 ci-workflow → 5 submit |
| deep-linking | 0 collect-domain+routes → 1 aasa+assetlinks → 2 router-config → 3 fallback-web → 4 push-integration → 5 test-commands |

## D3 — Decisión: framework support multi-stack

Cada skill detecta el framework en Phase 0 y ramifica: **React Native Expo** (default), **bare RN**,
**Flutter**. Las librerías por framework están tabuladas en M10 §2 por skill. El skill **MUST NOT**
asumir framework — lo detecta de `package.json`/`pubspec.yaml` o lo pregunta.

## D4 — Decisión: contratos bilaterales = schema devops-mobile.md

Los 5 contratos siguen el patrón YAML de `agents/_common/contracts/devops-mobile.md`:
Propósito → tabla "Escenarios de Interacción" (Escenario·Iniciador·Receptor·Tipo·Bloquea) →
secciones por escenario con Request/Response Format YAML → Security Escalation con veto @security.

## D5 — Decisión: gates CASTLE declarativos por skill

Cada skill declara su línea CASTLE ACTIVO (de M10 §8). No hay enforcement runtime en el plugin —
el gate se materializa como sección declarativa en el SKILL.md y se verifica en `/sdd-verify` contra
la tabla §8. Los gates numéricos mobile (`bundle`, `cold_start`, `crash_free`, etc.) son para el
proyecto destino (quality-gates.yaml), no para el plugin.

## D6 — Decisión: integración cross-skill documentada, no acoplada

`deep-linking` referencia el handler de `push-notifications` de forma opcional (detecta si está
instalado). `biometric-auth` detecta auth existente de `/auth-in-one-command`. Ningún skill importa
código de otro — la integración es por detección + documentación, manteniendo cada skill standalone.

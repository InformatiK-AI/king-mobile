# Tasks — M10 Mobile Vertical Expansion

> 18 tareas atómicas (M10 §6), agrupadas por sprint. Granularidad: **un worktree + merge a develop por sprint**.
> Por sprint: `worktree create` → `sdd-apply` (tareas) → `/review` → `/fix` (si aplica) → `/sdd-verify` → `/merge` a develop.
> Acceptance por skill: Gherkin de M10 §7. Gates: M10 §8.

## Sprint 1 — Push Notifications  ·  worktree `feature/m10-sprint1`
- [x] T01: `skills/mobile-push-notifications/SKILL.md` — 6 fases (Phase 0 session + detect, client, token-registry, opt-in, test-send). DONE.
- [x] T02: `commands/mobile-push-notifications.md`. DONE. Merge 5a3f235.
- Acceptance: M10 §7 / M-85.1 · CASTLE C·S·E

## Sprint 2 — Biometric Auth  ·  worktree `feature/m10-sprint2`
- [x] T03: `skills/mobile-biometric-auth/SKILL.md` — Keychain/Keystore, bloqueo AsyncStorage, fallback 3 intentos, enrollment 7d. DONE.
- [x] T04: `commands/mobile-biometric-auth.md`. DONE. Merge 856843b. CASTLE S·T·E (corrige §2 del plan).
- Acceptance: M10 §7 / M-85.3 · CASTLE S·T·E

## Sprint 3 — Analytics  ·  worktree `feature/m10-sprint3`
- [x] T05: `skills/mobile-analytics/SKILL.md` — eventos tipados SaaS/Consumer, screen tracking auto, PII masking, consent gate. DONE.
- [x] T06: `commands/mobile-analytics.md`. DONE. Merge cdab84a. CASTLE C·S·T·E (§8).
- Acceptance: M10 §7 / M-85.4 · CASTLE C·S·E

## Sprint 4 — Contratos bilaterales  ·  worktree `feature/m10-sprint4`
- [ ] T07 (1h): `agents/_common/contracts/mobile-developer.md` (schema = devops-mobile.md).
- [ ] T08 (1h): `agents/_common/contracts/mobile-frontend.md` + `mobile-performance.md`.
- [ ] T09 (1h): `agents/_common/contracts/mobile-security.md` + `mobile-qa.md`.
- Acceptance: M10 §7 / "Contratos bilaterales mobile.md"

## Sprint 5 — Offline Sync  ·  worktree `feature/m10-sprint5`
- [ ] T10 (2h): `skills/mobile-offline-sync/SKILL.md` — Phases 0-3 (detect+schema, local-db, outbox, sync-engine).
- [ ] T11 (2h): Completar Phases 4-6 (conflict-resolution, ui-indicator, test-offline) + `commands/mobile-offline-sync.md`.
- [ ] T12 (1h): Editar `agents/mobile.md` — añadir sección "Contratos bilaterales activos" con los 5 contratos.
- Acceptance: M10 §7 / M-85.2 · CASTLE C·S·T·L

## Sprint 6 — App Store Submit + Deep Linking  ·  worktree `feature/m10-sprint6`
- [ ] T13 (2h): `skills/mobile-app-store-submit/SKILL.md` — rejection checker + Phases 0-3.
- [ ] T14 (2h): Completar Phases 4-5 (ci-workflow, submit) + `commands/mobile-app-store-submit.md`.
- [ ] T15 (2h): `skills/mobile-deep-linking/SKILL.md` — AASA + assetlinks + Phases 0-5.
- [ ] T16 (1h): `commands/mobile-deep-linking.md`.
- Acceptance: M10 §7 / M-85.5 + M-85.6 · CASTLE C·S·T·E / C·S

## Sprint 7 — Knowledge + Template Spec  ·  worktree `feature/m10-sprint7`
- [ ] T17 (2h): `knowledge/_inject/mobile-production-patterns.md` (≤120 líneas) — patrones consolidados de los 6 skills.
- [ ] T18 (1h): `knowledge/_inject/mobile-saas-starter-spec.md` — spec template V8 (stack, features, done).
- Cierre: `/sdd-verify` global + `/sdd-archive`.

---

**Review Workload Forecast**: cada sprint produce 1-4 archivos Markdown/YAML (~150-400 líneas c/u).
Sin riesgo de budget de código (no es código runtime). `delivery_strategy: sprint-by-sprint` ya resuelve
el split — cada sprint es un merge independiente y reviewable. No se requiere `size:exception`.

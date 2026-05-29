---
name: mobile-offline-sync
description: "Implementar offline-first con outbox pattern. Usar cuando se necesite: sincronización offline, persistencia local mobile, outbox pattern, WatermelonDB, Drift Flutter, sync contra Supabase, conflict resolution timestamp-based, SyncStatusBanner, offline-first app, no data loss mobile."
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

# /mobile-offline-sync

Implementa sincronización offline-first con WatermelonDB (RN) o Drift/SQLite (Flutter), outbox pattern para mutaciones sin red, motor de sync con retry exponencial, conflict resolution timestamp-based y un `<SyncStatusBanner>` no intrusivo.

## Instrucciones

1. Invocar el skill usando la herramienta Skill con `skill: king-mobile:mobile-offline-sync`
2. El skill detecta automáticamente el framework (Expo / bare RN / Flutter) si no se especifica
3. Seguir todas las fases del skill en orden:
   - Phase 0 (Load Context) → Phase 1 (detect framework + collect schema) → Phase 2 (setup local DB) → Phase 3 (outbox pattern) → Phase 4 (sync engine) → Phase 5 (conflict resolution) → Phase 6 (UI indicator) → Phase N+1
4. El argumento `--framework` omite la detección interactiva
5. Prerequisito: proyecto mobile scaffoldeado (`/mobile-scaffold`)

## Output esperado

- Schema DB local: `src/db/schema.ts` (WatermelonDB) o `lib/db/database.dart` (Drift)
- Motor de sync con outbox, retry exponencial y detección de red: `src/db/sync.ts`
- Modelos por entidad detectada del schema: `src/db/models/`
- Hook de estado de sync: `src/hooks/useSyncStatus.ts` (`syncing | synced | pending | offline`)
- Componente `<SyncStatusBanner>` visible solo con cambios pendientes
- Test automatizado: `src/tests/offline-sync.test.ts` (5 ops CRUD offline → reconexión → 5 sync sin pérdida + 0 conflictos)

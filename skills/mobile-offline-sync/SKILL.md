---
name: mobile-offline-sync
version: 1.0
api_version: 1.0.0
description: "Implementa sincronización offline-first con WatermelonDB (React Native) o Drift/SQLite (Flutter), outbox pattern para mutaciones offline, y conflict resolution basado en timestamp. Genera schema local, motor de sync con retry exponencial, hook de estado de sync, y componente SyncStatusBanner no intrusivo. Usar cuando: agregar offline support, persistencia local mobile, outbox pattern, sync contra Supabase, sync contra API custom, conflict resolution mobile, WatermelonDB react native, Drift Flutter, offline-first app."
---

# /mobile-offline-sync — Sincronización Offline-First con Outbox Pattern

Implementa offline-first sobre una app mobile existente (generada por `/mobile-scaffold`): DB local con WatermelonDB (RN) o Drift (Flutter), outbox pattern para mutaciones sin red, motor de sync con retry exponencial, conflict resolution timestamp-based, y un `<SyncStatusBanner>` no intrusivo.

## Knowledge Injection

| File | Purpose | Required | Source |
|------|---------|----------|--------|
| `.king/knowledge/stack.md` | Framework mobile del proyecto | Yes | project |
| `knowledge/_inject/mobile-native-essentials.md` | Versiones RN/Flutter, native bridges | Yes | framework |
| `knowledge/_inject/mobile-production-patterns.md` | Patrones de sync, retry, conflict resolution | Optional | framework |

## QUICK REFERENCE

### BLOCKING CONDITIONS
> Si alguna es TRUE, DETENER inmediatamente

- No existe proyecto mobile scaffoldeado (sin `package.json`/`pubspec.yaml` ni estructura mobile reconocible)
- Framework no detectable y modo no-interactivo activo
- No existe schema de entidades de negocio detectable (ni en código fuente ni en `.king/knowledge/`) — el outbox necesita saber qué tablas registrar
- El proyecto ya tiene una solución de sync incompatible en producción sin ruta de migración clara

### ABSOLUTE RESTRICTIONS
> Comportamientos absolutamente prohibidos — sin excepciones

- NUNCA perder una mutación offline: toda operación entra al outbox antes de intentar el envío
- NUNCA ejecutar la cola outbox en orden distinto a FIFO: el orden de mutaciones importa
- NUNCA omitir el campo `updated_at` en registros sincronizables — es el árbitro del conflict resolution
- NUNCA resolver conflictos silenciosamente sin registrar en `.king/sync/conflict-log.json`
- NUNCA exponer el `SyncStatusBanner` en estado normal (sin pendientes) — es no intrusivo por diseño

### REQUIRED OUTPUTS

> Ver `skills/_shared/lifecycle-outputs.md` para la convención de rutas de sesión

- Schema DB local: `src/db/schema.ts` (WatermelonDB) o `lib/db/database.dart` (Drift)
- Motor de sync con outbox: `src/db/sync.ts` (RN) o `lib/db/sync_engine.dart` (Flutter)
- Modelos por entidad: `src/db/models/` (RN) o `lib/db/models/` (Flutter)
- Hook de estado: `src/hooks/useSyncStatus.ts` — expone `syncing | synced | pending | offline`
- Componente de UI: `src/components/SyncStatusBanner.tsx` (visible solo con pendientes)
- Test automatizado: `src/tests/offline-sync.test.ts`
- Session document en `.king/sessions/`

### PHASES OVERVIEW

```
PHASE 0           PHASE 1          PHASE 2          PHASE 3          PHASE 4              PHASE 5         PHASE 6        PHASE N+1
SESSION        →  DETECT        →  SETUP         →  OUTBOX        →  SYNC             →  CONFLICT     →  UI          →  SESSION
(registry)        FRAMEWORK        LOCAL DB          PATTERN           ENGINE               RESOLUTION      INDICATOR      (escribir
                  + COLLECT        (schema +         (interface        (red detect +        (timestamp-      (Banner +      session)
                  SCHEMA           migrations)       + persistence)    FIFO queue +         based LWW +      hook)
                                                                       retry exp.)          conflict log)
```

---

## AGENTES INVOLUCRADOS

- **@mobile** → Validación de constraints de plataforma (detección de red, background sync, batería)
- **@developer** → Backend API — endpoints que reciben el outbox y devuelven cambios desde `last_sync_at`
- **@architect** → Escalación si el outbox pattern requiere cambio en el schema de la API existente

## CASTLE ACTIVO: C·_·S·T·L·_

> **Nota**: §2 del plan declara `C·S·T`. La tabla §8 (autoritativa) declara `C·_·S·T·L·_` añadiendo **L (Local persistence / no data loss)**. Se usa §8.

- **C (Contracts)**: Interface `OutboxEntry` tipada; `useSyncStatus` expone union type `SyncState`; motor de sync cumple contrato de vaciado FIFO antes de pull
- **S (Security)**: Sin datos sensibles en el conflict-log; el outbox no persiste tokens ni credenciales; los payloads respetan el schema de la entidad — sin campos extras no declarados
- **T (Testing)**: `src/tests/offline-sync.test.ts` simula 5 ops CRUD sin red → reconexión → 5 cambios sincronizados sin pérdida + 0 conflictos; motor de sync inyectable con mock de red
- **L (Local persistence)**: Toda mutación offline escribe en DB local ANTES de intentar sync — garantía de no data loss; outbox sobrevive reinicios de la app; retry exponencial con tope de 5 intentos

---

## Phase 0: Load Context

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase 0

---

## Phase 1: Detect Framework + Collect Schema

Detectar el framework mobile y recolectar el schema de entidades existente.

**Detección** (de `package.json` / `pubspec.yaml` / `.king/knowledge/stack.md`):

| Framework | Señal | Librería local | Sync strategy |
|-----------|-------|---------------|---------------|
| React Native Expo | `expo` en dependencies | WatermelonDB (`@nozbe/watermelondb`) | Sync adapter contra Supabase / custom API |
| React Native bare | `react-native` sin `expo` | WatermelonDB (`@nozbe/watermelondb`) | Sync adapter contra Supabase / custom API |
| Flutter | `pubspec.yaml` presente | Drift (SQLite) + `drift_sync` pattern | Sync manual contra Supabase / custom API |

Si el framework no se detecta y el modo es interactivo, preguntar. Si es no-interactivo → BLOCKING CONDITION.

**Recolección de schema**: buscar entidades de negocio en:
1. `src/types/` o `src/models/` — interfaces TypeScript (RN)
2. `lib/models/` — clases Dart (Flutter)
3. `.king/knowledge/` — documentación de dominio
4. Si no se encuentra schema → preguntar al usuario las entidades principales antes de continuar (BLOCKING si no-interactivo)

---

## Phase 2: Setup Local DB

Generar el schema de la DB local y las migraciones iniciales según framework.

| Framework | Archivo generado | Contenido |
|-----------|-----------------|-----------|
| React Native | `src/db/schema.ts` | `appSchema` WatermelonDB con `tableSchema` por entidad; campos `updated_at`, `is_deleted` obligatorios |
| Flutter | `lib/db/database.dart` | `@DriftDatabase` con tablas Drift; columnas `updatedAt`, `isDeleted` obligatorias |

Cada tabla sincronizable **MUST** tener:
- `id` — UUID string (generado en cliente, no autoincrement del servidor)
- `updated_at` — timestamp del cliente (árbitro de conflict resolution)
- `is_deleted` — soft delete para propagación de eliminaciones via sync

**Generación de modelos** (`src/db/models/` o `lib/db/models/`): un archivo por entidad detectada en Phase 1.

---

## Phase 3: Outbox Pattern

Generar la implementación del outbox: la cola de mutaciones pendientes de sync.

**Interface obligatoria** (`src/db/outbox.ts` o `lib/db/outbox.dart`):

```typescript
interface OutboxEntry {
  id: string;
  table: string;
  operation: "create" | "update" | "delete";
  payload: Record<string, unknown>;
  createdAt: Date;
  retryCount: number;
  status: "pending" | "syncing" | "failed";
}
```

**Comportamiento del outbox**:
- Toda mutación (create/update/delete) escribe en la DB local Y en el outbox en la misma transacción atómica
- El outbox persiste en la DB local — sobrevive reinicios de la app
- El outbox es de solo-append: nunca se modifica una entrada existente — se marca como `syncing` → `failed` o se elimina al confirmarse en el servidor
- Cola FIFO: las entradas se procesan en orden de `createdAt` — nunca reordenar

---

## Phase 4: Sync Engine

Generar el motor de sync (`src/db/sync.ts` o `lib/db/sync_engine.dart`) con detección de red, procesamiento de la cola outbox, y pull de cambios remotos.

**Arquitectura del flujo de sync**:

```
[User Action]
     ↓
[Local DB (WatermelonDB/Drift)]  ← escribe inmediatamente (offline OK)
     ↓
[Outbox Queue]  ← pendiente de sync cuando haya red
     ↓
[Sync Engine]  ← detecta conexión + ejecuta cola en orden FIFO
     ↓
[Backend API]  ← aplica cambios en servidor
     ↓
[Conflict Resolution]  ← timestamp-based: last-write-wins con merge de campos no conflictivos
     ↓
[Pull Changes]  ← trae cambios del servidor desde el último sync timestamp
```

**Detección de red**:
- RN: `@react-native-community/netinfo` → listener `addEventListener('change', handler)`
- Flutter: `connectivity_plus` → `onConnectivityChanged` stream

**Procesamiento del outbox** (FIFO, obligatorio):
1. Al detectar conectividad: obtener entradas con `status: "pending"` ordenadas por `createdAt ASC`
2. Para cada entrada: marcar `status: "syncing"` → enviar al backend → éxito: eliminar entrada; fallo: incrementar `retryCount`, marcar `status: "failed"`, schedule retry con backoff exponencial
3. Retry exponencial: `delay = Math.min(2^retryCount * 1000ms, 30000ms)` — máximo 5 intentos antes de marcar permanentemente como `failed` y notificar al usuario

**Pull de cambios remotos**: tras vaciar el outbox, llamar al endpoint de pull con `?since=last_sync_timestamp`; aplicar cambios remotos a la DB local pasando por Conflict Resolution (Phase 5); actualizar `last_sync_timestamp` persistido localmente.

---

## Phase 5: Conflict Resolution

Implementar la estrategia last-write-wins con audit log.

**Estrategia timestamp-based**:
- Cada record tiene `updated_at` (timestamp del cliente) y `server_updated_at` (timestamp del servidor)
- Si `updated_at` local > `server_updated_at`: se aplica el cambio local (el cliente ganó)
- Si `server_updated_at` > `updated_at` local: se aplica el cambio del servidor y se notifica al usuario
- Campos no en conflicto (modificados solo en uno de los dos lados): merge — se aplican sin conflicto

**Registro de conflictos** (obligatorio, no omitir):
- Cada conflicto real (server ganó sobre cambio local) se registra en `.king/sync/conflict-log.json`:

```json
{
  "timestamp": "ISO-8601",
  "table": "nombre_tabla",
  "recordId": "uuid",
  "localValue": { "field": "valor_local" },
  "serverValue": { "field": "valor_servidor" },
  "resolution": "server_wins"
}
```

- El log es append-only; no se elimina entradas; solo para auditoría — no bloquea el flujo de sync

---

## Phase 6: UI Indicator

Generar el hook de estado y el componente `<SyncStatusBanner>`.

**Hook `useSyncStatus`** (`src/hooks/useSyncStatus.ts`):

```typescript
type SyncState = "syncing" | "synced" | "pending" | "offline";

interface SyncStatus {
  state: SyncState;
  pendingCount: number;
  lastSyncedAt: Date | null;
}

function useSyncStatus(): SyncStatus;
```

**Componente `<SyncStatusBanner>`** (`src/components/SyncStatusBanner.tsx`):
- Visible SOLO cuando `state === "pending"` o `state === "syncing"` — **no visible** en `synced` ni `offline` sin pendientes
- Texto: `"Sincronizando..."` (syncing) o `"X cambios pendientes"` (pending)
- No bloquea la interacción — banner no intrusivo (bottom o top, no modal)
- Accesible: `accessibilityRole="status"`, `accessibilityLiveRegion="polite"`

---

## FINAL CHECKPOINT

Antes de terminar, verificar:

- [ ] Framework detectado; librería local instalada (WatermelonDB o Drift)
- [ ] Schema generado con `id` UUID, `updated_at`, `is_deleted` en todas las tablas sincronizables
- [ ] Modelos generados por entidad detectada del schema del proyecto
- [ ] Interface `OutboxEntry` implementada; outbox persiste en DB local (sobrevive reinicios)
- [ ] Toda mutación escribe en DB local + outbox en transacción atómica
- [ ] Cola outbox procesada en FIFO por `createdAt ASC`
- [ ] Retry exponencial implementado: `2^n * 1000ms`, tope 30s, máximo 5 intentos
- [ ] Conflict resolution: `updated_at` vs `server_updated_at`; conflictos registrados en `.king/sync/conflict-log.json`
- [ ] `useSyncStatus` expone union type `SyncState` correcto
- [ ] `<SyncStatusBanner>` visible SOLO con pendientes/syncing; no visible en estado normal
- [ ] `src/tests/offline-sync.test.ts` generado: 5 ops CRUD offline → reconexión → 5 sync sin pérdida + 0 conflictos
- [ ] Session document creado en `.king/sessions/`

---

## Execution Summary

| Field | Value |
|-------|-------|
| Status | `COMPLETE` \| `PARTIAL` \| `BLOCKED` |
| CASTLE Verdict | _(copiar del assessment — C·S·T·L, usando §8)_ |
| Artifacts | _(schema, sync engine, outbox, useSyncStatus, SyncStatusBanner, test)_ |
| Next Recommended | `/mobile-app-store-submit` (submisión a stores) o `/mobile-deep-linking` (rutas deep link) |
| Risks | _(riesgos activos o "None")_ |

---

## Phase N+1: Write Session

> Seguir instrucciones de `skills/session-management/SKILL.md` → Phase N+1

## Phase N+2: Guide Next Step

| Condición | Próximo Skill |
|-----------|---------------|
| Offline sync configurado y se quiere subir a stores | `/mobile-app-store-submit` |
| Se quiere navegación por deep links desde notificaciones | `/mobile-deep-linking` |

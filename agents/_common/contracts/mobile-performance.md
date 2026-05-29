# Mobile-Performance Contract

## Propósito
Define el protocolo de interacción entre @mobile y @performance para establecer presupuestos de rendimiento adaptados a constraints móviles (battery, JS thread, bundle size, cold start) y para reportar profiling con evidencia medible antes de cada release.

---

## Escenarios de Interacción

| Escenario | Iniciador | Receptor | Tipo | Bloquea |
|-----------|-----------|----------|------|---------|
| Entrega de presupuestos globales de latencia y payload | @performance | @mobile | Budget Handoff | Sí |
| Battery profiling report de feature nueva | @mobile | @performance | Profiling Report | No |
| Revisión de JS thread budget excedido | @performance | @mobile | Budget Violation | Sí |
| Bundle size o cold start fuera de target | @mobile | @performance | Threshold Alert | Sí |
| Datos de telemetría con PII de usuario en profiling | @mobile o @performance | @security | Security Escalation | Sí — TOTAL |

---

## Performance Budget Handoff: Presupuestos Globales

### Cuándo @performance entrega presupuestos a @mobile

```yaml
type: "performance_budget_handoff"
from: "@performance"
to: "@mobile"

version: "{semver o sprint ID}"
baseline_device: "Pixel 3a (3GB RAM, Android 11) | iPhone 12 (iOS 16)"
low_end_device: "Redmi 9 (2GB RAM, Android 10) | iPhone SE 2020 (iOS 16)"

latency_budgets:
  api_response_p95_ms: 800
  api_response_p99_ms: 2000
  local_render_after_api_ms: 100  # tiempo desde data disponible hasta frame visible
  navigation_transition_ms: 300

payload_budgets:
  api_response_json_kb: 50        # por endpoint, sin comprimir
  api_response_json_compressed_kb: 15
  image_asset_max_kb: 200
  total_screen_payload_kb: 500

bundle_budgets:
  js_bundle_total_kb: 1800        # comprimido (gzip)
  js_bundle_initial_chunk_kb: 400 # chunk inicial (TTI crítico)
  native_binary_delta_mb: 10      # delta máximo por update de OTA

cold_start_targets_ms:
  ios_time_to_interactive: 2000
  android_time_to_interactive: 2500
  low_end_device_multiplier: 1.5  # aplicar a ambos en low-end

battery_budgets:
  background_cpu_max_percent: 5
  foreground_session_mah_per_hour: 120  # estimación en sesión activa típica
  gps_continuous_mah_per_hour: 80
```

### Response Format (@mobile → @performance)

```yaml
type: "performance_budget_ack"
from: "@mobile"
to: "@performance"

status: "ACCEPTED | NEEDS_NEGOTIATION"

current_baseline_measurements:
  js_bundle_kb: 1650
  cold_start_ios_ms: 1800
  cold_start_android_ms: 2200
  battery_foreground_mah_per_hour: 105

negotiation_items:  # solo si status es NEEDS_NEGOTIATION
  - budget: "{nombre del presupuesto}"
    requested: "{valor actual}"
    proposed_target: "{valor negociado}"
    rationale: "{justificación técnica}"
```

---

## Battery Profiling Report: Feature Nueva

### Cuándo @mobile emite profiling hacia @performance

```yaml
type: "battery_profiling_report"
from: "@mobile"
to: "@performance"

feature: "{nombre de la feature}"
profiling_tool: "Xcode Energy Organizer | Android Battery Historian | Perfetto"
device: "{modelo y OS usado para el profiling}"

measurements:
  cpu_usage_percent:
    idle: 2
    active: 18
    peak: 45
  wake_locks_held_ms: 0         # Android — debe ser 0 en background salvo justificación
  background_fetch_frequency_min: 15  # mínimo recomendado por iOS BGAppRefresh
  gps_usage: "none | on_demand | continuous"
  network_requests_per_session: 8
  large_payload_requests: 0     # requests > 500KB

within_budget: true|false
budget_violations:
  - metric: "{nombre de la métrica}"
    measured: "{valor medido}"
    budget: "{valor límite}"
    delta: "{diferencia}"
    fix_proposed: "{optimización propuesta}"
```

### Response Format (@performance → @mobile)

```yaml
type: "battery_profiling_review"
from: "@performance"
to: "@mobile"

verdict: "PASS | FAIL | CONDITIONAL"

approved_for_release: true|false

required_optimizations:  # solo si verdict es FAIL o CONDITIONAL
  - metric: "{nombre de la métrica}"
    priority: "BLOCKING | HIGH | MEDIUM"
    optimization: "{acción específica requerida}"
    deadline: "{sprint o fecha}"
```

---

## JS Thread Budget Review: Violación Detectada

### Cuándo @performance alerta a @mobile por JS thread excedido

```yaml
type: "js_thread_budget_violation"
from: "@performance"
to: "@mobile"

component: "{nombre del componente o screen}"
measured_frame_time_ms: 32      # >16ms = jank (drop bajo 60fps)
measured_long_task_ms: 180      # >50ms bloquea main thread
profiling_trace: "{path o URL al trace de Flipper/Perfetto}"

thread_breakdown:
  js_execution_ms: 120
  layout_ms: 25
  paint_ms: 15
  gc_ms: 20

verdict: "EXCEEDS_BUDGET"
blocking_release: true|false
```

### Response Format (@mobile → @performance)

```yaml
type: "js_thread_budget_violation_response"
from: "@mobile"
to: "@performance"

acknowledgment: true
root_cause: "{análisis de causa raíz}"

optimization_plan:
  - action: "{ej: memoizar componente X con React.memo}"
    expected_reduction_ms: 40
    effort: "LOW | MEDIUM | HIGH"
  - action: "{ej: mover cálculo Y a un worker thread con JSI}"
    expected_reduction_ms: 80
    effort: "HIGH"

estimated_resolution_sprint: "{sprint ID o fecha estimada}"
```

---

## Security Escalation: PII en Datos de Profiling

### Triggers (cualquier agente detona esto)

Detener inmediatamente y escalar a @security si se detecta:
- Datos de profiling o telemetría que contienen identificadores de usuario, emails, o tokens de sesión
- Traces de red con payloads que exponen información personal en herramientas de profiling
- Battery Historian u otras herramientas mostrando contenido de requests con datos sensibles

### Formato de escalación

```yaml
type: "security_escalation"
from: "@mobile | @performance"
to: "@security"

trigger: "pii_in_profiling_data | session_token_in_trace | sensitive_payload_in_telemetry"
location: "{herramienta y sección donde se detectó el dato sensible}"
evidence: |
  {Descripción del dato sensible detectado — NO incluir el dato real aquí}

action: "STOP — no continuar hasta resolución de @security"
```

> @security tiene veto total en este escenario. Su decisión es FINAL e irrevocable.

---

## Ver también

- **Mobile-Developer Contract**: `contracts/mobile-developer.md`
- **Mobile-Security Contract**: `contracts/mobile-security.md`
- **Mobile-QA Contract**: `contracts/mobile-qa.md`
- **Escalation Matrix**: `_common/escalation-matrix.md`

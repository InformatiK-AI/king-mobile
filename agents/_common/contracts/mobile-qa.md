# Mobile-QA Contract

## Propósito
Define el protocolo de interacción entre @mobile y @qa para coordinar la device matrix de cobertura (iOS 16-18, Android 10-15), garantizar testing en dispositivos de gama baja, y verificar compliance de accesibilidad con VoiceOver (iOS) y TalkBack (Android) antes de cada release.

---

## Escenarios de Interacción

| Escenario | Iniciador | Receptor | Tipo | Bloquea |
|-----------|-----------|----------|------|---------|
| Entrega de test plan base para feature móvil | @qa | @mobile | Test Plan Handoff | No |
| Entrega de device matrix para el release | @mobile | @qa | Device Matrix Spec | Sí |
| Cobertura en dispositivo de gama baja | @mobile | @qa | Low-End Coverage Report | Sí |
| A11y VoiceOver/TalkBack checklist | @mobile | @qa | Accessibility Report | Sí |
| Hallazgo de datos de usuario PII en logs de test | @qa o @mobile | @security | Security Escalation | Sí — TOTAL |

---

## Test Plan Handoff: Base para Feature Móvil

### Cuándo @qa entrega test plan a @mobile

```yaml
type: "test_plan_handoff"
from: "@qa"
to: "@mobile"

feature: "{nombre de la feature}"
sprint: "{sprint ID}"

acceptance_criteria:
  - id: "AC-01"
    description: "{criterio funcional}"
    mobile_specific_notes: "{consideración particular para mobile, ej: comportamiento en landscape}"
  - id: "AC-02"
    description: "{criterio funcional}"
    mobile_specific_notes: "{ej: estado offline requerido}"

out_of_scope_for_mobile:
  - "{funcionalidad cubierta por @qa genérico, no específica de plataforma}"

expected_mobile_additions:
  - "Device matrix coverage"
  - "Low-end device regression"
  - "VoiceOver/TalkBack a11y verification"
  - "Offline/connectivity edge cases"
```

### Response Format (@mobile → @qa)

```yaml
type: "test_plan_ack"
from: "@mobile"
to: "@qa"

status: "ACCEPTED | NEEDS_CLARIFICATION"

added_mobile_scenarios:
  - id: "MOB-AC-01"
    based_on: "AC-01"
    scenario: "{escenario móvil adicional: ej: 'AC-01 en iPhone SE 2020 con iOS 16 en modo landscape'}"
  - id: "MOB-AC-02"
    based_on: "AC-02"
    scenario: "{ej: 'AC-02 con conexión intermitente simulada (Network Link Conditioner 3G)'}"

questions:  # solo si status es NEEDS_CLARIFICATION
  - "{pregunta específica sobre el test plan}"
```

---

## Device Matrix Spec: Cobertura Requerida para el Release

### Cuándo @mobile emite la device matrix hacia @qa

```yaml
type: "device_matrix_spec"
from: "@mobile"
to: "@qa"

release_version: "{semver}"

ios_devices:
  - device: "iPhone 15 Pro"
    os: "iOS 18.0"
    priority: "P1"
    test_type: "smoke + regression"
  - device: "iPhone 13"
    os: "iOS 17.4"
    priority: "P1"
    test_type: "smoke + regression"
  - device: "iPhone SE (3rd gen)"
    os: "iOS 16.7"   # mínimo soportado
    priority: "P1 — low-end iOS"
    test_type: "regression + performance"
  - device: "iPad Air (5th gen)"
    os: "iOS 17.4"
    priority: "P2"
    test_type: "smoke"

android_devices:
  - device: "Pixel 8"
    os: "Android 14 (API 34)"
    priority: "P1"
    test_type: "smoke + regression"
  - device: "Samsung Galaxy S23"
    os: "Android 13 (API 33)"
    priority: "P1"
    test_type: "smoke + regression"
  - device: "Pixel 3a"
    os: "Android 12 (API 31)"
    priority: "P1 — low-end baseline"
    test_type: "regression + performance + battery"
  - device: "Redmi 9"
    os: "Android 10 (API 29)"  # mínimo soportado
    priority: "P1 — critical low-end"
    test_type: "regression + performance"
  - device: "Samsung Galaxy A13"
    os: "Android 11 (API 30)"
    priority: "P2"
    test_type: "smoke"

emulator_coverage:
  - simulator: "iPhone SE (3rd gen) — iOS 16"
    use_for: "CI automated tests"
  - emulator: "Pixel 3a API 29"
    use_for: "CI automated tests"

os_version_matrix_rationale:
  ios_minimum: "iOS 16 — ~5% market share del base instalada"
  android_minimum: "Android 10 (API 29) — ~8% market share del base instalada"
  ios_maximum: "iOS 18 (current)"
  android_maximum: "Android 15 (API 35)"
```

### Response Format (@qa → @mobile)

```yaml
type: "device_matrix_coverage_report"
from: "@qa"
to: "@mobile"

status: "FULL_COVERAGE | PARTIAL_COVERAGE | BLOCKED"

coverage_summary:
  devices_tested: 7
  devices_required: 9
  coverage_percent: 78

missing_coverage:  # solo si status no es FULL_COVERAGE
  - device: "{modelo}"
    os: "{versión}"
    reason: "{dispositivo no disponible en lab / emulador con bug conocido}"
    mitigation: "{alternativa propuesta}"

results_by_device:
  - device: "iPhone SE (3rd gen) iOS 16.7"
    result: "PASS | FAIL | BLOCKED"
    issues: []
  - device: "Redmi 9 Android 10"
    result: "PASS | FAIL | BLOCKED"
    issues:
      - "{descripción del issue encontrado}"
```

---

## Low-End Device Coverage: Reporte de Gama Baja

### Cuándo @mobile emite reporte de cobertura low-end

```yaml
type: "low_end_device_coverage_report"
from: "@mobile"
to: "@qa"

feature: "{nombre de la feature}"
test_device: "Redmi 9 (2GB RAM, Snapdragon 662, Android 10)"

performance_on_low_end:
  cold_start_ms: 2800
  screen_transition_ms: 380
  scroll_fps: 45    # ≥30fps aceptable en low-end
  memory_usage_mb: 180
  within_budget: true|false

issues_on_low_end:
  - issue: "{descripción del problema}"
    severity: "CRITICAL | HIGH | MEDIUM | LOW"
    reproducible: true|false
    workaround: "{workaround si existe}"

verdict: "PASS | FAIL | CONDITIONAL"
blocking_release: true|false  # true si algún issue es CRITICAL
```

### Response Format (@qa → @mobile)

```yaml
type: "low_end_coverage_review"
from: "@qa"
to: "@mobile"

verdict: "ACCEPTED | REJECTED | NEEDS_RETEST"

blocking_issues:
  - issue: "{descripción}"
    ticket: "{ID de ticket creado}"
    priority: "P0 | P1"

approved_for_release_on_low_end: true|false
```

---

## A11y Checklist: VoiceOver y TalkBack

### Cuándo @mobile emite reporte de accesibilidad

```yaml
type: "accessibility_report"
from: "@mobile"
to: "@qa"

feature: "{nombre de la feature}"
tester: "@mobile"

voiceover_ios:
  enabled_during_test: true
  os_version: "iOS 17.4"
  device: "iPhone 13"

  checks:
    - id: "VO-01"
      description: "Todos los elementos interactivos tienen accessibilityLabel descriptivo"
      result: "PASS | FAIL"
      notes: "{observación si FAIL}"
    - id: "VO-02"
      description: "Focus order es lógico y sigue flujo visual de izquierda a derecha, top a bottom"
      result: "PASS | FAIL"
      notes: "{observación si FAIL}"
    - id: "VO-03"
      description: "Imágenes decorativas tienen accessibilityElementsHidden = true"
      result: "PASS | FAIL"
    - id: "VO-04"
      description: "Formularios: cada input tiene accessibilityLabel + accessibilityHint"
      result: "PASS | FAIL"
    - id: "VO-05"
      description: "Alertas y modals anuncian correctamente al aparecer (accessibilityViewIsModal)"
      result: "PASS | FAIL"
    - id: "VO-06"
      description: "Touch targets ≥44x44pt en todos los elementos activables"
      result: "PASS | FAIL"

talkback_android:
  enabled_during_test: true
  api_level: 31
  device: "Pixel 3a"

  checks:
    - id: "TB-01"
      description: "Todos los elementos interactivos tienen contentDescription no nulo"
      result: "PASS | FAIL"
      notes: "{observación si FAIL}"
    - id: "TB-02"
      description: "Focus order correcto con explore-by-touch"
      result: "PASS | FAIL"
    - id: "TB-03"
      description: "ImageView decorativos tienen importantForAccessibility=no"
      result: "PASS | FAIL"
    - id: "TB-04"
      description: "EditText tienen hint y labelFor configurados"
      result: "PASS | FAIL"
    - id: "TB-05"
      description: "Snackbars y Toasts son anunciados por TalkBack"
      result: "PASS | FAIL"
    - id: "TB-06"
      description: "Touch targets ≥48x48dp en todos los elementos activables"
      result: "PASS | FAIL"

wcag_level: "AA"
overall_result: "PASS | FAIL | CONDITIONAL"
blocking_findings:
  - id: "{VO/TB ID}"
    description: "{descripción del fallo}"
    severity: "CRITICAL | HIGH"
    required_fix: "{corrección exacta}"
```

### Response Format (@qa → @mobile) — Validación A11y

```yaml
type: "accessibility_report_review"
from: "@qa"
to: "@mobile"

verdict: "APPROVED | REJECTED | CONDITIONAL"

# REJECTED bloquea el release — accesibilidad es criterio no negociable

additional_findings:
  - id: "{ID nuevo hallazgo}"
    description: "{issue detectado por @qa en revisión independiente}"
    severity: "CRITICAL | HIGH | MEDIUM"

a11y_approved_for_release: true|false
castle_layer_t_status: "FORTIFIED | CONDITIONAL | BREACHED"
```

---

## Security Escalation: PII en Logs de Test

### Triggers (cualquier agente detona esto)

Detener inmediatamente y escalar a @security si se detecta:
- Logs de test o reportes de QA que contienen emails, tokens de sesión, o datos de usuario reales
- Screenshots de dispositivos físicos con PII visible capturadas en el pipeline de CI
- Datos de producción usados en entorno de testing sin anonimización

### Formato de escalación

```yaml
type: "security_escalation"
from: "@qa | @mobile"
to: "@security"

trigger: "pii_in_test_logs | pii_in_ci_screenshots | production_data_in_test_env"
location: "{herramienta, reporte, o pipeline donde se detectó}"
evidence: |
  {Descripción del hallazgo — NO incluir el dato PII real aquí}

action: "STOP — no continuar hasta resolución de @security"
```

> @security tiene veto total en este escenario. Su decisión es FINAL e irrevocable.

---

## Ver también

- **Mobile-Developer Contract**: `contracts/mobile-developer.md`
- **Mobile-Security Contract**: `contracts/mobile-security.md`
- **Mobile-Performance Contract**: `contracts/mobile-performance.md`
- **Escalation Matrix**: `_common/escalation-matrix.md`

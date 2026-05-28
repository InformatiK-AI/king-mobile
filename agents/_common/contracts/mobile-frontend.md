# Mobile-Frontend Contract

## Propósito
Define el protocolo de interacción entre @mobile y @frontend para sincronizar design system tokens hacia la capa nativa, y establecer criterios objetivos que determinan cuándo una UI debe ser web-responsive versus componente nativo o WebView shell justificado.

---

## Escenarios de Interacción

| Escenario | Iniciador | Receptor | Tipo | Bloquea |
|-----------|-----------|----------|------|---------|
| Entrega de design system tokens para plataforma nativa | @frontend | @mobile | Token Handoff | No |
| Decisión web-responsive vs componente nativo | @mobile | @frontend | Platform Decision Request | Sí |
| Justificación de WebView shell para feature específica | @mobile | @frontend | WebView Justification | Sí |
| Tokens con datos PII o lógica de negocio embebida | @frontend o @mobile | @security | Security Escalation | Sí — TOTAL |

---

## Design Token Handoff: Colores, Tipografía y Espaciado

### Cuándo @frontend entrega tokens a @mobile

```yaml
type: "design_token_handoff"
from: "@frontend"
to: "@mobile"

version: "{semver del design system, ej: 2.3.0}"
source: "{path al token source, ej: design-tokens/tokens.json}"

colors:
  primary: "#1A73E8"
  primary_dark: "#1558B0"    # para dark mode
  secondary: "#FF6B35"
  surface: "#FFFFFF"
  surface_dark: "#1C1C1E"
  on_primary: "#FFFFFF"
  error: "#D93025"
  warning: "#F29900"
  success: "#1E8E3E"

typography:
  font_family_default: "Inter"
  font_family_mono: "JetBrains Mono"
  scale:
    - name: "display_large"
      size_sp: 57
      line_height: 64
      weight: 400
    - name: "title_large"
      size_sp: 22
      line_height: 28
      weight: 700
    - name: "body_medium"
      size_sp: 14
      line_height: 20
      weight: 400
    - name: "label_small"
      size_sp: 11
      line_height: 16
      weight: 500

spacing:
  base_unit: 4  # px
  scale: [0, 4, 8, 12, 16, 24, 32, 48, 64]

border_radius:
  small: 4
  medium: 8
  large: 16
  pill: 9999

dark_mode_supported: true|false
```

### Response Format (@mobile → @frontend)

```yaml
type: "design_token_handoff_response"
from: "@mobile"
to: "@frontend"

status: "ACCEPTED | ISSUES_FOUND"

native_mapping:
  ios:
    color_format: "UIColor (hex) o SwiftUI Color"
    font_handling: "custom font registrado en Info.plist"
    dynamic_type_compatible: true|false
  android:
    color_format: "Color resource en res/values/colors.xml"
    font_handling: "font family en res/font/"
    material_you_compatible: true|false

issues:  # solo si status es ISSUES_FOUND
  - token: "{nombre del token}"
    platform: "ios | android | both"
    issue: "{descripción: contraste insuficiente en dark mode, tamaño fuera de escala nativa, etc.}"
    fix: "{corrección requerida}"

a11y_check:
  contrast_ratio_aa_compliant: true|false
  minimum_touch_target_respected: true|false  # 44x44px iOS / 48x48dp Android
```

---

## Platform Decision: Web-Responsive vs Componente Nativo

### Cuándo @mobile emite decisión hacia @frontend

```yaml
type: "platform_decision_request"
from: "@mobile"
to: "@frontend"

feature: "{nombre de la feature o componente}"
component: "{nombre del componente bajo evaluación}"

evaluation_criteria:
  requires_native_api: true|false    # cámara, GPS, biometría, push, NFC
  performance_critical: true|false   # animaciones 60fps, listas largas
  offline_required: true|false
  platform_hig_compliance: true|false  # debe seguir Human Interface Guidelines / Material

proposed_approach: "web_responsive | native_component | webview_shell"
```

### Response Format (@mobile → @frontend) — Decisión Final

```yaml
type: "platform_decision_response"
from: "@mobile"
to: "@frontend"

component: "{nombre del componente}"
verdict: "WEB_RESPONSIVE | NATIVE_COMPONENT | WEBVIEW_JUSTIFIED"

rationale: |
  {Explicación técnica de 2-3 oraciones. Ej: "El componente requiere acceso a la cámara
  nativa y animaciones a 60fps consistentes en gama baja — web_responsive no es viable
  sin jank en Android 10 con RAM ≤ 2GB."}

implementation_notes:
  ios: "{consideraciones específicas iOS}"
  android: "{consideraciones específicas Android}"

performance_targets:
  frame_rate: "60fps"
  interaction_response_ms: 100
  low_end_device_tested: true|false  # Pixel 3a o equivalente ≤ 3GB RAM

fallback_if_webview: "{URL de fallback en caso de fallo de WebView}"
```

---

## WebView Shell Justification: Feature en WebView

### Cuándo @mobile justifica uso de WebView

```yaml
type: "webview_justification"
from: "@mobile"
to: "@frontend"

feature: "{nombre de la feature}"
url_pattern: "https://app.example.com/feature/{param}"

justification:
  reason: "feature_flag_controlled | legal_content | low_churn_screen | third_party_widget"
  native_blocker: "{por qué no se puede implementar nativo ahora}"

requirements:
  javascript_enabled: true
  dom_storage_enabled: true|false
  allow_file_access: false  # SIEMPRE false salvo excepción documentada
  mixed_content: "NEVER_ALLOW"
  certificate_pinning: true|false

navigation_rules:
  allowed_domains:
    - "app.example.com"
    - "cdn.example.com"
  blocked_redirects_to_external: true
  deep_link_intercept: true  # interceptar deep links dentro del WebView

security_review_required: true  # @security debe aprobar si allow_file_access o mixed_content != NEVER_ALLOW
```

### Response Format (@frontend → @mobile)

```yaml
type: "webview_justification_ack"
from: "@frontend"
to: "@mobile"

status: "APPROVED | REJECTED | ESCALATED_TO_SECURITY"
notes: "{observaciones de diseño o UX relevantes}"
```

---

## Security Escalation: Datos Sensibles en Tokens o WebView Config

### Triggers (cualquier agente detona esto)

Detener inmediatamente y escalar a @security si se detecta:
- Token de design system que contiene lógica de autenticación o endpoint de API embebido
- WebView con `allow_file_access: true` o `mixed_content` distinto de `NEVER_ALLOW`
- Token de color o tipografía que en realidad es una clave de API ofuscada

### Formato de escalación

```yaml
type: "security_escalation"
from: "@frontend | @mobile"
to: "@security"

trigger: "sensitive_data_in_token | unsafe_webview_config | obfuscated_api_key"
location: "{archivo y sección donde se detectó}"
evidence: |
  {Fragmento exacto que genera la alerta}

action: "STOP — no continuar hasta resolución de @security"
```

> @security tiene veto total en este escenario. Su decisión es FINAL e irrevocable.

---

## Ver también

- **Mobile-Developer Contract**: `contracts/mobile-developer.md`
- **Mobile-Performance Contract**: `contracts/mobile-performance.md`
- **Mobile-Security Contract**: `contracts/mobile-security.md`
- **Escalation Matrix**: `_common/escalation-matrix.md`

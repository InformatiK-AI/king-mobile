# Mobile-Developer Contract

## Propósito
Define el protocolo de interacción entre @mobile y @developer para coordinar el handoff de feature specs hacia constraints de plataforma nativa: permisos del sistema, push token lifecycle, y validación de deep link schemas antes de implementación.

---

## Escenarios de Interacción

| Escenario | Iniciador | Receptor | Tipo | Bloquea |
|-----------|-----------|----------|------|---------|
| Handoff de deep link schema para nueva feature | @developer | @mobile | Schema Review Request | Sí |
| Especificación de permisos nativos requeridos | @mobile | @developer | Native Permission Spec | Sí |
| Push token handling contract para feature de notificaciones | @developer | @mobile | Token Contract Request | No |
| Credencial o token expuesto en spec de feature | @developer o @mobile | @security | Security Escalation | Sí — TOTAL |

---

## Deep Link Schema Review: Handoff de Rutas

### Cuándo @developer solicita a @mobile

```yaml
type: "deep_link_schema_review_request"
from: "@developer"
to: "@mobile"

feature: "{nombre de la feature}"
proposed_routes:
  - path: "/product/:id"
    params:
      - name: "id"
        type: "string"
        required: true
  - path: "/checkout/:orderId"
    params:
      - name: "orderId"
        type: "string"
        required: true

auth_required: true|false
fallback_url: "https://app.example.com/{path}"
```

### Response Format (@mobile → @developer)

```yaml
type: "deep_link_schema_review_response"
from: "@mobile"
to: "@developer"

verdict: "APPROVED | REJECTED | CONDITIONAL"

platform_validations:
  ios:
    aasa_path: "/.well-known/apple-app-site-association"
    associated_domain: "applinks:app.example.com"
    aasa_verified: true|false
  android:
    asset_links_path: "/.well-known/assetlinks.json"
    package_name: "{com.example.app}"
    asset_links_verified: true|false

issues:  # solo si verdict es REJECTED o CONDITIONAL
  - route: "{path}"
    issue: "{descripción del problema}"
    fix: "{corrección requerida}"

required_changes:
  info_plist: []   # entradas requeridas en iOS
  android_manifest: []  # intent-filter requeridos en Android
```

---

## Native Permission Spec: Permisos Requeridos por Feature

### Cuándo @mobile emite hacia @developer

```yaml
type: "native_permission_spec"
from: "@mobile"
to: "@developer"

feature: "{nombre de la feature}"
permissions_required:
  ios:
    - key: "NSCameraUsageDescription"
      value: "La app necesita acceso a la cámara para escanear códigos QR"
      required: true
    - key: "NSLocationWhenInUseUsageDescription"
      value: "La app usa tu ubicación para mostrarte tiendas cercanas"
      required: false
  android:
    - permission: "android.permission.CAMERA"
      protection_level: "dangerous"
      rationale: "Requerido para escanear códigos QR en el flujo de checkout"
    - permission: "android.permission.ACCESS_FINE_LOCATION"
      protection_level: "dangerous"
      rationale: "Requerido para búsqueda de tiendas cercanas"

runtime_request_flow:
  - trigger: "{evento que dispara el pedido de permiso, ej: user taps 'Scan QR'}"
  - on_denied: "{comportamiento cuando el usuario rechaza}"
  - on_permanently_denied: "{deep link a Settings de la app}"

minimum_sdk_impact:
  ios_minimum: "16.0"
  android_minimum_api: 29
```

### Response Format (@developer → @mobile)

```yaml
type: "native_permission_spec_ack"
from: "@developer"
to: "@mobile"

status: "ACCEPTED | NEEDS_CLARIFICATION"

implementation_notes:
  - "{observación de implementación relevante}"

questions:  # solo si status es NEEDS_CLARIFICATION
  - "{pregunta específica}"
```

---

## Push Token Contract: Lifecycle de Tokens de Notificación

### Cuándo @developer solicita a @mobile

```yaml
type: "push_token_contract_request"
from: "@developer"
to: "@mobile"

feature: "{nombre de la feature}"
token_storage_proposed: "AsyncStorage | Keychain | Keystore | Backend only"
token_refresh_strategy: "on_app_foreground | on_token_expired | manual"
revocation_required: true|false
```

### Response Format (@mobile → @developer)

```yaml
type: "push_token_contract_response"
from: "@mobile"
to: "@developer"

verdict: "APPROVED | BLOCKED"

# BLOCKED si token_storage_proposed es AsyncStorage — anti-patrón crítico de seguridad
reason_if_blocked: "{explicación técnica}"

approved_storage:
  ios: "Keychain (com.example.app.push_token)"
  android: "EncryptedSharedPreferences o Keystore"

ttl_policy:
  max_age_days: 30
  refresh_trigger: "on_app_foreground"
  revocation_endpoint: "POST /api/push/revoke"

backend_contract:
  registration_endpoint: "POST /api/push/register"
  payload_schema:
    token: "string"
    platform: "ios | android"
    device_id: "string (UUID)"
    user_id: "string | null"
```

---

## Security Escalation: Token o Credencial Expuesto

### Triggers (cualquier agente detona esto)

Detener inmediatamente y escalar a @security si se detecta:
- Push token o auth token hardcodeado en spec o código
- Deep link schema que expone datos de sesión en la URL sin firma
- API shape que transmite credenciales por query params en lugar de headers

### Formato de escalación

```yaml
type: "security_escalation"
from: "@developer | @mobile"
to: "@security"

trigger: "hardcoded_token | session_data_in_url | credentials_in_query_params"
location: "{archivo y línea o sección de spec donde se detectó}"
evidence: |
  {Fragmento exacto que genera la alerta}

action: "STOP — no continuar hasta resolución de @security"
```

> @security tiene veto total en este escenario. Su decisión es FINAL e irrevocable.

---

## Ver también

- **Mobile-Security Contract**: `contracts/mobile-security.md`
- **Mobile-QA Contract**: `contracts/mobile-qa.md`
- **DevOps-Mobile Contract**: `contracts/devops-mobile.md`
- **Escalation Matrix**: `_common/escalation-matrix.md`

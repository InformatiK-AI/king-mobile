# Mobile-Security Contract

## Propósito
Define el protocolo de interacción entre @mobile y @security para validar el uso correcto de Keystore/Keychain, aplicar certificate pinning, verificar detección de jailbreak/root, y certificar compliance con OWASP MASVS antes de cada release. @security tiene veto absoluto y su decisión en cualquier escenario bloqueante es FINAL e irrevocable.

---

## Escenarios de Interacción

| Escenario | Iniciador | Receptor | Tipo | Bloquea |
|-----------|-----------|----------|------|---------|
| Revisión de Keychain/Keystore usage en feature de autenticación | @mobile | @security | Security Review Request | Sí |
| Certificate pinning requirement para endpoint crítico | @security | @mobile | Pinning Directive | Sí |
| Jailbreak/root detection review antes de release | @mobile | @security | Detection Review | Sí |
| OWASP MASVS gate pre-release | @security | @mobile | MASVS Gate | Sí — TOTAL |
| Credencial o token encontrado fuera de almacén seguro | @mobile o @security | @security | Security Escalation | Sí — TOTAL |
| Security gate de CASTLE no cumplido | @security | @mobile | CASTLE Gate Violation | Sí — TOTAL |

---

## Keychain/Keystore Review: Almacenamiento Seguro de Credenciales

### Cuándo @mobile solicita revisión a @security

```yaml
type: "secure_storage_review_request"
from: "@mobile"
to: "@security"

feature: "{nombre de la feature}"
data_classification:
  - item: "session_token"
    sensitivity: "CRITICAL"
    proposed_storage: "Keychain (kSecClassGenericPassword) | EncryptedSharedPreferences"
  - item: "biometric_enrolled_flag"
    sensitivity: "HIGH"
    proposed_storage: "Keychain (kSecAttrAccessibleWhenUnlockedThisDeviceOnly)"
  - item: "push_token"
    sensitivity: "HIGH"
    proposed_storage: "Keychain | Keystore"

explicitly_excluded:
  - storage: "AsyncStorage"
    reason: "plaintext on-disk — PROHIBIDO para tokens de sesión"
  - storage: "UserDefaults / SharedPreferences"
    reason: "no cifrado — PROHIBIDO para datos de autenticación"

biometric_protection:
  ios_la_context: true|false    # LAContext para acceso biométrico
  android_biometric_prompt: true|false

backup_exclusion:
  ios_icloud_backup: false      # SIEMPRE false para tokens críticos
  android_allow_backup: false   # SIEMPRE false para tokens críticos
```

### Response Format (@security → @mobile)

```yaml
type: "secure_storage_review_response"
from: "@security"
to: "@mobile"

verdict: "APPROVED | REJECTED | CONDITIONAL"

# REJECTED o CONDITIONAL bloquea el release hasta resolución

violations:  # solo si verdict es REJECTED o CONDITIONAL
  - item: "{nombre del dato}"
    issue: "{descripción de la violación}"
    severity: "CRITICAL | HIGH | MEDIUM"
    required_fix: "{corrección exacta requerida}"

approved_items:
  - item: "{nombre del dato}"
    approved_storage: "{almacén aprobado}"
    access_control: "{política de acceso aprobada}"

conditions:  # solo si verdict es CONDITIONAL
  - "{condición que debe cumplirse antes del release}"

castle_layer_c_status: "FORTIFIED | CONDITIONAL | BREACHED"
```

---

## Certificate Pinning: Directiva para Endpoint Crítico

### Cuándo @security emite directiva a @mobile

```yaml
type: "certificate_pinning_directive"
from: "@security"
to: "@mobile"

endpoints:
  - url: "https://api.example.com"
    pin_type: "public_key_hash | certificate_hash"
    pins:
      primary: "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
      backup: "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
    failure_action: "BLOCK | LOG_AND_BLOCK"

  - url: "https://auth.example.com"
    pin_type: "public_key_hash"
    pins:
      primary: "sha256/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC="
      backup: "sha256/DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD="
    failure_action: "BLOCK"

rotation_policy:
  notify_days_before_expiry: 30
  responsible_agent: "@devops + @mobile"

implementation:
  ios: "URLSession con URLSessionDelegate.urlSession(_:didReceive:completionHandler:)"
  android: "OkHttp CertificatePinner o TrustKit"
  react_native: "react-native-ssl-pinning o configuración nativa por plataforma"

blocking: true  # SIEMPRE true — sin pinning en endpoints críticos = BREACHED
```

### Response Format (@mobile → @security)

```yaml
type: "certificate_pinning_implementation_report"
from: "@mobile"
to: "@security"

status: "IMPLEMENTED | IN_PROGRESS | BLOCKED"

implementation_details:
  ios_method: "{método usado}"
  android_method: "{método usado}"
  network_security_config_updated: true|false  # Android

test_results:
  pinning_enforced: true|false
  backup_pin_tested: true|false
  invalid_cert_blocked: true|false  # test con cert inválido debe dar error

issues_if_blocked:
  - "{descripción del bloqueo técnico}"
```

---

## Jailbreak/Root Detection Review

### Cuándo @mobile solicita revisión a @security

```yaml
type: "jailbreak_root_detection_review"
from: "@mobile"
to: "@security"

implementation:
  ios:
    method: "DTTJailbreakDetection | custom checks"
    checks:
      - "cydia_installed"
      - "suspicious_files_present"  # /etc/apt, /private/var/lib/apt
      - "can_write_outside_sandbox"
      - "suspicious_url_schemes"    # cydia://, sileo://
    action_on_detection: "BLOCK_APP | DEGRADE_FEATURES | LOG_ONLY"

  android:
    method: "RootBeer | SafetyNet Attestation API | Play Integrity API"
    checks:
      - "root_management_apps"      # SuperSU, Magisk
      - "ro_build_tags_test_keys"
      - "dangerous_props_set"
      - "busybox_installed"
    attestation: "PLAY_INTEGRITY | SAFETYNET_DEPRECATED"
    action_on_detection: "BLOCK_APP | DEGRADE_FEATURES | LOG_ONLY"

response_to_detection:
  block_sensitive_features: true
  log_event: true
  pii_in_log: false   # NUNCA loguear PII en detección de root
```

### Response Format (@security → @mobile)

```yaml
type: "jailbreak_root_detection_verdict"
from: "@security"
to: "@mobile"

verdict: "APPROVED | REJECTED | CONDITIONAL"

required_changes:  # solo si verdict no es APPROVED
  - check: "{nombre del check}"
    issue: "{problema con la implementación actual}"
    fix: "{corrección requerida}"

minimum_required_action_on_detection: "BLOCK_APP | DEGRADE_FEATURES"
# LOG_ONLY NUNCA es aceptable para apps con datos financieros, médicos, o de autenticación crítica

castle_layer_c_status: "FORTIFIED | CONDITIONAL | BREACHED"
```

---

## OWASP MASVS Gate: Pre-Release

### Cuándo @security ejecuta el gate antes de release

```yaml
type: "masvs_gate_review"
from: "@security"
to: "@mobile"

release_version: "{semver}"
masvs_level: "L1 | L2"  # L2 para apps con datos sensibles

checklist:
  MASVS_STORAGE:
    - id: "MSTG-STORAGE-1"
      description: "Datos sensibles no en logs del sistema"
      status: "PASS | FAIL | N/A"
    - id: "MSTG-STORAGE-2"
      description: "No datos sensibles en AsyncStorage/UserDefaults/SharedPreferences"
      status: "PASS | FAIL | N/A"
    - id: "MSTG-STORAGE-3"
      description: "No datos sensibles en archivos de caché de la app"
      status: "PASS | FAIL | N/A"

  MASVS_NETWORK:
    - id: "MSTG-NETWORK-1"
      description: "Datos cifrados en tránsito con TLS 1.2+"
      status: "PASS | FAIL | N/A"
    - id: "MSTG-NETWORK-3"
      description: "Certificate pinning implementado para endpoints críticos"
      status: "PASS | FAIL | N/A"

  MASVS_PLATFORM:
    - id: "MSTG-PLATFORM-1"
      description: "App solicita solo permisos mínimos necesarios"
      status: "PASS | FAIL | N/A"
    - id: "MSTG-PLATFORM-2"
      description: "Todos los inputs de terceros validados y sanitizados"
      status: "PASS | FAIL | N/A"

  MASVS_CODE:
    - id: "MSTG-CODE-1"
      description: "App firmada con certificado válido"
      status: "PASS | FAIL | N/A"
    - id: "MSTG-CODE-8"
      description: "Detección de debugging en builds de producción"
      status: "PASS | FAIL | N/A"
```

### Response Format (@security → @mobile) — Veredicto MASVS

```yaml
type: "masvs_gate_verdict"
from: "@security"
to: "@mobile"

verdict: "PASS | FAIL"

# FAIL bloquea el release TOTALMENTE — veto @security

blocking_findings:
  - id: "{MSTG-ID}"
    description: "{descripción del fallo}"
    severity: "CRITICAL | HIGH"
    required_fix: "{corrección exacta antes del release}"
    deadline: "BEFORE_RELEASE"

non_blocking_findings:
  - id: "{MSTG-ID}"
    description: "{descripción}"
    severity: "MEDIUM | LOW"
    ticket: "{ID de ticket para sprint siguiente}"

castle_layer_c_status: "FORTIFIED | CONDITIONAL | BREACHED"
release_approved: true|false

# Si release_approved es false: STOP — no release hasta nueva revisión por @security
```

---

## Security Escalation: Violación Crítica de Seguridad

### Triggers (cualquier agente detona esto)

Detener inmediatamente y escalar a @security si se detecta:
- Token de sesión o credencial fuera de Keychain/Keystore (en AsyncStorage, logs, o archivos de caché)
- Certificate pinning ausente en endpoint con datos financieros, médicos, o de autenticación
- Jailbreak/root detection ausente o reducida a `LOG_ONLY` en app con datos críticos
- Cualquier MSTG con severidad CRITICAL marcado como FAIL

### Formato de escalación

```yaml
type: "security_escalation"
from: "@mobile | @qa | @developer"
to: "@security"

trigger: "credential_outside_secure_storage | missing_certificate_pinning | weak_jailbreak_detection | masvs_critical_fail"
location: "{archivo, línea, o componente donde se detectó}"
evidence: |
  {Descripción técnica exacta de la violación — sin incluir credenciales reales}

castle_impact: "BREACHED"
action: "STOP — no continuar hasta resolución de @security"
```

> @security tiene veto total en todos los escenarios de este contrato. Ningún release puede proceder con un finding CRITICAL sin aprobación explícita de @security. Su decisión es FINAL e irrevocable.

---

## Ver también

- **Mobile-Developer Contract**: `contracts/mobile-developer.md`
- **Mobile-QA Contract**: `contracts/mobile-qa.md`
- **DevOps-Mobile Contract**: `contracts/devops-mobile.md`
- **Escalation Matrix**: `_common/escalation-matrix.md`

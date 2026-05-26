# DevOps-Mobile Contract

## Propósito
Define el protocolo de interacción entre @devops y @mobile para coordinación de pipelines de distribución móvil, validación de signing/credentials, y escalación de seguridad en el contexto de `/mobile-deploy`.

---

## Escenarios de Interacción

| Escenario | Iniciador | Receptor | Tipo | Bloquea |
|-----------|-----------|----------|------|---------|
| Validación de Fastlane lanes en pipeline | @devops | @mobile | Review Request | Sí |
| Alineación de secrets entre Fastfile y YAML | @mobile | @devops | Credential Validation | Sí |
| Credencial expuesta o step inseguro detectado | @devops o @mobile | @security | Security Escalation | Sí — TOTAL |

---

## Pipeline Review Request: Validación de Lanes

### Cuándo @devops solicita a @mobile

```yaml
type: "pipeline_review_request"
from: "@devops"
to: "@mobile"

workflow_draft: "{path al YAML borrador}"
lanes:
  - name: "{nombre de la lane}"
    platform: "ios|android"
    action: "build_app|upload_to_testflight|upload_to_play_store"
```

### Response Format (@mobile → @devops)

```yaml
type: "pipeline_review_response"
from: "@mobile"
to: "@devops"

verdict: "PASS | FAIL"

corrections:  # solo si verdict es FAIL
  - lane: "{nombre de la lane}"
    issue: "{descripción del problema}"
    fix: "{corrección exacta requerida}"

verified:
  export_method_is_app_store: true|false   # debe ser "app-store", no "development"
  signing_configured: true|false
  lane_params_correct: true|false
```

---

## Credential Validation Request: Alineación de Secrets

### Cuándo @mobile solicita a @devops

```yaml
type: "credential_validation_request"
from: "@mobile"
to: "@devops"

expected_env_vars:
  - name: "APP_STORE_CONNECT_KEY_ID"
    source: "ENV[\"APP_STORE_CONNECT_KEY_ID\"]"
  - name: "{NOMBRE_VARIABLE}"
    source: "{ENV[\"...\"] en Fastfile}"
```

### Response Format (@devops → @mobile)

```yaml
type: "credential_validation_response"
from: "@devops"
to: "@mobile"

status: "ALIGNED | MISMATCH"

mismatches:  # solo si status es MISMATCH
  - expected: "{nombre en Fastfile}"
    found_in_yaml: "{nombre en secrets.XXX del workflow}"
    fix: "{corrección exacta}"
```

---

## Security Escalation: Credencial Expuesta

### Triggers (cualquier agente detona esto)

Detener inmediatamente y escalar a @security si se detecta:
- Referencia a un valor de credencial hardcodeado en lugar de un nombre de secret
- Step con `echo`, `cat`, o `printenv` sobre variables de credentials
- Flag `--verbose` en una lane de Fastlane en un job de producción

### Formato de escalación

```yaml
type: "security_escalation"
from: "@devops | @mobile"
to: "@security"

trigger: "hardcoded_credential | credential_leak_step | verbose_production"
location: "{archivo y línea o step donde se detectó}"
evidence: |
  {Fragmento exacto que genera la alerta}

action: "STOP — no continuar hasta resolución de @security"
```

> @security tiene veto total en este escenario. Su decisión es FINAL e irrevocable.

---

## Ver también

- **DevOps-Developer Contract**: `contracts/devops-developer.md`
- **DevOps-Security Contract**: `contracts/devops-security.md`
- **Escalation Matrix**: `_common/escalation-matrix.md`

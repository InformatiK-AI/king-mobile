# mobile-deploy — IOS (Phase 3)

> Sub-archivo de `SKILL.md`. Cargado via PHASE ROUTER para Phase 3.
> NUNCA ejecutar directamente — siempre invocado desde `skills/mobile-deploy/SKILL.md`.

---

## PHASE 3: iOS Credentials & Signing

### GATE IN
- [ ] `--target` = `ios` | `both`
- [ ] Phase 2 completada — secrets validados (o lista de faltantes conocida)

### MUST DO
> Todas las acciones son OBLIGATORIAS

#### Paso 3.1: App Store Connect API Key

Presentar al usuario las siguientes instrucciones paso a paso:

1. Ir a [appstoreconnect.apple.com](https://appstoreconnect.apple.com) → Users & Access → Integrations → App Store Connect API
2. Crear nueva API Key:
   - Name: `{nombre-proyecto}-deploy`
   - Access: **Developer** (NO App Manager, NO Admin)
   - Confirmar: habilita upload de builds + consulta de apps; NO gestiona cuenta ni team
3. Descargar `AuthKey_{KeyID}.p8`:
   - ADVERTENCIA: **DESCARGA ÚNICA** — Este archivo solo puede descargarse UNA vez. Si se pierde, debés revocar la key y crear una nueva.
   - Guardar en un lugar seguro ANTES de continuar
4. Anotar los siguientes valores (necesarios para los secrets):
   - **Key ID**: string de 10 caracteres alfanuméricos (visible en la lista de API Keys)
   - **Issuer ID**: UUID visible en la parte superior de la página de API Keys
5. Convertir el archivo `.p8` a base64:
   - macOS/Linux:
     ```bash
     base64 -i AuthKey_{KeyID}.p8 | tr -d '\n'
     ```
   - Windows PowerShell:
     ```powershell
     [Convert]::ToBase64String([IO.File]::ReadAllBytes('AuthKey_{KeyID}.p8'))
     ```
6. Configurar los tres secrets en GitHub:
   ```bash
   gh secret set APP_STORE_CONNECT_KEY_ID --body "{KeyID}"
   gh secret set APP_STORE_CONNECT_ISSUER_ID --body "{IssuerID}"
   gh secret set APP_STORE_CONNECT_KEY_CONTENT  # pegar el base64 cuando lo solicite
   ```
7. Agregar `AuthKey_*.p8` al `.gitignore` del proyecto **inmediatamente**:
   ```bash
   echo 'AuthKey_*.p8' >> .gitignore
   git add .gitignore && git commit -m "chore: ignore App Store Connect API key files"
   ```

#### Paso 3.2: Distribution Certificate & Provisioning Profile

> Para `expo-managed`: EAS gestiona el signing automáticamente — saltar este paso y continuar al Paso 3.3.

**Certificate (solo `rn-bare` y `flutter`):**

1. Abrir Keychain Access en Mac e ir a developer.apple.com/account → Certificates
2. Verificar que existe un "Apple Distribution" certificate vigente (no vencido):
   - Si existe y es válido: continuar al paso 4 (exportar)
3. Si no existe o está vencido, crear uno nuevo:
   - En Keychain Access → Certificate Assistant → Request a Certificate From a Certificate Authority
   - Generar un archivo CSR (.certSigningRequest)
   - Subir el CSR en developer.apple.com → Certificates → + → Apple Distribution
   - Descargar e instalar el certificado resultante en Keychain Access
4. Exportar el certificado como .p12:
   - Keychain Access → click derecho sobre el certificado "Apple Distribution" → Export
   - Formato: **Personal Information Exchange (.p12)**
   - Establecer un password fuerte y guardarlo (se necesitará para el secret `DIST_CERT_PASSWORD`)
5. Convertir a base64 y configurar los secrets:
   ```bash
   base64 -i Distribution.p12 | tr -d '\n' | gh secret set DISTRIBUTION_CERT
   gh secret set DIST_CERT_PASSWORD --body "{password}"
   ```

**Provisioning Profile (solo `rn-bare` y `flutter`):**

1. Ir a developer.apple.com/account → Profiles → + → Distribution → App Store Connect
2. Seleccionar el App ID que coincide con el bundle ID del proyecto
3. Seleccionar el Distribution Certificate recién creado o verificado
4. Descargar `{AppName}.mobileprovision`
5. Convertir a base64 y configurar el secret:
   ```bash
   base64 -i {AppName}.mobileprovision | tr -d '\n' | gh secret set PROVISIONING_PROFILE
   ```

#### Paso 3.3: GitHub Environment con protecciones

Crear el environment `production-ios` y configurar sus protecciones:

```bash
# Crear environment via gh CLI
gh api repos/{owner}/{repo}/environments/production-ios \
  --method PUT \
  --field deployment_branch_policy='{"protected_branches":false,"custom_branch_policies":true}'
```

Luego configurar en GitHub UI (Settings → Environments → production-ios):
- **Required reviewers**: 1 (el owner o tech lead del proyecto)
- **Deployment branch policy**: only `main` branch

#### Paso 3.4: eas.json verificación (solo expo-managed)

Verificar que `eas.json` tiene configurada la sección `submit` para iOS:

```json
{
  "submit": {
    "production": {
      "ios": {
        "appleId": "",
        "ascAppId": "",
        "appleTeamId": ""
      }
    }
  }
}
```

Instrucción: completar los valores con los datos de App Store Connect:
- `appleId`: el email de la cuenta de Apple Developer
- `ascAppId`: el App ID numérico visible en App Store Connect → App Information
- `appleTeamId`: el Team ID visible en developer.apple.com/account → Membership

### CHECKPOINT
> Verificar antes de continuar a Phase 4 o Phase 5

- [ ] `APP_STORE_CONNECT_KEY_ID` configurado como GitHub Secret
- [ ] `APP_STORE_CONNECT_ISSUER_ID` configurado como GitHub Secret
- [ ] `APP_STORE_CONNECT_KEY_CONTENT` configurado como GitHub Secret
- [ ] `AuthKey_*.p8` agregado al `.gitignore` del proyecto
- [ ] Para `rn-bare` y `flutter`: `DISTRIBUTION_CERT` configurado (certificate válido y no vencido)
- [ ] Para `rn-bare` y `flutter`: `DIST_CERT_PASSWORD` configurado
- [ ] Para `rn-bare` y `flutter`: `PROVISIONING_PROFILE` configurado
- [ ] GitHub Environment `production-ios` creado con required reviewers y branch policy `main`
- [ ] Para `expo-managed`: `eas.json` tiene sección `submit.production.ios` completa

### IF FAILS
```
Key ID no encontrado:
  → Instrucción: ir a appstoreconnect.apple.com → Users & Access → Integrations →
    App Store Connect API. El Key ID es el string de 10 caracteres alfanuméricos
    visible en la columna "Key ID" de cada API Key listada.
  → El Issuer ID está en la parte superior de esa misma página, sobre la tabla de keys.

Certificate vencido:
  → BLOQUEANTE — Apple no permite renovar certificados; hay que crear uno nuevo.
  → Proceso completo de renovación:
      1. Ir a developer.apple.com/account → Certificates
      2. Revocar el certificado vencido (click en el cert → Revoke)
      3. Crear un nuevo CSR desde Keychain Access (Certificate Assistant →
         Request a Certificate From a Certificate Authority)
      4. Subir el CSR en developer.apple.com → Certificates → + → Apple Distribution
      5. Descargar e instalar en Keychain Access
      6. Exportar como .p12 (Keychain → click derecho → Export → formato .p12)
      7. Convertir a base64 y actualizar el secret:
           base64 -i Distribution.p12 | tr -d '\n' | gh secret set DISTRIBUTION_CERT
      8. Actualizar también DIST_CERT_PASSWORD si el password cambió
  → No continuar hasta que el certificado nuevo esté configurado en GitHub Secrets

Provisioning Profile incompatible:
  → Verificar que el bundle ID del profile coincide EXACTAMENTE con el del proyecto:
      rn-bare: android/app/build.gradle → applicationId / ios/*/Info.plist → CFBundleIdentifier
      flutter: android/app/build.gradle → applicationId
  → Un bundle ID con diferencia de mayúsculas o punto extra genera fallo de signing
  → Si el App ID no existe en developer.apple.com, crearlo primero en
    Certificates, Identifiers & Profiles → Identifiers → +
  → Descargar y configurar el Provisioning Profile de nuevo una vez alineado el bundle ID
```

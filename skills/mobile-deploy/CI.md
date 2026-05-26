<!-- CI.md — mobile-deploy Phases 5-7 -->
<!-- PHASE ROUTER: cargar este archivo cuando fase activa = 5, 6 o 7 -->

## Phase 5: GitHub Actions Workflows

### GATE IN

- [ ] Phase 3 completada (si `--target ios | both`)
- [ ] Phase 4 completada (si `--target android | both`)
- [ ] Tipo de proyecto conocido: `expo-managed` | `rn-bare` | `flutter`
- [ ] `--target` resuelto: `ios` | `android` | `both`

**IF FAILS:** Volver a DISCOVERY.md y completar las phases anteriores antes de continuar.

---

### MUST DO

#### Reglas de generación (aplican a TODOS los workflows)

Cada workflow generado DEBE incluir:

1. `permissions: {}` o `permissions: contents: read` explícito en cada job
2. Step de validación de secrets con nombre `Validate required secrets` ANTES de cualquier operación que los consuma. El step de validación DEBE ser el primero en la lista `steps:` del job, inmediatamente después del step `actions/checkout`.
3. Step con `if: failure()` al final para notificación de errores
4. `environment:` asociado a `production-ios` o `production-android` según corresponda
5. Credenciales ÚNICAMENTE como `${{ secrets.NOMBRE }}` — jamás como valores literales

> Las ABSOLUTE RESTRICTIONS de `SKILL.md` aplican a todo lo generado en esta fase — consultar SKILL.md si hay dudas.

---

#### PATH A — expo-managed

Generar `.github/workflows/ios-deploy.yml`:

```yaml
name: iOS Deploy (EAS)
on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  deploy-ios:
    runs-on: ubuntu-latest
    environment: production-ios
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Validate required secrets
        env:
          ASC_KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          ASC_ISSUER: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          ASC_KEY: ${{ secrets.APP_STORE_CONNECT_KEY_CONTENT }}
          EXP_TOKEN: ${{ secrets.EXPO_TOKEN }}
        run: |
          [ -z "$ASC_KEY_ID" ] && echo "::error::APP_STORE_CONNECT_KEY_ID not configured" && exit 1
          [ -z "$ASC_ISSUER" ] && echo "::error::APP_STORE_CONNECT_ISSUER_ID not configured" && exit 1
          [ -z "$ASC_KEY" ] && echo "::error::APP_STORE_CONNECT_KEY_CONTENT not configured" && exit 1
          [ -z "$EXP_TOKEN" ] && echo "::error::EXPO_TOKEN not configured" && exit 1
          echo "All required secrets present"

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - uses: expo/expo-github-action@v8
        with:
          expo-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build and submit to TestFlight
        env:
          APP_STORE_CONNECT_KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          APP_STORE_CONNECT_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_KEY_CONTENT }}
        run: eas build --platform ios --non-interactive --auto-submit

      - name: Notify on failure
        if: failure()
        run: echo "::error::iOS EAS deployment failed — check build at expo.dev/accounts/{account}/projects/{project}/builds"
```

Generar `.github/workflows/android-deploy.yml` para expo-managed:

```yaml
name: Android Deploy (EAS)
on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  deploy-android:
    runs-on: ubuntu-latest
    environment: production-android
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Validate required secrets
        env:
          EXP_TOKEN: ${{ secrets.EXPO_TOKEN }}
          GP_KEY: ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
        run: |
          [ -z "$EXP_TOKEN" ] && echo "::error::EXPO_TOKEN not configured" && exit 1
          [ -z "$GP_KEY" ] && echo "::error::GOOGLE_PLAY_JSON_KEY not configured" && exit 1
          echo "All required secrets present"

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - uses: expo/expo-github-action@v8
        with:
          expo-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build and submit to Google Play
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
          GOOGLE_PLAY_JSON_KEY: ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
        run: eas build --platform android --non-interactive --auto-submit

      - name: Notify on failure
        if: failure()
        run: echo "::error::Android EAS deployment failed — check build at expo.dev"
```

---

#### PATH B — rn-bare y flutter

Generar `.github/workflows/ios-deploy.yml`:

```yaml
name: iOS Deploy (Fastlane)
on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  deploy-ios:
    runs-on: macos-latest
    environment: production-ios
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Validate required secrets
        env:
          KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          ISSUER: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_KEY_CONTENT }}
          CERT: ${{ secrets.DISTRIBUTION_CERT }}
          PROFILE: ${{ secrets.PROVISIONING_PROFILE }}
        run: |
          [ -z "$KEY_ID" ] && echo "::error::APP_STORE_CONNECT_KEY_ID missing" && exit 1
          [ -z "$CERT" ] && echo "::error::DISTRIBUTION_CERT missing" && exit 1
          [ -z "$PROFILE" ] && echo "::error::PROVISIONING_PROFILE missing" && exit 1
          echo "Secrets validated"

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Install iOS signing
        env:
          DISTRIBUTION_CERT: ${{ secrets.DISTRIBUTION_CERT }}
          DIST_CERT_PASSWORD: ${{ secrets.DIST_CERT_PASSWORD }}
          PROVISIONING_PROFILE: ${{ secrets.PROVISIONING_PROFILE }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          echo "$DISTRIBUTION_CERT" | base64 --decode > "$RUNNER_TEMP/dist.p12"
          security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security import "$RUNNER_TEMP/dist.p12" -k build.keychain \
            -P "$DIST_CERT_PASSWORD" -T /usr/bin/codesign -T /usr/bin/security
          security list-keychains -s build.keychain
          security set-keychain-settings -lut 21600 build.keychain
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "$PROVISIONING_PROFILE" | base64 --decode \
            > ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision
      - name: Deploy to App Store
        env:
          APP_STORE_CONNECT_KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          APP_STORE_CONNECT_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_KEY_CONTENT }}
        run: bundle exec fastlane release_ios

      - name: Cleanup signing artifacts
        if: always()
        run: rm -f "$RUNNER_TEMP/dist.p12"

      - name: Notify on failure
        if: failure()
        run: echo "::error::iOS Fastlane deployment failed — check logs above"
```

Generar `fastlane/Fastfile`:

```ruby
default_platform(:ios)

platform :ios do
  desc "Deploy to TestFlight via App Store Connect API"
  lane :release_ios do
    app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_KEY_CONTENT"],
      is_key_content_base64: true,
      in_house: false
    )
    build_app(
      scheme: "{AppName}",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "{bundle.id}" => "profile.mobileprovision"
        }
      }
    )
    upload_to_testflight(skip_waiting_for_build_processing: true)
  end
end

platform :android do
  desc "Deploy to Google Play Internal Testing"
  lane :release_android do
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )
    upload_to_play_store(
      track: "internal",
      json_key_data: ENV["GOOGLE_PLAY_JSON_KEY"]
    )
  end
end
```

Generar `fastlane/Appfile`:

```ruby
app_identifier("{bundle.id}")
apple_id("{apple-id@email.com}")

for_platform :android do
  package_name("{com.example.app}")
end
```

Generar `.github/workflows/android-deploy.yml` para rn-bare/flutter:

```yaml
name: Android Deploy (Fastlane)
on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  deploy-android:
    runs-on: ubuntu-latest
    environment: production-android
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Validate required secrets
        env:
          KS: ${{ secrets.KEYSTORE_BASE64 }}
          KS_PASS: ${{ secrets.KEYSTORE_PASSWORD }}
          GP_KEY: ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
        run: |
          [ -z "$KS" ] && echo "::error::KEYSTORE_BASE64 missing" && exit 1
          [ -z "$KS_PASS" ] && echo "::error::KEYSTORE_PASSWORD missing" && exit 1
          [ -z "$GP_KEY" ] && echo "::error::GOOGLE_PLAY_JSON_KEY missing" && exit 1
          echo "Secrets validated"

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Setup Android signing
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
        run: |
          echo "$KEYSTORE_BASE64" | base64 --decode > "$RUNNER_TEMP/release.keystore"

      - name: Deploy to Google Play
        env:
          KEYSTORE_PATH: ${{ runner.temp }}/release.keystore
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          GOOGLE_PLAY_JSON_KEY: ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
        run: bundle exec fastlane release_android

      - name: Cleanup signing artifacts
        if: always()
        run: rm -f "$RUNNER_TEMP/release.keystore"

      - name: Notify on failure
        if: failure()
        run: echo "::error::Android Fastlane deployment failed"
```

---

### CHECKPOINT Phase 5

- [ ] Workflows generados según tipo de proyecto (PATH A o PATH B)
- [ ] Cada job tiene `permissions: contents: read` explícito
- [ ] Cada workflow tiene step `Validate required secrets` antes de operaciones de deploy
- [ ] Cada workflow tiene step con `if: failure()` al final
- [ ] Ningún workflow contiene credenciales hardcodeadas ni logging de secrets
- [ ] Para PATH B: `fastlane/Fastfile` y `fastlane/Appfile` generados con placeholders

---

## Phase 6: CASTLE S Gate (@security)

### GATE IN

- [ ] Phase 5 completada — todos los workflows han sido generados

### IF FAILS
> ❌ What to do when CHECKPOINT Phase 5 fails

ERROR: Workflow generation incomplete or non-compliant
Cause: Generated YAML is syntactically invalid, a job lacks `permissions:`, a workflow is missing the `if: failure()` step, or credentials appear hardcoded.
Recovery:
  [ ] Option A: If YAML is invalid — open the generated file, identify the malformed block (missing indent, unclosed quote), fix it, and verify with `yamllint .github/workflows/`
  [ ] Option B: If `permissions:` is missing from a job — add `permissions: contents: read` under the `job-name:` key, before `steps:`
  [ ] Option C: If hardcoded credentials detected — replace the literal value with `${{ secrets.SECRET_NAME }}`, rotate the exposed credential immediately, and re-run the check

---

### MUST DO

@security ejecuta las siguientes verificaciones sobre los workflows generados.
Si algún check falla, corregir el workflow y re-ejecutar los checks antes de continuar.

**S-1 (BLOQUEANTE) — Sin logging de credenciales:**

```bash
grep -rn "printenv\|echo.*secrets\|cat.*\.p8\|cat.*\.json\|cat.*\.p12\|set -x\|--verbose" .github/workflows/ fastlane/
```

Si hay matches: DETENER. Identificar el step responsable, eliminar el logging, y re-ejecutar S-1.

**S-2 (BLOQUEANTE) — Sin valores hardcodeados:**

```bash
grep -rn "BEGIN.*KEY\|BEGIN.*CERTIFICATE\|AKIA[0-9A-Z]\|ghp_\|xoxb-" .github/workflows/ fastlane/
```

Si hay matches: DETENER. Reemplazar el valor literal por `${{ secrets.NOMBRE }}` y re-ejecutar S-2.

**S-3 (REQUERIDO) — Permissions explícito en todos los jobs:**

Verificar manualmente que cada `jobs.<job-id>:` tiene `permissions:` declarado.
Valores aceptados: `permissions: {}` o `permissions: contents: read`.

**S-4 (REQUERIDO) — Step de failure notification en todos los workflows:**

Verificar que cada workflow contiene al menos un step con `if: failure()`.

**S-5 (REQUERIDO) — Archivos de credenciales en .gitignore:**

Verificar que `.gitignore` contiene (o agregar si falta):

```
# Mobile signing artifacts
*.p8
*.p12
*.jks
*.keystore
google-services.json
GoogleService-Info.plist
AuthKey_*.p8
```

---

### CHECKPOINT Phase 6

- [ ] S-1: 0 matches de logging de credenciales en `.github/workflows/` y `fastlane/`
- [ ] S-2: 0 matches de valores hardcodeados en `.github/workflows/` y `fastlane/`
- [ ] S-3: `permissions:` explícito en cada job de cada workflow
- [ ] S-4: step `if: failure()` presente en cada workflow
- [ ] S-5: archivos de credenciales cubiertos en `.gitignore`

### IF FAILS
> ❌ What to do when CHECKPOINT Phase 6 fails

ERROR: CASTLE S gate failed — one or more security checks did not pass
Cause: A workflow contains credential logging patterns (S-1), hardcoded values (S-2), missing `permissions:` (S-3), missing failure notification (S-4), or credentials not in `.gitignore` (S-5).
Recovery:
  [ ] Option A: For S-1 (credential logging) — search the generated YAML for `printenv`, `echo`, `cat *.p8` patterns and remove or replace them; never use `set -x` in steps where secrets are in scope
  [ ] Option B: For S-2 (hardcoded values) — replace every literal credential with `${{ secrets.NAME }}`; rotate any exposed credential immediately before continuing
  [ ] Option C: For S-5 (.gitignore missing entries) — add the missing entries to `.gitignore` in the project root; the required entries are: `*.p8`, `*.p12`, `*.keystore`, `*.jks`, `google-services.json`, `GoogleService-Info.plist`, `AuthKey_*.p8`

---

## Phase 7: Report

### GATE IN

- [ ] Phase 6 completada — todos los checks del CASTLE S gate pasaron

**IF FAILS:** No generar el reporte hasta que el gate de seguridad esté limpio.

---

### MUST DO

**1. Generar `CREDENTIALS-SETUP.md` en la raíz del proyecto:**

El archivo se genera con fechas calculadas desde la fecha de ejecución del skill (hoy).
- Sustituir `{fecha-hoy}` por la fecha actual en todos los campos `Configured`.
- **iOS Distribution Certificate**: si Phase 2 detectó la fecha de expiración real via `openssl x509 -noout -enddate`, usar esa fecha en la columna `Expires` en lugar de `{fecha-hoy + 1 año}`.
- Si Phase 2 detectó que el cert vence en **< 30 días**: marcar la fila con `⚠️ EXPIRA PRONTO` y agregar alerta en la columna `Renewal Required` con la fecha exacta.
- Para los demás campos sin detección de expiración real: usar `{fecha-hoy + 1 año}` como default.

```markdown
# Credentials Maintenance

> Generado por King Framework `/mobile-deploy`. Actualizar las fechas al renovar.

## Expiration Schedule

| Credential | Configured | Expires | Renewal Required |
|---|---|---|---|
| iOS Distribution Certificate | {fecha-hoy} | {cert-expiry-real ó fecha-hoy + 1 año} | developer.apple.com → Certificates |
| iOS Provisioning Profile | {fecha-hoy} | {fecha-hoy + 1 año} | developer.apple.com → Profiles |
| App Store Connect API Key | {fecha-hoy} | Never (rotate every 12 months) | appstoreconnect.apple.com → Integrations |
| Android Keystore | {fecha-hoy} | **NEVER ROTATE** | Backup only — rotation = losing Play Store app |
| Google Play Service Account | {fecha-hoy} | Never (rotate every 12 months) | console.cloud.google.com → IAM |

> **Nota de generación — iOS Distribution Certificate**:
> - Reemplazar `{cert-expiry-real ó fecha-hoy + 1 año}` con la fecha detectada por openssl en Phase 2.
> - Si la fecha detectada es **< 30 días desde hoy**: cambiar la fila a:
>   `| iOS Distribution Certificate | {fecha-hoy} | ⚠️ EXPIRA PRONTO — {cert-expiry-real} | **Renovar URGENTE** — developer.apple.com → Certificates |`
> - Si no se detectó la fecha real (openssl no disponible): usar `{fecha-hoy + 1 año}` y agregar WARN en el session doc.

## Rotation Procedures

### iOS Distribution Certificate (annual)
1. Create new CSR in Keychain Access
2. Upload to developer.apple.com → Certificates → + → Apple Distribution
3. Export as .p12 and update `DISTRIBUTION_CERT` secret in GitHub
4. Regenerate Provisioning Profile associated with this certificate
5. Update `PROVISIONING_PROFILE` secret

### App Store Connect API Key (recommended every 12 months)
1. Create new key in App Store Connect → Integrations → App Store Connect API
2. Update all three secrets: KEY_ID, ISSUER_ID, KEY_CONTENT
3. Revoke the old key after confirming the new one works

### Android Keystore
⚠️ DO NOT ROTATE. The keystore is tied to the app's identity in Play Store.
Focus on backup security instead:
- Store in at least 2 offline locations
- Consider Google Play App Signing (Google manages the upload key)

### Google Play Service Account (recommended every 12 months)
1. Create new service account in Google Cloud Console
2. Invite in Play Console with same limited permissions
3. Update `GOOGLE_PLAY_JSON_KEY` secret
4. Delete old service account after confirming the new one works
```

**2. Mostrar resumen de lo configurado:**

```
## Deploy Setup — Summary

### Workflows generated
- .github/workflows/ios-deploy.yml     ✓  (si --target ios | both)
- .github/workflows/android-deploy.yml ✓  (si --target android | both)
- fastlane/Fastfile                    ✓  (si PATH B)
- fastlane/Appfile                     ✓  (si PATH B)

### Secrets required
iOS:
  - APP_STORE_CONNECT_KEY_ID          ✓ / ✗
  - APP_STORE_CONNECT_ISSUER_ID       ✓ / ✗
  - APP_STORE_CONNECT_KEY_CONTENT     ✓ / ✗
  - DISTRIBUTION_CERT                 ✓ / ✗  (solo PATH B)
  - DIST_CERT_PASSWORD                ✓ / ✗  (solo PATH B)
  - PROVISIONING_PROFILE              ✓ / ✗  (solo PATH B)
  - EXPO_TOKEN                        ✓ / ✗  (solo PATH A)

Android:
  - GOOGLE_PLAY_JSON_KEY              ✓ / ✗
  - KEYSTORE_BASE64                   ✓ / ✗  (solo PATH B)
  - KEYSTORE_PASSWORD                 ✓ / ✗  (solo PATH B)
  - KEY_ALIAS                         ✓ / ✗  (solo PATH B)
  - KEY_PASSWORD                      ✓ / ✗  (solo PATH B)
  - EXPO_TOKEN                        ✓ / ✗  (solo PATH A)

### GitHub Environments created
  - production-ios                    ✓ / pendiente
  - production-android                ✓ / pendiente

### CREDENTIALS-SETUP.md
  - Generated at repo root            ✓

### Next step
  Push to main to trigger the first automated deploy:
    git push origin main
```

Si algún secret aparece como ✗, mostrar el comando correspondiente:

```bash
gh secret set APP_STORE_CONNECT_KEY_ID --body "valor"
```

---

### CHECKPOINT Phase 7

- [ ] `CREDENTIALS-SETUP.md` generado con fechas calculadas desde la fecha actual
- [ ] Resumen mostrado con estado real de workflows, secrets y environments
- [ ] Instrucción de próximo paso (push a main) incluida en el output

### IF FAILS
> ❌ What to do when CHECKPOINT fails

ERROR: Report not generated — `CREDENTIALS-SETUP.md` absent or summary incomplete
Cause: Write operation failed for `CREDENTIALS-SETUP.md`, or secrets status could not be determined at report time.
Recovery:
  [ ] Option A: If `CREDENTIALS-SETUP.md` write failed (permissions or path error), create the file manually at the project root using the template in MUST DO step 1 — substitute today's date for `{fecha-hoy}` and `{fecha-hoy + 1 año}` accordingly
  [ ] Option B: If `gh secret list` failed (gh CLI not authenticated), mark each secret as `✗` in the summary and include the `gh secret set` commands — do NOT omit the summary section
  [ ] Option C: If any secret is confirmed missing at this stage, output the report with status PARTIAL and include the full list of missing secrets with their corresponding `gh secret set` commands — the report must be emitted even when configuration is incomplete
- [ ] Secrets faltantes informados con comando `gh secret set` correspondiente

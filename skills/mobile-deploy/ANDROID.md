# mobile-deploy — ANDROID (Phase 4)

> Sub-archivo de `SKILL.md`. Cargado via PHASE ROUTER para Phase 4.
> NUNCA ejecutar directamente — siempre invocado desde `skills/mobile-deploy/SKILL.md`.

---

## PHASE 4: Android Credentials & Signing

### GATE IN
- [ ] `--target` = `android` | `both`
- [ ] Phase 2 completada — secrets validados

### MUST DO
> Todas las acciones son OBLIGATORIAS

#### Paso 4.1: Android Keystore

> **ADVERTENCIA CRITICA — LEER ANTES DE CONTINUAR**
>
> El Android Keystore es **IRREMPLAZABLE** para una app en produccion.
> Google Play vincula cada update de la app a la firma del keystore.
> Si se pierde o se compromete, la app **no puede recibir updates** en Play Store.
>
> BACKUP OBLIGATORIO antes de cualquier otro paso:
> 1. Guardar el archivo `.keystore` en almacenamiento offline (USB, disco externo)
> 2. Guardar en un password manager (1Password, Bitwarden, etc.)
> 3. Considerar Google Cloud KMS o similar para proyectos de produccion
>
> NUNCA commitear el keystore al repositorio.

1. [ ] **Verificar si ya existe un keystore**:
   - Buscar archivos `*.keystore` o `*.jks` en el directorio del proyecto
   - Si existe: confirmar con el usuario que es el keystore de produccion activo antes de continuar

2. [ ] **Si keystore NO existe, generarlo**:
   ```bash
   keytool -genkey -v \
     -keystore {nombre-app}-release.keystore \
     -alias {nombre-app}-key \
     -keyalg RSA -keysize 2048 \
     -validity 10000 \
     -dname "CN={Nombre}, OU={Equipo}, O={Organizacion}, L={Ciudad}, S={Estado}, C={Pais}"
   ```
   - `{nombre-app}`: nombre del paquete en minusculas con guiones (ej: `my-app`)
   - `validity 10000`: ~27 anos — Google Play requiere validez hasta al menos 2033

3. [ ] **Confirmar backup del keystore** — preguntar explicitamente al usuario:
   > "Antes de continuar, confirmas que hiciste backup del keystore en al menos 2 ubicaciones
   > (offline + password manager)?"
   - NO continuar hasta que el usuario confirme explicitamente

4. [ ] **Convertir a base64 y configurar secrets**:

   macOS/Linux:
   ```bash
   base64 -i {nombre-app}-release.keystore | tr -d '\n' | gh secret set KEYSTORE_BASE64
   gh secret set KEYSTORE_PASSWORD --body "{password}"
   gh secret set KEY_ALIAS --body "{nombre-app}-key"
   gh secret set KEY_PASSWORD --body "{key-password}"
   ```

   Windows PowerShell:
   ```powershell
   [Convert]::ToBase64String([IO.File]::ReadAllBytes('{nombre-app}-release.keystore')) | gh secret set KEYSTORE_BASE64
   gh secret set KEYSTORE_PASSWORD --body "{password}"
   gh secret set KEY_ALIAS --body "{nombre-app}-key"
   gh secret set KEY_PASSWORD --body "{key-password}"
   ```

5. [ ] **Agregar al `.gitignore`** — verificar que estas entradas existen; agregarlas si no:
   ```
   *.keystore
   *.jks
   keystore.properties
   ```

---

#### Paso 4.2: Google Play Service Account

1. [ ] Google Play Console → Setup (menu izquierdo) → API Access

2. [ ] Si no hay proyecto vinculado:
   - Clic en "Link to a Google Cloud project"
   - Crear un proyecto nuevo o vincular uno existente

3. [ ] En Google Cloud Console (`console.cloud.google.com`):
   - IAM & Admin → Service Accounts → Create Service Account
   - Name: `{nombre-app}-play-deploy`
   - **No asignar roles a nivel de proyecto** — dejar vacio "Grant this service account access to project"

4. [ ] Crear JSON key:
   - Clic en la service account → Keys → Add Key → Create new key → JSON
   - Descargar el archivo JSON generado

5. [ ] En Google Play Console → Users & Permissions → Invite new user:
   - Email: la service account (`{name}@{project}.iam.gserviceaccount.com`)
   - Permissions: seleccionar la app especifica → asignar SOLO:
     - "Release to production, exclude devices, and use app signing by Google Play"
   - NO asignar: financial reports, store listing edit, user management

6. [ ] **Configurar secret** (sin base64 — Fastlane espera JSON raw):
   ```bash
   gh secret set GOOGLE_PLAY_JSON_KEY < {nombre-app}-play-deploy.json
   ```

7. [ ] Agregar al `.gitignore`:
   ```
   *-play-deploy.json
   *service-account*.json
   ```

---

#### Paso 4.3: Verificar build.gradle (targetSdkVersion)

1. [ ] Leer `android/app/build.gradle` y verificar:
   - `targetSdkVersion` = 35 (requerido por Google Play desde agosto 2025)
   - `minSdkVersion` >= 24
   - `compileSdkVersion` = 35

2. [ ] Si `targetSdkVersion` < 35 → emitir WARNING con instruccion exacta:
   ```groovy
   // En android/app/build.gradle — cambiar a:
   android {
       compileSdkVersion 35
       defaultConfig {
           targetSdkVersion 35
           minSdkVersion 24
       }
   }
   ```

---

#### Paso 4.4: GitHub Environment con protecciones

1. [ ] Crear el environment `production-android`:
   ```bash
   gh api repos/{owner}/{repo}/environments/production-android \
     --method PUT \
     --field deployment_branch_policy='{"protected_branches":false,"custom_branch_policies":true}'
   ```

2. [ ] Configurar en GitHub UI (Settings → Environments → production-android):
   - Required reviewers: 1
   - Deployment branch policy: only `main`

### CHECKPOINT
> Verificar antes de continuar a Phase 5

- [ ] Keystore generado o existente y confirmado como activo
- [ ] Backup del keystore realizado y confirmado explicitamente por el usuario
- [ ] 4 secrets Android configurados: `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`
- [ ] `GOOGLE_PLAY_JSON_KEY` configurado con scope minimo (solo la app, no la cuenta)
- [ ] `*.keystore`, `*.jks`, `keystore.properties` en `.gitignore`
- [ ] `targetSdkVersion` = 35 verificado en `android/app/build.gradle`
- [ ] GitHub Environment `production-android` creado con protecciones

### IF FAILS
```
Keystore perdido:
  → NO HAY RECOVERY para apps ya en produccion
  → Documentar que la app existente no puede recibir updates via ese keystore
  → Para proyectos nuevos o apps sin publicar: generar nuevo keystore y registrar la app desde cero
  → Escalar al usuario antes de cualquier accion — esta decision no es reversible

Service account sin permisos en Play Console:
  → Verificar que el email invitado coincide exactamente con el de la service account
     ({name}@{project}.iam.gserviceaccount.com)
  → Verificar que los permisos se asignaron a nivel de app (no cuenta global)
  → Esperar hasta 24 horas — los permisos de Play Console pueden tardar en propagarse

targetSdkVersion < 35 con builds existentes en produccion:
  → WARNING — no bloquear, pero documentar
  → Instruccion: actualizar build.gradle, hacer build de prueba en internal testing
     antes de promotion a production
  → Google Play rechazara nuevos builds con targetSdkVersion < 35 desde agosto 2025
```

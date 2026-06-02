# open-design-coolify — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Crear el repositorio `GrupoReacciona/open-design-coolify` con todos los ficheros necesarios para desplegar Open Design en Coolify usando la imagen prebuilt oficial, con HTTPS automático, OD_API_TOKEN obligatorio y Basic Auth en Traefik.

**Architecture:** Un recurso "Docker Compose" de Coolify apuntando a este repo. Coolify sustituye `SERVICE_FQDN_OPEN_DESIGN_7456` con el dominio asignado y genera el routing Traefik+TLS automáticamente. El contenedor usa la imagen prebuilt `docker.io/vanjayak/open-design:latest`, hardening completo (read-only, tmpfs, no-new-privileges, mem/pids limits) y un volumen nombrado para persistencia. La doble capa de seguridad (OD_API_TOKEN + Basic Auth) se configura vía variables de entorno en Coolify, nunca en el repo.

**Tech Stack:** Docker Compose v2, Traefik (gestionado por Coolify), imagen `docker.io/vanjayak/open-design:latest` (Node 24 + Express + SQLite), GitHub CLI (`gh`).

---

## File Map

| Fichero | Acción | Responsabilidad |
|---|---|---|
| `docker-compose.yml` | Crear | Definición del servicio: imagen, expose, variables Coolify, hardening, volumen, healthcheck, label Basic Auth |
| `.env.example` | Crear | Documentar todas las variables con descripción y cómo generarlas (sin valores reales) |
| `.gitignore` | Crear | Ignorar `.env`, `*.env`, `node_modules`, artefactos locales |
| `README.md` | Crear | Guía paso a paso de despliegue, activación de Basic Auth, actualización y backup |
| `docs/superpowers/specs/` | Ya existe | Spec de diseño (ya commiteado en Task 1) |
| `docs/superpowers/plans/` | Ya existe | Este plan |

---

## Task 1: Inicializar repo git standalone y commitear spec

**Files:**
- Create: `.gitignore`

> El directorio `/Users/cgomez/proyectos/open-design` vive dentro del repo git del home (`/Users/cgomez`). Necesitamos inicializarlo como repo propio para que Coolify pueda clonarlo independientemente.

- [ ] **Step 1: Inicializar nuevo repo git en el directorio del proyecto**

```bash
cd /Users/cgomez/proyectos/open-design
git init
git checkout -b main
```

Expected: `Initialized empty Git repository in .../open-design/.git/`

- [ ] **Step 2: Crear `.gitignore`**

Crear `/Users/cgomez/proyectos/open-design/.gitignore` con este contenido exacto:

```gitignore
.env
*.env
*.env.local
open_design_data/
node_modules/
.DS_Store
```

- [ ] **Step 3: Commitear spec y plan existentes**

```bash
git add .gitignore docs/
git commit -m "chore: init repo with design spec and implementation plan"
```

Expected: commit con 3 ficheros (`.gitignore`, spec `.md`, plan `.md`).

---

## Task 2: Crear docker-compose.yml

**Files:**
- Create: `docker-compose.yml`

> **Convención Coolify SERVICE_FQDN:** El nombre del servicio `open-design` se normaliza a `OPEN_DESIGN` (mayúsculas, guiones → guiones bajos). La variable mágica para el dominio en el puerto 7456 es `SERVICE_FQDN_OPEN_DESIGN_7456`. La URL completa es `SERVICE_URL_OPEN_DESIGN`.

- [ ] **Step 1: Crear `docker-compose.yml`**

Crear `/Users/cgomez/proyectos/open-design/docker-compose.yml` con este contenido exacto:

```yaml
name: open-design

services:
  open-design:
    container_name: open-design
    image: ${OPEN_DESIGN_IMAGE:-docker.io/vanjayak/open-design:latest}
    restart: always
    expose:
      - "7456"
    environment:
      SERVICE_FQDN_OPEN_DESIGN_7456: ""
      NODE_ENV: production
      NODE_OPTIONS: ${NODE_OPTIONS:---max-old-space-size=192}
      OD_BIND_HOST: 0.0.0.0
      OD_ALLOWED_ORIGINS: ${SERVICE_URL_OPEN_DESIGN:-}
      OD_PORT: 7456
      OD_WEB_PORT: 7456
      OD_API_TOKEN: ${OD_API_TOKEN:?Set OD_API_TOKEN in Coolify Environment Variables (openssl rand -hex 32)}
      OD_CODEX_SANDBOX: ${OD_CODEX_SANDBOX:-}
    labels:
      - "traefik.http.middlewares.od-basicauth.basicauth.users=${OD_BASICAUTH_USERS}"
    volumes:
      - open_design_data:/app/.od
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    mem_limit: ${OPEN_DESIGN_MEM_LIMIT:-384m}
    pids_limit: 256
    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "fetch('http://127.0.0.1:7456/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
        ]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s

volumes:
  open_design_data:
```

- [ ] **Step 2: Validar sintaxis del compose**

```bash
cd /Users/cgomez/proyectos/open-design
OD_API_TOKEN=dummy OD_BASICAUTH_USERS=dummy docker compose config --quiet
```

Expected: salida vacía (sin errores). Si falla con "service 'open-design' has neither an image nor a build", asegúrate de que el campo `image:` está escrito correctamente.

- [ ] **Step 3: Verificar que la variable requerida falla correctamente sin valor**

```bash
docker compose config --quiet 2>&1 | head -5
```

Expected: error mencionando `OD_API_TOKEN` y el mensaje `Set OD_API_TOKEN in Coolify Environment Variables`.

- [ ] **Step 4: Commit**

```bash
git add docker-compose.yml
git commit -m "feat: add Coolify-adapted docker-compose.yml"
```

---

## Task 3: Crear .env.example

**Files:**
- Create: `.env.example`

- [ ] **Step 1: Crear `.env.example`**

Crear `/Users/cgomez/proyectos/open-design/.env.example` con este contenido exacto:

```bash
# ============================================================
# open-design-coolify — Environment Variables
# ============================================================
# Set these in Coolify → Resource → Environment Variables.
# NEVER commit a .env file with real values to git.
# ============================================================

# REQUIRED: API token para el daemon de Open Design.
# Genera uno con: openssl rand -hex 32
OD_API_TOKEN=

# REQUIRED: Hash de usuario/contraseña para Basic Auth en Traefik.
# Genera uno con: htpasswd -nbB yourusername yourpassword
# IMPORTANTE: al pegar en Coolify, escapa los $ duplicándolos: $ → $$
# Ejemplo del formato resultante: admin:$$2y$$12$$abc...xyz
OD_BASICAUTH_USERS=

# OPTIONAL: Tag de imagen Docker. Default: latest
# Útil para fijar una versión concreta: docker.io/vanjayak/open-design:1.2.3
OPEN_DESIGN_IMAGE=docker.io/vanjayak/open-design:latest

# OPTIONAL: Límite de memoria del contenedor. Default: 384m
# Sube a 512m o más si haces exports grandes o múltiples agentes en paralelo.
OPEN_DESIGN_MEM_LIMIT=384m

# OPTIONAL: Tamaño máximo del heap de Node.js dentro del contenedor. Default: 192
# Debe ser < OPEN_DESIGN_MEM_LIMIT (en MiB).
NODE_OPTIONS=--max-old-space-size=192

# OPTIONAL: Override del sandbox de Codex CLI.
# Establecer a "danger-full-access" SOLO si Codex falla con errores de sandbox
# en instalaciones de un único usuario de confianza. No usar en entornos compartidos.
OD_CODEX_SANDBOX=
```

- [ ] **Step 2: Verificar que .env no está en git tracking**

```bash
echo "test=value" > /tmp/test-env-check.env
cp /tmp/test-env-check.env /Users/cgomez/proyectos/open-design/.env
git status | grep "\.env"
```

Expected: `.env` aparece como "untracked" pero NO en la lista de ficheros a commitear (por el `.gitignore`). Eliminar el fichero de prueba:

```bash
rm /Users/cgomez/proyectos/open-design/.env
```

- [ ] **Step 3: Commit**

```bash
git add .env.example
git commit -m "docs: add .env.example with all configurable variables"
```

---

## Task 4: Crear README.md

**Files:**
- Create: `README.md`

- [ ] **Step 1: Crear `README.md`**

Crear `/Users/cgomez/proyectos/open-design/README.md` con este contenido exacto:

```markdown
# open-design-coolify

Configuración de despliegue de [Open Design](https://github.com/nexu-io/open-design) en [Coolify](https://coolify.io/).

> Open Design es un workspace de diseño local-first y open-source desarrollado por [nexu-io](https://github.com/nexu-io), licenciado bajo Apache-2.0. Este repositorio contiene únicamente configuración de despliegue.

---

## Requisitos

- Instancia de Coolify funcionando con un dominio wildcard configurado
- `htpasswd` instalado localmente (`apt install apache2-utils` / `brew install httpd`)
- Acceso al panel de Coolify

---

## Despliegue inicial (5 pasos)

### Paso 1 — Crear recurso en Coolify

1. En Coolify: **New Resource → Docker Compose**
2. Seleccionar **Git Repository** y pegar la URL:
   ```
   https://github.com/GrupoReacciona/open-design-coolify
   ```
3. Branch: `main`
4. Coolify detectará el `docker-compose.yml` automáticamente.

### Paso 2 — Asignar dominio

En la pestaña **Domains**, asignar un dominio al servicio `open-design` (p.ej. `design.tudominio.com`).

### Paso 3 — Configurar variables de entorno

En la pestaña **Environment Variables**, añadir:

| Variable | Cómo generarla | Ejemplo del formato |
|---|---|---|
| `OD_API_TOKEN` | `openssl rand -hex 32` | `a3f8c2...` (64 chars hex) |
| `OD_BASICAUTH_USERS` | `htpasswd -nbB usuario contraseña` | `admin:$$2y$$12$$abc...xyz` |

> ⚠️ **Importante con `OD_BASICAUTH_USERS`:** el hash generado por `htpasswd` contiene signos `$`. Al pegarlo en Coolify, **duplica cada `$`** (escríbelo como `$$`). De lo contrario Coolify interpreta `$x` como una variable de entorno vacía y el hash queda roto.
>
> Ejemplo: si `htpasswd` genera `admin:$2y$12$abc`, pega `admin:$$2y$$12$$abc`.

### Paso 4 — Desplegar

Haz clic en **Deploy**. El contenedor arrancará, descargará la imagen y estará saludable en ~20 segundos (el healthcheck lo verifica).

### Paso 5 — Activar Basic Auth (única vez tras el primer deploy)

El middleware `od-basicauth` está definido pero aún no está asociado al router autogenerado por Coolify. Sigue estos pasos:

1. En el recurso, abre **Edit Compose → Show Deployable Compose**
2. Busca la label con `traefik.http.routers.<nombre-generado>.rule` — apunta `<nombre-generado>`
3. En tu fork de este repo (o directamente en el editor de Coolify), añade esta línea al bloque `labels:` del servicio:
   ```yaml
   - "traefik.http.routers.<nombre-generado>.middlewares=od-basicauth"
   ```
   Sustituyendo `<nombre-generado>` por el valor real (p.ej. `https-0-abc123def456`).
4. Haz **Redeploy**

A partir de este momento, todos los accesos requieren las credenciales de Basic Auth.

---

## Actualizar Open Design

Para actualizar a una versión nueva de la imagen:

1. En Coolify → Environment Variables, cambia `OPEN_DESIGN_IMAGE` al tag deseado:
   ```
   docker.io/vanjayak/open-design:1.2.3
   ```
2. Haz clic en **Redeploy**

Para seguir siempre `latest`, deja `OPEN_DESIGN_IMAGE` sin definir (o con el valor `docker.io/vanjayak/open-design:latest`) y haz Redeploy cuando quieras actualizar.

---

## Persistencia de datos

Los proyectos y el estado SQLite del daemon se almacenan en el volumen Docker `open_design_data` montado en `/app/.od` dentro del contenedor. Los datos **sobreviven** a reinicios del contenedor y actualizaciones de imagen.

### Hacer backup

Ejecuta esto en el host donde corre Coolify:

```bash
docker run --rm \
  --volumes-from open-design \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/open-design-backup-$(date +%Y%m%d).tar.gz /app/.od
```

---

## Seguridad

Este despliegue usa dos capas de protección:

| Capa | Mecanismo | Descripción |
|---|---|---|
| 1 | `OD_API_TOKEN` | El daemon rechaza cualquier llamada API sin token válido. El contenedor no arranca si no está definido. |
| 2 | Basic Auth (Traefik) | HTTP Basic Auth delante de todo el tráfico, antes de que llegue al daemon. Se activa en el Paso 5. |

> ⚠️ Este compose no expone ningún puerto directamente al host (`expose:` en lugar de `ports:`). Todo el tráfico va a través de Traefik. No modifiques esto.

> ⚠️ `OD_CODEX_SANDBOX=danger-full-access` solo debe usarse en instalaciones de un único usuario de confianza donde Codex falla con errores de sandbox. No lo actives en entornos compartidos.

---

## Atribución

Open Design es software de [nexu-io/open-design](https://github.com/nexu-io/open-design), licenciado bajo **Apache-2.0**.  
Este repositorio contiene únicamente configuración de despliegue y no redistribuye el software original.
```

- [ ] **Step 2: Verificar que el README tiene todas las secciones**

```bash
grep "^## " /Users/cgomez/proyectos/open-design/README.md
```

Expected output:
```
## Requisitos
## Despliegue inicial (5 pasos)
## Actualizar Open Design
## Persistencia de datos
## Seguridad
## Atribución
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add full deployment guide for Coolify"
```

---

## Task 5: Publicar en GitHub y verificar repo

**Files:** ninguno nuevo (solo push)

- [ ] **Step 1: Verificar estado del repo local antes de publicar**

```bash
cd /Users/cgomez/proyectos/open-design
git log --oneline
```

Expected: 4 commits en orden (init → docker-compose → .env.example → README).

- [ ] **Step 2: Crear repo en GitHub y hacer push**

```bash
gh repo create GrupoReacciona/open-design-coolify \
  --public \
  --description "Coolify deployment config for Open Design (nexu-io/open-design)" \
  --source=. \
  --remote=origin \
  --push
```

Expected: URL del repo nuevo, p.ej. `https://github.com/GrupoReacciona/open-design-coolify`.

> Si no tienes permisos para crear repos en `GrupoReacciona`, sustitúyelo por `cegomez-gr` (tu cuenta personal) y transfiere el repo a la org después desde GitHub.

- [ ] **Step 3: Verificar que el repo está accesible y el compose se ve correctamente**

```bash
gh repo view GrupoReacciona/open-design-coolify --web
```

Confirma visualmente en el navegador que:
- `docker-compose.yml` aparece en la raíz
- `README.md` se renderiza correctamente
- `.env.example` está visible
- `.env` NO aparece

- [ ] **Step 4: Verificar que el remote está configurado**

```bash
git remote -v
```

Expected:
```
origin  git@github.com:GrupoReacciona/open-design-coolify.git (fetch)
origin  git@github.com:GrupoReacciona/open-design-coolify.git (push)
```

---

## Notas post-despliegue

### Verificar convención SERVICE_FQDN si Coolify no rutea

Si al desplegar Coolify no asigna dominio automáticamente al servicio, puede ser que la convención del nombre de variable sea distinta. Comprueba en "Show Deployable Compose" qué variables mágicas ha sustituido Coolify. Si usa un nombre distinto (p.ej. `SERVICE_FQDN_OPENDESIGN_7456` sin guion bajo), actualiza el `docker-compose.yml` en consecuencia y haz Redeploy.

La convención documentada es: nombre del servicio en mayúsculas con guiones reemplazados por guiones bajos. Para el servicio `open-design` → `OPEN_DESIGN`.

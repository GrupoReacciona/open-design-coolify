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

> ⚠️ **Tras el deploy, el servicio es accesible públicamente hasta que completes el Paso 5.** El API Token (`OD_API_TOKEN`) protege la API, pero la interfaz web queda expuesta. Completa el Paso 5 inmediatamente después.

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

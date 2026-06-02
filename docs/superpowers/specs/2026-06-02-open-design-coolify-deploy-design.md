# Design Spec: open-design-coolify — Repositorio de despliegue para Coolify

**Fecha:** 2026-06-02  
**Estado:** Aprobado  
**Software upstream:** [nexu-io/open-design](https://github.com/nexu-io/open-design) (Apache-2.0)  
**Repositorio destino:** `GrupoReacciona/open-design-coolify`

---

## Objetivo

Crear un repositorio git mínimo que permita desplegar [Open Design](https://open-design.ai/) en Coolify con:
- Imagen oficial prebuilt (`docker.io/vanjayak/open-design:latest`)
- HTTPS y routing automáticos vía Traefik de Coolify
- Doble capa de seguridad: `OD_API_TOKEN` obligatorio + Basic Auth en Traefik
- Hardening completo del contenedor (read-only, tmpfs, no-new-privileges, mem/pids limits)
- Persistencia de datos en volumen Docker declarado como código
- README con guía paso a paso para desplegar y mantener

---

## Estructura del repositorio

```
open-design-coolify/
├── docker-compose.yml      # recurso "Docker Compose" de Coolify
├── .env.example            # documenta variables requeridas/opcionales
├── .gitignore              # ignora .env y artefactos locales
└── README.md               # guía de despliegue, seguridad y mantenimiento
```

---

## docker-compose.yml

### Imagen

Imagen prebuilt oficial. El tag se controla vía variable para facilitar actualizaciones:

```yaml
image: ${OPEN_DESIGN_IMAGE:-docker.io/vanjayak/open-design:latest}
```

No se incluye `build:` (no se compila desde fuente).

### Networking con Coolify

En lugar de `ports:`, se usa `expose:` más la variable mágica `SERVICE_FQDN` para que Traefik de Coolify enrute el tráfico y termine TLS:

```yaml
expose:
  - "7456"
environment:
  SERVICE_FQDN_OPENDESIGN_7456: ""
```

`SERVICE_FQDN_OPENDESIGN_7456` es sustituida por Coolify con el FQDN asignado al servicio. Coolify usa el nombre del servicio (`open-design` en el compose) para generar la variable; el sufijo `_7456` indica el puerto interno.

`OD_ALLOWED_ORIGINS` se alimenta de `SERVICE_URL_OPENDESIGN` para que el CORS apunte al dominio generado:

```yaml
OD_ALLOWED_ORIGINS: ${SERVICE_URL_OPENDESIGN:-}
```

### Seguridad — capa 1: OD_API_TOKEN

El daemon exige token. Si no está definido, el compose falla al arrancar con mensaje de error explícito:

```yaml
OD_API_TOKEN: ${OD_API_TOKEN:?Set OD_API_TOKEN in Coolify Environment Variables (openssl rand -hex 32)}
```

### Seguridad — capa 2: Basic Auth (Traefik label)

Se añade un middleware `basicauth` vía label de Traefik. El hash de usuarios viene de la variable `OD_BASICAUTH_USERS` (configurada en Coolify, **no** en el repo):

```yaml
labels:
  - "traefik.http.middlewares.od-basicauth.basicauth.users=${OD_BASICAUTH_USERS}"
```

> **Nota de implementación:** Coolify autogenera el nombre del router (visible en "Show Deployable Compose"). El paso de asociar el middleware al router se documenta en README como un paso de una sola vez post-primer-deploy, ya que requiere conocer el nombre de router autogenerado.

### Hardening del contenedor

Se mantiene todo el hardening del `docker-compose.yml` oficial:

```yaml
read_only: true
tmpfs:
  - /tmp
security_opt:
  - no-new-privileges:true
mem_limit: ${OPEN_DESIGN_MEM_LIMIT:-384m}
pids_limit: 256
```

### Persistencia

Volumen nombrado para los proyectos y estado SQLite del daemon:

```yaml
volumes:
  - open_design_data:/app/.od

volumes:
  open_design_data:
```

### Healthcheck

Sondea el endpoint `/api/health` interno cada 30 segundos:

```yaml
healthcheck:
  test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:7456/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 20s
```

### Reinicio automático

```yaml
restart: always
```

---

## .env.example

Documenta las variables que hay que rellenar en "Environment Variables" de Coolify. **Este archivo nunca lleva valores reales.** Incluye:

| Variable | Requerida | Descripción |
|---|---|---|
| `OD_API_TOKEN` | Sí | Token de seguridad del daemon. `openssl rand -hex 32` |
| `OD_BASICAUTH_USERS` | Sí | Hash htpasswd para Basic Auth. `htpasswd -nbB usuario pass` |
| `OPEN_DESIGN_IMAGE` | No | Tag de imagen. Default: `docker.io/vanjayak/open-design:latest` |
| `OPEN_DESIGN_MEM_LIMIT` | No | Límite de memoria del contenedor. Default: `384m` |

---

## .gitignore

```
.env
*.env
open_design_data/
```

---

## README.md — Contenido

El README cubre en orden:

1. **Qué es esto** — Una línea + enlace a nexu-io/open-design (atribución Apache-2.0)
2. **Requisitos** — Coolify funcionando con un dominio wildcard configurado
3. **Despliegue paso a paso:**
   - Crear recurso "Docker Compose" en Coolify apuntando al repo
   - Asignar dominio al servicio
   - Generar `OD_API_TOKEN` (`openssl rand -hex 32`) y añadirlo en Environment Variables
   - Generar hash Basic Auth (`htpasswd -nbB usuario password`) y añadir `OD_BASICAUTH_USERS`
   - Hacer Deploy
4. **Paso único post-deploy — activar Basic Auth:**
   - Ir a "Show Deployable Compose" para ver el nombre de router autogenerado
   - Añadir label al servicio asociando el middleware `od-basicauth` al router
   - Hacer Redeploy
5. **Actualizar imagen** — Cambiar `OPEN_DESIGN_IMAGE` a nuevo tag → Redeploy
6. **Persistencia y backups** — Volumen `open_design_data`; cómo hacer backup con `docker cp` o `docker run --volumes-from`
7. **Notas de seguridad** — Resumen de las dos capas; advertencia sobre `OD_CODEX_SANDBOX`

---

## Decisiones clave y su justificación

| Decisión | Alternativa descartada | Razón |
|---|---|---|
| Imagen prebuilt | Build desde fuente | Más rápido, sin toolchain Node/pnpm en Coolify |
| `expose:` + `SERVICE_FQDN` | `ports: "0.0.0.0:7456:7456"` | Traefik gestiona TLS y routing; no exponer puerto crudo |
| `OD_BASICAUTH_USERS` como env var | Hash hardcodeado en el compose | El repo es público; los secretos van en Coolify |
| Middleware label en compose | Solo configurar en Traefik UI de Coolify | Config como código, reproducible |
| Paso manual de asociar router (post-deploy) | Pre-cablear nombre de router | El nombre lo genera Coolify; no es conocible antes del primer deploy |

---

## Lo que este repo NO hace

- No gestiona el agente de IA ni sus credenciales (BYOK de cada usuario)
- No compila Open Design desde fuente
- No configura Coolify ni Traefik más allá del servicio (wildcard DNS, certificados, etc.)
- No toma decisiones autónomas de infraestructura: todo cambio requiere confirmación explícita

---

## Atribución

Open Design es software de [nexu-io/open-design](https://github.com/nexu-io/open-design), licenciado bajo Apache-2.0. Este repositorio contiene únicamente configuración de despliegue.

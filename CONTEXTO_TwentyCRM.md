# CONTEXTO PROYECTO · creador-crm (Twenty CRM)

> Estado vivo de este proyecto. Copia y pega este archivo completo como contexto en otro chat de Claude para continuar el trabajo sin pérdida de información.
>
> Última actualización: 2026-08-18 — **primer arranque + gobernanza de licencias**: se arrancó por primera vez el stack vía Docker Compose (`packages/twenty-docker/docker-compose.yml`); se auditó el repositorio en busca de archivos con comentario `/* @license Enterprise */` (315 encontrados, ver sección 3); se documentó en `CLAUDE.md` la decisión de no borrarlos y la norma de no configurar nunca `ENTERPRISE_KEY` salvo compra explícita de licencia para un cliente concreto (ver sección 4). Se crearon este archivo y la sección de gobernanza de `CLAUDE.md` en la misma sesión.

---

# 1. QUÉ ES ESTE PROYECTO

`creador-crm` es un fork propio de [Twenty](https://twenty.com) (CRM open-source, licencia AGPLv3 con módulos comerciales marcados aparte), clonado dentro de `proyectos-code/` como carpeta hermana de `creador-web/`. Repositorio propio: `https://github.com/Alexitrup/twenty-crm-base` (git anidado — tiene su propio `.git`, independiente del repo `proyectos-code`; `proyectos-code` lo ve como carpeta sin trackear, no como submódulo).

Pensado como base para ofrecer CRM autoalojado a clientes de `creador-web/`, cada uno con su propia instancia cuando corresponda — no un multi-tenant compartido por defecto (aunque Twenty soporta modo multi-workspace, ver `packages/twenty-docs/developers/self-host/capabilities/setup.mdx`).

La guía técnica de Twenty para trabajar en su propio código (comandos `nx`, arquitectura, convenciones de estilo) vive en `CLAUDE.md` de este mismo proyecto, sección "Guía técnica de Twenty (upstream, sin modificar)" — no se duplica aquí.

---

# 2. INFRAESTRUCTURA — ESTADO ACTUAL

**Arrancado por primera vez el 2026-08-18**, vía Docker Compose (`packages/twenty-docker/docker-compose.yml`), con Docker Desktop en la máquina local.

Servicios y estado tras el primer arranque:

| Servicio | Imagen | Estado | Puerto |
|---|---|---|---|
| `server` | `twentycrm/twenty:latest` | healthy | `3000` (host) → `3000` |
| `worker` | `twentycrm/twenty:latest` | up | — |
| `db` | `postgres:16` | healthy | interno (`5432`, no publicado al host) |
| `redis` | `redis` | healthy | interno (`6379`, no publicado al host) |

Acceso: `http://localhost:3000`.

**`.env` de `packages/twenty-docker/`:** generado a partir de `.env.example` en esta misma sesión. No está commiteado (upstream lo ignora vía `.gitignore`). Contiene un `ENCRYPTION_KEY` propio generado con `openssl rand -base64 32` — **si se pierde, se pierde el acceso a todo secreto cifrado en la base de datos** (tokens OAuth, variables de configuración sensibles, etc. — ver `packages/twenty-docs/developers/self-host/capabilities/setup.mdx`). Pendiente: guardar una copia de `ENCRYPTION_KEY` en un gestor de secretos fuera del repositorio.

`SERVER_URL` y `STORAGE_TYPE=local` quedan en sus valores de `.env.example` (desarrollo local, sin S3). `ENTERPRISE_KEY` no está definida — ver sección 4.

---

# 3. ARCHIVOS BAJO LICENCIA ENTERPRISE — INVENTARIO

Verificado el 2026-08-18 con `grep -rl "@license Enterprise"` sobre todo el repo (excluyendo `.yarn/`). **315 archivos totales.**

| Carpeta | Archivos |
|---|---|
| `packages/twenty-server/src/engine/core-modules/billing` | 119 |
| `packages/twenty-front/src/modules/settings` | 45 |
| `packages/twenty-server/src/engine/metadata-modules` | 25 |
| `packages/twenty-server/src/engine/core-modules/billing-webhook` | 22 |
| `packages/twenty-server/src/engine/core-modules/usage` | 15 |
| `packages/twenty-server/src/engine/core-modules/sso` | 14 |
| `packages/twenty-server/src/engine/core-modules/event-logs` | 12 |
| `packages/twenty-server/src/engine/workspace-manager` (row-level permissions) | 12 |
| `packages/twenty-server/src/engine/core-modules/auth` | 10 |
| `packages/twenty-server/src/engine/core-modules/enterprise` | 9 |
| `packages/twenty-shared/src/types` (row-level permissions) | 5 |
| resto (jwt, emailing-domain, cloudflare, dns-manager, twenty-orm, front/auth, front/object-record, front/pages, server/database/commands, test, LICENSE) | 27 |

Resumen por paquete: 256 en `twenty-server`, 52 en `twenty-front`, 5 en `twenty-shared`, 2 sueltos (un test de integración y el propio `LICENSE`).

**Mecanismo de gating:** `EnterprisePlanService` (`packages/twenty-server/src/engine/core-modules/enterprise/services/enterprise-plan.service.ts`) valida `ENTERPRISE_KEY` como JWT firmado contra la API de Twenty. Sin clave válida, `isValid()` devuelve `false` y las funciones quedan inactivas — el código está presente pero no se ejecuta. Decisión y razones completas en `CLAUDE.md`, sección "Decisión: no borrar los archivos con `/* @license Enterprise */`".

**No se ha borrado ni modificado ninguno de estos 315 archivos.**

---

# 4. LICENCIAS ENTERPRISE POR CLIENTE

Norma completa en `CLAUDE.md`, sección "Norma: `ENTERPRISE_KEY` nunca se configura salvo compra explícita para un cliente concreto". Esta tabla es el registro vivo de excepciones — se actualiza cada vez que un cliente compre o cancele una licencia Enterprise.

| Cliente | Instancia | `ENTERPRISE_KEY` configurada | Fecha de alta | Notas |
|---|---|---|---|---|
| — | — | — | — | Ningún cliente tiene licencia Enterprise comprada a día de hoy (2026-08-18). |

---

# 5. ESTRUCTURA DE CARPETAS VIGENTE

```
proyectos-code/                    ← raíz del repositorio Git (proyectos-code, no incluye creador-crm)
├── CLAUDE.md                      ← instrucciones de sistema (ramas, Vercel, estructura)
├── CONTEXTO.md                    ← estado general de sistema
├── creador-web/                   ← proyecto hermano: agencia de webs con IA
└── creador-crm/                   ← este proyecto — git anidado propio, no trackeado por proyectos-code
    ├── CLAUDE.md                  ← gobernanza propia + guía técnica upstream de Twenty (sin modificar)
    ├── CONTEXTO_TwentyCRM.md      ← este archivo
    └── packages/
        ├── twenty-server/         ← backend NestJS
        ├── twenty-front/          ← frontend React
        ├── twenty-shared/         ← tipos/utilidades comunes (MIT)
        ├── twenty-docker/         ← docker-compose.yml, docker-compose.dev.yml, .env.example, .env (local, no versionado)
        └── ...                    ← resto de paquetes del monorepo Nx de Twenty (ver CLAUDE.md, "Package Structure")
```

---

# 6. PENDIENTES

- [ ] Guardar `ENCRYPTION_KEY` del `.env` local en un gestor de secretos (fuera del repositorio).
- [ ] Decidir y documentar el modelo de despliegue por cliente (instancia dedicada vs. multi-workspace) antes de dar de alta al primer cliente real.
- [ ] Configurar `SERVER_URL`, `STORAGE_TYPE` (S3 en vez de local) y credenciales SMTP reales antes de cualquier despliegue que no sea puramente local/desarrollo.

---

*Fin del contexto.*

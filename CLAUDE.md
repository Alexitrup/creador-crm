# CLAUDE.md — creador-crm

Este documento tiene dos partes. La primera es la **gobernanza de este proyecto** dentro de `proyectos-code/` (nuestra, se mantiene y actualiza aquí). La segunda — desde "Guía técnica de Twenty (upstream, sin modificar)" — es la documentación original que trae el propio repositorio de Twenty para trabajar en su código (comandos de build/test/lint, arquitectura, convenciones); no se toca salvo que cambie aguas arriba, y se sincroniza vía `git pull` desde `origin` (fork propio: `https://github.com/Alexitrup/twenty-crm-base`).

---

## GOBERNANZA — decisiones de este proyecto

### Qué es `creador-crm`

Instancia self-hosted de [Twenty](https://twenty.com) (CRM open-source, AGPLv3 + módulos bajo licencia comercial marcados aparte), clonada como fork propio y ejecutada vía Docker Compose (`packages/twenty-docker/docker-compose.yml`). Pensada como base para ofrecer CRM a clientes de `creador-web/`: cada cliente, cuando corresponda, tendría su propia instancia/despliegue — no un multi-tenant compartido por defecto.

### Arranque local

```bash
cd packages/twenty-docker
docker compose up -d      # requiere .env (ver .env.example) — ya existe uno local, no versionado
docker compose ps         # server, worker, db, redis
```

Server en `http://localhost:3000`. El `.env` local se generó a partir de `.env.example` con un `ENCRYPTION_KEY` propio (`openssl rand -base64 32`); no está commiteado (cae bajo el `.gitignore` del propio repo de Twenty).

### Decisión: no borrar los archivos con `/* @license Enterprise */`

315 archivos del código fuente llevan este comentario (verificado el 2026-08-18 con `grep -rl "@license Enterprise"`, excluyendo `.yarn/`). Reparto principal:

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

**Por qué se conservan:** Twenty gatea estas funciones en tiempo de ejecución mediante la variable `ENTERPRISE_KEY` (JWT firmado, validado en `EnterprisePlanService.hasValidSignedEnterpriseKey()` / `isValid()`, en `packages/twenty-server/src/engine/core-modules/enterprise/services/enterprise-plan.service.ts`). Sin esa clave, el código queda compilado pero inactivo — no aporta ninguna función extra estando presente. Borrarlo solo generaría divergencia con el upstream de Twenty (conflictos en cada `git pull`/actualización) sin ninguna ganancia funcional, y complicaría reactivarlo el día que un cliente sí compre licencia.

**Cómo aplicarla en la práctica:** nunca tocar ni eliminar archivos marcados `@license Enterprise` como parte de una "limpieza" o "reducir código no usado". Si en algún momento se vuelve a cuestionar, releer esta sección antes de actuar.

### Norma: `ENTERPRISE_KEY` nunca se configura salvo compra explícita para un cliente concreto

- Por defecto, ninguna instancia de `creador-crm` — ni la de desarrollo/pruebas ni ninguna futura de cliente — debe tener `ENTERPRISE_KEY` en su `.env` ni en su `docker-compose`. El `docker-compose.yml` de este repo no la referencia en absoluto; así debe mantenerse mientras esta norma siga vigente.
- La única excepción es cuando un cliente concreto compra explícitamente una licencia Enterprise de Twenty para su propio despliegue. En ese caso: (1) la clave se configura solo en la instancia de ESE cliente, nunca en la de desarrollo ni en la de otro cliente; (2) se documenta qué cliente, qué instancia y fecha de alta de la licencia — aquí mientras no haya `CONTEXTO_[Cliente].md` propio, y en ese archivo cuando exista.
- **Por qué:** activar funciones Enterprise (billing, SSO, permisos a nivel de fila, límites de uso, etc.) sin que el cliente las haya pagado sería usar código bajo licencia comercial sin la licencia correspondiente — no es una cuestión técnica sino contractual/legal, y la clave es exactamente el mecanismo que Twenty usa para verificarlo contra su propia API.
- Ningún cliente tiene, a día de hoy (2026-08-18), licencia Enterprise comprada. Ver `CONTEXTO_TwentyCRM.md` para el estado vivo de este listado.

---

## Guía técnica de Twenty (upstream, sin modificar)

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Twenty is an open-source CRM built with modern technologies in a monorepo structure. The codebase is organized as an Nx workspace with multiple packages.

## Key Commands

### Development
```bash
# Start development environment (frontend + backend + worker)
yarn start

# Individual package development
npx nx start twenty-front     # Start frontend dev server
npx nx start twenty-server    # Start backend server
npx nx run twenty-server:worker  # Start background worker
```

### Testing
```bash
# Preferred: run a single test file (fast)
npx jest path/to/test.test.ts --config=packages/PROJECT/jest.config.mjs

# Run all tests for a package
npx nx test twenty-front      # Frontend unit tests
npx nx test twenty-server     # Backend unit tests
npx nx run twenty-server:test:integration:with-db-reset  # Integration tests with DB reset
# To run an individual test or a pattern of tests, use the following command:
cd packages/{workspace} && npx jest "pattern or filename"

# Storybook
npx nx storybook:build twenty-front
npx nx storybook:test twenty-front

# When testing the UI end to end, click on "Continue with Email" and use the prefilled credentials.
```

### Code Quality
```bash
# Linting (diff with main - fastest, always prefer this)
npx nx lint:diff-with-main twenty-front
npx nx lint:diff-with-main twenty-server
npx nx lint:diff-with-main twenty-front --configuration=fix  # Auto-fix

# Linting (full project - slower, use only when needed)
npx nx lint twenty-front
npx nx lint twenty-server

# Type checking
npx nx typecheck twenty-front
npx nx typecheck twenty-server

# Format code
npx nx fmt twenty-front
npx nx fmt twenty-server
```

### Build
```bash
# Build packages (twenty-shared must be built first)
npx nx build twenty-shared
npx nx build twenty-front
npx nx build twenty-server
```

### Database Operations
```bash
# Database management
npx nx database:reset twenty-server         # Reset database
npx nx run twenty-server:database:init:prod # Initialize database
npx nx run twenty-server:database:migrate:prod # Run instance commands (fast only)

# Generate an instance command (fast or slow)
npx nx run twenty-server:database:migrate:generate --name <name> --type <fast|slow>
```

### Database Inspection (Postgres MCP)

A read-only Postgres MCP server is configured in `.mcp.json`. Use it to:
- Inspect workspace data, metadata, and object definitions while developing
- Verify migration results (columns, types, constraints) after running migrations
- Explore the multi-tenant schema structure (core, metadata, workspace-specific schemas)
- Debug issues by querying raw data to confirm whether a bug is frontend, backend, or data-level
- Inspect metadata tables to debug GraphQL schema generation issues

This server is read-only — for write operations (reset, migrations, sync), use the CLI commands above.

### GraphQL
```bash
# Generate GraphQL types (run after schema changes)
npx nx run twenty-front:graphql:generate
npx nx run twenty-front:graphql:generate --configuration=metadata
```

## Architecture Overview

### Tech Stack
- **Frontend**: React 18, TypeScript, Jotai (state management), Linaria (styling), Vite
- **Backend**: NestJS, TypeORM, PostgreSQL, Redis, GraphQL (with GraphQL Yoga)
- **Monorepo**: Nx workspace managed with Yarn 4

### Package Structure
```
packages/
├── twenty-front/          # React frontend application
├── twenty-server/         # NestJS backend API
├── twenty-ui/             # Shared UI components library
├── twenty-shared/         # Common types and utilities
├── twenty-emails/         # Email templates with React Email
├── twenty-website/    # Next.js marketing website
├── twenty-docs/           # Documentation website
├── twenty-zapier/         # Zapier integration
└── twenty-e2e-testing/    # Playwright E2E tests
```

### Key Development Principles
- **Functional components only** (no class components)
- **Named exports only** (no default exports)
- **Types over interfaces** (except when extending third-party interfaces)
- **String literals over enums** (except for GraphQL enums)
- **No 'any' type allowed** — strict TypeScript enforced
- **Event handlers preferred over useEffect** for state updates
- **Props down, events up** — unidirectional data flow
- **Composition over inheritance**
- **No abbreviations** in variable names (`user` not `u`, `fieldMetadata` not `fm`)

### Naming Conventions
- **Variables/functions**: camelCase
- **Constants**: SCREAMING_SNAKE_CASE
- **Types/Classes**: PascalCase (suffix component props with `Props`, e.g. `ButtonProps`)
- **Files/directories**: kebab-case with descriptive suffixes (`.component.tsx`, `.service.ts`, `.entity.ts`, `.dto.ts`, `.module.ts`)
- **TypeScript generics**: descriptive names (`TData` not `T`)

### File Structure
- Components under 300 lines, services under 500 lines
- Components in their own directories with tests and stories
- Use `index.ts` barrel exports for clean imports
- Import order: external libraries first, then internal (`@/`), then relative

### Comments
- Use short-form comments (`//`), not JSDoc blocks
- Explain WHY (business logic), not WHAT
- Do not comment obvious code
- Multi-line comments use multiple `//` lines, not `/** */`

### State Management
- **Jotai** for global state: atoms for primitive state, selectors for derived state, atom families for dynamic collections
- Component-specific state with React hooks (`useState`, `useReducer` for complex logic)
- GraphQL cache managed by Apollo Client
- Use functional state updates: `setState(prev => prev + 1)`

### Backend Architecture
- **NestJS modules** for feature organization
- **TypeORM** for database ORM with PostgreSQL
- **GraphQL** API with code-first approach
- **Redis** for caching and session management
- **BullMQ** for background job processing

### Database & Upgrade Commands
- **PostgreSQL** as primary database
- **Redis** for caching and sessions
- **ClickHouse** for analytics (when enabled)
- When changing entity files, generate an **instance command** (`database:migrate:generate --name <name> --type <fast|slow>`)
- **Fast** instance commands handle schema changes; **slow** ones add a `runDataMigration` step for data backfills
- **Workspace commands** iterate over all active/suspended workspaces for per-workspace upgrades
- Commands use `@RegisteredInstanceCommand` and `@RegisteredWorkspaceCommand` decorators for automatic discovery
- Include both `up` and `down` logic in instance commands
- Never delete or rewrite committed instance command `up`/`down` logic
- See `packages/twenty-server/docs/UPGRADE_COMMANDS.md` for full documentation

### Utility Helpers
Use existing helpers from `twenty-shared` instead of manual type guards:
- `isDefined()`, `isNonEmptyString()`, `isNonEmptyArray()`

## Development Workflow

IMPORTANT: Use Context7 for code generation, setup or configuration steps, or library/API documentation. Automatically use the Context7 MCP tools to resolve library IDs and get library docs without waiting for explicit requests.

### Before Making Changes
1. Always run linting (`lint:diff-with-main`) and type checking after code changes
2. Test changes with relevant test suites (prefer single-file test runs)
3. Ensure instance commands are generated for entity changes (`database:migrate:generate`)
4. Check that GraphQL schema changes are backward compatible
5. Run `graphql:generate` after any GraphQL schema changes

### Code Style Notes
- Use **Linaria** for styling with zero-runtime CSS-in-JS (styled-components pattern)
- Follow **Nx** workspace conventions for imports
- Use **Lingui** for internationalization
- Apply security first, then formatting (sanitize before format)

### Testing Strategy
- **Test behavior, not implementation** — focus on user perspective
- **Test pyramid**: 70% unit, 20% integration, 10% E2E
- Query by user-visible elements (text, roles, labels) over test IDs
- Use `@testing-library/user-event` for realistic interactions
- Descriptive test names: "should [behavior] when [condition]"
- Clear mocks between tests with `jest.clearAllMocks()`

## Dev Environment Setup

All dev environments (Claude Code web, Cursor, local) use one script:

```bash
bash packages/twenty-utils/setup-dev-env.sh
```

This handles everything: starts Postgres + Redis (auto-detects local services vs Docker), creates databases, copies `.env` files, and initializes the database schema (runs migrations) on a fresh database. Idempotent — safe to run multiple times.

- `--docker` — force Docker mode (uses `packages/twenty-docker/docker-compose.dev.yml`)
- `--down` — stop services
- `--reset` — wipe data and restart fresh
- **Skip the setup script** for tasks that only read code — architecture questions, code review, documentation, etc.

**Note:** CI workflows (GitHub Actions) manage services via Actions service containers and run setup steps individually — they don't use this script.

## Important Files
- `nx.json` - Nx workspace configuration with task definitions
- `tsconfig.base.json` - Base TypeScript configuration
- `package.json` - Root package with workspace definitions
- `.cursor/rules/` - Detailed development guidelines and best practices

# Semana 1 — Fundamentos: monorepo, auth, dominio (22 h)

**Hito H1:** la app arranca en un teléfono físico, un usuario se registra, inicia sesión, y el
backend desplegado responde `/health` por HTTPS.

## PROMPT DE ARRANQUE

```
/semana 1
```

Equivalente sin skill:

> Trabaja la Semana 1 según `tasks/semana-1.md`. Lee `tasks/ESTADO.md` y `AGENTS.md`, y solo las
> secciones de `PLAN_DESARROLLO.md` que cada tarea indica. Delega cada tarea a un subagente con la
> plantilla de `CLAUDE.md` §5, verifica con comandos reales, y actualiza bitácora y `ESTADO.md` al
> cerrar. No abras la Semana 2 sin el gate en verde.

## TAREAS

| # | Tarea | Plan § | Modelo | Estado | h |
|---|---|---|---|---|---|
| 1.1 | Monorepo pnpm + Turborepo: `packages/config`, `packages/shared` (enums, socket-events, DTOs zod), `packages/fare-engine` con vitest | §1.3, §10.2 | Sonnet | ⬜ | 2.0 |
| 1.2 | Scaffold NestJS en `apps/api`: env por zod, `ConfigModule`, `AllExceptionsFilter`, `LoggingInterceptor`, `ZodValidationPipe`, CORS, módulo `health` | §1.3, §10.3 | Sonnet | ⬜ | 2.5 |
| 1.3 | Prisma: `schema.prisma` completo, conexión a Supabase, migración inicial + migración SQL manual para `geography` e índices GIST | §2.1, §2.2, §2.3 | Sonnet | ⛔ Supabase | 3.0 |
| 1.4 | `seed.ts`: `fare_configs` `chincha-v1`, `service_areas` con el polígono, 1 admin, 3 drivers verificados (2 con `yape_phone`, 1 sin él), 2 clientes | §2.2, §10.4 | Sonnet | ⛔ valores de negocio | 1.5 |
| 1.5 | Módulo `auth`: registro, OTP de teléfono (mock en consola), login, refresh rotatorio, argon2id, `JwtAuthGuard`, `RolesGuard`, `@CurrentUser()` | §4.1, §6 | Sonnet | ⬜ | 4.0 |
| 1.6 | Módulo `users`: `GET/PATCH /users/me`, `switch-role`, `push-token`, CRUD de `saved_places` | §4.2 | Sonnet | ⬜ | 2.0 |
| 1.7 | Scaffold Expo: expo-router, NativeWind v4, tokens de tema, `api-client.ts` con refresh, `expo-secure-store`, store de sesión Zustand | §1.3 | Sonnet | ⬜ | 3.5 |
| 1.8 | Pantallas `(auth)`: login, registro, verificación de teléfono + `ui/` base (Button, Input, Sheet, Badge, Skeleton). **Exigir skill `frontend-design`** | §1.3 | Sonnet + skill | ⬜ | 2.5 |
| 1.9 | Deploy: `Dockerfile` multi-stage, `docker-compose.yml` (api + redis), Caddy con TLS, `deploy.sh`; verificar `/health` por HTTPS | §1.1 | Sonnet | ⛔ droplet | 1.0 |

Orden sugerido: 1.1 → 1.2 → 1.7 (no dependen de cuentas) → 1.3 → 1.4 → 1.5 → 1.6 → 1.8 → 1.9.

## Cómo obtener `DATABASE_URL` y `DIRECT_URL` (desbloquea 1.3)

En Supabase: proyecto → **Connect** → pestaña **ORMs** → Prisma. Dan dos cadenas:

```
# pooler, puerto 6543 — la que usa la app en runtime
DATABASE_URL="postgresql://postgres.<ref>:<password>@aws-0-sa-east-1.pooler.supabase.com:6543/postgres?pgbouncer=true&connection_limit=1"
# conexión directa, puerto 5432 — la que usa prisma migrate
DIRECT_URL="postgresql://postgres.<ref>:<password>@aws-0-sa-east-1.pooler.supabase.com:5432/postgres"
```

Reglas:
- Las dos van en `.env`, nunca en `.env.example` ni en el repo (`.gitignore` ya cubre `.env`).
- `schema.prisma` declara `url = env("DATABASE_URL")` y `directUrl = env("DIRECT_URL")`. Sin
  `directUrl`, `prisma migrate` falla contra el pooler.
- Si la contraseña trae `@`, `#` o `/`, hay que codificarla en URL o la cadena se rompe.
- Antes de usarla: Database → Extensions → habilitar **postgis**. Sin eso, la migración de columnas
  `geography` falla.
- Verificación: `pnpm --filter api prisma validate` y `prisma migrate dev` sobre la DB vacía.

## Contratos que esta semana congela

- Enums de `packages/shared/src/enums.ts` idénticos a §2.1, incluido
  `PaymentStatus = PENDING | CONFIRMED | NOT_RECEIVED`.
- `driver_profiles.yape_phone` existe desde la migración inicial (ADR-09).
- `packageManager: "pnpm@12"`, `.nvmrc` → `22`, Docker → `node:22-alpine`.
- Formato de error uniforme de §4: `{ statusCode, code, message, timestamp }`.

## BITÁCORA

<!-- append-only, ≤6 líneas por sesión, citar archivo:línea, pegar salida literal de comandos -->

## GATE DE CIERRE

Comandos (pegar salida literal en la bitácora):

```bash
pnpm turbo typecheck
pnpm turbo lint
pnpm --filter @delivery/fare-engine test
pnpm --filter api prisma validate
```

- [ ] `pnpm turbo typecheck lint` sin errores ni warnings nuevos
- [ ] `prisma validate` OK y la migración aplica sobre una DB limpia
- [ ] `psql` confirma los índices GIST sobre `driver_locations.geom` y `service_areas.polygon`
- [ ] `GET /health` responde 200 por HTTPS en el droplet, con checks de DB y Redis
- [ ] Registro + login + refresh funcionan contra el backend desplegado (no localhost)
- [ ] La app corre en un **teléfono físico** y completa el flujo de registro
- [ ] `.env.example` contiene todas las variables usadas; ningún secreto en el repo
- [ ] Revisión de diff con `cavecrew-reviewer`; findings resueltos o movidos a Deuda
- [ ] `ESTADO.md` actualizado y commit `feat(semana-1): fundamentos` propuesto al usuario

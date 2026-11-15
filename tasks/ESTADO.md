# Estado del proyecto

**Semana activa:** 0 — Preparación (no hay código todavía)
**Tarea activa:** ninguna. Siguiente: `1.1` Monorepo pnpm + Turborepo
**Última sesión:** 2026-11-14
**Horas consumidas en la semana activa:** 0 / 3 (Fase 0)

## Entorno verificado

| Herramienta | Versión local | Nota |
|---|---|---|
| Node | v24.15.0 | plan exige ≥ 22 ✅ |
| pnpm | 12.10.1 | plan decía `pnpm@9` → se usa `pnpm@12` |
| git | 2.54.0 | repo inicializado. Rama `main`, commit `3b9abfa` «inicio de proyecto», árbol limpio |
| eas-cli | no instalado | solo bloquea la tarea 4.9 |

## Bloqueantes

- [ ] **Supabase:** proyecto en `sa-east-1`, PostGIS habilitado, bucket `documents` privado.
      Entregar `DATABASE_URL` y `DIRECT_URL`. → bloquea `1.3`
- [ ] **Google Cloud:** Places API (New), Geocoding, Routes, Maps SDK Android + API key
      restringida. → bloquea `2.2`
- [ ] **DigitalOcean + dominio:** droplet Ubuntu 24.04 y `api.<dominio>` apuntado. → bloquea `1.9`
- [ ] **Valores de `fare_configs` para Chincha** (decisión de negocio, no técnica).
      Marcadores del plan: base S/ 5.00 · 2 km incluidos · S/ 1.20/km · nocturno S/ 2.00 ·
      comisión 15 %. → bloquea `1.4`
- [ ] **Polígono del área de servicio.** Basta un rectángulo de Chincha Alta + Chincha Baja +
      Pueblo Nuevo + Sunampe para arrancar. → bloquea `1.4`
- [ ] **Cuentas Expo/EAS y Google Play ($25).** → bloquea `4.9` y `5.7`

## Deuda abierta

- `.nvmrc` queda en `22` y el Dockerfile usa `node:22-alpine` aunque local corre Node 24. Alinear
  si aparece una incompatibilidad.
- La tarea `1.1` **no** debe reescribir el `.gitignore` existente; solo añadir lo que falte para el
  monorepo.

## Convenciones operativas autorizadas

- **Rama por semana:** `feat/semana-N`, creada automáticamente por la skill `/semana` sin preguntar.
  Merge a `main` con `--no-ff` solo cuando el gate cierra en verde. Los commits sí requieren OK.
- `.gitignore` ya cubre `node_modules/`, `.env*` (salvo `.env.example`), `dist/`, `.turbo/`,
  `.expo/`, `android/`, `ios/`, `*.keystore`, `*.jks`, `*.apk`, `*.aab`, logs y basura de editor.
- **Toda la planeación está versionada** (`docs/`, `tasks/`, `.claude/`, `AGENTS.md`, `CLAUDE.md`,
  `PLAN_DESARROLLO.md`). Solo `.claude/settings.local.json` queda fuera del repo.
- Remoto: `origin` → `https://github.com/Case1490/ChinchaYa.git`, rama `main`.
- **Los commits no llevan línea `Co-Authored-By`.** Instrucción explícita del usuario (2026-11-14).

## Decisiones de negocio ya cerradas

- **12-nov-2026:** pago contra entrega, efectivo o Yape al número del driver. Sin POS ni pasarela
  en ninguna fase (comisión 4–5 %). Registrado como ADR-09 en `PLAN_DESARROLLO.md` §1.2.

# AGENTS.md — Reglas de trabajo para agentes de IA en `app-delivery`

Este archivo es el contrato técnico que cualquier agente (Claude Code, Cursor, Copilot, Gemini) debe respetar al escribir código en este repositorio. Es la fuente de verdad sobre convenciones. El documento de arquitectura y el plan de fases viven en `PLAN_DESARROLLO.md`; el alcance comercial en `docs/ALCANCE_TECNICO.md`.

---

## 1. Qué es este proyecto

App móvil de encargos punto A → punto B para la provincia de Chincha, Perú. Una sola aplicación Expo con dos modos conmutados por rol (Cliente y Driver), un backend NestJS con Socket.io para dispatch y tracking en tiempo real, y PostgreSQL con PostGIS para consultas de proximidad.

**Lee `PLAN_DESARROLLO.md` antes de escribir código.** El esquema de base de datos, el contrato de API, el contrato de eventos WebSocket y la fórmula de tarifas ya están definidos ahí. No los reinventes ni los "mejores" sin pedirlo.

---

## 2. Stack fijo (no sustituir sin aprobación explícita)

| Capa | Tecnología |
|---|---|
| Monorepo | pnpm workspaces + Turborepo |
| Mobile | Expo (SDK más reciente estable), expo-router, NativeWind v4 |
| Estado mobile | Zustand (global) + TanStack Query (datos de servidor) |
| Backend | NestJS 11, Node 22, TypeScript strict |
| Realtime | Socket.io con adaptador Redis |
| Base de datos | PostgreSQL 15 + PostGIS 3 (Supabase) |
| ORM | Prisma; SQL crudo tipado (`$queryRaw`) solo para geoespacial |
| Validación | zod en ambos extremos, schemas compartidos en `packages/shared` |
| Mapas | Google Maps Platform (Places New, Geocoding, Routes, Maps SDK Android) |
| Archivos | Supabase Storage (buckets privados + URLs firmadas) |
| Push | Expo Push Notifications (FCM por debajo) |
| Tests | vitest (lógica pura), supertest (e2e de API) |

---

## 3. Comandos

```bash
pnpm install                                # instalar todo el workspace
pnpm turbo dev                              # api + mobile en paralelo
pnpm --filter api dev                       # solo backend
pnpm --filter mobile start                  # solo Expo
pnpm turbo typecheck                        # obligatorio antes de entregar
pnpm turbo lint
pnpm --filter @delivery/fare-engine test
pnpm --filter api prisma migrate dev --name <nombre>
pnpm --filter api db:seed
```

Nunca uses `npm` ni `yarn` en este repo. Nunca ejecutes `prisma db push` contra una base de datos que no sea local.

---

## 4. Reglas no negociables

### 4.1 Dinero
- Todo monto es `numeric(10,2)` en Postgres y se maneja como número con 2 decimales redondeado explícitamente. **Nunca** acumules flotantes sin redondear.
- La moneda es `PEN`. Formato de visualización: `S/ 12.50`.
- La tarifa se calcula **solo** en `packages/fare-engine`. El backend la invoca; la app la muestra. Si necesitas la tarifa en la app, pídela a `POST /orders/quote` — no dupliques la fórmula en el cliente.
- La tarifa se **congela** al crear el pedido (`fare_config_id` + `fare_breakdown` en `orders`). Jamás recalcules la tarifa de un pedido existente.
- **El cobro es contra entrega y directo al driver** (efectivo o Yape al número del driver). La plataforma no procesa pagos: no hay pasarela, ni POS, ni payouts, ni carga de comprobante por el cliente. No agregues ninguna de esas piezas sin cambiar antes el ADR-09 de `PLAN_DESARROLLO.md`.
- La comisión es deuda del driver: se asienta como `COMMISSION_DEBIT` en `driver_ledger` al entregar. **Nunca** la descuentes de `payments.amount`, que es el monto íntegro que el cliente paga al driver.
- `payments.status` solo pasa a `CONFIRMED` dentro de la misma transacción que marca el pedido como `DELIVERED`. Un pedido entregado con pago en `PENDING` es un bug.

### 4.2 Geoespacial
- Orden de coordenadas en PostGIS: **longitud primero** → `ST_MakePoint(lng, lat)`. En TypeScript usamos `{ lat, lng }`. Esta inversión es la fuente de bugs #1; sé explícito en los nombres.
- SRID siempre `4326`. Tipo de columna siempre `geography`, no `geometry`.
- Para buscar por radio usa `ST_DWithin(geom, punto::geography, metros)`. **Nunca** `ST_Distance(...) < metros` — eso no usa el índice GIST.
- Toda consulta geoespacial nueva va en `modules/tracking/geo.repository.ts` o `modules/dispatch/`, nunca inline en un controlador.

### 4.3 Estados del pedido
- Las transiciones pasan **siempre** por `OrdersService.transition()`. Prohibido hacer `prisma.order.update({ data: { status } })` desde cualquier otro lugar.
- Toda transición escribe en `order_status_history` dentro de la **misma transacción**.
- `DELIVERED` exige `otp_validated_at IS NOT NULL` **y** confirmación de cobro del driver (`paymentReceived: true` en `verify-otp`). No hay excepciones ni banderas de desarrollo que lo salten.
- Impago: el driver reporta `POST /orders/:id/fail` con `reason: 'NO_PAYMENT'` → pedido `FAILED`, `payments.status = NOT_RECEIVED`, sin asientos en `driver_ledger`.

### 4.4 OTP de entrega
- `orders.delivery_otp` **nunca** se serializa en una respuesta dirigida a un driver ni en un evento WebSocket del namespace `/driver`. La serialización por rol vive en `modules/orders/orders.serializer.ts`; úsala.
- Compara el OTP en tiempo constante. Máximo 5 intentos por pedido.

### 4.5 Seguridad
- Secretos solo por variables de entorno. Si añades una variable, actualiza `.env.example` y `config/env.schema.ts` en el mismo cambio.
- La app móvil **nunca** llama a Google Maps API directamente para autocompletado, geocoding o rutas: todo pasa por los endpoints `/geo/*`. La API key del SDK nativo de mapas es la única que vive en el cliente y está restringida por paquete + SHA-1.
- Toda ruta es autenticada por defecto. Exponer una ruta pública requiere el decorador `@Public()` explícito.
- No registres en logs: contraseñas, tokens, OTP, números de teléfono completos (incluido `yape_phone`), ni URLs firmadas de documentos de drivers.
- `driver_profiles.yape_phone` se expone al cliente solo cuando su pedido está en `PICKED_UP` o posterior y eligió `YAPE`. Nunca en listados, ofertas ni perfiles públicos.

### 4.6 Realtime
- El WebSocket es un canal de **notificación**, no la fuente de verdad. Tras reconectar, la app debe reconstruir su estado con `GET /orders/active` y `GET /orders/:id/track`.
- Los nombres de eventos se definen **solo** en `packages/shared/src/socket-events.ts`. Prohibido escribir el string de un evento a mano en la app o en el gateway.
- Emisiones de GPS: máximo 1 cada 3 s y solo si el desplazamiento supera 15 m.

---

## 5. Convenciones de código

### 5.1 General
- TypeScript `strict: true`. Prohibido `any`; usa `unknown` y estrecha el tipo. Prohibido `@ts-ignore` sin un comentario que explique por qué y un issue asociado.
- Nada de barrel files (`index.ts` que re-exporta todo) dentro de `apps/*`; sí en `packages/*`, que son librerías.
- Comentarios: solo para explicar el *por qué* de una decisión no obvia. Prohibido el comentario que narra la línea siguiente.
- Código y comentarios en **inglés**. Textos visibles al usuario en **español peruano** (centralizados en `src/i18n/es.ts` en la app).

### 5.2 Backend
- Un módulo por dominio. `controller` (HTTP), `service` (reglas de negocio), `repository` (acceso a datos cuando hay SQL crudo), `dto` (zod).
- Los controladores no contienen lógica de negocio ni llaman a Prisma directamente.
- Errores de dominio: lanza excepciones HTTP de Nest con un `code` estable en el cuerpo (`ORDER_ALREADY_TAKEN`, `OUT_OF_SERVICE_AREA`, `INVALID_TRANSITION`, `OTP_LOCKED`). La app móvil ramifica por `code`, nunca por el texto del mensaje.
- Toda operación que toca más de una tabla va dentro de `prisma.$transaction`.

### 5.3 Mobile
- Pantallas en `app/` solo componen; la lógica vive en hooks de `src/features/<dominio>/`.
- Datos de servidor → TanStack Query con claves de query estructuradas (`['order', id]`). Estado de UI y sesión → Zustand. No dupliques datos de servidor en Zustand.
- Estilos con clases de NativeWind. Prohibido `StyleSheet.create` salvo para animaciones o sombras que NativeWind no cubra.
- Los colores, radios y tipografías vienen de `src/theme/tokens.ts`. Prohibido un hex suelto en un componente.
- Todo estado asíncrono necesita sus tres vistas: carga (skeleton, no spinner centrado), error (con reintento) y vacío (con acción sugerida).

### 5.4 Diseño visual
Aplica la skill `frontend-design` al crear o modificar vistas. Restricciones específicas de este producto:
- **Prohibido:** degradados morados o azul-violeta, sombras genéricas difusas, layouts de tarjetas centradas sin jerarquía, emojis como iconografía de producto.
- La app la usan motociclistas en movimiento, bajo el sol de Chincha, con una sola mano: objetivos táctiles ≥ 48 px en las pantallas del driver, acción primaria siempre en el tercio inferior, contraste mínimo AA, tipografía de números grande y tabular.
- Un solo acento de color por pantalla. El mapa es el protagonista; la UI flota sobre él en hojas (sheets), no lo tapa.

---

## 6. Base de datos

- Cada cambio de esquema necesita una migración Prisma versionada. Nunca edites una migración ya aplicada; crea una nueva.
- Las columnas `geography` se declaran como `Unsupported("geography(Point,4326)")` en `schema.prisma` y se crean/alteran por SQL manual dentro de la migración.
- Índices obligatorios al añadir una consulta nueva: si filtras por una columna en un camino caliente, el índice va en la misma migración.
- Nombres: tablas en `snake_case` plural, columnas en `snake_case`, modelos Prisma en `PascalCase` singular con `@@map`.

---

## 7. Qué hacer cuando algo no está claro

1. Busca la respuesta en `PLAN_DESARROLLO.md` (secciones 1 a 6).
2. Busca un patrón ya implementado en el repo e imítalo.
3. Si sigue ambiguo y la decisión cambia el contrato de API, el esquema de DB o la fórmula de tarifas: **pregunta antes de implementar**.
4. Si es una decisión interna y reversible: elige la opción más simple, impleméntala y déjala documentada en el reporte.

---

## 8. Checklist antes de reportar una tarea como terminada

- [ ] `pnpm turbo typecheck` sin errores nuevos.
- [ ] `pnpm turbo lint` sin warnings nuevos.
- [ ] Tests de lógica pura añadidos o actualizados si tocaste `fare-engine` o `transition()`.
- [ ] `.env.example` y `env.schema.ts` actualizados si añadiste una variable.
- [ ] Migración Prisma creada si cambiaste el esquema.
- [ ] Endpoints nuevos documentados en `docs/api.http` con un ejemplo real.
- [ ] Reporte con la **salida real** de los comandos ejecutados, no un resumen de lo que debería haber pasado.
- [ ] Si algo quedó incompleto o roto, dilo explícitamente. Un reporte que oculta un fallo es peor que la tarea sin hacer.

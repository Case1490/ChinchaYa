# Plan de Arquitectura y Desarrollo — App de Encargos Punto A → Punto B (Chincha, Perú)

> **Documento maestro de ingeniería.** Versión 1.0 — 11 de noviembre de 2026
> Autor: Lead Software Architect
> Estado: Aprobado para iniciar Fase 1

---

## 0. Resumen ejecutivo y supuestos

**Producto:** aplicación móvil única (Expo/React Native) con dos modos de operación conmutados por rol (Cliente / Driver). El Cliente define origen (A), destino (B) y describe el encargo en texto libre; el sistema calcula una tarifa estimada por distancia y difunde el pedido por WebSocket a los drivers disponibles dentro de un radio geográfico. El Driver acepta, ejecuta y liquida la entrega validando un código OTP de 4 dígitos.

**Alcance geográfico del MVP:** provincia de Chincha, Ica (Chincha Alta, Chincha Baja, Pueblo Nuevo, Sunampe, Grocio Prado, Tambo de Mora). Se define un *geofence* poligonal; pedidos fuera del polígono se rechazan en validación.

### 0.1 Supuestos declarados (importante)

| # | Supuesto | Implicancia |
|---|---|---|
| S1 | 22 h/semana efectivas de programación | Capacidad total 4 semanas = **88 h** |
| S2 | El MVP completo *publicado en Play Store* no cabe en 88 h | Ver §0.2: se entrega **MVP funcional end-to-end en 88 h** y se reservan **2 semanas adicionales (44 h)** para hardening + publicación |
| S3 | Un solo desarrollador (tú) + Claude Code como ejecutor | Las tareas están redactadas como prompts atómicos delegables |
| S4 | Sin pasarela de pago, sin facturación electrónica, sin panel web de admin en el MVP | La administración se hace con consultas SQL + Supabase Studio en Fase 1–6 |
| S6 | **Pago contra entrega, directo al driver** (efectivo o Yape al número del driver). Decisión de negocio del 12-nov-2026: POS y tarjetas cobran 4–5 % y quedan descartados en todas las fases | La plataforma no mueve dinero: no hay payouts, no hay comprobante subido por el cliente, no hay bucket `receipts`. El driver cobra el 100 % y queda debiendo la comisión (`driver_ledger`). El cierre del pedido exige confirmación de cobro del driver |
| S5 | Cuenta Google Play Console ($25) y cuenta Google Cloud con billing activo ya contratadas antes de la Semana 5 | Bloqueante de publicación |

### 0.2 Corrección honesta de cronograma

Un MVP de delivery en tiempo real con tracking GPS, dispatch por WebSocket, OTP, cobro contra entrega y publicación en Play Store es realísticamente **120–140 h** para un dev solo. El plan de abajo divide así:

- **Semanas 1–4 (85 h de alcance sobre 88 h de capacidad):** núcleo funcional completo y demostrable (happy path end-to-end sobre dispositivo físico, backend desplegado).
- **Semanas 5–6 (44 h):** estabilización, casos borde, permisos de background location, build AAB firmado, ficha de Play Store y revisión.

Si el plazo de 4 semanas es inamovible, el recorte recomendado está en §7.4 (MVP Mínimo Viable Real, 84 h).

---

## 1. Arquitectura de software

### 1.1 Vista de alto nivel

```
┌────────────────────────────────────────────────────────────────────┐
│                     APP EXPO (React Native)                        │
│  ┌──────────────────────┐        ┌──────────────────────┐          │
│  │  Modo CLIENTE        │        │  Modo DRIVER         │          │
│  │  - Selector A/B      │        │  - Toggle online     │          │
│  │  - Descripción       │        │  - Inbox de ofertas  │          │
│  │  - Tarifa estimada   │        │  - Navegación GPS    │          │
│  │  - Tracking en vivo  │        │  - Captura OTP       │          │
│  │  - OTP + pago        │        │  - Cobro + comisión  │          │
│  └──────────┬───────────┘        └──────────┬───────────┘          │
│             └──────────┬──────────────────-─┘                      │
│        Zustand (estado) + TanStack Query (cache HTTP)              │
│        socket.io-client (canal realtime)  expo-location (GPS)      │
└───────────────┬──────────────────────────────┬─────────────────────┘
                │ HTTPS/REST                   │ WSS (Socket.io)
┌───────────────▼──────────────────────────────▼─────────────────────┐
│                      API NestJS (Node 22, TS)                      │
│  Módulos: auth · users · drivers · orders · dispatch · tracking    │
│           fares · payments · ratings · uploads · notifications     │
│  Guards: JwtAuthGuard · RolesGuard · WsJwtGuard                    │
│  Gateways Socket.io: /client · /driver (namespaces)                │
│  Adapter Redis (socket.io-redis) → escalado horizontal futuro      │
└──────┬──────────────┬──────────────┬──────────────┬────────────────┘
       │              │              │              │
┌──────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐ ┌─────▼─────────────┐
│ PostgreSQL  │ │   Redis    │ │  Supabase  │ │ Google Maps       │
│  + PostGIS  │ │ geo-cache  │ │  Storage   │ │ Platform          │
│  (Supabase) │ │ + pub/sub  │ │(documentos)│ │ Places/Geocoding/ │
│   Prisma    │ │            │ │            │ │ Routes            │
└─────────────┘ └────────────┘ └────────────┘ └───────────────────┘
                                                      │
                                              ┌───────▼─────────┐
                                              │ Expo Push (FCM) │
                                              └─────────────────┘
```

### 1.2 Decisiones de arquitectura (ADR resumidos)

| ID | Decisión | Alternativa descartada | Razón |
|---|---|---|---|
| ADR-01 | **Supabase Postgres (con PostGIS) + NestJS en droplet DigitalOcean $6/mes** | Supabase Edge Functions puras | Socket.io necesita un proceso Node persistente; las Edge Functions son stateless. Supabase aporta DB gestionada, backups y Storage sin ops. |
| ADR-02 | **JWT propio (access 15 min + refresh 30 d) emitido por NestJS** | Supabase Auth | El handshake de Socket.io, los roles y el `driverId` se resuelven con un solo secreto; evita acoplar sesiones de dos sistemas. |
| ADR-03 | **Prisma ORM + SQL crudo para geo** | TypeORM | DX y migraciones superiores. PostGIS se consulta con `$queryRaw` tipado; la columna `geography` se declara vía migración manual y se expone como `Unsupported("geography(Point,4326)")`. |
| ADR-04 | **Una sola app con renderizado condicional por rol** | Dos apps separadas | 1 build, 1 ficha de Play Store, 1 pipeline. El árbol de navegación se elige en el root según `user.activeRole`. |
| ADR-05 | **Dispatch por *broadcast* secuencial con ventanas de 20 s** | Asignación directa al más cercano | Con pocos drivers (<30) el broadcast por olas maximiza la tasa de aceptación sin un motor de matching complejo. Ver §3.3. |
| ADR-06 | **Google Maps Platform** | Mapbox | Mejor cobertura de direcciones y POIs en Perú (Places Autocomplete), y `react-native-maps` usa el proveedor nativo de Google en Android sin configuración extra. |
| ADR-07 | **Tracking GPS vía WebSocket, no persistido en caliente** | INSERT por cada ping | Posición viva en Redis (`TTL 60 s`) + muestreo en DB cada 15 s para auditoría. Reduce escrituras ~10×. |
| ADR-08 | **NativeWind v4** | StyleSheet / Tamagui | Consistencia de design tokens y velocidad de maquetación. |
| ADR-09 | **Cobro contra entrega directo al driver (efectivo o Yape al número del driver); la plataforma solo asienta la comisión en `driver_ledger`** | Pasarela/POS (Niubiz, Culqi, Izipay) o Yape a la cuenta de la plataforma con liquidación semanal | Decisión de negocio: la pasarela cobra 4–5 % por operación sobre un ticket de ~S/ 8, insostenible para el margen. Cobrar a la cuenta de la plataforma obligaría a un módulo de payouts, conciliación diaria y custodia de fondos de terceros (+8–12 h y riesgo operativo). Con pago al driver, el flujo de dinero queda fuera del sistema y el software solo necesita registrar la deuda de comisión. Consecuencia: el riesgo de impago lo absorbe el driver en el momento de la entrega, por eso `DELIVERED` exige su confirmación explícita de cobro. |

### 1.3 Estructura de carpetas del monorepo

Gestor: **pnpm workspaces + Turborepo**.

```
app-delivery/
├─ pnpm-workspace.yaml
├─ turbo.json
├─ package.json                       # scripts raíz: dev, build, lint, typecheck, db:*
├─ .nvmrc                             # 22
├─ .env.example
├─ .gitignore
├─ README.md
├─ PLAN_DESARROLLO.md                 # este documento
├─ AGENTS.md                          # reglas para cualquier agente IA
├─ CLAUDE.md                          # reglas específicas de Claude Code
│
├─ apps/
│  ├─ api/                            # NestJS
│  │  ├─ src/
│  │  │  ├─ main.ts
│  │  │  ├─ app.module.ts
│  │  │  ├─ common/
│  │  │  │  ├─ decorators/            # @CurrentUser, @Roles, @Public
│  │  │  │  ├─ guards/                # jwt-auth, roles, ws-jwt, throttler
│  │  │  │  ├─ filters/               # all-exceptions, prisma-exception
│  │  │  │  ├─ interceptors/          # logging, transform-response
│  │  │  │  ├─ pipes/                 # zod-validation.pipe
│  │  │  │  └─ utils/                 # geo.util, otp.util, money.util
│  │  │  ├─ config/                   # configuration.ts + env.schema.ts (zod)
│  │  │  ├─ infra/
│  │  │  │  ├─ prisma/                # prisma.service.ts, prisma.module.ts
│  │  │  │  ├─ redis/                 # redis.service.ts (ioredis)
│  │  │  │  ├─ storage/               # supabase-storage.service.ts
│  │  │  │  ├─ maps/                  # google-maps.service.ts (routes, geocode)
│  │  │  │  └─ push/                  # expo-push.service.ts
│  │  │  └─ modules/
│  │  │     ├─ auth/                  # controller, service, strategies, dto
│  │  │     ├─ users/
│  │  │     ├─ drivers/               # perfil, documentos, online/offline, wallet
│  │  │     ├─ orders/                # CRUD + máquina de estados
│  │  │     ├─ dispatch/              # motor de ofertas + dispatch.gateway.ts
│  │  │     ├─ tracking/              # tracking.gateway.ts + geo repository
│  │  │     ├─ fares/                 # fare.service.ts + fare-config
│  │  │     ├─ payments/              # cobro contra entrega: efectivo / yape
│  │  │     ├─ ratings/
│  │  │     └─ health/
│  │  ├─ prisma/
│  │  │  ├─ schema.prisma
│  │  │  ├─ migrations/
│  │  │  └─ seed.ts
│  │  ├─ test/                        # e2e (supertest)
│  │  ├─ Dockerfile
│  │  └─ tsconfig.json
│  │
│  └─ mobile/                         # Expo (expo-router)
│     ├─ app/                         # file-based routing
│     │  ├─ _layout.tsx               # providers + gate de sesión
│     │  ├─ index.tsx                 # splash / redirect por rol
│     │  ├─ (auth)/
│     │  │  ├─ login.tsx
│     │  │  ├─ register.tsx
│     │  │  └─ verify-phone.tsx
│     │  ├─ (client)/
│     │  │  ├─ _layout.tsx            # tabs cliente
│     │  │  ├─ home.tsx               # mapa + selector A/B
│     │  │  ├─ new-order/
│     │  │  │  ├─ pick-points.tsx
│     │  │  │  ├─ describe.tsx
│     │  │  │  └─ confirm.tsx         # tarifa + método de pago
│     │  │  ├─ order/[id].tsx         # tracking en vivo + OTP
│     │  │  ├─ history.tsx
│     │  │  └─ profile.tsx
│     │  ├─ (driver)/
│     │  │  ├─ _layout.tsx
│     │  │  ├─ dashboard.tsx          # toggle online + ofertas
│     │  │  ├─ offer/[id].tsx         # aceptar/rechazar con countdown
│     │  │  ├─ run/[id].tsx           # navegación por etapas + OTP input
│     │  │  ├─ earnings.tsx
│     │  │  └─ profile.tsx
│     │  └─ (shared)/
│     │     └─ settings.tsx
│     ├─ src/
│     │  ├─ components/
│     │  │  ├─ ui/                    # Button, Input, Sheet, Badge, Skeleton
│     │  │  ├─ map/                   # MapCanvas, RoutePolyline, DriverMarker
│     │  │  ├─ order/                 # FareBreakdown, StatusTimeline, OtpDisplay
│     │  │  └─ driver/                # OfferCard, OnlineToggle, StageButton
│     │  ├─ features/                 # hooks de dominio por feature
│     │  │  ├─ auth/
│     │  │  ├─ orders/
│     │  │  ├─ dispatch/
│     │  │  └─ tracking/
│     │  ├─ lib/
│     │  │  ├─ api-client.ts          # axios + refresh interceptor
│     │  │  ├─ socket.ts              # singleton socket.io-client
│     │  │  ├─ query-client.ts
│     │  │  ├─ location.ts            # expo-location foreground/background
│     │  │  ├─ storage.ts             # expo-secure-store
│     │  │  └─ notifications.ts
│     │  ├─ store/                    # zustand: session, activeOrder, driverState
│     │  ├─ theme/                    # tokens.ts (colores, radios, tipografía)
│     │  └─ types/
│     ├─ assets/
│     ├─ app.config.ts                # plugins, permisos, API keys por env
│     ├─ eas.json                     # perfiles development / preview / production
│     ├─ tailwind.config.js
│     ├─ babel.config.js
│     └─ metro.config.js
│
├─ packages/
│  ├─ shared/                         # @delivery/shared — tipos y contratos
│  │  └─ src/
│  │     ├─ enums.ts                  # OrderStatus, Role, PaymentMethod...
│  │     ├─ dto/                      # schemas zod compartidos API↔app
│  │     ├─ socket-events.ts          # contrato tipado de eventos WS
│  │     └─ index.ts
│  ├─ fare-engine/                    # @delivery/fare-engine — cálculo puro
│  │  └─ src/
│  │     ├─ fare.ts                   # calculateFare()
│  │     ├─ geo.ts                    # haversine, bbox, isInsideGeofence
│  │     ├─ config.ts                 # FareConfig por defecto (Chincha)
│  │     └─ fare.test.ts              # vitest
│  └─ config/                         # @delivery/config — tooling compartido
│     ├─ eslint/
│     ├─ prettier/
│     └─ tsconfig/                    # base.json, nest.json, expo.json
│
└─ docs/
   ├─ ALCANCE_TECNICO.md              # documento comercial (fuente del PDF)
   ├─ ALCANCE_TECNICO.pdf
   ├─ api.http                        # colección REST manual
   └─ adr/
```

---

## 2. Modelo de datos (PostgreSQL 15 + PostGIS 3)

### 2.1 Enums

```sql
CREATE TYPE user_role        AS ENUM ('CLIENT','DRIVER','ADMIN');
CREATE TYPE user_status      AS ENUM ('PENDING','ACTIVE','SUSPENDED','BANNED');
CREATE TYPE driver_status    AS ENUM ('OFFLINE','ONLINE','BUSY','SUSPENDED');
CREATE TYPE vehicle_type     AS ENUM ('MOTORCYCLE','BICYCLE','CAR','WALK');
CREATE TYPE order_status     AS ENUM (
  'DRAFT','SEARCHING','ACCEPTED','ARRIVED_PICKUP','PICKED_UP',
  'ARRIVED_DROPOFF','DELIVERED','CANCELLED_BY_CLIENT',
  'CANCELLED_BY_DRIVER','EXPIRED','FAILED'
);
CREATE TYPE payment_method   AS ENUM ('CASH','YAPE');
-- Cobro contra entrega: PENDING hasta que el driver confirme el cobro al entregar.
-- NOT_RECEIVED = el driver declaró que el cliente no pagó (incidencia, el pedido va a FAILED).
CREATE TYPE payment_status   AS ENUM ('PENDING','CONFIRMED','NOT_RECEIVED');
CREATE TYPE offer_status     AS ENUM ('SENT','ACCEPTED','REJECTED','TIMEOUT','SUPERSEDED');
CREATE TYPE ledger_kind      AS ENUM ('COMMISSION_DEBIT','TOPUP_CREDIT','ADJUSTMENT','TRIP_EARNING');
```

### 2.2 Tablas

#### `users`
| Columna | Tipo | Notas |
|---|---|---|
| `id` | `uuid` PK `default gen_random_uuid()` | |
| `phone` | `varchar(15)` UNIQUE NOT NULL | formato E.164 `+519XXXXXXXX`. Identificador primario de login |
| `email` | `varchar(255)` UNIQUE NULL | opcional |
| `password_hash` | `text` NOT NULL | argon2id |
| `full_name` | `varchar(120)` NOT NULL | |
| `dni` | `varchar(8)` NULL | requerido solo para DRIVER |
| `avatar_url` | `text` NULL | Supabase Storage |
| `roles` | `user_role[]` NOT NULL DEFAULT `'{CLIENT}'` | un usuario puede ser CLIENT y DRIVER |
| `active_role` | `user_role` NOT NULL DEFAULT `'CLIENT'` | determina el árbol de navegación |
| `status` | `user_status` NOT NULL DEFAULT `'PENDING'` | |
| `phone_verified_at` | `timestamptz` NULL | |
| `rating_avg` | `numeric(3,2)` NOT NULL DEFAULT 5.00 | desnormalizado |
| `rating_count` | `int` NOT NULL DEFAULT 0 | |
| `created_at` / `updated_at` | `timestamptz` | |

Índices: `idx_users_phone`, `idx_users_status`.

#### `refresh_tokens`
`id uuid PK` · `user_id uuid FK→users ON DELETE CASCADE` · `token_hash text NOT NULL` · `device_id varchar(64)` · `expires_at timestamptz NOT NULL` · `revoked_at timestamptz NULL` · `created_at`.
Índice: `idx_refresh_user_active (user_id) WHERE revoked_at IS NULL`.

#### `driver_profiles` (1:1 con users)
| Columna | Tipo | Notas |
|---|---|---|
| `id` | `uuid` PK | |
| `user_id` | `uuid` UNIQUE FK→users | |
| `status` | `driver_status` DEFAULT `'OFFLINE'` | |
| `vehicle_type` | `vehicle_type` NOT NULL | |
| `plate` | `varchar(10)` NULL | |
| `vehicle_desc` | `varchar(80)` NULL | «Honda Wave roja» |
| `license_url` / `soat_url` / `id_card_url` | `text` NULL | documentos en Storage |
| `verified_at` | `timestamptz` NULL | habilitación manual por admin |
| `max_radius_m` | `int` DEFAULT 4000 | radio que acepta el driver |
| `yape_phone` | `varchar(15)` NULL | número de Yape donde el driver recibe el pago del cliente. Obligatorio para recibir pedidos con `payment_method = 'YAPE'`; si es NULL el dispatch solo le ofrece pedidos en efectivo |
| `wallet_balance` | `numeric(10,2)` DEFAULT 0 | negativo = debe comisión a la plataforma. El driver cobra el 100 % al cliente, por eso este saldo es casi siempre negativo |
| `completed_orders` | `int` DEFAULT 0 | |
| `last_seen_at` | `timestamptz` NULL | |
| `created_at` / `updated_at` | | |

#### `driver_locations` (posición viva, 1 fila por driver — UPSERT)
| Columna | Tipo | Notas |
|---|---|---|
| `driver_id` | `uuid` PK FK→driver_profiles | |
| `geom` | `geography(Point,4326)` NOT NULL | **fuente de verdad geoespacial** |
| `heading` | `numeric(5,2)` NULL | grados 0–360 |
| `speed_kph` | `numeric(5,2)` NULL | |
| `accuracy_m` | `numeric(6,2)` NULL | |
| `is_available` | `boolean` NOT NULL DEFAULT false | `status = ONLINE` y sin pedido |
| `updated_at` | `timestamptz` NOT NULL DEFAULT now() | |

```sql
CREATE INDEX idx_driver_loc_geom ON driver_locations USING GIST (geom);
CREATE INDEX idx_driver_loc_avail ON driver_locations (is_available, updated_at DESC);
```

#### `saved_places` (direcciones frecuentes del cliente)
`id uuid PK` · `user_id uuid FK` · `label varchar(40)` · `address_text text` · `reference varchar(160)` *(«portón azul, 2do piso»)* · `geom geography(Point,4326)` · `created_at`.

#### `orders` (agregado central)
| Columna | Tipo | Notas |
|---|---|---|
| `id` | `uuid` PK | |
| `code` | `varchar(10)` UNIQUE | legible: `CHI-4F2A` |
| `client_id` | `uuid` FK→users | |
| `driver_id` | `uuid` NULL FK→driver_profiles | NULL mientras `SEARCHING` |
| `status` | `order_status` NOT NULL DEFAULT `'SEARCHING'` | |
| `errand_description` | `text` NOT NULL | lo que pide el cliente; máx 500 chars |
| `errand_requires_purchase` | `boolean` DEFAULT false | «compra X cosa» |
| `purchase_budget` | `numeric(10,2)` NULL | monto que adelanta el driver |
| `package_size` | `varchar(10)` NULL | `S` / `M` / `L` |
| **Origen** | | |
| `pickup_address` | `text` NOT NULL | |
| `pickup_reference` | `varchar(160)` NULL | |
| `pickup_geom` | `geography(Point,4326)` NOT NULL | |
| `pickup_contact_name` | `varchar(80)` NULL | |
| `pickup_contact_phone` | `varchar(15)` NULL | |
| **Destino** | | |
| `dropoff_address` | `text` NOT NULL | |
| `dropoff_reference` | `varchar(160)` NULL | |
| `dropoff_geom` | `geography(Point,4326)` NOT NULL | |
| `dropoff_contact_name` | `varchar(80)` NULL | |
| `dropoff_contact_phone` | `varchar(15)` NULL | |
| **Tarifa** | | |
| `distance_m` | `int` NOT NULL | distancia de ruta (Routes API) |
| `duration_s` | `int` NOT NULL | ETA estimado |
| `route_polyline` | `text` NULL | polyline codificada para dibujar |
| `fare_config_id` | `uuid` FK→fare_configs | tarifa congelada al crear |
| `fare_base` | `numeric(10,2)` NOT NULL | |
| `fare_distance` | `numeric(10,2)` NOT NULL | |
| `fare_surcharges` | `numeric(10,2)` NOT NULL DEFAULT 0 | |
| `fare_total` | `numeric(10,2)` NOT NULL | lo que paga el cliente |
| `fare_breakdown` | `jsonb` NOT NULL | desglose auditable (ver §3.2) |
| `commission_rate` | `numeric(4,3)` NOT NULL | ej. 0.150 |
| `commission_amount` | `numeric(10,2)` NOT NULL | |
| `driver_payout` | `numeric(10,2)` NOT NULL | `fare_total - commission_amount` |
| **Entrega / OTP** | | |
| `delivery_otp` | `char(4)` NOT NULL | visible solo al cliente dueño |
| `otp_attempts` | `smallint` NOT NULL DEFAULT 0 | bloqueo a los 5 |
| `otp_validated_at` | `timestamptz` NULL | |
| **Ciclo de vida** | | |
| `searching_expires_at` | `timestamptz` NOT NULL | now() + 5 min |
| `accepted_at` / `picked_up_at` / `delivered_at` / `cancelled_at` | `timestamptz` NULL | |
| `cancel_reason` | `varchar(160)` NULL | |
| `created_at` / `updated_at` | | |

```sql
CREATE INDEX idx_orders_client      ON orders (client_id, created_at DESC);
CREATE INDEX idx_orders_driver      ON orders (driver_id, created_at DESC);
CREATE INDEX idx_orders_searching   ON orders (status, searching_expires_at) WHERE status = 'SEARCHING';
CREATE INDEX idx_orders_pickup_geom ON orders USING GIST (pickup_geom);
-- Invariante: un driver no puede tener dos pedidos activos
CREATE UNIQUE INDEX uq_driver_active_order ON orders (driver_id)
  WHERE status IN ('ACCEPTED','ARRIVED_PICKUP','PICKED_UP','ARRIVED_DROPOFF');
```

#### `order_status_history` (auditoría inmutable)
`id bigserial PK` · `order_id uuid FK ON DELETE CASCADE` · `from_status order_status NULL` · `to_status order_status NOT NULL` · `actor_id uuid NULL` · `actor_role user_role NULL` · `geom geography(Point,4326) NULL` *(dónde ocurrió)* · `metadata jsonb` · `created_at`.

#### `order_tracks` (muestreo de ruta, 1 fila cada ~15 s)
`id bigserial PK` · `order_id uuid FK` · `driver_id uuid` · `geom geography(Point,4326)` · `heading numeric(5,2)` · `speed_kph numeric(5,2)` · `recorded_at timestamptz`.
Índice: `idx_tracks_order (order_id, recorded_at)`. Retención 30 días (job de limpieza).

#### `order_offers` (bitácora del dispatch)
`id uuid PK` · `order_id uuid FK` · `driver_id uuid FK` · `wave smallint NOT NULL` · `distance_m int` · `eta_s int` · `status offer_status DEFAULT 'SENT'` · `sent_at timestamptz` · `responded_at timestamptz NULL` · `expires_at timestamptz`.
Índice único: `uq_offer (order_id, driver_id)`.

#### `payments`
| Columna | Tipo | Notas |
|---|---|---|
| `id` | `uuid` PK | |
| `order_id` | `uuid` UNIQUE FK | 1:1 con el pedido en el MVP |
| `method` | `payment_method` NOT NULL | elegido por el cliente al crear el pedido |
| `amount` | `numeric(10,2)` NOT NULL | `fare_total + purchase_budget`. Es lo que el cliente paga **al driver** contra entrega |
| `status` | `payment_status` NOT NULL DEFAULT `'PENDING'` | |
| `yape_target_phone` | `varchar(15)` NULL | copia de `driver_profiles.yape_phone` al momento de aceptar la oferta; se congela para que un cambio posterior del driver no altere pedidos pasados |
| `confirmed_by` | `uuid` NULL | driver que confirmó el cobro, o admin en una corrección |
| `confirmed_at` | `timestamptz` NULL | |
| `not_received_reason` | `varchar(160)` NULL | motivo declarado por el driver cuando el cliente no paga |
| `created_at` / `updated_at` | | |

> No hay columnas de comprobante (`yape_receipt_url`, `yape_operation_code`): el driver verifica el abono en su propia app de Yape. El sistema nunca es parte del movimiento de dinero. Tampoco hay `payer_moment`: todo cobro es contra entrega, en el punto B.

#### `driver_ledger` (comisiones y saldos)
`id bigserial PK` · `driver_id uuid FK` · `order_id uuid NULL FK` · `kind ledger_kind NOT NULL` · `amount numeric(10,2) NOT NULL` *(signo incluido)* · `balance_after numeric(10,2) NOT NULL` · `note varchar(160)` · `created_at`.

#### `ratings`
`id uuid PK` · `order_id uuid FK` · `rater_id uuid FK→users` · `ratee_id uuid FK→users` · `score smallint CHECK (score BETWEEN 1 AND 5)` · `tags text[]` *(`puntual`, `amable`, `cuidó el pedido`)* · `comment varchar(300) NULL` · `created_at`.
Único: `uq_rating (order_id, rater_id)`.

#### `fare_configs` (tarifario versionado)
| Columna | Tipo | Valor inicial Chincha |
|---|---|---|
| `id` | `uuid` PK | |
| `name` | `varchar(40)` | `chincha-v1` |
| `currency` | `char(3)` | `PEN` |
| `base_fare` | `numeric(10,2)` | `5.00` |
| `included_km` | `numeric(5,2)` | `2.00` |
| `per_km` | `numeric(10,2)` | `1.20` |
| `min_fare` | `numeric(10,2)` | `5.00` |
| `max_fare` | `numeric(10,2)` | `60.00` |
| `night_surcharge` | `numeric(10,2)` | `2.00` (22:00–05:59) |
| `purchase_fee_rate` | `numeric(4,3)` | `0.050` sobre `purchase_budget` |
| `purchase_fee_min` | `numeric(10,2)` | `1.50` |
| `size_m_surcharge` | `numeric(10,2)` | `1.50` |
| `size_l_surcharge` | `numeric(10,2)` | `3.00` |
| `peak_multiplier` | `numeric(4,2)` | `1.00` (palanca manual) |
| `commission_rate` | `numeric(4,3)` | `0.150` |
| `rounding_step` | `numeric(10,2)` | `0.50` |
| `search_radius_m` | `int` | `4000` |
| `is_active` | `boolean` | `true` |
| `valid_from` / `valid_to` | `timestamptz` | |

#### `service_areas` (geofence)
`id uuid PK` · `name varchar(60)` · `polygon geography(Polygon,4326) NOT NULL` · `fare_config_id uuid FK` · `is_active boolean`.
Índice GIST sobre `polygon`.

#### `push_tokens`
`id uuid PK` · `user_id uuid FK` · `expo_token varchar(200) UNIQUE` · `platform varchar(10)` · `last_used_at timestamptz`.

#### `otp_codes` (verificación de teléfono en registro — distinto del OTP de entrega)
`id uuid PK` · `phone varchar(15)` · `code_hash text` · `purpose varchar(20)` · `attempts smallint DEFAULT 0` · `expires_at timestamptz` · `consumed_at timestamptz NULL`.

### 2.3 Diagrama de relaciones

```
users 1─1 driver_profiles 1─1 driver_locations
  │              │
  │              ├─* order_offers *─1 orders
  │              ├─* driver_ledger
  │              └─* orders (driver_id)
  ├─* orders (client_id)
  ├─* saved_places
  ├─* refresh_tokens
  ├─* push_tokens
  └─* ratings (rater_id / ratee_id)

orders 1─1 payments
orders 1─* order_status_history
orders 1─* order_tracks
orders 1─* ratings
orders *─1 fare_configs
service_areas *─1 fare_configs
```

### 2.4 Máquina de estados del pedido

```
DRAFT ──create──▶ SEARCHING
SEARCHING ──driver acepta──▶ ACCEPTED
SEARCHING ──timeout 5 min / sin drivers──▶ EXPIRED
SEARCHING ──cliente cancela──▶ CANCELLED_BY_CLIENT      (sin penalidad)
ACCEPTED ──driver llega a A──▶ ARRIVED_PICKUP
ACCEPTED ──driver cancela──▶ SEARCHING                   (re-dispatch, wave+1)
ACCEPTED ──cliente cancela──▶ CANCELLED_BY_CLIENT        (penalidad si >2 min)
ARRIVED_PICKUP ──driver recoge──▶ PICKED_UP
PICKED_UP ──driver llega a B──▶ ARRIVED_DROPOFF
ARRIVED_DROPOFF ──OTP válido + cobro confirmado──▶ DELIVERED
ARRIVED_DROPOFF ──5 OTP fallidos / incidencia / NO_PAYMENT──▶ FAILED
```

Reglas duras aplicadas en `OrdersService.transition()`:
1. Toda transición se valida contra una tabla de adyacencia; una transición ilegal lanza `409 Conflict`.
2. Toda transición escribe en `order_status_history` dentro de la **misma transacción** Prisma.
3. `DELIVERED` exige `otp_validated_at IS NOT NULL`.
4. `DELIVERED` exige además cobro confirmado. El driver envía `POST /orders/:id/verify-otp` con `{ code, paymentReceived: true }`: en la misma transacción se valida el OTP y se marca `payments.status = 'CONFIRMED'` con `confirmed_by = driverId`. **Un pedido entregado nunca queda con `payments.status = 'PENDING'`.**
5. Si el driver declara que no le pagaron, no usa `verify-otp`: usa `POST /orders/:id/fail` con `{ reason: 'NO_PAYMENT', detail }` → `payments.status = 'NOT_RECEIVED'`, orden a `FAILED`, **sin** asientos en `driver_ledger` (no se cobra comisión de un pedido impago) y con registro en `order_status_history` incluyendo `geom`.
6. `DELIVERED` dispara: inserta `driver_ledger` (`TRIP_EARNING` + `COMMISSION_DEBIT`); incrementa `completed_orders`; libera al driver (`BUSY` → `ONLINE`, `is_available = true`).
7. Un cliente solo puede tener **1 pedido activo** a la vez en el MVP.

---

## 3. Algoritmo de tarifas y lógica geoespacial

### 3.1 Fórmula

```
distancia_km = distance_m / 1000

km_facturables = max(0, distancia_km − included_km)

subtotal_distancia = km_facturables × per_km

recargos =
      (hora_local ∈ [22:00, 06:00)  ? night_surcharge   : 0)
    + (package_size = 'M'           ? size_m_surcharge  : 0)
    + (package_size = 'L'           ? size_l_surcharge  : 0)
    + (requires_purchase            ? max(purchase_fee_min,
                                          purchase_budget × purchase_fee_rate) : 0)

bruto = (base_fare + subtotal_distancia + recargos) × peak_multiplier

tarifa = clamp(redondear(bruto, rounding_step), min_fare, max_fare)

comision     = redondear(tarifa × commission_rate, 0.10)   // deuda del driver con la plataforma
pago_driver  = tarifa − comision                           // ganancia neta que se le muestra en la oferta
cobro_total  = tarifa + (requires_purchase ? purchase_budget : 0)
```

`cobro_total` es el monto exacto que el **driver cobra al cliente contra entrega** (efectivo o Yape), y es el valor de `payments.amount`. La `comision` no se descuenta de ahí: se asienta como `COMMISSION_DEBIT` en `driver_ledger` y el driver la liquida al negocio por fuera de la app.

`redondear(x, step)` = `Math.ceil(x / step) * step` → siempre a favor del driver, en múltiplos de S/ 0.50 (práctico con monedas peruanas).

### 3.2 Implementación de referencia (`packages/fare-engine/src/fare.ts`)

```ts
import type { FareConfig, FareInput, FareResult } from './types';

const roundUpTo = (value: number, step: number) =>
  Math.ceil((value + Number.EPSILON) / step) * step;

const money = (value: number) => Math.round(value * 100) / 100;

export function calculateFare(input: FareInput, cfg: FareConfig): FareResult {
  const distanceKm = input.distanceMeters / 1000;
  const billableKm = Math.max(0, distanceKm - cfg.includedKm);
  const distanceFare = money(billableKm * cfg.perKm);

  const lines: Array<{ code: string; label: string; amount: number }> = [
    { code: 'BASE', label: `Tarifa base (${cfg.includedKm} km incluidos)`, amount: cfg.baseFare },
    { code: 'DISTANCE', label: `${billableKm.toFixed(2)} km adicionales × S/ ${cfg.perKm}`, amount: distanceFare },
  ];

  // Recargo nocturno — hora local de Lima (UTC-5), sin DST.
  const hour = input.localHour;
  if (hour >= 22 || hour < 6) {
    lines.push({ code: 'NIGHT', label: 'Recargo nocturno', amount: cfg.nightSurcharge });
  }

  if (input.packageSize === 'M') {
    lines.push({ code: 'SIZE_M', label: 'Paquete mediano', amount: cfg.sizeMSurcharge });
  } else if (input.packageSize === 'L') {
    lines.push({ code: 'SIZE_L', label: 'Paquete grande', amount: cfg.sizeLSurcharge });
  }

  if (input.requiresPurchase) {
    const budget = input.purchaseBudget ?? 0;
    const fee = money(Math.max(cfg.purchaseFeeMin, budget * cfg.purchaseFeeRate));
    lines.push({ code: 'PURCHASE_FEE', label: 'Gestión de compra', amount: fee });
  }

  const surcharges = money(
    lines.filter((l) => l.code !== 'BASE' && l.code !== 'DISTANCE')
         .reduce((sum, l) => sum + l.amount, 0),
  );

  const gross = money((cfg.baseFare + distanceFare + surcharges) * cfg.peakMultiplier);
  const total = money(Math.min(cfg.maxFare, Math.max(cfg.minFare, roundUpTo(gross, cfg.roundingStep))));

  const commission = money(roundUpTo(total * cfg.commissionRate, 0.1));
  const driverPayout = money(total - commission);
  const chargeTotal = money(total + (input.requiresPurchase ? (input.purchaseBudget ?? 0) : 0));

  return {
    currency: cfg.currency,
    distanceMeters: input.distanceMeters,
    durationSeconds: input.durationSeconds,
    base: cfg.baseFare,
    distance: distanceFare,
    surcharges,
    total,
    commissionRate: cfg.commissionRate,
    commission,
    driverPayout,
    chargeTotal,
    breakdown: { configName: cfg.name, peakMultiplier: cfg.peakMultiplier, lines },
  };
}
```

**Casos de prueba obligatorios (vitest):**

| Caso | distance_m | Extras | Esperado |
|---|---|---|---|
| Viaje mínimo | 800 | — | S/ 5.00 (`min_fare`) |
| 2 km exactos | 2000 | — | S/ 5.00 |
| 3.5 km | 3500 | — | base 5 + 1.5×1.2 = 6.80 → **7.00** |
| 6 km nocturno | 6000 | 23:00 | 5 + 4.8 + 2 = 11.80 → **12.00** |
| Compra S/ 40 | 3000 | purchase | 5 + 1.2 + max(1.5, 2.0) = 8.20 → **8.50**; cobro total **48.50** |
| Tope | 60000 | — | **60.00** (`max_fare`) |

> **Decisión de producto:** la tarifa se **congela** en el momento de crear el pedido (`fare_config_id` + `fare_breakdown`). Un cambio posterior de tarifario no altera pedidos en curso.

### 3.3 Distancia: estrategia de dos niveles

```
1. Intentar Google Routes API (computeRoutes, travelMode=TWO_WHEELER)
   → distance_m real por calle + duration_s + polyline
2. Fallback (timeout 2.5 s, cuota agotada o error):
   haversine(A,B) × FACTOR_VIAL (1.35 para la trama urbana de Chincha)
   duration_s ≈ distance_m / 6.94  (25 km/h promedio en moto)
   flag `fare_breakdown.estimationMode = 'HAVERSINE'`
```

Haversine de referencia:

```ts
export function haversineMeters(a: LatLng, b: LatLng): number {
  const R = 6_371_000;
  const toRad = (d: number) => (d * Math.PI) / 180;
  const dLat = toRad(b.lat - a.lat);
  const dLng = toRad(b.lng - a.lng);
  const lat1 = toRad(a.lat);
  const lat2 = toRad(b.lat);
  const h = Math.sin(dLat / 2) ** 2 + Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(h));
}
```

**Control de costos de Maps:** cachear el par `(A,B)` redondeado a 4 decimales (~11 m) en Redis con TTL de 6 h; usar *session tokens* en Places Autocomplete; `fields` restringidos en Place Details.

### 3.4 Búsqueda por radio con PostGIS

Consulta central del dispatch — drivers disponibles a menos de 4 km del punto A, ordenados por distancia:

```sql
SELECT
  dp.id                              AS driver_id,
  dp.user_id,
  u.full_name,
  u.rating_avg,
  dp.vehicle_type,
  ST_Y(dl.geom::geometry)            AS lat,
  ST_X(dl.geom::geometry)            AS lng,
  ST_Distance(dl.geom, $1::geography)::int AS distance_m
FROM driver_locations dl
JOIN driver_profiles dp ON dp.id = dl.driver_id
JOIN users           u  ON u.id  = dp.user_id
WHERE dl.is_available = true
  AND dp.status       = 'ONLINE'
  AND dp.verified_at IS NOT NULL
  AND u.status        = 'ACTIVE'
  AND dl.updated_at  > now() - interval '90 seconds'   -- heartbeat vivo
  AND ST_DWithin(dl.geom, $1::geography, $2)           -- $2 = radio en metros
  AND dp.id <> ALL($3::uuid[])                         -- excluir ya ofertados
  AND ($5::boolean = false OR dp.yape_phone IS NOT NULL) -- $5 = el pedido se paga por Yape
ORDER BY distance_m ASC, u.rating_avg DESC
LIMIT $4;
```

Claves de rendimiento:
- `ST_DWithin` con `geography` **sí** usa el índice GIST (a diferencia de `ST_Distance(...) < x`).
- El filtro por `updated_at` descarta drivers con la app muerta.
- `$5` evita ofrecer un pedido con `payment_method = 'YAPE'` a un driver que no declaró su número de Yape: no podría cobrarlo.
- `$1` se construye con `ST_SetSRID(ST_MakePoint(lng, lat), 4326)::geography` — **longitud primero**.

Validación de geofence al crear pedido:

```sql
SELECT sa.id, sa.fare_config_id
FROM service_areas sa
WHERE sa.is_active
  AND ST_Covers(sa.polygon, $1::geography)
  AND ST_Covers(sa.polygon, $2::geography)   -- A y B dentro del área
LIMIT 1;
```

### 3.5 Motor de dispatch (olas progresivas)

```
crearPedido(order):
  validar geofence  → 422 si A o B fuera de Chincha
  calcular tarifa   → congelar en el pedido
  status = SEARCHING, searching_expires_at = now() + 5 min
  encolar waveDispatch(order.id, wave = 1)

waveDispatch(orderId, wave):
  si order.status != 'SEARCHING'  → terminar
  si now() > searching_expires_at → status = EXPIRED; emitir order:expired; terminar

  radio      = min(4000 + (wave - 1) * 2000, 10000)   # 4km → 6km → 8km → 10km
  excluidos  = drivers con oferta previa en este pedido
  candidatos = queryRadio(pickup_geom, radio, excluidos, limit = 5)

  si candidatos vacío:
      si wave >= 4 → status = EXPIRED; notificar al cliente
      sino         → esperar 15 s; waveDispatch(orderId, wave + 1)

  para cada candidato:
      insertar order_offers(status = SENT, expires_at = now() + 20 s)
      emitir WS 'order:offer' al room driver:<id>
      enviar push si el socket está desconectado

  esperar 20 s → marcar ofertas sin respuesta como TIMEOUT
  si nadie aceptó → waveDispatch(orderId, wave + 1)

aceptarOferta(driverId, orderId):   # carrera resuelta por la DB
  BEGIN
    UPDATE orders
       SET status = 'ACCEPTED', driver_id = $driverId, accepted_at = now()
     WHERE id = $orderId AND status = 'SEARCHING'      -- gana el primer UPDATE
    RETURNING *;
    si 0 filas → ROLLBACK; responder 409 'ORDER_ALREADY_TAKEN'
    UPDATE order_offers SET status = 'ACCEPTED', responded_at = now() WHERE ...
    UPDATE order_offers SET status = 'SUPERSEDED' WHERE order_id = $orderId AND status = 'SENT'
    UPDATE driver_profiles SET status = 'BUSY' WHERE id = $driverId
    UPDATE driver_locations SET is_available = false WHERE driver_id = $driverId
    UPDATE payments SET yape_target_phone = (SELECT yape_phone FROM driver_profiles
                                             WHERE id = $driverId)
     WHERE order_id = $orderId AND method = 'YAPE'      -- congela a quién se paga
    INSERT INTO order_status_history (...)
  COMMIT
  emitir 'order:accepted' al cliente y 'offer:revoked' a los demás drivers
```

El temporizador vive en un servicio `DispatchScheduler` basado en `@nestjs/schedule` + una cola en Redis (claves `dispatch:wave:<orderId>` con `ZADD` por timestamp). Un `@Cron('*/5 * * * * *')` drena los vencimientos. Esto evita BullMQ en el MVP y sigue siendo recuperable tras un reinicio.

---

## 4. Contrato de API REST

Base: `https://api.<dominio>/v1`. Auth: `Authorization: Bearer <access_token>`.
Formato de error uniforme: `{ "statusCode": 409, "code": "ORDER_ALREADY_TAKEN", "message": "...", "timestamp": "..." }`.

### 4.1 Auth
| Método | Ruta | Body / Query | Respuesta | Rol |
|---|---|---|---|---|
| POST | `/auth/register` | `{ phone, fullName, password, role }` | `{ userId, otpSent: true }` | público |
| POST | `/auth/verify-phone` | `{ phone, code }` | `{ accessToken, refreshToken, user }` | público |
| POST | `/auth/login` | `{ phone, password, deviceId }` | `{ accessToken, refreshToken, user }` | público |
| POST | `/auth/refresh` | `{ refreshToken }` | `{ accessToken, refreshToken }` | público |
| POST | `/auth/logout` | `{ refreshToken }` | `204` | auth |
| POST | `/auth/forgot-password` | `{ phone }` | `{ otpSent }` | público |
| POST | `/auth/reset-password` | `{ phone, code, newPassword }` | `204` | público |

### 4.2 Usuarios y perfil
| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| GET | `/users/me` | perfil + roles + rating | auth |
| PATCH | `/users/me` | `{ fullName?, email?, avatarUrl? }` | auth |
| POST | `/users/me/switch-role` | `{ role: 'CLIENT' \| 'DRIVER' }` → nuevo JWT | auth |
| POST | `/users/me/push-token` | `{ expoToken, platform }` | auth |
| GET | `/users/me/saved-places` | lista | CLIENT |
| POST | `/users/me/saved-places` | `{ label, addressText, reference, lat, lng }` | CLIENT |
| DELETE | `/users/me/saved-places/:id` | | CLIENT |

### 4.3 Drivers
| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| POST | `/drivers/apply` | `{ vehicleType, plate, vehicleDesc, dni }` → perfil `PENDING` | auth |
| GET | `/drivers/me` | perfil, status, wallet, métricas | DRIVER |
| PATCH | `/drivers/me` | `{ vehicleDesc?, plate?, maxRadiusM?, yapePhone? }` — `yapePhone` en formato `+519XXXXXXXX`; sin él el driver solo recibe pedidos en efectivo | DRIVER |
| POST | `/drivers/me/documents` | multipart (`license`, `soat`, `idCard`) | DRIVER |
| POST | `/drivers/me/status` | `{ status: 'ONLINE' \| 'OFFLINE', lat, lng }` | DRIVER |
| GET | `/drivers/me/earnings?from&to` | `{ total, commission, net, trips[] }` | DRIVER |
| GET | `/drivers/me/ledger?cursor` | movimientos de billetera | DRIVER |
| GET | `/drivers/nearby?lat&lng&radius` | **solo para el mapa del cliente** (posiciones difusas, sin identidad) | CLIENT |

### 4.4 Pedidos
| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| POST | `/orders/quote` | `{ pickup{lat,lng}, dropoff{lat,lng}, requiresPurchase?, purchaseBudget?, packageSize? }` → `FareResult` **sin crear nada** | CLIENT |
| POST | `/orders` | crea el pedido + dispara dispatch → `{ order, otp }` | CLIENT |
| GET | `/orders/active` | pedido activo del usuario (cliente o driver) | auth |
| GET | `/orders/:id` | detalle (vista recortada según rol; el OTP solo al cliente dueño) | auth |
| GET | `/orders?status&cursor&limit` | historial paginado por cursor | auth |
| POST | `/orders/:id/cancel` | `{ reason }` | CLIENT \| DRIVER |
| POST | `/orders/:id/accept` | driver toma el pedido | DRIVER |
| POST | `/orders/:id/reject` | `{ offerId }` | DRIVER |
| POST | `/orders/:id/arrived-pickup` | `{ lat, lng }` | DRIVER |
| POST | `/orders/:id/picked-up` | `{ lat, lng, note? }` | DRIVER |
| POST | `/orders/:id/arrived-dropoff` | `{ lat, lng }` | DRIVER |
| POST | `/orders/:id/verify-otp` | `{ code, paymentReceived: true }` → `DELIVERED` + `payments.status = CONFIRMED`, o `{ attemptsLeft }`. `paymentReceived !== true` devuelve `422 PAYMENT_NOT_CONFIRMED` | DRIVER |
| POST | `/orders/:id/resend-otp` | re-emite el OTP al cliente por WS/push | CLIENT |
| POST | `/orders/:id/fail` | `{ reason, detail? }` → `FAILED`. `reason ∈ { NO_ONE_AT_DROPOFF, ADDRESS_NOT_FOUND, CLIENT_UNREACHABLE, NO_PAYMENT }`; con `NO_PAYMENT` marca `payments.status = NOT_RECEIVED` y no asienta comisión | DRIVER |
| GET | `/orders/:id/track` | últimos N puntos de `order_tracks` (reconexión) | auth |

### 4.5 Pagos (cobro contra entrega)
| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| GET | `/payments/order/:orderId` | `{ method, amount, status, yapeTargetPhone }`. Para el cliente, `yapeTargetPhone` solo se expone cuando el pedido está en `PICKED_UP` o posterior y `method = 'YAPE'` | auth |
| POST | `/payments/:id/confirm` | confirmación de cobro fuera del flujo normal de entrega (corrección de un pedido cerrado como `NO_PAYMENT` que después sí se pagó) → `CONFIRMED` + asientos de `driver_ledger` | ADMIN |

> El camino normal **no** pasa por un endpoint de pagos: el cobro se confirma dentro de `POST /orders/:id/verify-otp` (§2.4, regla 4), en la misma transacción que la entrega. No existe `/payments/:id/yape-proof` ni carga de comprobante: el pago va al Yape del driver y él lo verifica en su propia aplicación.

### 4.6 Ratings, geo y utilidades
| Método | Ruta | Descripción | Rol |
|---|---|---|---|
| POST | `/ratings` | `{ orderId, score, tags?, comment? }` | auth |
| GET | `/geo/autocomplete?q&sessionToken` | proxy de Places (restringido a Chincha vía `locationRestriction`) | auth |
| GET | `/geo/place/:placeId?sessionToken` | detalle → `{ address, lat, lng }` | auth |
| GET | `/geo/reverse?lat&lng` | geocodificación inversa | auth |
| GET | `/geo/service-area` | polígono del área de servicio (GeoJSON) para pintar el mapa | público |
| GET | `/config/fare` | tarifario activo (para mostrar «desde S/ 5.00») | público |
| GET | `/health` | liveness + checks de DB y Redis | público |

**Rate limits** (`@nestjs/throttler`): `/auth/*` 10 req/min por IP; `/orders` POST 5/min por usuario; `/geo/autocomplete` 30/min por usuario.

---

## 5. Contrato de WebSockets (Socket.io)

### 5.1 Conexión

```
URL:        wss://api.<dominio>/realtime
Transports: ['websocket']            // sin polling en RN
Auth:       { auth: { token: '<accessToken>' } }   // validado en WsJwtGuard
Namespaces: /client   y   /driver    (el rol se deriva del JWT, no del cliente)
```

Rooms automáticos al conectar:
- Cliente: `user:<userId>` y, si tiene pedido activo, `order:<orderId>`.
- Driver: `driver:<driverProfileId>`, `geo:chincha` (difusión de ofertas) y `order:<orderId>` si está en carrera.

### 5.2 Eventos Cliente → Servidor

| Evento | Payload | Efecto |
|---|---|---|
| `client:subscribe_order` | `{ orderId }` | une al room `order:<id>`; devuelve snapshot |
| `client:unsubscribe_order` | `{ orderId }` | sale del room |
| `client:ping` | `{ ts }` | ack de latencia |

### 5.3 Eventos Driver → Servidor

| Evento | Payload | Efecto |
|---|---|---|
| `driver:go_online` | `{ lat, lng }` | UPSERT `driver_locations`, `is_available = true`, entra a `geo:chincha` |
| `driver:go_offline` | `{}` | `is_available = false`, sale del room |
| `driver:location` | `{ lat, lng, heading, speedKph, accuracyM, ts }` | UPSERT posición viva (throttle 1/3 s); si hay pedido activo → reenvía a `order:<id>` y muestrea a `order_tracks` cada 15 s |
| `driver:accept_offer` | `{ orderId, offerId }` | ejecuta `aceptarOferta()`; ack `{ ok, order }` o `{ ok: false, code: 'ORDER_ALREADY_TAKEN' }` |
| `driver:reject_offer` | `{ orderId, offerId }` | marca `REJECTED` y adelanta la ola |
| `driver:stage` | `{ orderId, stage }` | atajo WS equivalente a los POST de etapa (REST sigue siendo la fuente autoritativa) |

### 5.4 Eventos Servidor → Driver

| Evento | Payload |
|---|---|
| `order:offer` | `{ offerId, orderId, code, errandDescription, requiresPurchase, purchaseBudget, pickup{address,reference,lat,lng,distanceM,etaS}, dropoff{address,lat,lng}, distanceM, durationS, farePayout, paymentMethod, amountToCollect, expiresAt }` — `amountToCollect` = `cobro_total` de §3.1, lo que el driver cobrará contra entrega |
| `offer:revoked` | `{ orderId, reason: 'TAKEN' \| 'CANCELLED' \| 'TIMEOUT' }` |
| `order:cancelled_by_client` | `{ orderId, reason, compensation }` |
| `driver:wallet_updated` | `{ balance, lastEntry }` |
| `system:force_offline` | `{ reason }` |

### 5.5 Eventos Servidor → Cliente

| Evento | Payload |
|---|---|
| `order:searching` | `{ orderId, wave, driversNotified, expiresAt }` |
| `order:accepted` | `{ orderId, driver{ id, fullName, ratingAvg, vehicleType, vehicleDesc, plate, phone, avatarUrl, yapePhone }, amountToPay, etaToPickupS }` — `yapePhone` se envía **solo** si `paymentMethod = 'YAPE'`; en efectivo va `null` |
| `order:driver_location` | `{ orderId, lat, lng, heading, etaS, ts }` *(≈1 cada 3 s)* |
| `order:status_changed` | `{ orderId, from, to, at, actorRole }` |
| `order:arrived_pickup` | `{ orderId, at }` |
| `order:picked_up` | `{ orderId, at }` |
| `order:arrived_dropoff` | `{ orderId, at, otpReminder: true }` |
| `order:delivered` | `{ orderId, at, fareTotal, requiresRating: true }` |
| `order:expired` | `{ orderId, reason: 'NO_DRIVERS' }` |
| `order:failed` | `{ orderId, reason }` |
| `payment:status_changed` | `{ orderId, paymentId, status }` |

### 5.6 Reglas de resiliencia

1. **Reconexión:** `socket.io-client` con `reconnectionDelayMax: 10000`. Al reconectar, la app llama `GET /orders/active` y `GET /orders/:id/track` para reconstruir el estado; el WebSocket **nunca** es la única fuente de verdad.
2. **Push como respaldo:** si no hay socket para un `driverId`, la oferta se envía por Expo Push con `priority: high` y `channelId: 'offers'`.
3. **Throttle de GPS:** la app emite máximo 1 punto cada 3 s y solo si el desplazamiento supera 15 m; en background, `expo-task-manager` con `distanceInterval: 50`.
4. **Idempotencia:** los POST de etapa aceptan el header `Idempotency-Key`; repetir una etapa ya aplicada devuelve `200` con el estado actual, no `409`.

---

## 6. Seguridad y cumplimiento (MVP)

| Riesgo | Mitigación |
|---|---|
| OTP filtrado | `delivery_otp` nunca se serializa al driver (interceptor de respuesta recorta el campo por rol); 5 intentos máx.; sin OTP no hay `DELIVERED` |
| Driver falsea la entrega | Toda transición registra `geom`; `arrived-dropoff` valida `ST_Distance(pos, dropoff) < 150 m` (advertencia, no bloqueo, por deriva de GPS) |
| Comprobante Yape falso | Riesgo eliminado por diseño: no se acepta ninguna captura como prueba de pago. El driver verifica el abono en su propia app de Yape y confirma con `paymentReceived: true` en `verify-otp` |
| Cliente que no paga contra entrega | Sin `paymentReceived: true` no hay `DELIVERED`. La incidencia `NO_PAYMENT` queda en `order_status_history` con `geom` y hora; el admin suspende al cliente reincidente vía SQL |
| Driver que confirma un cobro que no recibió | Solo se perjudica a sí mismo: la comisión se asienta igual al entregar. La asimetría es intencional y es la razón de que la confirmación la haga el driver y no el cliente |
| Robo de cuenta | argon2id, refresh tokens rotatorios hasheados, revocación por dispositivo |
| Abuso de la API de Maps | El cliente **nunca** llama a Google directo: todo pasa por `/geo/*` con rate limit y cache; API key restringida por paquete Android + SHA-1 |
| Datos personales (Ley 29733, Perú) | Teléfono del contraparte visible solo durante el pedido activo; `order_tracks` purgado a los 30 días; política de privacidad publicada (requisito de Play Store) |
| Secretos en el repo | `.env` fuera de git; secretos de build en EAS Secrets; `app.config.ts` lee de `process.env` |

---

## 7. Roadmap y WBS atómico

Convención: cada tarea es un prompt ejecutable para Claude Code. `[Hh]` = estimación en horas. Total Semanas 1–4 = **85 h** (22 + 22 + 22.5 + 18.5), tras aplicar ADR-09; las 3 h netas liberadas quedan como colchón dentro de la capacidad de 88 h.

### FASE 0 — Preparación (no programación, 3 h, hacer antes de la Semana 1)

| # | Tarea | Horas |
|---|---|---|
| 0.1 | Crear proyecto en Google Cloud; habilitar Places API (New), Geocoding API, Routes API, Maps SDK for Android; API key restringida | 1.0 |
| 0.2 | Crear proyecto Supabase (región `sa-east-1`); habilitar extensión PostGIS; crear bucket `documents` (privado) para licencia/SOAT/DNI de los drivers. **No** se crea bucket `receipts`: no hay comprobantes de pago | 0.5 |
| 0.3 | Crear droplet DigitalOcean Ubuntu 24.04 ($6); instalar Docker + Caddy; apuntar subdominio `api.<dominio>` | 1.0 |
| 0.4 | Crear cuenta Expo/EAS y cuenta Google Play Console ($25) | 0.5 |

---

### SEMANA 1 — Fundamentos: monorepo, auth, dominio (22 h)

**Hito H1:** la app arranca en un teléfono físico, un usuario se registra, inicia sesión, y el backend desplegado responde `/health` por HTTPS.

| # | Tarea atómica (prompt para Claude Code) | Horas |
|---|---|---|
| 1.1 | Inicializar monorepo pnpm + Turborepo con `apps/`, `packages/`, `turbo.json`, `.nvmrc`, `.gitignore`, ESLint/Prettier/TS compartidos en `packages/config` | 2.0 |
| 1.2 | Scaffold NestJS en `apps/api` con validación de env por zod, `ConfigModule` global, `AllExceptionsFilter`, `LoggingInterceptor`, `ZodValidationPipe`, CORS y módulo `health` | 2.5 |
| 1.3 | Configurar Prisma: `schema.prisma` completo de §2, conexión a Supabase, migración inicial, migración SQL manual para columnas `geography` + índices GIST | 3.0 |
| 1.4 | `seed.ts`: `fare_configs` `chincha-v1`, `service_areas` con el polígono de Chincha, 1 admin, 3 drivers verificados (2 con `yape_phone`, 1 sin él para probar el filtro de pedidos en efectivo), 2 clientes de prueba | 1.5 |
| 1.5 | Módulo `auth`: registro, OTP de teléfono (mock en consola en el MVP), login, refresh rotatorio, argon2id, `JwtAuthGuard`, `RolesGuard`, `@CurrentUser()` | 4.0 |
| 1.6 | Módulo `users`: `GET/PATCH /users/me`, `switch-role` (reemite JWT), `push-token`, CRUD de `saved_places` | 2.0 |
| 1.7 | Scaffold Expo en `apps/mobile` con expo-router, NativeWind v4, tokens de tema en `src/theme`, `api-client.ts` con interceptor de refresh, `expo-secure-store`, store de sesión en Zustand | 3.5 |
| 1.8 | Pantallas `(auth)`: login, registro, verificación de teléfono. Componentes `ui/` base (Button, Input, Sheet, Badge, Skeleton) aplicando la skill `frontend-design` | 2.5 |
| 1.9 | Deploy: `Dockerfile` multi-stage para la API, `docker-compose.yml` (api + redis), Caddy con TLS automático, script `deploy.sh`; verificar `/health` por HTTPS | 1.0 |

---

### SEMANA 2 — Geo, tarifas y creación de pedidos (22 h)

**Hito H2:** el cliente elige A y B en el mapa, escribe el encargo, ve la tarifa desglosada y crea un pedido que queda en `SEARCHING` con su OTP visible.

| # | Tarea atómica | Horas |
|---|---|---|
| 2.1 | `packages/fare-engine`: `calculateFare()`, `haversineMeters()`, `isInsideGeofence()`, config por defecto + suite vitest con los 6 casos de §3.2 | 3.0 |
| 2.2 | `GoogleMapsService` en la API: `computeRoutes` (TWO_WHEELER), `geocode`, `reverseGeocode`, `placesAutocomplete` con session token, cache Redis del par A/B (TTL 6 h), fallback haversine con `estimationMode` | 3.5 |
| 2.3 | Módulo `geo`: endpoints `/geo/autocomplete`, `/geo/place/:id`, `/geo/reverse`, `/geo/service-area`, con rate limit y `locationRestriction` a Chincha | 1.5 |
| 2.4 | Módulo `fares`: `POST /orders/quote` + `GET /config/fare`; validación de geofence con `ST_Covers`; errores `OUT_OF_SERVICE_AREA` | 1.5 |
| 2.5 | Módulo `orders` parte 1: `POST /orders` (transacción: tarifa congelada + OTP de 4 dígitos + `payments` PENDING + `order_status_history`), `GET /orders/active`, `GET /orders/:id` con serialización por rol, `GET /orders` paginado por cursor | 4.0 |
| 2.6 | `OrdersService.transition()`: tabla de adyacencia de estados, escritura de historial en la misma transacción, errores `409 INVALID_TRANSITION` | 2.0 |
| 2.7 | App: `MapCanvas` con `react-native-maps` (proveedor Google), permisos de ubicación en primer plano, centrado en la posición actual, polígono del área de servicio dibujado | 2.5 |
| 2.8 | App: flujo `new-order/pick-points` — búsqueda con autocompletado, arrastre del pin, usar «mi ubicación», direcciones guardadas, validación A ≠ B | 2.5 |
| 2.9 | App: `new-order/describe` (textarea + chips de ejemplo + toggle «requiere compra» con presupuesto + tamaño de paquete) y `new-order/confirm` con `FareBreakdown` y selector de método de pago (efectivo / Yape, ambos rotulados «contra entrega») | 1.5 |

---

### SEMANA 3 — Realtime: dispatch, tracking, OTP y cobro (22.5 h)

**Hito H3:** happy path end-to-end sobre dos teléfonos. El pedido llega al driver, lo acepta, el cliente ve el ícono moverse en el mapa y la entrega se cierra con OTP **y cobro confirmado**.

| # | Tarea atómica | Horas |
|---|---|---|
| 3.1 | Infra Socket.io: `RealtimeGateway` con namespaces `/client` y `/driver`, `WsJwtGuard`, gestión de rooms, `RedisIoAdapter`, registro de conexiones/desconexiones | 3.0 |
| 3.2 | `GeoRepository`: consulta de §3.4 con `$queryRaw` tipado, UPSERT de `driver_locations`, `POST /drivers/me/status`, `driver:go_online` / `go_offline` | 2.5 |
| 3.3 | `DispatchService`: olas progresivas de §3.5, `order_offers`, cola de temporizadores en Redis + `@Cron('*/5 * * * * *')`, expiración a los 5 min, aceptación atómica con `UPDATE ... WHERE status='SEARCHING'`, filtro de candidatos por `yape_phone` cuando `payment_method = 'YAPE'` y congelado de `payments.yape_target_phone` al aceptar | 4.5 |
| 3.4 | `TrackingService`: ingesta de `driver:location` con throttle, reenvío a `order:<id>`, muestreo a `order_tracks` cada 15 s, cálculo de ETA, `GET /orders/:id/track` | 2.5 |
| 3.5 | Endpoints de etapa: `arrived-pickup`, `picked-up`, `arrived-dropoff`, `fail`, `cancel` (con reglas de penalización) — con `Idempotency-Key` y validación de proximidad | 2.0 |
| 3.6 | `verify-otp`: comparación en tiempo constante, contador de intentos, bloqueo a los 5, exigencia de `paymentReceived: true` (`422 PAYMENT_NOT_CONFIRMED`), transición a `DELIVERED` con `payments.status = CONFIRMED` en la misma transacción, asientos en `driver_ledger`, liberación del driver, `resend-otp`, y `fail` con `reason: 'NO_PAYMENT'` (sin asiento de comisión) | 2.5 |
| 3.7 | App Driver: `dashboard` con `OnlineToggle`, suscripción WS, `OfferCard` con countdown de 20 s, aceptar/rechazar, manejo de `ORDER_ALREADY_TAKEN` y `offer:revoked` | 3.0 |
| 3.8 | App Driver: `run/[id]` — navegación por etapas (botón grande contextual), deep link a Google Maps, emisión de GPS en primer plano, teclado numérico de OTP | 2.5 |

---

### SEMANA 4 — Cobro contra entrega, ratings, pulido y cierre del MVP (18.5 h)

**Hito H4:** MVP funcional completo: cobro contra entrega confirmado por el driver, calificaciones, notificaciones push, historial, ganancias del driver y build interno distribuible (`preview`).

> La decisión de cobro contra entrega (ADR-09) elimina el módulo de comprobantes: `SupabaseStorageService` para recibos, hash SHA-256, `expo-image-picker` con compresión, pantalla de carga de captura y los estados `PROOF_UPLOADED` / `REJECTED`. Esta semana baja de 22 h a **18.5 h**; las 3.5 h liberadas quedan como colchón, no como alcance nuevo.

| # | Tarea atómica | Horas |
|---|---|---|
| 4.1 | App Cliente: `order/[id]` — tracking en vivo (marcador animado + polyline), `StatusTimeline`, `OtpDisplay` (card grande), datos del driver, botón de llamada, cancelar | 3.5 |
| 4.2 | Módulo `payments`: fila `payments` creada con el pedido, `GET /payments/order/:orderId` con exposición de `yapeTargetPhone` solo desde `PICKED_UP` y solo si `method = 'YAPE'`, congelado de `yape_target_phone` al aceptar la oferta, `POST /payments/:id/confirm` (solo ADMIN, corrección), evento `payment:status_changed` | 1.0 |
| 4.3 | App: cobro contra entrega — cliente ve monto exacto y, si es Yape, el número del driver con botón de copiar; driver ve «cobrar S/ X en efectivo \| por Yape» y el check **«recibí el pago»** junto al teclado del OTP, más el botón «no me pagaron» que dispara `fail` con `NO_PAYMENT`. Incluye el campo `yape_phone` en el registro y el perfil del driver | 1.5 |
| 4.4 | Módulo `ratings` + `POST /ratings`, recálculo de `rating_avg`, modal post-entrega bidireccional con chips de etiquetas | 2.0 |
| 4.5 | `ExpoPushService` + `notifications.ts`: registro de token, canal `offers` (alta prioridad, sonido), respaldo de ofertas, avisos de cambio de estado, deep link al pedido | 2.5 |
| 4.6 | App: `history` (cliente) y `earnings` + `ledger` (driver) con infinite scroll sobre cursor; pantalla de detalle de pedido pasado | 2.5 |
| 4.7 | Estados vacíos, skeletons, `ErrorBoundary`, banner de offline, toasts, revisión de accesibilidad (tamaños táctiles ≥ 44 px, contraste) | 2.0 |
| 4.8 | Pruebas manuales end-to-end con 2 teléfonos sobre el guión de 13 escenarios (happy path en efectivo, happy path por Yape, sin drivers, doble aceptación, OTP fallido, cliente que no paga → `NO_PAYMENT`, cancelaciones, reconexión, pérdida de señal) + corrección de bugs encontrados | 2.5 |
| 4.9 | `eas.json`, variables de entorno por perfil, build `preview` (APK interno) y verificación en dispositivo real | 1.0 |

---

### SEMANAS 5–6 — Hardening y publicación (44 h, recomendado)

**Hito H5:** AAB firmado aprobado y publicado en Google Play (pista de prueba cerrada → producción).

| # | Tarea | Horas |
|---|---|---|
| 5.1 | GPS en background: `expo-task-manager` + `expo-location` con `foregroundService`, permiso `ACCESS_BACKGROUND_LOCATION`, notificación persistente, justificación para Play Store | 6.0 |
| 5.2 | Robustez del dispatch: recuperación de pedidos huérfanos al reiniciar, drivers zombi (heartbeat), reintento de re-dispatch tras cancelación del driver, pruebas de carrera | 5.0 |
| 5.3 | Tests e2e de la API (supertest + Postgres de prueba): auth, creación, dispatch, OTP, cobro (`DELIVERED` imposible sin `paymentReceived`, `NO_PAYMENT` sin asiento de comisión). Objetivo: cobertura de los caminos críticos | 7.0 |
| 5.4 | Observabilidad: Sentry (app + API), logs estructurados con pino, `/metrics` básico, alerta por Telegram de errores 5xx | 4.0 |
| 5.5 | Panel de admin mínimo: aprobación de drivers, saldo de comisiones por driver, incidencias `NO_PAYMENT`, cancelación forzada (puede ser una CLI o un Next.js mínimo) | 6.0 |
| 5.6 | Preparación de Play Store: ícono adaptativo, splash, 6 capturas, texto de la ficha, política de privacidad publicada, cuestionario de seguridad de datos, declaración de permisos | 5.0 |
| 5.7 | Build `production` AAB firmado, carga a pista cerrada, prueba con 10 usuarios reales, corrección de hallazgos | 6.0 |
| 5.8 | Backups de DB, runbook de operación, documento de precios/parámetros ajustables, promoción a producción | 5.0 |

### 7.4 Plan de contingencia: MVP Mínimo Viable Real en 4 semanas (84 h)

Si el plazo es inamovible, **recortar** en este orden exacto:

Punto de partida actualizado: **85 h** en Semanas 1–4 (22 + 22 + 22.5 + 18.5), ya con el cobro contra entrega aplicado.

1. GPS en background (solo tracking en primer plano) → ahorra 6 h
2. ~~Comprobante Yape fotográfico → reemplazar por «el driver confirma que recibió el Yape»~~ → **ya aplicado** como decisión de negocio (ADR-09), no es un recorte disponible
3. Calificaciones → ahorra 2 h
4. Pantalla de ganancias/ledger → consultar por SQL → ahorra 2.5 h
5. Verificación de teléfono por OTP en registro → registro directo → ahorra 1.5 h
6. Historial paginado → últimos 20 fijos → ahorra 1 h

Lo que **nunca** se recorta: auth, geofence, tarifa congelada, dispatch con aceptación atómica, tracking en primer plano, OTP de entrega, confirmación de cobro del driver, asiento de comisión, máquina de estados con auditoría.

---

## 8. Definición de «Terminado» (Definition of Done)

Una tarea está terminada cuando:
1. `pnpm turbo typecheck lint` pasa sin errores ni warnings nuevos.
2. La lógica pura (fare-engine, transiciones) tiene test unitario.
3. Ningún secreto nuevo quedó hardcodeado; `.env.example` está actualizado.
4. El flujo se probó en un **dispositivo físico**, no solo en emulador (el GPS y los permisos se comportan distinto).
5. Los endpoints nuevos están en `docs/api.http` con un ejemplo real.
6. Si cambió el esquema, existe migración Prisma versionada (nunca `db push` contra producción).

---

## 9. Riesgos y mitigaciones

| Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|
| Direcciones de Chincha mal indexadas en Google Places | Alta | Alto | Campo obligatorio de **referencia** en texto libre + arrastre del pin en mapa; no depender solo del autocompletado |
| Costo de Google Maps por sobre el nivel gratuito | Media | Medio | Cache Redis, session tokens, fallback haversine, alerta de presupuesto en GCP a los $10 |
| Rechazo en Play Store por permiso de ubicación en background | Media | Alto | Declaración de uso en primer plano con `foregroundService` visible + video explicativo; si se rechaza, el MVP funciona solo en primer plano |
| Drivers sin smartphone capaz / datos móviles intermitentes | Alta | Alto | Carga útil mínima en WS, reconexión agresiva, respaldo por push, pantallas que funcionan con último estado conocido |
| Cliente que no paga contra entrega | Media | Medio | El driver verifica el abono en su propia app de Yape (o cuenta el efectivo) antes de cerrar; sin `paymentReceived: true` no hay `DELIVERED`. Incidencia `NO_PAYMENT` con hora y `geom`; suspensión del cliente reincidente vía SQL |
| Driver con `yape_phone` mal escrito o de otra persona | Media | Medio | Campo validado como `+519XXXXXXXX` en el registro, verificado por el admin junto con los documentos, y congelado en `payments.yape_target_phone` al aceptar la oferta para poder auditar a quién se pagó |
| Driver que acumula comisión y no la liquida | Alta | Medio | `wallet_balance` visible para el negocio y para el driver; umbral de deuda que bloquea el paso a `ONLINE` (parámetro operativo, no código nuevo); suspensión manual |
| Subestimación del cronograma | Alta | Medio | Hitos semanales demostrables; plan de recorte de §7.4 ya definido |
| Dos drivers aceptan el mismo pedido | Media | Alto | `UPDATE ... WHERE status='SEARCHING'` (bloqueo a nivel de fila) + índice único de pedido activo por driver |

---

## 10. Instrucciones para iniciar la Fase 1

### 10.1 Prerrequisitos locales

```powershell
node -v                 # debe ser >= 22
npm i -g pnpm@latest
npm i -g eas-cli
pnpm -v
```

### 10.2 Prompt #1 — Esqueleto del monorepo

> Copiar y pegar tal cual en Claude Code, en `C:\Users\HP\Desktop\app-delivery`:

```
Inicializa el monorepo base del proyecto app-delivery. Delega la ejecución a un
subagente y revisa el resultado. Requisitos exactos:

1. Raíz:
   - package.json con "packageManager": "pnpm@9", private: true, workspaces vía
     pnpm-workspace.yaml (apps/*, packages/*), y scripts raíz que deleguen en
     turbo: dev, build, lint, typecheck, test, clean.
   - turbo.json con pipeline: build (dependsOn ^build, outputs dist/**),
     dev (cache false, persistent true), lint, typecheck, test.
   - .nvmrc con 22, .gitignore (node_modules, dist, .env*, .expo, .turbo,
     *.keystore, *.jks), .editorconfig.
   - .env.example con TODAS las variables: DATABASE_URL, DIRECT_URL,
     JWT_ACCESS_SECRET, JWT_REFRESH_SECRET, JWT_ACCESS_TTL=15m,
     JWT_REFRESH_TTL=30d, REDIS_URL, GOOGLE_MAPS_API_KEY,
     SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY,
     SUPABASE_BUCKET_DOCUMENTS, PORT=3000, NODE_ENV, APP_TZ=America/Lima,
     DEFAULT_SEARCH_RADIUS_M=4000, OFFER_TIMEOUT_S=20, SEARCH_TIMEOUT_S=300.

2. packages/config: configuración compartida publicada como @delivery/config
   - tsconfig/base.json (strict true, target ES2022, moduleResolution bundler),
     tsconfig/nest.json, tsconfig/expo.json
   - eslint/base.cjs, eslint/nest.cjs, eslint/expo.cjs (flat config o legacy,
     elige una y sé consistente)
   - prettier/index.json (printWidth 100, singleQuote true, semi true)

3. packages/shared: paquete @delivery/shared, TypeScript puro, build con tsup.
   - src/enums.ts con TODOS los enums del plan: UserRole, UserStatus,
     DriverStatus, VehicleType, OrderStatus, PaymentMethod, PaymentStatus,
     OfferStatus, LedgerKind, PackageSize.
   - src/socket-events.ts con interfaces tipadas ClientToServerEvents,
     ServerToClientEvents, DriverToServerEvents según el contrato del plan.
   - src/dto/ con schemas zod: registerSchema, loginSchema, quoteOrderSchema,
     createOrderSchema, verifyOtpSchema (incluye paymentReceived: z.literal(true)),
     stageUpdateSchema, failOrderSchema (reason enum con NO_PAYMENT),
     ratingSchema. Exporta también los tipos inferidos.
   - src/index.ts re-exportando todo.

4. packages/fare-engine: paquete @delivery/fare-engine, depende de
   @delivery/shared, test con vitest.
   - src/types.ts, src/config.ts (FareConfig por defecto chincha-v1 con los
     valores de la tabla fare_configs del plan), src/geo.ts
     (haversineMeters, isInsideGeofence, bboxAround), src/fare.ts
     (calculateFare exactamente como en el plan).
   - src/fare.test.ts con los 6 casos de prueba de la tabla del plan.

5. Crea las carpetas apps/api y apps/mobile vacías con un .gitkeep (se
   scaffoldean en las tareas 1.2 y 1.7).

Al terminar: ejecuta pnpm install, pnpm turbo typecheck y pnpm --filter
@delivery/fare-engine test. Reporta la salida real de los tres comandos.
No instales dependencias que no se usen.
```

### 10.3 Prompt #2 — Backend NestJS + Prisma

```
Scaffold de apps/api (tareas 1.2 y 1.3 del plan). Delega a un subagente.

A) NestJS 11 con TypeScript, estructura de carpetas exacta de la sección 1.3
   del PLAN_DESARROLLO.md.
B) config/env.schema.ts validando el entorno con zod y fallando el arranque si
   falta una variable. ConfigModule global y tipado.
C) common/: AllExceptionsFilter y PrismaExceptionFilter que emitan el formato
   de error { statusCode, code, message, timestamp }; LoggingInterceptor con
   pino; ZodValidationPipe; decoradores @Public, @Roles, @CurrentUser.
D) infra/prisma: PrismaService con onModuleInit/enableShutdownHooks.
E) prisma/schema.prisma con TODAS las tablas, enums, índices y relaciones de
   la sección 2 del PLAN_DESARROLLO.md. Las columnas geography se declaran
   como Unsupported("geography(Point,4326)").
F) Una migración SQL manual que: cree la extensión postgis, convierta las
   columnas geography, y cree los índices GIST y los índices parciales únicos
   (uq_driver_active_order, idx_orders_searching).
G) Módulo health con GET /health comprobando DB y Redis.
H) main.ts: helmet, CORS, validación global, prefijo global /v1, shutdown
   hooks, puerto desde config.

Al terminar: pnpm --filter api build y pnpm --filter api prisma validate.
Pega la salida real. No ejecutes migraciones contra la base remota todavía.
```

### 10.4 Prompt #3 — Seeds y datos de Chincha

```
Implementa prisma/seed.ts (tarea 1.4). Debe insertar de forma idempotente
(upsert por clave natural):

1. fare_configs "chincha-v1" con los valores de la tabla del plan.
2. service_areas "Chincha" con un polígono geography que cubra Chincha Alta,
   Chincha Baja, Pueblo Nuevo, Sunampe, Grocio Prado y Tambo de Mora.
   Usa un polígono de al menos 8 vértices alrededor de lat -13.42, lng -76.13.
   Deja el arreglo de coordenadas en una constante separada y comentada para
   poder ajustarlo después.
3. 1 admin, 2 clientes y 3 drivers verificados con driver_locations dentro del
   polígono y a distancias distintas del centro (0.8 km, 2.5 km, 6 km) para
   poder probar las olas de dispatch.
4. Password de todos los usuarios de prueba: "Test1234!" hasheado con argon2id.

Agrega el script "db:seed" en apps/api/package.json y el bloque prisma.seed.
```

### 10.5 Orden de ejecución de la Semana 1

```
Prompt #1  → 1.1 monorepo                        (2.0 h)
Prompt #2  → 1.2 + 1.3 NestJS + Prisma           (5.5 h)
Prompt #3  → 1.4 seeds                           (1.5 h)
Prompt #4  → 1.5 módulo auth                     (4.0 h)
Prompt #5  → 1.6 módulo users                    (2.0 h)
Prompt #6  → 1.7 scaffold Expo + NativeWind      (3.5 h)
Prompt #7  → 1.8 pantallas de auth + UI base     (2.5 h)
Prompt #8  → 1.9 Docker + Caddy + deploy         (1.0 h)
                                            Total 22.0 h
```

---

## 11. Checklist de cierre del MVP

- [ ] Un cliente nuevo se registra, crea un pedido y lo recibe entregado sin intervención manual.
- [ ] El pedido se difunde a drivers reales en menos de 3 s desde su creación.
- [ ] Dos drivers aceptando a la vez: uno gana, el otro recibe `ORDER_ALREADY_TAKEN`.
- [ ] Sin drivers en 4 olas: el pedido expira y el cliente es notificado.
- [ ] El cliente ve la posición del driver actualizarse al menos cada 5 s.
- [ ] Sin el OTP correcto es **imposible** cerrar el pedido como entregado.
- [ ] Un pedido no pasa a `DELIVERED` sin `paymentReceived: true`; el impago queda como `FAILED` + `payments.status = NOT_RECEIVED` y sin asiento de comisión.
- [ ] El cliente ve el número de Yape del driver asignado solo desde `PICKED_UP` y solo si eligió Yape.
- [ ] `driver_ledger` cuadra: suma de comisiones = suma de `commission_amount` de pedidos entregados.
- [ ] La app sobrevive a perder el internet 60 s y recupera el estado al reconectar.
- [ ] AAB firmado instalable y funcional en un dispositivo limpio.

# Semana 2 — Geo, tarifas y creación de pedidos (22 h)

**Hito H2:** el cliente elige A y B en el mapa, escribe el encargo, ve la tarifa desglosada y crea
un pedido que queda en `SEARCHING` con su OTP visible.

**Precondición:** gate de la Semana 1 en verde + API key de Google Maps disponible.

## PROMPT DE ARRANQUE

```
/semana 2
```

## TAREAS

| # | Tarea | Plan § | Modelo | Estado | h |
|---|---|---|---|---|---|
| 2.1 | `packages/fare-engine`: `calculateFare()`, `haversineMeters()`, `isInsideGeofence()`, config por defecto + suite vitest con los 6 casos | §3.1, §3.2 | Sonnet | ⬜ | 3.0 |
| 2.2 | `GoogleMapsService`: `computeRoutes` (TWO_WHEELER), `geocode`, `reverseGeocode`, `placesAutocomplete` con session token, cache Redis del par A/B (TTL 6 h), fallback haversine con `estimationMode` | §3.3 | Sonnet | ⬜ | 3.5 |
| 2.3 | Módulo `geo`: `/geo/autocomplete`, `/geo/place/:id`, `/geo/reverse`, `/geo/service-area` con rate limit y `locationRestriction` a Chincha | §4.6 | Sonnet | ⬜ | 1.5 |
| 2.4 | Módulo `fares`: `POST /orders/quote` + `GET /config/fare`; geofence con `ST_Covers`; error `OUT_OF_SERVICE_AREA` | §3.4, §4.6 | Sonnet | ⬜ | 1.5 |
| 2.5 | Módulo `orders` parte 1: `POST /orders` (transacción: tarifa congelada + OTP 4 dígitos + `payments` PENDING + historial), `GET /orders/active`, `GET /orders/:id` serializado por rol, `GET /orders` por cursor | §2.2, §4.4 | Sonnet | ⬜ | 4.0 |
| 2.6 | `OrdersService.transition()`: tabla de adyacencia, historial en la misma transacción, `409 INVALID_TRANSITION` | §2.4 | Sonnet | ⬜ | 2.0 |
| 2.7 | App: `MapCanvas` con `react-native-maps`, permisos de ubicación en primer plano, centrado en posición actual, polígono del área dibujado | §1.3 | Sonnet + skill | ⬜ | 2.5 |
| 2.8 | App: `new-order/pick-points` — autocompletado, arrastre del pin, «mi ubicación», direcciones guardadas, validación A ≠ B | §1.3 | Sonnet + skill | ⬜ | 2.5 |
| 2.9 | App: `new-order/describe` + `new-order/confirm` con `FareBreakdown` y selector de método de pago (efectivo / Yape, ambos «contra entrega») | §1.3 | Sonnet + skill | ⬜ | 1.5 |

## Trampas a vigilar en el diff (`CLAUDE.md` §6)

1. `ST_MakePoint(lng, lat)` — longitud primero, en toda llamada a PostGIS y a Google.
2. Dinero redondeado en cada paso del desglose; nada de flotantes acumulados.
3. OTP recortado por rol en `orders.serializer.ts`; jamás en una respuesta al driver.
4. Recargo nocturno con hora de Lima: `TZ=America/Lima` en el contenedor, no `getHours()` en UTC.
5. La tarifa se congela en `fare_breakdown`; prohibido recalcularla al abrir un pedido.
6. `/geo/*` con cache y rate limit: cada llamada suelta a Google cuesta dinero real.

## BITÁCORA

<!-- append-only, ≤6 líneas por sesión -->

## GATE DE CIERRE

```bash
pnpm turbo typecheck
pnpm turbo lint
pnpm --filter @delivery/fare-engine test
```

- [ ] Los 6 casos de tarifa de §3.2 pasan con los valores reales de `fare_configs`
- [ ] `POST /orders/quote` devuelve el mismo total que `calculateFare()` para los mismos insumos
- [ ] Un punto fuera del polígono devuelve `OUT_OF_SERVICE_AREA`, no un precio
- [ ] El pedido creado guarda `fare_breakdown` y `fare_config_id`; reabrirlo no recalcula nada
- [ ] `GET /orders/:id` como driver **no** incluye `delivery_otp` (verificado con el token del driver)
- [ ] Una transición ilegal devuelve `409 INVALID_TRANSITION` y no deja fila en `order_status_history`
- [ ] Cache de rutas activa: dos cotizaciones del mismo par A/B generan **una** llamada a Routes
- [ ] En teléfono físico: elegir A y B, describir el encargo, ver el desglose y crear el pedido
- [ ] Revisión de diff con `cavecrew-reviewer`; findings resueltos o en Deuda
- [ ] `ESTADO.md` actualizado y commit `feat(semana-2): geo, tarifas y pedidos` propuesto

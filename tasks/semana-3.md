# Semana 3 — Realtime: dispatch, tracking, OTP y cobro (22.5 h)

**Hito H3:** happy path end-to-end sobre dos teléfonos. El pedido llega al driver, lo acepta, el
cliente ve el ícono moverse en el mapa, y la entrega se cierra con OTP **y cobro confirmado**.

**Precondición:** gate de la Semana 2 en verde. Dos teléfonos Android físicos disponibles.

## PROMPT DE ARRANQUE

```
/semana 3
```

## TAREAS

| # | Tarea | Plan § | Modelo | Estado | h |
|---|---|---|---|---|---|
| 3.1 | Infra Socket.io: `RealtimeGateway` con namespaces `/client` y `/driver`, `WsJwtGuard`, gestión de rooms, `RedisIoAdapter`, registro de conexión/desconexión | §5.1 | Sonnet | ⬜ | 3.0 |
| 3.2 | `GeoRepository`: consulta de §3.4 con `$queryRaw` tipado, UPSERT de `driver_locations`, `POST /drivers/me/status`, `driver:go_online` / `go_offline` | §3.4, §4.3 | Sonnet | ⬜ | 2.5 |
| 3.3 | `DispatchService`: olas progresivas, `order_offers`, temporizadores en Redis + `@Cron('*/5 * * * * *')`, expiración a 5 min, aceptación atómica, filtro por `yape_phone` si el pedido es Yape, congelado de `payments.yape_target_phone` | §3.5 | Sonnet | ⬜ | 4.5 |
| 3.4 | `TrackingService`: ingesta de `driver:location` con throttle, reenvío a `order:<id>`, muestreo a `order_tracks` cada 15 s, ETA, `GET /orders/:id/track` | §5.3, §5.6 | Sonnet | ⬜ | 2.5 |
| 3.5 | Endpoints de etapa: `arrived-pickup`, `picked-up`, `arrived-dropoff`, `fail`, `cancel` con `Idempotency-Key` y validación de proximidad | §4.4 | Sonnet | ⬜ | 2.0 |
| 3.6 | `verify-otp`: comparación en tiempo constante, 5 intentos, exigencia de `paymentReceived: true` (`422 PAYMENT_NOT_CONFIRMED`), `DELIVERED` + `payments.status = CONFIRMED` en la misma transacción, asientos en `driver_ledger`, liberación del driver, `resend-otp`, `fail` con `NO_PAYMENT` sin comisión | §2.4, §4.4, §4.5 | Sonnet | ⬜ | 2.5 |
| 3.7 | App Driver: `dashboard` con `OnlineToggle`, suscripción WS, `OfferCard` con countdown de 20 s, aceptar/rechazar, `ORDER_ALREADY_TAKEN`, `offer:revoked` | §5.4 | Sonnet + skill | ⬜ | 3.0 |
| 3.8 | App Driver: `run/[id]` — botón contextual por etapa, deep link a Google Maps, GPS en primer plano, teclado numérico de OTP | §1.3 | Sonnet + skill | ⬜ | 2.5 |

## Trampas a vigilar en el diff

1. **Aceptación de oferta:** debe ser `UPDATE orders SET ... WHERE id = ? AND status = 'SEARCHING'`
   verificando filas afectadas. Un `SELECT` previo seguido de `UPDATE` permite doble aceptación.
2. `ST_DWithin`, nunca `ST_Distance(...) < x` — el segundo ignora el índice GIST.
3. Rooms de Socket.io abandonados en `disconnect`; listeners registrados una sola vez, no por render.
4. `useEffect` con cleanup en la suscripción al socket y en el watcher de `expo-location`.
5. Estado del pedido mutado fuera de `transition()` → historial incompleto.
6. `DELIVERED` sin `paymentReceived: true` es un bug, no un atajo de pruebas.

## BITÁCORA

<!-- append-only, ≤6 líneas por sesión -->

## GATE DE CIERRE

```bash
pnpm turbo typecheck
pnpm turbo lint
```

- [ ] Pedido creado se difunde a los drivers en **< 3 s** (medido, no estimado)
- [ ] Dos drivers aceptando a la vez: uno gana, el otro recibe `ORDER_ALREADY_TAKEN`
- [ ] Sin drivers tras 4 olas: el pedido pasa a `EXPIRED` y el cliente es notificado
- [ ] El cliente ve la posición del driver actualizarse al menos cada 5 s
- [ ] Sin OTP correcto es **imposible** cerrar el pedido; 5 intentos bloquean
- [ ] `DELIVERED` exige `paymentReceived: true`; sin él responde `422 PAYMENT_NOT_CONFIRMED`
- [ ] `fail` con `NO_PAYMENT`: pedido `FAILED`, pago `NOT_RECEIVED`, **cero** filas nuevas en `driver_ledger`
- [ ] Un pedido con método Yape no se ofrece a un driver sin `yape_phone`
- [ ] `payments.yape_target_phone` quedó congelado al aceptar la oferta
- [ ] Happy path completo entre **dos teléfonos físicos**, de punta a punta
- [ ] Reconexión: matar el socket 60 s y recuperar estado con `GET /orders/active` + `/track`
- [ ] Revisión de diff con `cavecrew-reviewer`; findings resueltos o en Deuda
- [ ] `ESTADO.md` actualizado y commit `feat(semana-3): dispatch, tracking y entrega` propuesto

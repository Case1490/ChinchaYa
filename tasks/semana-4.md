# Semana 4 — Cobro contra entrega, ratings, pulido y cierre del MVP (18.5 h)

**Hito H4:** MVP funcional completo: cobro contra entrega confirmado por el driver, calificaciones,
notificaciones push, historial, ganancias del driver y build interno distribuible (`preview`).

> ADR-09 eliminó el módulo de comprobantes (storage de recibos, hash SHA-256, `expo-image-picker`,
> estados `PROOF_UPLOADED`/`REJECTED`). Esta semana tiene 18.5 h de alcance sobre 22 h de
> capacidad. Las 3.5 h libres son **colchón**, no alcance nuevo.

**Precondición:** gate de la Semana 3 en verde.

## PROMPT DE ARRANQUE

```
/semana 4
```

## TAREAS

| # | Tarea | Plan § | Modelo | Estado | h |
|---|---|---|---|---|---|
| 4.1 | App Cliente: `order/[id]` — tracking en vivo (marcador animado + polyline), `StatusTimeline`, `OtpDisplay`, datos del driver, botón de llamada, cancelar | §5.5 | Sonnet + skill | ⬜ | 3.5 |
| 4.2 | Módulo `payments`: `GET /payments/order/:orderId` con `yapeTargetPhone` expuesto solo desde `PICKED_UP` y solo si el método es Yape, `POST /payments/:id/confirm` (ADMIN, corrección), evento `payment:status_changed` | §4.5, §2.2 | Sonnet | ⬜ | 1.0 |
| 4.3 | App: cobro contra entrega — cliente ve monto exacto y número de Yape con botón de copiar; driver ve «cobrar S/ X» + check **«recibí el pago»** junto al OTP + botón «no me pagaron» (`NO_PAYMENT`). Incluye `yape_phone` en registro y perfil del driver | §4.3, §4.5 | Sonnet + skill | ⬜ | 1.5 |
| 4.4 | Módulo `ratings`: `POST /ratings`, recálculo de `rating_avg`, modal post-entrega bidireccional con chips | §4.6 | Sonnet | ⬜ | 2.0 |
| 4.5 | `ExpoPushService` + `notifications.ts`: registro de token, canal `offers` (alta prioridad, sonido), respaldo de ofertas, avisos de estado, deep link al pedido | §5.6 | Sonnet | ⬜ | 2.5 |
| 4.6 | App: `history` (cliente) y `earnings` + `ledger` (driver) con infinite scroll por cursor; detalle de pedido pasado | §4.3, §4.4 | Sonnet + skill | ⬜ | 2.5 |
| 4.7 | Estados vacíos, skeletons, `ErrorBoundary`, banner de offline, toasts, accesibilidad (táctil ≥ 44 px, contraste) | — | Sonnet + skill | ⬜ | 2.0 |
| 4.8 | Pruebas manuales end-to-end con 2 teléfonos sobre el guión de 13 escenarios + corrección de bugs | §11 | — | ⬜ | 2.5 |
| 4.9 | `eas.json`, variables por perfil, build `preview` (APK interno) y verificación en dispositivo real | — | Sonnet | ⛔ cuenta EAS | 1.0 |

## Guión de los 13 escenarios (tarea 4.8)

1. Happy path en efectivo · 2. Happy path por Yape · 3. Sin drivers disponibles (expira) ·
4. Doble aceptación simultánea · 5. OTP incorrecto 5 veces · 6. Cliente que no paga → `NO_PAYMENT` ·
7. Cancelación del cliente antes de aceptar · 8. Cancelación del cliente después de aceptar
(penalidad) · 9. Cancelación del driver → re-dispatch · 10. Pérdida de señal del driver 60 s ·
11. Pedido fuera del área de servicio · 12. Encargo con compra y presupuesto ·
13. Driver sin `yape_phone` que no debe recibir pedidos Yape.

Registrar el resultado de cada escenario en la bitácora: `OK` o el bug encontrado.

## BITÁCORA

<!-- append-only, ≤6 líneas por sesión -->

## GATE DE CIERRE

```bash
pnpm turbo typecheck
pnpm turbo lint
pnpm --filter @delivery/fare-engine test
```

- [ ] Los 13 escenarios de 4.8 ejecutados, con resultado registrado uno por uno
- [ ] `driver_ledger` cuadra: suma de comisiones = suma de `commission_amount` de pedidos entregados
- [ ] Ningún pedido `DELIVERED` quedó con `payments.status = 'PENDING'` (verificar por SQL)
- [ ] El cliente **no** ve el `yape_phone` del driver antes de `PICKED_UP` ni en pedidos en efectivo
- [ ] Push de oferta llega con el socket cerrado y abre el pedido correcto por deep link
- [ ] Calificación bidireccional actualiza `rating_avg` de ambas partes
- [ ] APK `preview` instalado en un dispositivo limpio completa el flujo sin herramientas de dev
- [ ] Revisión de diff con `cavecrew-reviewer`; findings resueltos o en Deuda
- [ ] `ESTADO.md` actualizado y commit `feat(semana-4): cobro, ratings y cierre del MVP` propuesto

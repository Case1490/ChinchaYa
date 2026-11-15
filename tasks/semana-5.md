# Semana 5 — Hardening: background GPS, robustez, tests y observabilidad (22 h)

**Hito H5a:** el MVP resiste uso real: GPS en segundo plano, sin pedidos huérfanos, con tests e2e
de los caminos críticos y errores visibles en Sentry.

**Precondición:** gate de la Semana 4 en verde y APK `preview` funcionando.

> Las Semanas 5 y 6 son las 44 h de `PLAN_DESARROLLO.md` §7 partidas en dos tramos de 22 h.

## PROMPT DE ARRANQUE

```
/semana 5
```

## TAREAS

| # | Tarea | Plan § | Modelo | Estado | h |
|---|---|---|---|---|---|
| 5.1 | GPS en background: `expo-task-manager` + `expo-location` con `foregroundService`, permiso `ACCESS_BACKGROUND_LOCATION`, notificación persistente, justificación para Play Store | §5.6 | Sonnet | ⬜ | 6.0 |
| 5.2 | Robustez del dispatch: recuperación de pedidos huérfanos al reiniciar, drivers zombi por heartbeat, re-dispatch tras cancelación del driver, pruebas de carrera | §3.5 | Sonnet | ⬜ | 5.0 |
| 5.3 | Tests e2e de la API (supertest + Postgres de prueba): auth, creación, dispatch, OTP, cobro (`DELIVERED` imposible sin `paymentReceived`; `NO_PAYMENT` sin asiento de comisión) | §8 | Sonnet | ⬜ | 7.0 |
| 5.4 | Observabilidad: Sentry (app + API), logs estructurados con pino, `/metrics` básico, alerta por Telegram de errores 5xx | §6 | Sonnet | ⬜ | 4.0 |

## Trampas a vigilar

1. Permisos de Android faltantes en `app.config.ts` tras añadir la capacidad nativa de background.
2. Fuga de memoria: tareas de `expo-task-manager` registradas varias veces.
3. Pedido huérfano: driver que pierde la app y deja el pedido en `ACCEPTED` para siempre.
4. Logs que filtran OTP, teléfonos completos o `yape_phone` (`AGENTS.md` §4.5).

## BITÁCORA

<!-- append-only, ≤6 líneas por sesión -->

## GATE DE CIERRE

```bash
pnpm turbo typecheck
pnpm turbo lint
pnpm --filter api test:e2e
```

- [ ] Suite e2e verde y cubriendo: doble aceptación, expiración por olas, OTP bloqueado,
      `DELIVERED` sin pago rechazado, `NO_PAYMENT` sin comisión
- [ ] Tracking sigue emitiendo con la app en segundo plano y la pantalla apagada 10 min
- [ ] Matar la API con un pedido en `ACCEPTED` y reiniciar: el pedido se recupera, no queda huérfano
- [ ] Driver que pierde conexión 3 min sale de `is_available` y su pedido se re-despacha
- [ ] Un error 5xx forzado aparece en Sentry y dispara la alerta de Telegram
- [ ] `grep` de los logs no muestra OTP, teléfonos completos ni `yape_phone`
- [ ] Revisión de diff con `cavecrew-reviewer`; findings resueltos o en Deuda
- [ ] `ESTADO.md` actualizado y commit `chore(semana-5): hardening y observabilidad` propuesto

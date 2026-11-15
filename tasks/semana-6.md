# Semana 6 — Admin mínimo, ficha de tienda y publicación (22 h)

**Hito H5b:** AAB firmado aprobado y publicado en Google Play (pista cerrada → producción), con
runbook de operación entregado al negocio.

**Precondición:** gate de la Semana 5 en verde. Cuenta Google Play Console activa.

## PROMPT DE ARRANQUE

```
/semana 6
```

## TAREAS

| # | Tarea | Plan § | Modelo | Estado | h |
|---|---|---|---|---|---|
| 5.5 | Panel de admin mínimo: aprobación de drivers, saldo de comisiones por driver, incidencias `NO_PAYMENT`, cancelación forzada (CLI o Next.js mínimo) | §2.4 | Sonnet | ⬜ | 6.0 |
| 5.6 | Play Store: ícono adaptativo, splash, 6 capturas, texto de la ficha, política de privacidad publicada, cuestionario de seguridad de datos, declaración de permisos | — | Sonnet + skill | ⬜ | 5.0 |
| 5.7 | Build `production` AAB firmado, carga a pista cerrada, prueba con 10 usuarios reales, corrección de hallazgos | — | Sonnet | ⬜ | 6.0 |
| 5.8 | Backups de DB verificados, runbook de operación, documento de precios y parámetros ajustables, promoción a producción | §7 (E-07) | Sonnet | ⬜ | 5.0 |

## Riesgo principal de la semana

El permiso `ACCESS_BACKGROUND_LOCATION` es la causa más común de rechazo en la revisión de Google.
Si lo rechazan, la app debe seguir operando con tracking en primer plano **sin perder
funcionalidad esencial** (`ALCANCE_TECNICO.md` §10). Verificarlo antes de enviar, no después.

## BITÁCORA

<!-- append-only, ≤6 líneas por sesión -->

## GATE DE CIERRE — Checklist del MVP (`PLAN_DESARROLLO.md` §11)

- [ ] Un cliente nuevo se registra, crea un pedido y lo recibe entregado sin intervención manual
- [ ] El pedido se difunde a drivers reales en menos de 3 s
- [ ] Dos drivers aceptando a la vez: uno gana, el otro recibe `ORDER_ALREADY_TAKEN`
- [ ] Sin drivers en 4 olas: el pedido expira y el cliente es notificado
- [ ] El cliente ve la posición del driver actualizarse al menos cada 5 s
- [ ] Sin el OTP correcto es imposible cerrar el pedido como entregado
- [ ] Un pedido no pasa a `DELIVERED` sin `paymentReceived: true`; el impago queda `FAILED` +
      `NOT_RECEIVED` y sin asiento de comisión
- [ ] El cliente ve el `yape_phone` del driver solo desde `PICKED_UP` y solo si eligió Yape
- [ ] `driver_ledger` cuadra con las comisiones de los pedidos entregados
- [ ] La app sobrevive 60 s sin internet y recupera el estado al reconectar
- [ ] AAB firmado instalable y funcional en un dispositivo limpio
- [ ] Backups automáticos de Supabase verificados con una restauración de prueba
- [ ] Entregables E-01 a E-09 de `ALCANCE_TECNICO.md` §7 completos
- [ ] Credenciales y accesos transferidos al propietario del negocio

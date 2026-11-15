---
name: semana
description: Ejecuta o cierra una semana de desarrollo del proyecto app-delivery según tasks/semana-N.md. Usar para "/semana N" (arrancar o continuar el trabajo de esa semana) y "/semana N gate" (correr el gate de cierre). Mantiene el contexto mínimo leyendo solo tasks/ESTADO.md y el archivo de la semana, nunca PLAN_DESARROLLO.md completo.
---

# Ejecución semanal de app-delivery

Argumento: número de semana (1–6), opcionalmente seguido de `gate`.
Sin argumento: deducir la semana activa de `tasks/ESTADO.md`.

## Modo trabajo — `/semana N`

1. **Leer, en este orden y nada más:** `tasks/ESTADO.md`, `tasks/semana-N.md`. `AGENTS.md` y
   `CLAUDE.md` ya están en contexto. **Prohibido leer `PLAN_DESARROLLO.md` completo.**
2. **Rama de la semana, sin preguntar.** Autorizado por el usuario el 2026-11-14:
   `git checkout -b feat/semana-N` si no existe; si existe, `git checkout feat/semana-N`. Nunca
   trabajar sobre `main`. El merge a `main` ocurre solo con el gate en verde. Los *commits* siguen
   necesitando OK explícito del usuario; crear la rama, no.
3. Confirmar en **una línea**: semana, rama, tarea que se retoma, y bloqueantes que la afectan.
4. **Bloqueantes de credenciales.** Si la tarea activa está `⛔` por una credencial o cuenta que
   falta, no la fuerces y no inventes valores:
   - pide el dato **concreto** con su formato exacto y, si el archivo semanal trae un bloque de
     instrucciones (p. ej. «Cómo obtener `DATABASE_URL`»), reprodúcelo en 3–5 líneas;
   - cuando el usuario lo pegue, escríbelo en `.env` — **nunca** en `.env.example`, nunca en código,
     nunca en la bitácora ni en `ESTADO.md`; en el chat, cita solo el host, con la contraseña
     enmascarada;
   - verifica con un comando (`pnpm --filter api prisma db execute --stdin <<< "select 1"` o
     `prisma validate`) y recién entonces desbloquea la tarea;
   - mientras no llegue, pasa a la siguiente tarea desbloqueada de la tabla y dilo. Si todas las
     pendientes están bloqueadas, repórtalo y detente.
5. Por cada tarea, en el orden sugerido del archivo semanal:
   - leer **solo** las secciones de `PLAN_DESARROLLO.md` que la fila indica en la columna `Plan §`;
   - delegar a un subagente con la plantilla de `CLAUDE.md` §5, con el modelo de la columna
     `Modelo` (`Sonnet + skill` = exigir la skill `frontend-design` en el prompt);
   - marcar la fila `🔄` al empezar;
   - al volver el subagente, **leer el diff real**, no su resumen;
   - ejecutar los comandos de verificación y leer la salida;
   - si falla, reasignar al mismo subagente con las fallas concretas; no parchear a mano;
   - marcar `✅` solo con comandos en verde.
6. Vigilar en cada diff la lista de trampas del archivo semanal y `CLAUDE.md` §6.
7. Al cerrar la sesión (o al quedarte sin tareas desbloqueadas):
   - añadir entrada a la **BITÁCORA** del archivo semanal: fecha, tareas cerradas, salida literal de
     los comandos, desvíos. Máximo 6 líneas;
   - reescribir `tasks/ESTADO.md` (tarea activa, horas consumidas, bloqueantes, deuda);
   - si un desvío contradice `PLAN_DESARROLLO.md`, corregir el plan en el mismo cierre;
   - proponer el commit. **Nunca commitear sin OK explícito del usuario.**

## Modo gate — `/semana N gate`

1. Leer `tasks/semana-N.md` y ejecutar **todos** los comandos del bloque GATE DE CIERRE.
2. Pegar la salida literal en la bitácora. Si no ejecutaste algo, decirlo; no inferir resultados.
3. Marcar cada casilla del gate solo con evidencia. Los puntos que exigen dispositivo físico o
   verificación manual se le piden al usuario explícitamente; no se marcan por cuenta propia.
4. Lanzar `cavecrew-reviewer` sobre el diff de la semana. Findings: resolver, o mover a la sección
   **Deuda** de `ESTADO.md` con justificación.
5. Veredicto en una línea: **VERDE** (semana cerrada, siguiente semana habilitada) o **ROJO** con la
   lista exacta de lo que falta. Gate rojo = no se abre la semana siguiente (`CLAUDE.md` §2).
6. Con el gate en **VERDE**, proponer el merge de `feat/semana-N` a `main` (`--no-ff`) y esperar el
   OK del usuario. Con el gate en **ROJO**, la rama se queda abierta.

## Economía de contexto

- No pegues en el hilo principal archivos completos, diffs enteros ni logs largos. Cita por
  `archivo:línea`.
- Resume cada subagente en 3–6 líneas.
- El archivo semanal y `ESTADO.md` son la memoria del proyecto: si algo importa mañana, va escrito
  ahí, no en el chat.

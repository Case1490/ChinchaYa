# tasks/ — Protocolo de ejecución por semanas

Este directorio es la memoria operativa del proyecto. Existe para que ninguna sesión necesite
leer `PLAN_DESARROLLO.md` completo (66 KB ≈ 18k tokens) y para que el avance quede registrado
fuera del contexto del chat.

## Archivos

| Archivo | Qué es | Quién lo escribe |
|---|---|---|
| `ESTADO.md` | Fuente única de verdad del «dónde estamos». Máximo 40 líneas. | Claude, al cerrar cada sesión |
| `semana-N.md` | Tareas de la semana, bitácora y gate de cierre. | Claude, durante la semana |
| `README.md` | Este protocolo. | Humano |

Los subagentes **nunca** escriben en `tasks/`. Solo el hilo principal.

## Protocolo de sesión

### Al abrir
1. Leer `tasks/ESTADO.md` y `tasks/semana-N.md` de la semana activa. **Nada más.**
2. Confirmar en una línea la tarea en la que se retoma el trabajo.
3. Si hay un bloqueante abierto que afecta la tarea activa, decirlo antes de empezar.

Nunca se lee `PLAN_DESARROLLO.md` entero. Cada tarea del archivo semanal indica sus secciones
(`§2.2`, `§4.4`); se leen solo esas, y solo cuando se va a delegar esa tarea.

### Al delegar
Un subagente por tarea, con la plantilla de `CLAUDE.md` §5. El prompt incluye:
- rutas exactas de archivos a crear o modificar;
- las secciones del plan que debe leer, citadas por número;
- archivos existentes a imitar como patrón;
- criterio de aceptación con comandos concretos.

El subagente devuelve **≤10 líneas**: archivos tocados, salida de los comandos, y lo que no hizo.
Prohibido pegar diffs completos o logs largos en el hilo principal.

### Al cerrar sesión
1. Añadir entrada a la bitácora de `semana-N.md`: fecha, tareas cerradas, salida literal de los
   comandos, desvíos respecto al plan. Máximo 6 líneas.
2. Reescribir `ESTADO.md` (tarea activa, horas, bloqueantes, deuda).
3. Proponer commit. Nunca commitear sin OK explícito del usuario.

## Reglas duras

1. Una tarea no se marca ✅ sin salida literal de comando en la bitácora. «Funciona» no es evidencia.
2. Gate de la semana en rojo = la semana no cierra y no se abre la siguiente (`CLAUDE.md` §2).
3. Todo hallazgo que no se atiende ahora va a la sección **Deuda** de `ESTADO.md`. Nada se pierde
   en el chat.
4. Desvíos respecto a `PLAN_DESARROLLO.md` se anotan en la bitácora **y** se corrige el plan en el
   mismo cierre de sesión. El plan no puede quedar mintiendo.
5. Horas: cada tarea trae su estimación. Si una tarea se pasa más del 50 %, se anota en la bitácora
   y se reporta; no se compensa silenciosamente recortando otra.
6. Verificación en **dispositivo físico** desde la Semana 1. El emulador miente con GPS y permisos.

## Estados de tarea

`⬜` pendiente · `🔄` en curso · `✅` cerrada y verificada · `⛔` bloqueada (indicar por qué) ·
`➖` recortada por decisión explícita (indicar dónde quedó registrada)

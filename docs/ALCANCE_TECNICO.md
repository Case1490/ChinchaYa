# Alcance Técnico del Proyecto

## Aplicación móvil de encargos y delivery punto A → punto B
### Provincia de Chincha, Ica — Perú

---

**Documento:** Alcance Técnico — MVP
**Versión:** 1.1
**Fecha:** 14 de noviembre de 2026
**Preparado por:** Lead Software Architect
**Modalidad:** Producto Mínimo Viable (MVP) para lanzamiento local

> **Cambios de la versión 1.1 (decisión de negocio del 12 de noviembre de 2026).**
> El modelo de cobro queda definido como **pago contra entrega**, en efectivo o por Yape, siempre **directo al repartidor**. No se integra POS ni pasarela de tarjetas, en ninguna fase, porque cobran entre 4 % y 5 % de comisión sobre cada pedido. Como consecuencia: el repartidor cobra el 100 % del pedido y queda debiendo la comisión de la plataforma (ya previsto en R-15 y S-10); y desaparece la carga de comprobante de Yape por parte del cliente, reemplazada por la confirmación del repartidor, que verifica el abono en su propia aplicación de Yape. Secciones afectadas: 2.1 (C-08, C-14), 2.2 (R-01, R-12), 2.4, 3 (F-01, F-14), 4, 5.2, 7 (E-07), 8.2, 10 y 11.

---

## 1. Resumen del producto

Se desarrollará una aplicación móvil para Android que permite a cualquier persona en la provincia de Chincha solicitar que un repartidor recoja algo en un punto de origen y lo entregue en un punto de destino dentro de la misma ciudad.

A diferencia de las aplicaciones de delivery de comida, el producto no depende de un catálogo de comercios: el usuario simplemente **marca dos puntos en el mapa y describe en sus propias palabras qué necesita** — recoger unas llaves olvidadas, llevar un documento, comprar algo en una tienda específica, enviar un paquete a un familiar.

El sistema calcula automáticamente el precio según la distancia real de recorrido, emite la solicitud en tiempo real a los repartidores disponibles más cercanos, permite al cliente seguir el trayecto en vivo sobre el mapa, y cierra la entrega con un código de confirmación de 4 dígitos que solo el cliente conoce.

El cobro es **contra entrega**: el cliente paga al repartidor en el momento de recibir, en efectivo o por Yape al número del propio repartidor. La plataforma no procesa dinero y no usa POS ni pasarela de tarjetas; registra la comisión que el repartidor le debe por cada pedido entregado.

### 1.1 Propuesta de valor

| Para el cliente | Para el repartidor |
|---|---|
| Precio conocido antes de solicitar, sin regateo | Pedidos cercanos, sin desplazamientos en vacío |
| Seguimiento en vivo del repartidor | Ingresos visibles por carrera y por día |
| Confirmación de entrega por código, no por confianza | Ruta guiada con navegación integrada |
| Cobertura local real, con referencias de dirección en lenguaje cotidiano | Control total sobre cuándo estar disponible |

---

## 2. Alcance funcional del MVP

### 2.1 Aplicación del Cliente

| # | Funcionalidad | Descripción |
|---|---|---|
| C-01 | Registro e inicio de sesión | Alta con número de celular, nombre y contraseña. Verificación del número por código. |
| C-02 | Selección de punto de recojo (A) | Búsqueda de dirección con autocompletado, uso de la ubicación actual, o ajuste manual del pin sobre el mapa. |
| C-03 | Selección de punto de entrega (B) | Mismo mecanismo que el punto A, con validación de que ambos puntos estén dentro del área de servicio. |
| C-04 | Campo de referencia | Texto libre adicional por punto («portón azul, al lado de la bodega»), indispensable en zonas con numeración irregular. |
| C-05 | Descripción del encargo | Texto libre de hasta 500 caracteres, con ejemplos sugeridos para orientar al usuario. |
| C-06 | Encargos con compra | Indicación de que el repartidor debe comprar algo, con el monto aproximado a adelantar. |
| C-07 | Cotización previa | Precio desglosado antes de confirmar: tarifa base, kilómetros adicionales y recargos aplicables. |
| C-08 | Elección de método de pago | Efectivo o Yape, en ambos casos **contra entrega** y pagado directamente al repartidor. No hay pago anticipado ni cobro con tarjeta. |
| C-09 | Direcciones guardadas | Lugares frecuentes con etiqueta propia (casa, trabajo, tienda). |
| C-10 | Búsqueda de repartidor en vivo | Pantalla de espera con indicación de cuántos repartidores fueron notificados. |
| C-11 | Seguimiento en tiempo real | Posición del repartidor sobre el mapa, actualizada de forma continua, con tiempo estimado de llegada. |
| C-12 | Datos del repartidor asignado | Nombre, calificación, vehículo, placa y botón de llamada directa. |
| C-13 | Código de entrega | Código de 4 dígitos visible en la app, que el cliente entrega verbalmente al repartidor al momento de recibir. |
| C-14 | Datos de pago del repartidor | Al momento de la entrega, la app muestra el número de Yape del repartidor asignado y el monto exacto a pagar, para yapear en el acto. El cliente no sube ninguna captura. |
| C-15 | Cancelación | Cancelación del pedido con registro del motivo. |
| C-16 | Historial de pedidos | Listado de pedidos anteriores con su detalle y precio final. |
| C-17 | Calificación del repartidor | Puntuación de 1 a 5 con etiquetas rápidas y comentario opcional. |
| C-18 | Notificaciones | Avisos de repartidor asignado, llegada al punto de recojo y entrega completada. |

### 2.2 Aplicación del Repartidor

| # | Funcionalidad | Descripción |
|---|---|---|
| R-01 | Registro como repartidor | Datos personales, DNI, tipo de vehículo, placa y **número de Yape** donde recibirá los pagos de los clientes. |
| R-02 | Carga de documentos | Licencia, SOAT y documento de identidad para habilitación. |
| R-03 | Habilitación manual | El repartidor opera recién cuando la administración valida sus documentos. |
| R-04 | Disponibilidad | Interruptor para ponerse en línea o fuera de línea. |
| R-05 | Recepción de pedidos cercanos | Alerta en tiempo real de pedidos dentro del radio de búsqueda, con sonido y notificación de alta prioridad. |
| R-06 | Detalle de la oferta | Descripción del encargo, distancia al punto de recojo, distancia total, ganancia neta y método de pago. |
| R-07 | Aceptación con tiempo límite | Ventana de 20 segundos para aceptar; si no responde, el pedido pasa al siguiente repartidor. |
| R-08 | Navegación por etapas | Botón contextual único por etapa: en camino al recojo → llegué → recogí → llegué al destino. |
| R-09 | Ruta en Google Maps | Apertura directa de la navegación hacia el punto correspondiente. |
| R-10 | Transmisión de ubicación | Envío automático de posición mientras la carrera está activa. |
| R-11 | Validación por código | Ingreso del código de 4 dígitos del cliente para cerrar la entrega. Sin código correcto, la carrera no se cierra. |
| R-12 | Confirmación de cobro | En la misma pantalla del código, el repartidor confirma que recibió el efectivo o que vio el Yape abonado en su propia aplicación. Sin esa confirmación la entrega no se cierra; si el cliente no paga, el repartidor reporta la incidencia «no me pagaron». |
| R-13 | Reporte de incidencia | Registro de entrega fallida con motivo (nadie en destino, dirección inexistente, cliente no responde, **cliente no pagó**). |
| R-14 | Ganancias | Resumen por día y por periodo: bruto, comisión y neto. |
| R-15 | Saldo de comisiones | Estado de cuenta de las comisiones adeudadas a la plataforma. |
| R-16 | Calificación del cliente | Puntuación recíproca al finalizar. |

### 2.3 Lógica del sistema

| # | Componente | Descripción |
|---|---|---|
| S-01 | Cálculo de tarifa | Tarifa base con kilómetros incluidos, más un valor por kilómetro adicional, más recargos por horario nocturno, tamaño del paquete y gestión de compra. Precio redondeado en múltiplos de S/ 0.50. |
| S-02 | Distancia real de recorrido | Cálculo sobre la red vial mediante el servicio de rutas de Google, no en línea recta. Incluye un método alternativo de respaldo si el servicio externo falla. |
| S-03 | Precio congelado | El precio queda fijado al momento de crear el pedido. Cambios posteriores en el tarifario no afectan pedidos en curso. |
| S-04 | Área de servicio | Delimitación geográfica de la provincia de Chincha. Pedidos con origen o destino fuera del área se rechazan automáticamente. |
| S-05 | Asignación por proximidad | Búsqueda de repartidores disponibles por radio geográfico real, ordenados por distancia al punto de recojo. |
| S-06 | Asignación por olas progresivas | Primera ola a 4 km; si nadie acepta, el radio se amplía progresivamente hasta 10 km en un máximo de 4 olas. El pedido expira a los 5 minutos sin repartidor. |
| S-07 | Prevención de doble asignación | Garantía a nivel de base de datos de que dos repartidores no pueden tomar el mismo pedido. |
| S-08 | Auditoría del ciclo de vida | Registro inmutable de cada cambio de estado del pedido, con fecha, responsable y coordenadas. |
| S-09 | Historial de recorrido | Muestreo periódico de la ruta recorrida, conservado 30 días para resolución de disputas. |
| S-10 | Comisiones | El repartidor cobra el 100 % del pedido al cliente. El sistema calcula automáticamente la comisión de la plataforma por carrera entregada y la registra como deuda en el estado de cuenta del repartidor, que la liquida al negocio por fuera de la app. La plataforma no retiene ni transfiere dinero. |
| S-11 | Tarifario configurable | Tarifa base, costo por kilómetro, recargos, comisión y radio de búsqueda ajustables sin necesidad de actualizar la aplicación. |

### 2.4 Administración (en el MVP)

La administración del MVP se realiza directamente sobre la base de datos mediante la consola de administración del proveedor (Supabase Studio) y consultas preparadas. Esto cubre: habilitación de repartidores, ajuste del tarifario, control del saldo de comisiones por repartidor, resolución de incidencias de pago y consulta de métricas operativas.

> **Nota:** un panel de administración web con interfaz propia está contemplado como **Fase 2**, posterior al MVP. Ver §4.

---

## 3. Fuera de alcance del MVP

Lo siguiente **no** forma parte de esta entrega. Se documenta explícitamente para evitar expectativas incorrectas.

| # | Excluido | Motivo / Fase sugerida |
|---|---|---|
| F-01 | Pasarela de pago en línea y POS (tarjeta, Niubiz, Culqi, Izipay) | **Descartado por decisión de negocio, en todas las fases.** Cobran entre 4 % y 5 % de comisión por operación, lo que se come el margen del servicio. El modelo es pago contra entrega: efectivo o Yape directo al repartidor. |
| F-02 | Integración con la API oficial de Yape o billeteras | No existe una API pública abierta para comercios pequeños. El repartidor verifica el abono en su propia aplicación de Yape y lo confirma en la app. |
| F-03 | Versión para iOS | El MVP se publica solo en Google Play. iOS requiere cuenta de desarrollador Apple ($99/año) y un ciclo de revisión propio. Fase 2. |
| F-04 | Panel de administración web con interfaz propia | Fase 2. |
| F-05 | Chat en vivo entre cliente y repartidor | Se resuelve con llamada telefónica directa en el MVP. Fase 2. |
| F-06 | Facturación electrónica / boletas SUNAT | Fase 3, sujeto a definición del modelo tributario del negocio. |
| F-07 | Programación de pedidos a futuro | Fase 2. |
| F-08 | Múltiples destinos en un mismo pedido | Fase 2. |
| F-09 | Cupones, promociones y códigos de referido | Fase 2. |
| F-10 | Firma digital o foto de prueba de entrega | El MVP valida con código de 4 dígitos. Fase 2. |
| F-11 | Asignación automática con algoritmo de optimización de flota | El MVP usa difusión por olas, adecuado para el volumen inicial. |
| F-12 | Soporte multi-ciudad | La arquitectura ya contempla áreas de servicio múltiples, pero el MVP se configura y prueba solo para Chincha. |
| F-13 | Seguro de encargos / cobertura de pérdidas | Decisión de negocio, no técnica. |
| F-14 | Carga de comprobante de Yape por el cliente | Innecesaria: el pago llega al Yape del repartidor, que lo verifica en su propia aplicación antes de confirmar la entrega. La plataforma no es parte del movimiento de dinero. |
| F-15 | Billetera y liquidación de pagos a repartidores (payouts) | El repartidor cobra directo del cliente; no hay dinero en tránsito que liquidar. Solo se registra la comisión que debe a la plataforma. |

---

## 4. Fases del producto

| Fase | Contenido | Estado |
|---|---|---|
| **Fase 1 — MVP** | Todo el alcance de la sección 2. Publicación en Google Play. | **Objeto de este documento** |
| Fase 2 — Crecimiento | Versión iOS, panel de administración web, chat interno, pedidos programados, cupones. *(Sin pasarela de pago: el modelo contra entrega es definitivo.)* | Posterior |
| Fase 3 — Escala | Facturación electrónica, expansión a otras provincias, optimización de asignación, programa de fidelización. | Posterior |

---

## 5. Arquitectura y tecnologías

### 5.1 Componentes del sistema

| Componente | Tecnología | Función |
|---|---|---|
| Aplicación móvil | React Native con Expo | Una sola aplicación que alterna entre modo Cliente y modo Repartidor según el rol de la cuenta. |
| Interfaz de usuario | NativeWind (Tailwind CSS) | Diseño propio, legible a plena luz del día y operable con una mano. |
| Servidor de aplicación | Node.js con NestJS | Reglas de negocio, cálculo de tarifas, asignación de pedidos, seguridad. |
| Comunicación en tiempo real | WebSockets (Socket.io) | Difusión de pedidos y seguimiento de ubicación con latencia de segundos. |
| Base de datos | PostgreSQL con extensión PostGIS | Almacenamiento y consultas geográficas de proximidad de alto rendimiento. |
| Infraestructura de datos | Supabase | Base de datos gestionada, respaldos automáticos y almacenamiento de archivos. |
| Servidor | DigitalOcean | Servidor dedicado para el servicio en tiempo real, con certificado HTTPS. |
| Mapas y direcciones | Google Maps Platform | Autocompletado de direcciones, geocodificación y cálculo de rutas reales. |
| Notificaciones | Expo Push Notifications | Alertas de alta prioridad a los repartidores. |
| Distribución | Google Play Console | Publicación del instalador firmado. |

### 5.2 Decisiones técnicas relevantes para el negocio

1. **Una sola aplicación para ambos roles.** Reduce a la mitad el costo de mantenimiento, publicación y actualizaciones. Un repartidor puede además solicitar encargos como cliente con la misma cuenta.
2. **Base de datos geográfica real (PostGIS).** La búsqueda de repartidores cercanos se resuelve con índices geoespaciales, no recorriendo la lista completa de repartidores. Esto mantiene la velocidad de asignación constante aunque la flota crezca de 10 a 500 repartidores.
3. **Precio basado en distancia de recorrido, no en línea recta.** Evita el reclamo más común en este tipo de servicios: que el precio no refleje el trayecto real.
4. **Tarifario configurable sin actualizar la app.** La tarifa base, el costo por kilómetro, los recargos y la comisión se ajustan en el servidor y toman efecto de inmediato en todos los dispositivos.
5. **Validación de entrega por código.** Elimina la disputa «yo sí entregué» / «a mí no me llegó» sin necesidad de fotografías ni firmas.
6. **Auditoría completa del pedido.** Cada cambio de estado se registra con hora y coordenadas, lo que permite resolver cualquier reclamo con evidencia.
7. **Pago contra entrega, sin intermediarios.** El dinero va del cliente al repartidor, en efectivo o por Yape. La plataforma no procesa pagos: no paga comisión de pasarela (4–5 % por operación con POS o tarjeta), no necesita cuenta de comercio, no mantiene saldos de terceros y no asume el riesgo de contracargos. El ingreso del negocio es la comisión por pedido entregado, que queda registrada como deuda del repartidor y se cobra por fuera de la app.

---

## 6. Cronograma estimado

Dedicación prevista: **22 horas semanales**, un desarrollador.

| Etapa | Periodo | Entregable verificable |
|---|---|---|
| Preparación de cuentas e infraestructura | Previo, 3 h | Servidor, base de datos y cuentas de servicio operativas. |
| **Semana 1** — Fundamentos | 22 h | La app instala en un teléfono real; un usuario se registra e inicia sesión; el servidor responde por HTTPS. |
| **Semana 2** — Mapas y tarifas | 22 h | El cliente selecciona A y B, describe el encargo, ve el precio desglosado y genera el pedido. |
| **Semana 3** — Tiempo real | 22 h | Flujo completo entre dos teléfonos: el pedido llega al repartidor, lo acepta, el cliente lo sigue en el mapa y la entrega se cierra con código. |
| **Semana 4** — Cobro y cierre funcional | 22 h | Cobro contra entrega confirmado por el repartidor, calificaciones, notificaciones, historial y ganancias. Instalador interno distribuible. |
| **Semanas 5–6** — Estabilización y publicación | 44 h | Ubicación en segundo plano, pruebas automatizadas, monitoreo de errores, ficha de tienda y publicación en Google Play. |
| **Total** | **~135 horas / 6 semanas** | Aplicación publicada y operativa. |

### 6.1 Observación sobre el plazo

El alcance de la sección 2, incluyendo publicación en Google Play, corresponde realísticamente a **6 semanas** con la dedicación indicada. Al cierre de la **Semana 4** se obtiene una aplicación **funcional y demostrable de punta a punta**, apta para pruebas con repartidores reales mediante instalación directa, pero aún no publicada en la tienda.

Si el plazo de 4 semanas fuese inamovible, se dispone de un plan de reducción de alcance acordado por prioridad, que preserva íntegramente el núcleo del servicio: autenticación, área de servicio, cálculo de tarifa, asignación de pedidos, seguimiento en vivo y validación de entrega por código.

---

## 7. Entregables

| # | Entregable |
|---|---|
| E-01 | Código fuente completo en un repositorio Git, organizado como monorepo. |
| E-02 | Aplicación Android firmada (`.aab`) publicada en Google Play, y `.apk` para instalación directa. |
| E-03 | Servidor de aplicación desplegado, con dominio propio y certificado HTTPS. |
| E-04 | Base de datos en producción con respaldos automáticos activos. |
| E-05 | Documento de arquitectura técnica (`PLAN_DESARROLLO.md`). |
| E-06 | Documentación de la interfaz de programación (endpoints y eventos en tiempo real). |
| E-07 | Guía de operación: cómo habilitar repartidores, ajustar tarifas, controlar el saldo de comisiones y resolver incidencias de pago. |
| E-08 | Credenciales y accesos de todos los servicios, transferidos al propietario del negocio. |
| E-09 | Ficha de tienda completa: ícono, capturas, descripción y política de privacidad publicada. |

---

## 8. Costos de operación

### 8.1 Costos de puesta en marcha (pago único)

| Concepto | Monto referencial (USD) |
|---|---|
| Cuenta de desarrollador de Google Play | 25.00 |
| Dominio propio (primer año) | 12.00 |
| **Total inicial** | **≈ 37.00** |

### 8.2 Costos recurrentes mensuales estimados

| Servicio | Plan | Monto referencial (USD/mes) |
|---|---|---|
| Servidor de aplicación (DigitalOcean) | Droplet básico | 6.00 |
| Base de datos y almacenamiento (Supabase) | Plan gratuito al inicio; plan Pro al crecer | 0.00 → 25.00 |
| Google Maps Platform | Según volumen; con caché y restricciones aplicadas | 0.00 → 30.00 |
| Notificaciones push | Incluido en el servicio de Expo | 0.00 |
| Compilación de instaladores (EAS) | Plan gratuito suficiente para el MVP | 0.00 |
| Comisión de procesamiento de pagos | No aplica: pago contra entrega, sin POS ni tarjetas | 0.00 |
| **Total estimado de arranque** | | **≈ 6.00 – 10.00** |
| **Total estimado con operación activa** | | **≈ 40.00 – 70.00** |

> **Efecto del cobro contra entrega.** Al no usar POS ni pasarela se evita la comisión de 4 % a 5 % por operación. Con un ticket promedio de S/ 8.00 y 300 pedidos al mes (S/ 2,400 transados), eso equivale a entre **S/ 96 y S/ 120 mensuales** que el negocio no paga, y que además escalan con el volumen, a diferencia de los costos de infraestructura de la tabla anterior.

> Los montos son referenciales a la fecha de este documento y están sujetos a los precios vigentes de cada proveedor. El consumo de Google Maps Platform es el único rubro que escala con el volumen de pedidos; la arquitectura aplica caché de rutas, restricción geográfica de búsquedas y un método de cálculo alternativo para contenerlo. Se recomienda configurar una alerta de presupuesto en Google Cloud desde el primer día.

---

## 9. Supuestos

| # | Supuesto |
|---|---|
| A-01 | El propietario del negocio provee y financia las cuentas de Google Cloud, Google Play, Supabase, DigitalOcean y el dominio. |
| A-02 | Los repartidores cuentan con un teléfono Android 9 o superior, con GPS funcional y plan de datos móviles. |
| A-03 | La captación, habilitación y gestión de los repartidores es responsabilidad del negocio, no del desarrollo. |
| A-04 | Los valores iniciales del tarifario (tarifa base, costo por kilómetro, recargos y comisión) son una decisión comercial del propietario; el desarrollo entrega el sistema que los aplica y permite ajustarlos. |
| A-05 | El contenido legal (términos y condiciones, política de privacidad) será validado por el propietario; el desarrollo entrega una versión base y la publica. |
| A-06 | El servicio de atención al usuario y la resolución de incidencias operativas quedan a cargo del negocio. |
| A-07 | La verificación del número de celular por SMS requiere contratar un proveedor de mensajería si se desea en producción; en el MVP se entrega el mecanismo listo para conectarlo. |
| A-08 | Cada repartidor tiene una cuenta de Yape activa a su nombre y declara ese número al registrarse. Si no la tiene, solo puede recibir pedidos en efectivo. |
| A-09 | El cobro de la comisión acumulada al repartidor (el saldo de R-15) es un proceso del negocio fuera de la app: la plataforma lo calcula y lo muestra, pero no lo cobra ni lo descuenta automáticamente. |

---

## 10. Riesgos identificados y mitigación

| Riesgo | Impacto | Mitigación aplicada |
|---|---|---|
| Direcciones de Chincha incompletas o mal indexadas en los mapas de Google | Alto | Campo obligatorio de referencia en texto libre y ajuste manual del pin sobre el mapa. El sistema no depende únicamente del autocompletado. |
| Conexión móvil intermitente de los repartidores | Alto | Carga de datos mínima en el canal en tiempo real, reconexión automática, respaldo por notificación push y recuperación del estado del pedido al reconectar. |
| Rechazo del permiso de ubicación en segundo plano por parte de Google Play | Medio | Justificación documentada con servicio en primer plano visible. Si se rechaza, la aplicación opera con seguimiento en primer plano sin pérdida de funcionalidad esencial. |
| Consumo de Google Maps por encima del nivel gratuito | Medio | Caché de rutas, restricción geográfica, método de cálculo alternativo y alerta de presupuesto. |
| Cliente que no paga al momento de la entrega | Medio | El repartidor confirma el cobro en la misma pantalla del código de entrega; sin esa confirmación el pedido no se cierra como entregado, queda registrado como incidencia «no me pagaron» con hora y coordenadas, y el cliente reincidente se suspende desde la administración. |
| Comprobante de Yape falsificado | Bajo | Riesgo prácticamente eliminado: el repartidor verifica el abono en su propia aplicación de Yape antes de entregar. No se aceptan capturas de pantalla como prueba de pago. |
| Repartidor que cobra y no reporta la comisión | Medio | La comisión se calcula y asienta automáticamente al entregar, no la declara el repartidor. El saldo adeudado es visible para el negocio y para el repartidor, y el incumplimiento se sanciona con suspensión de la cuenta. |
| Dos repartidores aceptando el mismo pedido | Alto | Resolución a nivel de base de datos mediante bloqueo de fila; técnicamente imposible la doble asignación. |
| Baja densidad inicial de repartidores | Alto | Ampliación progresiva del radio de búsqueda hasta 10 km y notificación de alta prioridad. Es un riesgo de negocio, no técnico. |

---

## 11. Criterios de aceptación

El MVP se considera entregado cuando se verifican, en dispositivos físicos reales, los siguientes puntos:

1. Un cliente nuevo se registra, crea un pedido y lo recibe entregado sin intervención manual del equipo técnico.
2. El pedido se difunde a los repartidores disponibles en menos de 3 segundos desde su creación.
3. Con dos repartidores aceptando simultáneamente, uno obtiene el pedido y el otro recibe un aviso claro de que ya fue tomado.
4. Si ningún repartidor acepta tras las cuatro olas de búsqueda, el pedido expira y el cliente es notificado.
5. El cliente observa la posición del repartidor actualizarse al menos cada 5 segundos durante la carrera.
6. Sin el código de 4 dígitos correcto es imposible marcar el pedido como entregado.
7. Un pedido no puede cerrarse como entregado sin que el repartidor confirme que recibió el pago; si declara que no lo recibió, el pedido queda registrado como incidencia de pago y no como entrega normal.
8. El estado de cuenta de comisiones del repartidor coincide exactamente con las comisiones de los pedidos entregados.
9. La aplicación tolera la pérdida de conexión durante 60 segundos y recupera el estado del pedido al reconectar.
10. El instalador firmado funciona en un dispositivo Android limpio, sin herramientas de desarrollo.

---

*Documento preparado para revisión y aprobación del alcance. Cualquier funcionalidad no listada en la sección 2 se considera fuera del alcance de la Fase 1 y será cotizada por separado.*

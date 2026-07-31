# Auditoría del Flujo de Checkout — Administración y Experiencia de Comprador

NOTA: esta es una fotografía histórica del 30 de julio de 2026. Los dos bugs críticos documentados aquí (métodos de pago sin protección contra quedar en cero activos, y campo `type` corrupto en payphone/paypal/bank_transfer) fueron corregidos el mismo día en una sesión posterior — ver commits `db0d968`, `f0a7a44`, `f690c6a` y `554656b`. El resto de hallazgos (reporte de abandono, actualización en vivo de estado, fricciones de UI, código muerto) seguían pendientes al momento de esta nota; consultar el historial de commits para su estado real. No usar como estado actual del código.

Auditoría realizada el 2026-07-30 sobre la gestión admin de órdenes/pagos (`/admin/orders`, `/admin/payment-methods`, `/admin/shipping`, `/admin/checkout`) y el flujo completo de compra (`/producto/:slug` → carrito → `/checkout` → confirmación → `/dashboard/orders`), con lectura completa de código en ambos repos y prueba real en navegador (sesión única que actúa como admin y como comprador, mismo patrón que auditorías anteriores de este proyecto). Diagnóstico únicamente — no se modificó ningún archivo de código.

Metodología: (1) mapeo de código mediante 6 sub-agentes de exploración en paralelo (backend: admin de órdenes, métodos de pago, checkout/validación, cupones/envío/carrito abandonado/email; frontend: pantallas admin de checkout, flujo de comprador), cada uno citando `archivo:línea`; (2) verificación cruzada en vivo de los hallazgos más importantes (no se tomó ningún hallazgo de agente como definitivo sin confirmarlo yo mismo cuando fue posible); (3) dos compras reales de punta a punta con transferencia bancaria (`#ORD-MS80K5HL-17A97A` y `#ORD-MS80QFXM-62057C`, ambas canceladas al terminar, documentadas como prueba); (4) 3 acciones administrativas reales sobre la primera orden (nota interna, cambio de estado a Pagado, cancelación); (5) inspección de red y de la API admin directamente vía `fetch()` en consola para contrastar lo que la UI muestra contra lo que el backend realmente devuelve.

---

## PARTE A — Mapeo del lado administrador

### A.1 Pantallas relacionadas con checkout en admin

| Pantalla | Ruta | Qué hace / qué datos maneja |
|---|---|---|
| Gestión de Órdenes | `/admin/orders` | Listado con filtros (estado, estado de pago, rango de fechas, búsqueda libre), acciones inline por fila (cambio de estado, cancelar, reversar Payphone, reembolsar PayPal, marcar listo para retiro), y detalle en modal. |
| Detalle de orden | `/admin/orders/:id` (modal `OrderDetailsModal.tsx`) | 4 tabs: Detalles (items, cliente, facturación, resumen financiero, **timeline operativo real** con `statusHistory`), Dirección (envío o datos de sucursal si es pickup), Despacho (courier externo vs. envío propio con PIN, transportista, tracking), Notas (notas internas). |
| Validación de entrega | `/admin/delivery-validation?orderId=:id` | Confirmar entrega interna con PIN de 6 dígitos, o entrega externa con datos de courier/tracking. |
| Entregas pendientes | `/admin/deliveries` | Lista de órdenes con acción de entrega pendiente (enviadas / listas para retiro); solo enrutamiento hacia validación. |
| Pagos → Métodos de Pago | `/admin/payment-methods?tab=methods` | CRUD de métodos de pago, activar/desactivar, reordenar, configuración de montos mín/máx y comisiones. |
| Pagos → Transferencias | `/admin/payment-methods?tab=transfers` | Conciliación real de transferencia bancaria: lista con polling cada 30s, validar/rechazar comprobante. |
| Pagos → Cuentas Bancarias | `/admin/payment-methods?tab=accounts` | CRUD de cuentas bancarias destino para transferencias, cuenta predeterminada. |
| Envíos | `/admin/shipping` | 4 tabs: Transportistas, Sucursales (con soporte de pickup, horarios), Zonas de Envío, Tarifas; más configuración global de umbral de envío gratis. |
| Checkout (reglas) | `/admin/checkout` | Hub de reglas globales: IVA, monto mínimo de pedido, habilitar/deshabilitar retiro en tienda (`pickup_enabled`), e Historial de cambios de configuración (real, no placeholder). Envío/Pagos/Descuentos son solo enlaces a sus propios módulos — el texto en pantalla dice explícitamente "Checkout ya no administra comisiones ni costos de cobro". |
| Cupones | `/admin/discounts/coupons` | CRUD de cupones (%, $, envío gratis; alcance global/categoría/producto/usuario), notificación por email a usuarios al crear. |
| Reportes → Ventas | `/admin/reports/sales` | Ventas diarias y **ventas por método de pago**, con datos reales de BD (no placeholder). |
| Reportes (hub) | `/admin/reports` | KPIs generales, demanda de búsqueda sin resultados, abuso de cupones, inteligencia comercial — todo con queries SQL reales. |

### A.2 Acciones reales probadas sobre una orden de prueba

Se creó una orden real de prueba comprando el producto más barato del catálogo (`dfsfdsdfsdsffddfsfds`, $11.50 + $6.00 envío + $2.40 IVA hasta $18.40 total, transferencia bancaria) → **`#ORD-MS80K5HL-17A97A`**. Sobre ella se ejecutaron 3 acciones reales desde `/admin/orders`:

1. **Agregar nota interna** (tab Notas del modal de detalle): `[PRUEBA] Orden de prueba creada durante auditoria de checkout. No es una compra real.` → toast "Nota agregada exitosamente", nota visible inmediatamente con timestamp.
2. **Cambiar estado a "Pagado"** vía el `<select>` inline de la fila: el dropdown solo ofreció las transiciones válidas desde `pending_transfer` (`Transferencia pendiente`, `Pago verificado`, `Pagado`) — confirma en vivo que `VALID_TRANSITIONS` (`order-status.validator.ts:18-29`) efectivamente restringe las opciones que ve el admin, no solo el backend. Aplicado sin error.
3. **Cancelar la orden**: el botón de cancelar abrió un modal que **exige un motivo obligatorio** antes de habilitar "Cancelar orden" — buen guardrail. Al confirmar: toast "Orden cancelada exitosamente" + una notificación en tiempo real separada ("Pedido cancelado: Tu pedido #ORD-MS80K5HL-17A97A ha sido cancelado. Motivo: ...") dirigida al mismo usuario (en este entorno de prueba el admin y el comprador son la misma cuenta), confirmando que el sistema de notificación al cliente se dispara de verdad, no solo cambia una columna en la base de datos.

Timeline operativo verificado también sobre una orden **real y ya entregada** (`#ORD-MQEJFQXR-69E1D0`) vía `GET /api/v1/admin/orders/:id`: el campo `statusHistory` trae registros completos con `previousStatus`/`newStatus`/`changedBy`/`reason`/`metadata` (incluye IP, user agent, método de confirmación) — y la UI lo renderiza correctamente como "Timeline operativo" ("Enviado → Entregado", "Procesado → Enviado") con motivo y hora. (Nota: en la orden de prueba antes de tocarla, `statusHistory` venía como `[]` — no porque el feature esté roto, sino porque el historial solo se escribe en transiciones *admin-driven*, y esa orden nunca había tenido ninguna hasta que yo mismo generé la primera con la acción #2 de arriba.)

### A.3 Métodos de pago

Configurados actualmente en este entorno: **Payphone, PayPal, Transferencia bancaria** — los 3 activos. No hay "Efectivo contra entrega" ni "Tarjeta de crédito/débito" configurados (existen como `type` soportado por el modelo, pero no hay fila creada para ellos en este entorno).

Cómo se activan/desactivan: `PATCH /admin/payment-methods/:id/toggle` (botón toggle en la lista) o `PUT /admin/payment-methods/:id` con `isActive` en el body (desde el formulario de edición) — **dos vías independientes**.

**Hallazgo crítico confirmado en código:** ninguna de las dos vías valida que quede al menos un método de pago activo en el sistema. `toggleStatus` (`admin-payment-methods.service.ts:258-272`) solo verifica, por método individual, que no haya órdenes/transacciones pendientes usando *ese* método específico (`assertMethodNotInActiveUse`, líneas 112-142) — no hay ningún chequeo global de "¿cuántos métodos quedan activos en total?". Peor aún, `updatePaymentMethod` (líneas 217-253) ni siquiera llama a esa verificación por-método: puede desactivar un método que sí tiene órdenes pendientes. Un admin puede, método por método (esperando a que cada uno quede sin actividad pendiente, algo trivial de lograr con paciencia), dejar la tienda sin ningún método de pago activo — el checkout del comprador simplemente mostraría una lista vacía en el paso de Pago, sin ninguna alerta preventiva al admin.

Verificación adicional en vivo: la columna "TIPO" en `/admin/payment-methods` muestra **"Efectivo" para los 3 métodos** (Payphone, PayPal y Transferencia bancaria), confirmado también contra la API (`GET /admin/payment-methods` → los 3 registros tienen `type: "cash"` en la base de datos, cuando deberían tener `payphone`/`paypal`/`transfer` según el comentario del propio modelo, `paymentMethod.model.ts:8`: *"'transfer', 'cash', 'paypal', 'credit_card', 'debit_card'"*). Verifiqué que esto **no** rompe la protección de "método en uso" (esa consulta usa `paymentMethodId` por UUID, no el campo `type`), así que es un bug de datos aislado y visual, pero engañoso: un admin que mire esa columna pensaría que Payphone y PayPal son pagos "en efectivo".

PayPal y Payphone verifican el pago **server-side siempre** (nunca confían en un estado que mande el navegador): capturan/confirman contra la API real de la pasarela, comparan el monto capturado contra `order.total` con tolerancia de 0.01, y usan locks pesimistas (`FOR UPDATE`) al marcar una orden como pagada (`paypal-payment.processor.ts:68-86,399-454`; `payphone-payment.processor.ts:379-511`).

### A.4 Conciliación de pago — transferencia bancaria

Flujo confirmado en código y en vivo (mismo test de la orden de prueba):

1. El comprador sube un comprobante real (archivo, no URL — `transfer-management.processor.ts:372-374` rechaza explícitamente URLs externas) vía `POST /payments/transfers/:orderId/submit`, límite de **3 intentos** por orden.
2. El admin ve la lista en `/admin/payment-methods?tab=transfers` (con **polling cada 30s**, `TransfersList.tsx:55`) y valida o rechaza desde `TransferValidationDialog.tsx`.
3. **Validar** (`PATCH /admin/payments/transfers/:transferId/validate`) usa locking pesimista explícito sobre `Order` y `OrderTransferDetail` dentro de una transacción — comentario en código: *"CRÍTICO: Validar transferencia con locking pesimista"* (`transfer-validation.service.ts:263-275`) — específicamente para que dos admins no puedan validar la misma transferencia dos veces en simultáneo. Exige que el **monto coincida exactamente** con el total de la orden antes de aceptar. Al validar: descuenta stock definitivamente, marca la orden `PAID`, y **sí registra en `OrderStatusHistory`**.
4. **Rechazar**: libera el stock reservado; si es el 3er intento fallido, cancela la orden automáticamente; si no, la reabre para que el comprador reintente.
5. **Timeout automático**: `order-payment-expiry.scheduler.ts` corre cada 15 min y cancela transferencias sin actividad por 30 min (usa `updatedAt`, no `createdAt`, para no penalizar reintentos recientes) — libera cupón, promoción y stock reservados. Única salvedad: este flujo automático **no** escribe en `OrderStatusHistory` (solo deja una nota de sistema en texto libre), a diferencia de todas las acciones admin-driven.

Existe además un endpoint separado y más laxo: `PATCH /orders/:id/confirm-payment` (confirmación manual sin exigir comprobante ni verificar monto, restringido a `bank_transfer`/`cash`) — pero **no tiene ningún botón que lo invoke** en `AdminOrders.tsx` ni `OrderDetailsModal.tsx` (búsqueda exhaustiva sin resultados). Es decir, la vía real de conciliación que usa el admin en pantalla es la de arriba (`transfers/:id/validate`, con evidencia y verificación de monto); este segundo endpoint es un camino "de emergencia" que hoy es inalcanzable desde la UI.

### A.5 Reportes / analítica de checkout

`/admin/reports/sales` — ventas diarias y ventas por método de pago **con datos reales de BD**, sin el patrón de placeholder detectado la semana pasada en popups (`"Visual Charts Placeholder"`, exclusivo de `PopupStats.tsx` — no aparece en ningún archivo de reportes de checkout/órdenes, confirmado por búsqueda exhaustiva en ambos repos). Los componentes manejan explícitamente el caso "sin datos" con un mensaje textual en vez de un gráfico decorativo vacío.

**Falta un reporte de "tasa de abandono de checkout"**, pese a que el dato ya se recolecta (ver Parte C.2) — sería trivial de construir sobre la tabla `checkout_sessions` existente, pero hoy no hay ningún endpoint ni pantalla que lo calcule.

---

## PARTE B — Mapeo del flujo de comprador

### B.1 Producto → carrito

Ambos botones existen y confirmados en vivo: **"Comprar ahora"** agrega el ítem al carrito sin abrir el drawer y navega directo a `/checkout` (o a `/login` con retorno a checkout si no hay sesión); **"Agregar al carrito"** agrega y muestra feedback in-place (botón cambia a "✓ Agregado al carrito", contador del header sube de 4→5 en la prueba real) sin navegar. Diferencia real de UX, no solo cosmética.

### B.2 Carrito

Edición de cantidad confirmada **no optimista**: en la prueba real, incrementar cantidad de 1→2 tardó ~1 segundo en reflejarse (llamada real al backend, sin debounce ni actualización inmediata) — clics rápidos consecutivos pueden disparar múltiples llamadas concurrentes sin protección visible (botones no se deshabilitan durante la carga). Eliminar producto (ícono de basura) funciona correctamente y confirmado en vivo.

Cupón: el drawer del carrito **solo muestra un indicador informativo** ("tienes cupones disponibles"), sin campo para aplicar — la aplicación real ocurre en el sidebar del paso de checkout (`CouponSelector.tsx`), con validación contra backend al presionar "Aplicar" (no mientras se escribe).

### B.3 Checkout paso a paso

**a) Facturación** — confirmado en vivo: banner "Datos cargados desde tu perfil", nombre/apellido/email/teléfono precargados, **2 identificaciones de facturación guardadas** ofrecidas como radio buttons (una marcada "Predeterminado") más opción de usar una nueva. No pide de más — los campos son razonables para facturación en Ecuador (cédula/RUC, razón social solo si RUC).

**b) Entrega** — confirmado en vivo: **"Recoger en tienda" sigue apareciendo deshabilitado** ("Esta opción está desactivada por ahora."), consistente con el hallazgo de la auditoría de content-pages. Confirmado en código que **no es código sin terminar**: el componente completo (selección de sucursal, fecha, franja horaria) existe y funciona, pero está gateado por el flag `pickup_enabled` (`useCheckoutPage.ts:129-130`), que viene en `false` por defecto (`SettingsContext.tsx:140`) y solo un admin puede activarlo desde `/admin/checkout`. Costo de envío calculado en tiempo real contra transportista+zona ($6.00 con Servientrega auto-seleccionado en la prueba, recalculado tras elegir dirección — no es un valor fijo).

**c) Pago** — confirmado en vivo: los 3 métodos configurados en A.3 (Payphone, PayPal, Transferencia bancaria) aparecen exactamente igual en el carrusel de selección del checkout, cargados dinámicamente desde el backend (no hardcodeados en frontend). Se llegó hasta el modal de confirmación con transferencia bancaria: al pulsar "Completar pedido" aparece primero un modal **"Confirmar pedido"** (con resumen y botón "Crear orden y continuar") **antes** de crear la orden — solo al confirmar ahí se dispara la creación real y aparece el segundo modal ("Completa tu transferencia", con cuentas bancarias y botones "Ya hice la transferencia"/"Lo haré después"). No existe "efectivo contra entrega" en este entorno para probarlo (no está configurado, ver A.3).

### B.4 Confirmación de pedido

`OrderConfirmation.tsx` muestra: stepper de progreso, tarjeta de PIN de entrega o tracking, instrucciones de transferencia si sigue pendiente, resumen completo de precio, y un aviso **"Confirmación enviada a [email]"** condicionado únicamente a que `order.billingEmail` exista — es decir, es un mensaje optimista de la UI, no una confirmación real de que el correo se envió. Verificado en backend que sí hay disparo automático de email al confirmarse el pago (vía `notificationService.notifyOrderConfirmed`, llamado desde los processors de PayPal y Payphone tras captura exitosa, y desde la creación de la orden para transferencia con un email distinto de "pendiente de comprobante"), con proveedor real (Brevo, no mock). Único matiz: la "plantilla" (`template: 'order-confirmed'`) es un identificador decorativo — el HTML real es el mismo layout genérico reutilizado para todas las notificaciones, con `title`/`message` distintos, no una plantilla de email de orden dedicada.

### B.5 Historial de órdenes del comprador

`/dashboard/orders` ("Mis pedidos") confirmado en vivo: lista completa con filtros por estado, stepper visual de progreso por orden (Pendiente → Procesando → Enviado → Entregado, con checks verdes en los pasos ya alcanzados), y tracking cuando aplica. **No hay polling ni WebSocket** — confirmado en código (`useOrderConfirmation.ts`, `MyOrdersPage.tsx`: ambos cargan una sola vez en `useEffect` al montar) — el comprador debe recargar la página para ver un cambio de estado hecho por el admin.

---

## PARTE C — Casos borde y validaciones

**C.1 — Stock agotado durante checkout.** Confirmado en código, doble revalidación con locks reales: el stock se revisa una vez al construir el snapshot de items (`order-checkout.service.ts:293-343`, con `SELECT ... FOR UPDATE` cuando es la confirmación real) y otra vez inmediatamente antes de decrementar (`order-inventory.service.ts:120-125,176-181`), ambas dentro de la misma transacción atómica. Si el stock se agotó entre que el usuario entró a checkout y confirmó, la orden falla con `AppError` 400 y todo se revierte (`rollback`) — no queda orden huérfana ni decremento fantasma. No se forzó este escenario en vivo (requeriría dos sesiones concurrentes compitiendo por el último ítem), pero la evidencia de código es sólida y específica (línea por línea).

**C.2 — Carrito abandonado.** **Contradice una sospecha previa de este proyecto ("código de abandono sin conectar"): sí está conectado end-to-end.** El scheduler arranca en el boot del servidor (`server.ts:90`), corre cada 2 horas, y el registro de sesión (`checkout_sessions`) se alimenta desde llamadas reales del frontend: confirmado en la captura de red durante mi propia prueba de checkout — `PUT /orders/checkout-session` se disparó automáticamente al entrar al flujo, sin que yo hiciera nada explícito para ello. El email de abandono usa el mismo proveedor real (Brevo).

**C.3 — Sesión expirada durante checkout.** Intenté simular esto corrompiendo el token de `localStorage` (`mutantGardenToken`) a mitad del formulario y completando el checkout de todas formas — **la orden se creó exitosamente pese al token corrupto**, lo que revela que la sesión real de autenticación depende de una cookie `httpOnly` (no accesible ni manipulable desde JavaScript del lado cliente), y que ese valor en `localStorage` no es lo que autentica las peticiones reales. **Limitación metodológica honesta:** no pude forzar una expiración de sesión real sin acceso a manipular cookies `httpOnly` o esperar el tiempo real de expiración, así que este caso queda **sin verificar en vivo**, solo con este hallazgo colateral. Sí confirmé, al intentar cancelar una orden vía `fetch()` directo sin pasar por la UI, que hay protección CSRF activa en los endpoints admin que cambian estado (`403 Token CSRF requerido`).

**C.4 — Doble submit.** No se probó con un doble-clic real cronometrado (difícil de garantizar vía automatización de navegador), pero la evidencia de código muestra **tres capas independientes** de protección: (1) un lock síncrono en frontend (`submitLockRef` en `useCheckoutHandlers.ts:89-95`, un `useRef` que bloquea reentradas antes de que la primera respuesta vuelva); (2) el modal de confirmación de pago deshabilita sus botones mientras `isLoading`; (3) a nivel de backend, una **constraint UNIQUE en base de datos** sobre `idempotency_keys.key` (`idempotency.service.ts:41-45,81-84`) — si dos requests llegan casi simultáneas, la segunda recibe un `409 CHECKOUT_CONFIRM_IN_PROGRESS` o la respuesta cacheada de la primera, nunca crea una segunda orden. La clave se genera por hash determinístico del payload si el cliente no manda una explícita, así que incluso un doble-submit accidental sin idempotency key coincide en la misma clave.

**C.5 — Cambio de precio entre carrito y checkout.** Confirmado en código y en la propia captura de red de mi compra de prueba: el frontend **nunca envía precio**, solo `{productId, quantity, variantId?}` (`checkoutPayload.ts:76-114`) — comentario explícito en el código: *"El pricing del checkout siempre viene del backend"*. El backend relee `Product`/`ProductVariant` de la base de datos dentro de la misma transacción de creación (`order-checkout.service.ts:307-353`) y jamás usa un total que mande el cliente. Riesgo de manipulación de precio del lado cliente: bajo, por diseño.

---

## PARTE D — Comparación código vs. tráfico real

**Hallazgo principal: el patrón de bug encontrado en productos (Zod `.superRefine()`/`.refine()` mal combinado con `.partial()`, y `.default()` en un schema base compartido enmascarando campos ausentes en updates parciales) NO existe en el módulo de órdenes/checkout — por una razón estructural: el módulo de órdenes no usa Zod en absoluto.** Confirmado con `grep -rn "z\.object\(|\.superRefine\(|\.partial\(\)|\.default\(" src/` en todo el backend: el único archivo que matchea en todo el proyecto sigue siendo `product.validation.ts` (ya corregido). La validación en `orders/` es 100% imperativa TypeScript (`if (!x) throw new AppError(...)`), duplicada en varios puntos (`checkout.validation.ts`, `service-order-creator.ts:179-237`, `checkout-draft.service.ts:539-560`).

El área más análoga estructuralmente — el merge de un "borrador de checkout" parcial contra la sesión persistida (`mergeDraftPayload`, `checkout-draft.service.ts:214-233`) — fue auditada específicamente por este riesgo: usa spread plano de objetos, **sin ningún `.default()` de negocio** que rellene campos ausentes con valores que después disparen reglas cruzadas espurias. Un `confirm` con un patch vacío sobre un draft nunca inicializado falla con un mensaje de campo faltante claro, no un 500 ni un default silencioso.

Payload real interceptado en mi propia compra de prueba (confirmado también por lectura de `checkoutPayload.ts`): items solo con `productId`/`quantity`/`variantId`, sin precio ni total — el backend es la única fuente de verdad para ambos, con revalidación server-side dentro de transacción (ver C.1 y C.5). No se encontraron discrepancias del tipo "campo fantasma que el frontend envía y el backend descarta silenciosamente" en el payload de checkout (a diferencia de `slugHistory` en productos) — el único campo puramente informativo es `subtotalHint`, usado solo para la sesión de carrito abandonado, nunca para calcular el total real.

---

## Hallazgos organizados

### BUGS CRÍTICOS

**#1 — Ningún método de pago está protegido de quedar en cero.** Ni `toggleStatus` ni `updatePaymentMethod` (`admin-payment-methods.service.ts:217-272`) verifican cuántos métodos quedarán activos tras la operación; `updatePaymentMethod` ni siquiera reutiliza el chequeo de "método en uso" que sí tiene `toggleStatus`. Un admin puede dejar la tienda entera sin forma de cobrar, sin ninguna advertencia.
- **Impacto: alto** — bloquea el checkout completo de la tienda, de forma silenciosa (el admin no se entera hasta que un comprador se queja).
- **Esfuerzo: bajo** — un `count` de métodos activos antes de desactivar el último, en ambos endpoints.

**#2 — El campo `type` de los 3 métodos de pago activos está corrupto en base de datos** (los tres tienen `type: "cash"` en vez de `payphone`/`paypal`/`transfer`), mostrando "Efectivo" para todos en la columna TIPO del admin. Confirmado que no rompe la protección de "método en uso" (esa consulta usa UUID), pero es un dato visiblemente incorrecto en una pantalla que un admin usa para tomar decisiones.
- **Impacto: medio** — no rompe funcionalidad hoy, pero es exactamente el tipo de corrupción de datos silenciosa que puede volverse crítica si algún flujo futuro empieza a confiar en `type`.
- **Esfuerzo: bajo** — corregir los 3 registros existentes + revisar el formulario de creación/edición para que no permita guardar un `type` inconsistente con el `code`.

### PASOS/CAMPOS QUE SOBRAN

- **`PATCH /orders/:id/confirm-payment`** (confirmación manual de pago sin exigir comprobante) está implementado en backend y en el servicio frontend, pero **no tiene ningún botón que lo invoque** en la UI admin actual — la conciliación real pasa por `transfers/:id/validate`, que es más estricta (exige comprobante y monto exacto). Código alcanzable solo por API directa, no desde el panel.
  - **Impacto: bajo. Esfuerzo: bajo** (eliminarlo, o conectarlo explícitamente como "vía de emergencia" documentada con un aviso de que se salta la verificación de monto).
- **`adminCheckoutService.updateCommissions`** (`PUT /admin/checkout/config/commissions`) existe en el servicio frontend pero el tab "Pagos" de `/admin/checkout` no tiene ningún formulario que lo llame — solo texto explicando que el ownership se movió a `payment_methods.config`.
  - **Impacto: bajo. Esfuerzo: bajo** (eliminar el método del servicio si de verdad quedó obsoleto).

### PASOS/CAMPOS QUE FALTAN

1. **Reporte de tasa de abandono de checkout.** El dato ya se recolecta en `checkout_sessions` (completedAt/notifiedAt) end-to-end, pero no hay ningún endpoint ni pantalla que lo calcule y muestre.
   - **Impacto: medio** (visibilidad de negocio perdida sobre un dato que ya existe). **Esfuerzo: bajo-medio** (una query de agregación + una tarjeta en el hub de reportes).
2. **Actualización en vivo del estado de orden para el comprador.** Ni `MyOrdersPage.tsx` ni `OrderConfirmation.tsx`/`useOrderConfirmation.ts` tienen polling ni WebSocket — el comprador ve un estado desactualizado hasta que recarga manualmente, incluso justo después de que un admin marque su pedido como enviado.
   - **Impacto: medio** (experiencia de "¿mi pedido avanzó?" poco confiable). **Esfuerzo: medio** (polling ligero cada X segundos en la página de confirmación, al menos mientras la orden esté en un estado no-final).
3. **Plantilla de email dedicada para confirmación de orden.** Hoy todo email transaccional (incluida la confirmación de compra) usa el mismo layout genérico con `title`/`message` variables — el campo `template: 'order-confirmed'` está definido pero nunca se usa para cargar contenido distinto.
   - **Impacto: bajo-medio** (calidad de comunicación con el cliente). **Esfuerzo: medio** (plantillas HTML por tipo de notificación).

### FRICCIÓN A REDISEÑAR

**#1 — "Recoger en tienda" apagado por defecto en un checkout que ya lo soporta por completo.** No es código incompleto (confirmado: sucursal, fecha y franja horaria funcionan de punta a punta), es una decisión de negocio actualmente en "apagado". Vale la pena confirmar con el negocio si esto es intencional hoy o quedó apagado desde una prueba — sobre todo si hay contenido de marketing (páginas de contenido, como se vio en la auditoría de content-pages) que insinúa que la opción existe.
- **Impacto: medio** (expectativa del comprador vs. realidad). **Esfuerzo: bajo** (activar el flag) — pero es una decisión de producto, no técnica; **detengo aquí y no recomiendo un lado sin que el negocio confirme**.

**#2 — Edición de cantidad en el carrito sin debounce ni bloqueo visual durante la llamada.** Clics rápidos en +/- para usuarios logueados disparan una llamada de red por clic sin protección visible contra carreras (los botones no se deshabilitan mientras `isLoading`).
- **Impacto: bajo-medio** (mayormente cosmético/de carga en el servidor, no corrompe datos porque cada llamada es idempotente en el valor final). **Esfuerzo: bajo** (debounce de ~300-400ms, ya usado en otro punto del propio checkout para el preview de pricing).

**#3 — Botón "Completar pedido" de escritorio/móvil sin `disabled={isSubmitting}` explícito.** La protección real contra doble-submit es sólida (ver Parte C.4: lock + idempotency key + constraint única), pero el botón en sí no se deshabilita visualmente en `CheckoutStepActions.tsx` — un usuario puede seguir haciendo clic sin feedback de que ya se está procesando (aunque no genere duplicados).
- **Impacto: bajo** (percepción de UI, no de datos). **Esfuerzo: bajo** (agregar el `disabled` que ya existe en el modal de confirmación, aquí también).

---

## Resumen para priorizar

| # | Hallazgo | Categoría | Impacto | Esfuerzo |
|---|---|---|---|---|
| 1 | Ningún método de pago protegido de quedar todos inactivos | Bug crítico | Alto | Bajo |
| 2 | `type` corrupto (="cash") en los 3 métodos de pago activos | Bug crítico | Medio | Bajo |
| — | Falta reporte de tasa de abandono de checkout (dato ya existe) | Falta | Medio | Bajo-medio |
| — | Sin actualización en vivo del estado de orden para el comprador | Falta | Medio | Medio |
| — | Plantilla de email de confirmación genérica, no dedicada | Falta | Bajo-medio | Medio |
| — | "Recoger en tienda" apagado por defecto (decisión de negocio, no bug) | Fricción — confirmar con negocio | Medio | Bajo (una vez decidido) |
| — | Edición de cantidad en carrito sin debounce/bloqueo visual | Fricción | Bajo-medio | Bajo |
| — | Botón "Completar pedido" sin `disabled` visual explícito | Fricción | Bajo | Bajo |
| — | `PATCH .../confirm-payment` sin botón en UI (código alcanzable solo por API) | Sobra | Bajo | Bajo |
| — | `updateCommissions` sin formulario en `/admin/checkout` | Sobra | Bajo | Bajo |

**Comparación con la auditoría de productos:** a diferencia de aquella, aquí **no se encontró ningún bug crítico que bloquee o corrompa datos de forma silenciosa a nivel de validación** — el módulo de checkout/órdenes resultó notablemente más robusto en los puntos que más importan (revalidación de precio y stock server-side con locks, idempotencia respaldada por constraint de base de datos, verificación real contra las pasarelas de pago). Los dos bugs críticos encontrados son de una naturaleza distinta: uno de **ausencia de guardrail de negocio** (métodos de pago) y otro de **integridad de datos silenciosa** (campo `type`), ninguno descubierto por accidente en el flujo feliz del comprador, sino por revisión deliberada de código administrativo.

**Recomendación de orden de ataque:** #1 (métodos de pago) primero por ser el único con potencial de tumbar el checkout completo de la tienda; #2 (dato `type`) junto con él por ser trivial una vez que se está tocando ese módulo; luego el reporte de abandono (impacto de negocio real, esfuerzo bajo) antes que las fricciones de UI, que pueden esperar a un sprint de pulido.

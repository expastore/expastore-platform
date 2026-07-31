# Auditoría del Wizard de Creación/Edición de Productos

NOTA: esta es una fotografía histórica del 28 de julio de 2026. Los hallazgos aquí documentados fueron corregidos en sesiones posteriores (ej. el bug crítico de `customShippingRate` con envío de tarifa fija en el commit `2da4b5f`, la fuga de estado del wizard entre aperturas en `6c33f7f`, y el resto de hallazgos de PASO 4 — checklist desincronizado, paso de precio sobrecargado, `ProductFormModal.tsx` muerto, `slugHistory` fantasma, `dealPosition` sin tope — en la tanda TAREA 1-7 de días siguientes). No usar como estado actual del código.

Auditoría realizada el 2026-07-28 sobre `/admin/products`, `ProductFormWizard.tsx`, `useProductForm.ts` y `productFormModel.ts`, con lectura completa de código y prueba real en navegador (sesión admin) creando, editando y duplicando un producto real (`Auditoria Producto Simple Test`, id `d9a45e94-0a02-43c2-8978-39065469cc4a`). Diagnóstico únicamente — no se modificó ningún archivo de código.

Metodología: (1) mapeo de código con lectura completa de los archivos fuente; (2) prueba end-to-end en navegador real (Chrome, sesión admin activa) creando un producto físico simple; (3) inspección de las peticiones HTTP reales (`XMLHttpRequest`/`fetch` interceptados en la página) para verificar qué se envía de verdad al backend, no solo lo que la UI muestra; (4) contraste con el validador Zod del backend (`product.validation.ts`). Donde no se completó una prueba en vivo (caso de variantes), se indica explícitamente y la evidencia es solo de código.

---

## PASO 1 — Mapeo del flujo tal como está en código

### 1.1 Pasos del wizard, en orden

El wizard tiene **9 pasos fijos** (`ProductFormWizard.tsx:31-64`, `STEPS`/`STEP_FIELDS`), confirmado en pantalla ("Paso X de 9"):

| # | Nombre | Componente | Campos |
|---|---|---|---|
| 1 | Tipo | `ProductTypeSection.tsx` | `productType` (Físico/Digital) |
| 2 | Basico | `BasicInfoSection.tsx` | `name`, `slug`, `categoryId`, `shortDescription`, `description`, `tags` |
| 3 | Imagenes | `ImagesSection.tsx` | `mainImage`, `mainImageAltText`, `images[]` (hasta 8), alt text por imagen |
| 4 | Precio | `PricingSection.tsx` | `price`, `comparePrice`, `purchaseCostInput`, `desiredProfitPercent`, `otherDirectCosts`, `operationalShippingCost`, `profitBasis`, `roundingMode`, `referencePaymentMethodCode`, checkboxes "Costo incluye IVA"/"Simular efectivo" |
| 5 | Inventario | `InventorySection.tsx` | `productMode` (simple/variantes), `trackInventory`, `lowStockThreshold`, `allowBackorder`, `stock`, `sku` |
| 6 | Variantes (opcional) | `VariantsSection.tsx` (~1200 líneas) | `variants[]` — solo aplica si `productMode==='variants'`; se **omite automáticamente** en modo simple |
| 7 | Entrega | `ShippingSection.tsx` + `WarrantySection.tsx` (físico) / `DigitalDeliverySection.tsx` (digital) | shipping type, tarifas, dimensiones, garantía |
| 8 | SEO | `SeoSection.tsx` | `slug` (editable), `seoMetaTitle`, `seoMetaDescription`, preview de Google |
| 9 | Publicacion | `VisibilitySection.tsx` + `PublishSection.tsx` | `visibility`, `dealPosition`, checklist final |

### 1.2 Obligatoriedad y momento de validación

No hay un schema declarativo (zod/yup) en frontend: `buildValidationIssues()` en `useProductForm.ts:609-744` es un único `if/push` imperativo. Momento de validación — **mixto, no es "por paso" ni "al final" de forma limpia**:

1. **"Siguiente"** → `validateStep(currentStep)`: valida **solo el paso actual**.
2. **"Guardar borrador"** (solo en creación) → valida explícitamente **solo pasos 1, 2 y 4**, ignorando 3/5/6/7/8/9.
3. **"Publicar producto"/"Guardar cambios"** (paso 9) → `validateForm()` sin argumento, valida todo de una vez. Es la única validación realmente completa.
4. El botón de submit además depende de un **segundo sistema de reglas independiente**: `buildPublishChecklist()` (`publishChecklist.ts:78-151`), que no está sincronizado con `buildValidationIssues`. Ejemplo confirmado en código: el checklist exige `stock > 0` para físicos (línea 62-73, estricto mayor-que), mientras que la validación de paso 5 solo exige `stock >= 0` (línea 652).

Campos requeridos por paso (frontend): nombre (≥3 caracteres) y categoría (paso 2); imagen principal solo en creación (paso 3); en modo simple: costo de compra, utilidad objetivo y método de cobro de referencia, más precio > 0 (paso 4); stock (paso 5, modo simple); al menos 1 variante válida (paso 6, si aplica); tipo de envío + tarifa/dimensiones según el tipo elegido (paso 7); nada obligatorio en SEO (paso 8); posición de oferta solo si visibilidad = "Oferta Destacada" (paso 9).

### 1.3 Navegación entre pasos

**No es estrictamente lineal ni totalmente libre — y además es inconsistente entre sesiones** (ver hallazgo crítico en PASO 4). En una sesión "fría" (recién cargada la página):

- Avanzar por primera vez solo es posible con "Siguiente", que exige que el paso actual pase `validateStep()`.
- Retroceder a un paso ya visitado es libre y **no se re-valida** ese paso hasta el submit final.
- Un tab nunca visitado permanece deshabilitado — no se puede "saltar en frío" (confirmado en vivo: al abrir "Editar producto" recién cargada la página, clicar directamente el tab "Precio" no hace nada; el modal se queda en "Paso 1 de 9").
- El paso 6 (Variantes) se salta automáticamente cuando el producto es modo simple.

### 1.4 Autoguardado / persistencia

**No existe autoguardado ni borrador local.** Confirmado por código (`grep` de `localStorage`/`sessionStorage` relacionado a producto: cero resultados) y confirmado en vivo: llenar el paso 2 con nombre "Prueba Cancelar A Medias", pulsar "Cancelar" y reabrir "Crear producto" deja el campo Nombre completamente vacío — no hay ningún rastro del texto escrito.

"Guardar borrador" **no es un draft de wizard**: es un POST real a `/admin/products` con `status=draft`; crea una fila real en la base de datos. Solo existe en creación, no en edición.

### 1.5 Clics/campos mínimos — producto físico simple

Contado en vivo, campo por campo, sobre el producto real creado:

| Paso | Interacciones requeridas |
|---|---|
| 1. Tipo | 0 (Físico/Simple ya viene por defecto) + 1 clic "Siguiente" |
| 2. Basico | Nombre (escribir) + Categoría (seleccionar) + 1 clic "Siguiente" |
| 3. Imagenes | 1 imagen principal (obligatoria) + 1 clic "Siguiente" |
| 4. Precio | Precio base + Costo de compra + Utilidad objetivo (método de cobro ya viene con default) + 1 clic "Siguiente" |
| 5. Inventario | Stock + 1 clic "Siguiente" |
| 6. Variantes | Se salta automáticamente |
| 7. Entrega | 1 clic para (re)seleccionar tipo de envío *— ver hallazgo crítico, es obligatorio hacer clic aunque "Costo fijo" ya se vea seleccionado* + tarifa (si aplica) + 1 clic "Siguiente" |
| 8. SEO | 0 campos + 1 clic "Siguiente" |
| 9. Publicacion | 0 campos (Visibilidad "Normal" por defecto) + 1 clic "Publicar producto" |

**Total: ~7 campos + 9 clics de navegación = 16 interacciones mínimas, en 9 pantallas**, para el producto más simple posible. Esto no cuenta los reintentos que un admin real necesitará por el bug del paso 7 (ver PASO 4, hallazgo #1).

### 1.6 Producto con variantes + varias imágenes (caso más común real)

No se completó una prueba en vivo de creación de variantes por alcance de tiempo — esto es solo evidencia de código, indicado explícitamente:

- En el paso 5, hay que cambiar "Modo de producto" a "Producto con variantes" (1 clic) — esto mueve `stock`/`sku` del nivel de producto al nivel de variante.
- El paso 6 (Variantes) deja de omitirse y se vuelve obligatorio: hay que definir atributos (ej. color/talla), generar combinaciones, y por cada variante activa completar precio > 0, stock ≥ 0 y SKU no vacío, sin combinaciones duplicadas (`useProductForm.ts:659-675`). `VariantsSection.tsx` soporta imagen por variante (`images: string[]`, `imageUrl`) además de las imágenes generales del paso 3 — es decir, **las imágenes se piden dos veces en dos lugares distintos** (imágenes generales del producto en paso 3, e imágenes por variante en paso 6), sin que el wizard explique la diferencia entre ambas.
- Con 4-6 variantes reales (ej. 3 colores x 2 tallas), el conteo de campos fácilmente supera 30-40 interacciones adicionales solo en el paso 6.

---

## PASO 2 — Prueba real en navegador, de punta a punta

Producto creado: físico, categoría Electrónica, 1 imagen principal + 3 adicionales (subidas manualmente por el usuario en el navegador), precio base $99.99, costo $50, utilidad 25%.

### 2.1 Fricciones observadas, con evidencia

**a) El campo "Utilidad objetivo (%)" no se limpia al hacer foco+escribir.** Al hacer clic en el campo (que mostraba "25" con apariencia de placeholder gris) y escribir "25", el resultado fue "2525" — el valor anterior no era un placeholder, era un valor real precargado, y el clic no seleccionó el contenido. Un admin que no revise el valor resultante puede terminar guardando una utilidad objetivo del 2525% sin darse cuenta, ya que el campo no tiene un tope máximo visible ni un `select-on-focus`.

**b) El paso 4 "Precio" es una calculadora financiera completa, no un formulario de precio simple.** Contiene 9 campos/controles (precio base, precio de comparación, costo de compra, utilidad objetivo, otros gastos directos, envío operativo absorbido, "calcular utilidad sobre", redondeo comercial, método de cobro de referencia) más 2 checkboxes ("Costo incluye IVA", "Simular efectivo") y un panel de resultados con "Utilidad neta estimada", "Margen sobre total cobrado" y avisos de redondeo aplicado. Para un admin nuevo que solo quiere poner un precio de venta, esto es significativamente más complejo de lo que el nombre del paso ("Precio") sugiere.

**c) El campo numérico de "Costo fijo de envío" se muestra con coma decimal ("4,99") por el locale `es-EC` del navegador**, aunque el valor interno (`input.value`) sí queda correctamente en formato punto ("4.99"). No es un bug de datos en sí (confirmado con inspección directa del DOM), pero es una fuente de confusión visual — un admin acostumbrado a "." como separador decimal puede pensar que escribió mal el número.

**d) Información pedida dos veces:** las imágenes se piden en el paso 3 (imagen principal + hasta 8 adicionales) y, en modo variantes, otra vez por variante en el paso 6 (ver 1.6), sin que la UI aclare cuál imagen se usa dónde ni si son complementarias o excluyentes.

**e) Texto ambiguo:** en el paso 9, el checklist muestra "OK · Configuracion de envio" en verde con el subtítulo "Selecciona el tipo de envio y completa los campos requeridos" — este mensaje aparece como **check verde exitoso incluso cuando la configuración de envío está rota** (ver PASO 4, hallazgo #1), lo cual es objetivamente engañoso.

**f) Botones sin distinción visual clara de riesgo:** "Guardar borrador" (ámbar) crea de verdad una fila en la base de datos con `status=draft`, algo que un admin podría no esperar de un botón que suena a "guardar progreso localmente".

### 2.2 Cosas que un admin novato probablemente no entendería a la primera

- Que "Guardar borrador" publica el producto en la base de datos (aunque oculto/no listado como activo) en vez de guardar el progreso del formulario.
- La diferencia entre "Costo incluye IVA" y "Simular efectivo" en el paso de precio, sin tooltip ni ejemplo.
- Por qué el checklist de publicación dice "OK" en envío y el botón "Publicar producto" falla igual (ver PASO 4).
- Que cerrar el modal sin guardar borra todo sin aviso — no hay ningún diálogo de confirmación tipo "¿Seguro que quieres salir? Perderás los cambios" al pulsar "Cancelar" o la X.

### 2.3 Editar un producto existente

Probado en frío (recarga completa de página antes de abrir "Editar"): el wizard se abre en **"Paso 1 de 9: Tipo"**, con los tabs 2-9 deshabilitados. Confirmado en vivo que hacer clic directo en el tab "Precio" no tiene ningún efecto — hay que pulsar "Siguiente" tres veces (validando Tipo, Basico, Imagenes) antes de poder tocar el precio. **No existe ningún atajo dentro del wizard para editar un solo campo.**

Sí existen 2 atajos **fuera** del wizard, confirmados en el código (`ProductTableSection.tsx`): edición inline de stock (solo para productos sin variantes) y toggle de estado Activo/Borrador con un clic, ambos en la tabla de listado.

### 2.4 Cancelar a medias y volver

Probado en vivo: escribir un nombre en el paso 2, pulsar "Cancelar", y volver a abrir "Crear producto" — el campo Nombre queda vacío (correcto, no hay fuga de datos entre sesiones), **pero el modal reabre directamente en "Paso 2 de 9: Basico" en vez de "Paso 1"**, con el paso 1 ("Tipo") marcado como completado (check verde) sin haberlo tocado en esta nueva sesión. Ver causa raíz en PASO 4, hallazgo #2 — es el mismo bug de estado que afecta a "Editar".

### 2.5 Duplicar producto

Probado en vivo sobre el producto recién creado: el botón "Duplicar producto" dispara un `window.confirm()` nativo del navegador (bloqueante), y al confirmar aparece el toast "Producto duplicado exitosamente" y se abre automáticamente el wizard en modo edición sobre la copia (nombre "Auditoria Producto Simple Test (Copia)", slug con sufijo "-copia", categoría heredada). Confirmado en código: es un clon real en base de datos (no un prefill en memoria), con `status` forzado a `DRAFT`, SKU y slug regenerados, variantes clonadas (incluidas inactivas), pero **sin heredar el historial de slugs**. Funciona bien como atajo para productos similares — es, junto con el toggle de estado y la edición inline de stock, de lo mejor resuelto del flujo.

Nota de auditoría: el `window.confirm()` bloqueante es una forma de confirmación anticuada frente al resto de la UI (que usa toasts y modales propios); no es inconsistente en el sentido de "roto", pero desentona con el resto del sistema de diseño.

---

## PASO 3 — Comparación wizard vs. modelo vs. backend

### 3.1 Payload real enviado (interceptado en vivo)

Con `shippingType='free'` (creación exitosa, HTTP 201), el `FormData` real enviado fue:

```
name, productType, shippingType, status, price, stock, categoryId,
trackInventory, allowBackorder, visibility, featured, isFeaturedDeal,
warrantyMonths, pricingInputs (JSON), mainImage (File), images (File),
newImageAltTexts (JSON)
```

Nótese que **`slugHistory` no se incluyó** porque el producto es nuevo (sin historial previo) — pero el código de `toFormData()` sí lo agrega cuando existe. Este campo se pierde igual en el backend (ver 3.2).

### 3.2 Discrepancias confirmadas (código + tráfico real)

| Campo | Wizard | Backend zod | Uso real | Veredicto |
|---|---|---|---|---|
| `slugHistory` | Se envía si hay historial (`useProductForm.ts:449-451`) | Ausente en `baseProductSchema` — se descarta en `parseAsync` sin `.passthrough()` | El controller nunca lo lee | **Feature fantasma**: el banner en `BasicInfoSection.tsx:126-129` promete redirección automática de slugs antiguos, pero el valor del cliente nunca llega a ningún lado. El historial real lo genera el backend de forma autónoma solo en `update`, cuando detecta cambio de slug (`service-admin-product-management.adapter.ts:575-584`) — desincronizado del array que ve el usuario en el formulario. |
| `featured`, `isFeaturedDeal` | Se envían como booleanos | No declarados, se descartan | El controller los recalcula desde `visibility` | Redundante pero inofensivo. |
| `costPrice` | Nunca se envía (el wizard usa `purchaseCostInput` dentro de `pricingInputs`) | Declarado, opcional | No usado por el controller | Campo backend huérfano. |
| `keepExistingImages` | Nunca se envía | Declarado, opcional | No usado | Campo backend huérfano/legacy. |
| Archivo/URL para producto digital | El wizard bloquea avanzar sin archivo o URL (`useProductForm.ts:713-718`) | El schema no lo exige (`digitalDetailsSchema` solo pide `digitalType`) | — | Un cliente API directo podría crear un producto digital sin contenido — el wizard es más estricto que el backend aquí. |
| `pricingInputs` completo | Requerido en frontend (costo, utilidad, método de pago) salvo modo variantes | Opcional en backend | El backend nunca exige pricing propio | El requisito de "costo + utilidad + método de pago" es una regla de negocio solo del frontend. |
| `dealPosition` rango 1-5 | La UI limita el `<select>` a 1-5 pero no valida rango explícito en `useProductForm.ts` | El create/update schema solo valida mínimo (`min:1`), sin máximo — el tope de 5 solo existe en el endpoint separado `/deal-slot`, que el wizard nunca llama | — | Un create/update directo con `dealPosition: 999` pasaría la validación del backend. |
| `comparePrice > price` en modo variantes | El frontend no valida esta regla cuando el precio viene derivado de variantes | El backend sí la valida siempre (`superRefine`, sin excepción de modo variantes) | — | Backend más estricto que frontend en este caso puntual. |

---

## PASO 4 — Hallazgos priorizados

### 4.1 PASOS/CAMPOS QUE SOBRAN

**Ninguno se identifica con evidencia sólida como completamente innecesario.** `ProductFormModal.tsx` es el único candidato real: es un formulario de una sola página que duplica la lógica de `ProductFormWizard.tsx` pero **no está importado en ningún lugar del código** (confirmado por búsqueda exhaustiva) — es código muerto.

- **Impacto: bajo** (no afecta a usuarios, solo a quien lee el código). **Esfuerzo: bajo** (eliminar el archivo).

### 4.2 PASOS/CAMPOS QUE FALTAN

1. **Ningún mecanismo de recuperación tras cierre accidental.** Confirmado en vivo: cancelar borra todo sin confirmación ni posibilidad de recuperar. Para un formulario de 9 pasos con carga de imágenes, esto es una pérdida de trabajo real y frecuente.
   - **Impacto: alto** (pérdida de trabajo del admin, especialmente en productos con variantes que toman varios minutos). **Esfuerzo: medio** (autosave a `localStorage` por producto en edición/creación, con expiración).

2. **Ningún diálogo de confirmación al cancelar con cambios sin guardar.** Un clic accidental en "Cancelar" o en la X, tras haber llenado varios pasos, borra todo sin preguntar.
   - **Impacto: medio**. **Esfuerzo: bajo** (un `window.confirm` o modal propio ya existe como patrón en "Duplicar", se puede reutilizar).

3. **Atajo para editar un solo campo sin pasar por los 9 pasos.** Ya existe para `stock` y `status` (edición inline en la tabla) — falta al menos para `price`, que es probablemente el campo que más se edita en el día a día de un catálogo.
   - **Impacto: medio-alto** (fricción diaria repetida). **Esfuerzo: medio** (edición inline similar a la de stock, reutilizando el patrón existente).

### 4.3 FRICCIÓN A REDISEÑAR (lo más importante de esta auditoría)

**#1 — BUG CRÍTICO confirmado en vivo: un producto físico con envío "Costo fijo" (la opción por defecto) nunca se puede publicar.**

Reproducido de forma consistente (4 intentos): al crear un producto físico dejando "Costo fijo" seleccionado (que es el valor por defecto al llegar al paso 7, sin que el admin haga ningún clic) y llenando el campo "Costo fijo de envío ($)" con un valor válido, el submit final falla siempre con `400 Errores de validacion: customShippingRate: El costo fijo de envio debe ser mayor a 0` — a pesar de que el campo se ve lleno en pantalla y el checklist del paso 9 muestra "OK · Configuracion de envio" en verde.

Causa raíz confirmada leyendo el código y comparando con el `FormData` real interceptado en la petición: `initialFormData.shippingType` es `'fixed'` por defecto (`productFormModel.ts:187`), pero `initialFormData.useCustomShippingRate` es `false` por defecto (línea 219) — son dos flags separados que deberían estar sincronizados. Solo se sincronizan dentro de `handleShippingTypeChange()` en `ShippingSection.tsx:57-68`, un handler que **solo se ejecuta cuando el admin hace clic en una tarjeta de tipo de envío**. Como "Costo fijo" ya viene preseleccionado por defecto, un admin que nunca hace clic en esa tarjeta (porque ya está seleccionada) deja `useCustomShippingRate=false` para siempre. Y `toFormData()` en `useProductForm.ts:501-502` decide si envía `customShippingRate` al backend usando exactamente esa condición rota:

```ts
if (formData.useCustomShippingRate && formData.customShippingRate) {
  data.append('customShippingRate', formData.customShippingRate);
}
```

Mientras que la validación del paso 7 (`useProductForm.ts:683-684`) usa la condición correcta (`formData.shippingType === 'fixed'`), por eso el wizard nunca muestra error al avanzar de paso — el error solo aparece en el toast del submit final, sin apuntar a ningún campo visible en pantalla en ese momento (el usuario ya está en el paso 9).

**Verificado el workaround:** si el admin hace clic manualmente en otra tarjeta de envío y luego vuelve a hacer clic en "Costo fijo", `useCustomShippingRate` se sincroniza correctamente y el submit funciona. Pero nada en la UI sugiere que este re-clic sea necesario.

- **Impacto: alto** — bloquea la creación de productos físicos con envío de tarifa fija (probablemente la opción de envío más común en un catálogo real) de forma silenciosa y no descubrible sin leer el código o interceptar la red.
- **Esfuerzo: bajo** — corrección de una línea: cambiar la condición de `useProductForm.ts:501` de `formData.useCustomShippingRate` a `formData.shippingType === 'fixed'` (igual que ya hace la validación en la línea 683), o eliminar el flag `useCustomShippingRate` por completo y depender solo de `shippingType`.

**#2 — BUG confirmado en vivo: el wizard no se desmonta entre aperturas, y el paso/estado de navegación se filtra entre sesiones distintas.**

`ProductFormWizard` (`ProductFormWizard.tsx:472-480`) usa el patrón `if (!isOpen) return null` **después** de declarar `useState(1)` para `currentStep`, lo que significa que React nunca destruye la instancia del componente al cerrar el modal — solo dejar de renderizarlo. Confirmado en vivo tres veces: (a) tras terminar de crear un producto en el paso 9, abrir "Editar" sobre otro producto abrió el modal directamente en "Paso 9 de 9" con todos los tabs habilitados; (b) tras cancelar una creación a medio llenar en el paso 2, reabrir "Crear producto" abrió directamente en "Paso 2 de 9" en vez de "Paso 1"; (c) al duplicar un producto, el modal de edición resultante abrió en "Paso 2 de 9" en vez de "Paso 1" — heredado de la sesión de cancelación anterior en la misma pestaña del navegador.

Esto hace que el comportamiento de navegación (secuencial vs. libre) sea **no determinista**: depende del historial de aperturas del wizard en la pestaña actual, no del producto ni de la acción en curso. Un admin puede terminar viendo pasos marcados como "completados" que en realidad nunca revisó para el producto que tiene abierto en ese momento — riesgo de guardar datos de un paso que parece validado pero nunca se tocó para ese producto específico.

- **Impacto: alto** — comportamiento impredecible del wizard, con riesgo de que un admin confíe en un check verde que no corresponde al producto actual.
- **Esfuerzo: bajo-medio** — mover el `useState` fuera de la rama condicional no alcanza (el problema es que el componente no se desmonta); la corrección real es resetear `currentStep`/`completedSteps` en un `useEffect` que dependa de `product?.id` y de la transición de `isOpen` a `true`, o forzar remount con `key={product?.id ?? 'new'}` desde `AdminProducts.tsx`.

**#3 — Checklist de publicación y validación de campos son dos sistemas de reglas independientes, no sincronizados.**

Confirmado en código (`stock > 0` en el checklist vs. `stock >= 0` en la validación) y observado en vivo con el bug de envío (#1): el checklist mostró "OK" en verde para una configuración de envío que en realidad bloqueaba la publicación. Esto rompe la confianza del admin en el checklist como fuente de verdad.

- **Impacto: medio-alto** (mina la confianza en la UI; en el caso #1 activamente engaña). **Esfuerzo: medio** — unificar ambos sistemas en una sola fuente de verdad (derivar el checklist de `buildValidationIssues` en vez de mantener reglas paralelas en `publishChecklist.ts`).

**#4 — Paso 4 "Precio" mezcla configuración de precio con una calculadora financiera completa.**

Ver evidencia en 2.1(b). Para el caso más simple (poner un precio de venta), el admin tiene que entender conceptos de "utilidad objetivo", "redondeo comercial", "calcular utilidad sobre costo base vs. venta neta" — útil para quien fija precios con margen, pero excesivo como paso obligatorio para alguien que solo quiere escribir "$19.99".

- **Impacto: medio** (fricción cognitiva, no bloquea). **Esfuerzo: medio** — hacer colapsable/opcional la sección de calculadora ("Precio avanzado ▾"), dejando visible por defecto solo precio base y precio de comparación.

**#5 — Información de imágenes pedida en dos lugares (paso 3 general + paso 6 por variante) sin explicar la diferencia.**

Ver 1.6 y 2.1(d). Solo verificado por código, no en vivo.

- **Impacto: bajo-medio**. **Esfuerzo: bajo** (agregar un texto explicativo en el paso 6: "estas imágenes son específicas de la variante; la imagen principal del paso 3 se usa como fallback").

**#6 — Edición de un solo campo obliga a atravesar los pasos 1→2→3 antes de llegar a "Precio", en sesión fría.**

Ver 2.3. Confirmado en vivo. Nota importante: en la práctica, si el admin ya abrió el wizard antes en la misma sesión de navegador (crear o editar otro producto), el bug #2 puede "arreglar" accidentalmente esta fricción dejando todos los tabs habilitados — pero esto es un efecto secundario de un bug, no un diseño intencional, y no se puede confiar en él.

- **Impacto: medio** (fricción diaria si se corrige el bug #2 sin agregar memoria de paso intencional). **Esfuerzo: medio** — al abrir en modo edición, marcar todos los pasos como completados intencionalmente (el producto ya tiene datos válidos guardados), permitiendo navegación libre real, no accidental.

---

## Resumen para priorizar

| # | Hallazgo | Categoría | Impacto | Esfuerzo |
|---|---|---|---|---|
| 1 | `customShippingRate` nunca se envía con "Costo fijo" por defecto → bloquea publicar productos físicos | Fricción/bug | Alto | Bajo |
| 2 | Wizard no se desmonta → estado de paso se filtra entre sesiones | Fricción/bug | Alto | Bajo-medio |
| 3 | Checklist de publicación desincronizado de la validación real | Fricción/bug | Medio-alto | Medio |
| 6 | Editar un campo obliga a pasos 1→2→3 en frío | Falta / fricción | Medio | Medio |
| — | Sin autosave ni confirmación al cancelar | Falta | Alto / Medio | Medio / Bajo |
| — | Sin atajo para editar precio fuera del wizard | Falta | Medio-alto | Medio |
| 4 | Paso Precio mezcla precio simple con calculadora financiera | Fricción | Medio | Medio |
| 5 | Imágenes pedidas dos veces sin explicación (paso 3 y 6) | Fricción | Bajo-medio | Bajo |
| — | `ProductFormModal.tsx` código muerto | Sobra | Bajo | Bajo |
| — | `slugHistory` es feature fantasma en el cliente | Backend/frontend | Bajo | Bajo (documentar o eliminar el banner) |
| — | `dealPosition` sin tope superior en create/update | Backend/frontend | Bajo | Bajo |

**Recomendación de orden de ataque:** hallazgos #1 y #2 primero (son bugs de una línea/pocas líneas con impacto alto y esfuerzo bajo — el mejor ratio posible), luego autosave/confirmación de cancelar (impacto alto, esfuerzo medio), y el resto según capacidad.

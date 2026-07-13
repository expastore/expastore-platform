# Orders

## Proposito

Gestiona creacion, persistencia, estados, historial y ciclo de vida de ordenes.

## Frontend involucrado

- `expastore-frontend/src/pages/MyOrdersPage.tsx`
- `expastore-frontend/src/pages/OrderConfirmation.tsx`
- `expastore-frontend/src/pages/OrderSuccess.tsx`
- `expastore-frontend/src/services/orderService.ts`

## Backend involucrado

- `expastore-backend/src/modules/orders/*`
- `expastore-backend/src/modules/payments/transfer-validation.service.ts`
- `expastore-backend/src/services/order-payment-expiry.scheduler.ts`

## Superficie funcional principal

- creacion de orden
- consulta de historial y detalle
- cancelacion
- checkout draft y session
- acceso digital y fulfillment posterior

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- checkout
- pagos
- shipping
- inventory
- delivery pins
- notificaciones

## Automatizaciones o eventos

- expiracion de pago
- cambio de estado
- generacion de PIN de entrega
- alertas por venta confirmada

## Deuda conocida

- el modulo orden cruza muchas responsabilidades
- hace falta mapa claro del lifecycle completo por metodo de pago

## Preguntas abiertas

- conviene separar mejor order creation, order payment lifecycle y fulfillment

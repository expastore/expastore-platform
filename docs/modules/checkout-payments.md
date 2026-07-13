# Checkout y Pagos

## Proposito

Gestiona el flujo de compra, datos de facturacion, direccion de envio, carriers, promociones y metodos de pago.

## Frontend involucrado

- `expastore-frontend/src/pages/Checkout/*`
- `expastore-frontend/src/hooks/useCheckoutData.ts`
- `expastore-frontend/src/hooks/useCheckoutValidation.ts`
- `expastore-frontend/src/hooks/useCheckoutPromotions.ts`
- `expastore-frontend/src/components/payment/*`
- `expastore-frontend/src/components/shipping/*`

## Backend involucrado

- `expastore-backend/src/modules/orders/*`
- `expastore-backend/src/modules/payments/*`
- `expastore-backend/src/modules/payment-methods/*`
- `expastore-backend/src/modules/billing/*`
- `expastore-backend/src/modules/shipping/*`
- `expastore-backend/src/services/paypal.service.ts`
- `expastore-backend/src/services/payphone.service.ts`

## Superficie funcional principal

- Frontend: `/checkout`, `/order-confirmation/:orderId`, callbacks de pago
- Backend: draft de checkout, ordenes, pagos, metodos de pago, billing y shipping

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- auth
- direcciones
- carriers y shipping rates
- promociones y cupones
- order lifecycle

## Automatizaciones o eventos

- expiracion de ordenes sin pago
- validacion de transferencias
- confirmacion de pago
- alertas Telegram y admin por ventas y transferencias

## Deuda conocida

- hay complejidad alta repartida entre hooks del checkout
- el estado local y el estado del backend necesitan documentacion mas fina

## Preguntas abiertas

- que partes del checkout deben persistirse localmente y cuales no
- como separar mejor orquestacion de UI frente a logica de negocio

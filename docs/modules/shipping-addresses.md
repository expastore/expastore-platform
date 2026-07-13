# Shipping y Direcciones

## Proposito

Gestiona direcciones del usuario, direccion predeterminada, carriers, zonas, tarifas y datos de entrega.

## Frontend involucrado

- `expastore-frontend/src/pages/Addresses.tsx`
- `expastore-frontend/src/components/layout/NavDireccion.tsx`
- `expastore-frontend/src/pages/Checkout/components/ShippingStep.tsx`
- `expastore-frontend/src/components/shipping/CarrierSelector.tsx`

## Backend involucrado

- `expastore-backend/src/modules/addresses/*`
- `expastore-backend/src/modules/shipping/*`
- `expastore-backend/src/modules/shipping/models/*`

## Superficie funcional principal

- direcciones del usuario
- seleccion de direccion predeterminada
- calculo de shipping
- puntos de retiro y disponibilidad

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- checkout
- orders
- admin shipping
- header de navegacion

## Automatizaciones o eventos

- seleccion de direccion predeterminada desde el header
- sincronizacion local con checkout

## Deuda conocida

- recien se elimino un page alterno de direcciones que no estaba conectado
- falta documentar bien la fuente de verdad entre header, dashboard y checkout

## Preguntas abiertas

- conviene centralizar estado de direcciones con contexto o query layer

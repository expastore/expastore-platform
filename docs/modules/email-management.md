# Email Management

## Proposito

Gestiona plantillas, logs, soporte por correo, replies rapidos, proveedores y webhooks de email.

## Frontend involucrado

- `expastore-frontend/src/pages/admin/AdminEmails.tsx`
- `expastore-frontend/src/pages/admin/tabs/*`
- `expastore-frontend/src/components/admin/*Email*`
- `expastore-frontend/src/components/admin/inbox/*`
- `expastore-frontend/src/services/admin/emailTemplates.ts`

## Backend involucrado

- `expastore-backend/src/modules/email-management/*`
- `expastore-backend/src/services/email.service.ts`
- `expastore-backend/src/services/email-providers/*`

## Superficie funcional principal

- administracion de emails y templates
- soporte y tickets
- webhooks de proveedor
- inbound email

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- auth admin
- support tickets
- alerts
- account security

## Automatizaciones o eventos

- envio transactional
- webhooks de Brevo
- inbound email
- tickets y quick replies

## Deuda conocida

- modulo grande y con varias responsabilidades
- falta separar mejor correo transaccional, inbox operativo y soporte

## Preguntas abiertas

- cual es el modelo objetivo para unificar email, ticketing y alertas

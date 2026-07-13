# Notifications y Alerts

## Proposito

Gestiona notificaciones del usuario, preferencias, WebSocket, Web Push y alertas operativas para admin y Telegram.

## Frontend involucrado

- `expastore-frontend/src/context/NotificationContext.tsx`
- `expastore-frontend/src/components/notifications/*`
- `expastore-frontend/src/pages/NotificationsPage.tsx`
- `expastore-frontend/src/pages/NotificationPreferencesPage.tsx`
- `expastore-frontend/src/services/socket.ts`

## Backend involucrado

- `expastore-backend/src/modules/notifications/*`
- `expastore-backend/src/services/socket.service.ts`
- `expastore-backend/src/services/web-push.service.ts`
- `expastore-backend/src/services/alert.service.ts`
- `expastore-backend/src/services/telegram.service.ts`
- `expastore-backend/src/modules/admin/models/adminAlert.model.ts`

## Superficie funcional principal

- notificaciones del usuario
- preferencias y web push
- eventos en tiempo real
- alertas operativas y administrativas

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- auth
- orders
- payments
- support
- popups y promociones

## Automatizaciones o eventos

- notificaciones en tiempo real
- push notifications
- alertas de transferencias
- alertas de ventas
- alertas de stock bajo

## Deuda conocida

- conviene distinguir mejor notificacion usuario vs alerta operativa admin
- hace falta mapa claro de eventos emitidos y consumidores

## Preguntas abiertas

- cual deberia ser el contrato central de eventos del sistema

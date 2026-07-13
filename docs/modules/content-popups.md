# Content y Popups

## Proposito

Gestiona contenido dinamico de la tienda, popups promocionales y parte del contenido editable desde admin.

## Frontend involucrado

- `expastore-frontend/src/components/popups/*`
- `expastore-frontend/src/services/popups.ts`
- `expastore-frontend/src/pages/admin/popups/*`
- `expastore-frontend/src/pages/admin/AdminContentManager.tsx`

## Backend involucrado

- `expastore-backend/src/modules/popups/*`
- `expastore-backend/src/modules/settings/*`
- `expastore-backend/src/modules/media/*`

## Superficie funcional principal

- popups activos para storefront
- configuracion publica y contenido editable
- flujos admin asociados a contenido, media y campañas

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- analytics
- notificaciones
- contenido de home y campañas

## Deuda conocida

- existio recientemente un problema de CSP y llamadas bloqueadas en cliente
- falta documentar mejor el ciclo popup activo, tracking y administracion

## Preguntas abiertas

- cual es la fuente de verdad de contenido marketing entre settings, popups y content manager

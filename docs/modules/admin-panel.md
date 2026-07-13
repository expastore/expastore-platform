# Admin

## Proposito

Agrupa la operacion administrativa de productos, ordenes, usuarios, descuentos, contenido, seguridad, reportes y mantenimiento.

## Frontend involucrado

- `expastore-frontend/src/components/layout/AdminLayout.tsx`
- `expastore-frontend/src/components/routes/AdminRoute.tsx`
- `expastore-frontend/src/pages/admin/*`
- `expastore-frontend/src/services/admin/*`

## Backend involucrado

- `expastore-backend/src/modules/admin/*`
- modulos especializados que exponen rutas admin

## Superficie funcional principal

- `/admin`
- `/admin/orders`
- `/admin/products`
- `/admin/users`
- `/admin/categories`
- `/admin/attributes`
- `/admin/payment-methods`
- `/admin/discounts/*`
- `/admin/popups`
- `/admin/notifications`
- `/admin/reports`
- `/admin/settings`
- `/admin/security`
- `/admin/backups`
- `/admin/content`

Nota: estas rutas del frontend representan areas del panel. El contrato HTTP oficial del backend vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- casi todos los modulos del sistema

## Automatizaciones o eventos

- alertas admin
- auditoria
- reportes
- mantenimiento
- backups

## Deuda conocida

- superficie muy grande
- conviene documentar permisos efectivos y mapeo frontend-backend
- hay mucha documentacion dispersa dentro de subcarpetas admin

## Preguntas abiertas

- como particionar el panel admin por dominios sin perder consistencia

# Catalogo y Producto

## Proposito

Gestiona catalogo publico, detalle de producto, categorias, atributos, promociones visibles y busqueda.

## Frontend involucrado

- `expastore-frontend/src/pages/Home.tsx`
- `expastore-frontend/src/pages/Catalog.tsx`
- `expastore-frontend/src/pages/ProductDetail.tsx`
- `expastore-frontend/src/pages/SearchResults.tsx`
- `expastore-frontend/src/components/products/*`
- `expastore-frontend/src/services/api.ts`

## Backend involucrado

- `expastore-backend/src/modules/products/*`
- `expastore-backend/src/modules/categories/*`
- `expastore-backend/src/modules/promotions/*`
- `expastore-backend/src/modules/coupons/*`

## Superficie funcional principal

- Frontend: `/`, `/productos`, `/categoria/:slug`, `/producto/:slug`, `/buscar`
- Backend: catalogo publico, categorias, busqueda, atributos visibles y promociones aplicables

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- pricing
- inventario y variantes
- promociones
- shipping rules por producto
- atributos para admin y PDP

## Automatizaciones o eventos

- featured deals
- productos destacados
- tracking de demanda o busquedas sin resultado

## Deuda conocida

- conviene revisar coherencia entre `services/api.ts` y servicios mas especializados
- hay bastante superficie de admin y producto repartida entre varias capas

## Preguntas abiertas

- que parte del pricing vive definitivamente en frontend y cual en backend
- como unificar mejor atributos, variantes y reglas de envio por producto

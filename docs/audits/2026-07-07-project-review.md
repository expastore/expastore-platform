# Project Review

NOTA: esta es una fotografía histórica del 7 de julio de 2026. Varios números reportados aquí ya cambiaron (ej. ProductDetail.tsx pasó de ~1535 a 977 líneas, AdminContentPagesWorkspace.tsx de ~2383 a 8 tras refactor). No usar como estado actual.

Auditoria realizada el 2026-07-07 sobre el repositorio ExpaStore.

## Resumen Ejecutivo

El proyecto esta bien separado en frontend y backend. El backend tiene una arquitectura mas madura: monolito modular con guardrails automatizados, tests amplios y cobertura importante de seguridad. El frontend esta en una migracion parcial hacia `app/pages/features/entities/shared`, pero todavia convive con una estructura legacy grande basada en `components`, `pages`, `hooks`, `services`, `types` y `utils`.

El mayor riesgo actual no es una falla estructural critica, sino la acumulacion de archivos muy grandes, responsabilidades mezcladas y duplicacion de ubicaciones durante la migracion. La recomendacion principal es consolidar la arquitectura gradualmente, con reglas para codigo nuevo y refactors pequenos por dominio.

## Tecnologia Detectada

Frontend:

- React 18, Vite, TypeScript, Tailwind.
- React Router 6, TanStack Query, Axios.
- Playwright para E2E.
- ESLint con `eslint-plugin-boundaries` en modo warning.

Backend:

- Node >= 22.12, Express, TypeScript.
- Sequelize + PostgreSQL.
- Redis opcional para rate limiting distribuido.
- Jest con `ts-jest`.
- OpenAPI/Swagger.
- Middlewares de seguridad: helmet, CORS, CSRF, sanitizacion, WAF, rate limiting, upload validation.

## Verificaciones Ejecutadas

- Backend `npm run check:architecture`: OK.
- Backend `npm run typecheck`: OK.
- Backend `npm.cmd test`: OK, 77 suites y 341 tests pasaron.
- Frontend `npm run typecheck`: OK.
- Frontend `npm run check:architecture`: OK sin warnings despues de corregir `Home.tsx`.
- Frontend E2E no ejecutado: requiere servidor/app y posiblemente estado de autenticacion. Hay suites en `expastore-frontend/e2e` y `expastore-frontend/e2e-prod`.

Nota: `npm test` fallo inicialmente en PowerShell por execution policy de `npm.ps1`; `npm.cmd test` funciono correctamente.

## Hallazgos Frontend

### Alto

Problema: Archivos de UI y paginas demasiado grandes.

- Archivos afectados:
  - `expastore-frontend/src/pages/admin/AdminContentPagesWorkspace.tsx` (~2383 lineas)
  - `expastore-frontend/src/pages/ProductDetail.tsx` (~1535 lineas)
  - `expastore-frontend/src/components/admin/sections/VariantsSection.tsx` (~1212 lineas)
  - `expastore-frontend/src/hooks/useProductForm.ts` (~1109 lineas)
- Riesgo: cambios lentos, regresiones UI, responsabilidades mezcladas y dificultad para probar.
- Solucion recomendada: extraer por dominio en pasos pequenos: `ui`, `model`, `api`, schemas y presentadores. Empezar por `useProductForm.ts` y `ProductDetail.tsx`.

Problema: Migracion FSD incompleta.

- Archivos/carpetas afectadas: `src/components`, `src/hooks`, `src/services`, `src/types`, `src/utils`, `src/features`.
- Riesgo: codigo nuevo puede seguir cayendo en carpetas legacy y aumentar la deuda.
- Solucion recomendada: para codigo nuevo usar `src/features/<dominio>/{ui,model,api}`, `src/entities` y `src/shared`. Mantener legacy funcionando y migrar solo cuando se toque el dominio.

Problema: Duplicacion transicional en infraestructura HTTP.

- Archivos afectados:
  - `expastore-frontend/src/services/apiClient.ts`
  - `expastore-frontend/src/shared/api/httpClient.ts`
  - `expastore-frontend/src/services/api/deviceIdentity.ts`
  - `expastore-frontend/src/shared/api/deviceIdentity.ts`
- Riesgo: futuros cambios pueden aplicarse a una copia incorrecta.
- Solucion recomendada: mantener `shared/api/httpClient.ts` como fuente canonica y convertir antiguos servicios en reexports hasta eliminar referencias.

### Medio

Problema: `pages/` contiene logica de dominio y admin pesada.

- Archivos afectados: `src/pages/admin/*`, `src/pages/Checkout/*`, `src/pages/ProductDetail.tsx`, `src/pages/WarrantyPage.tsx`.
- Riesgo: pages dejan de ser capa de composicion y se vuelven dificiles de reutilizar.
- Solucion recomendada: extraer hooks de estado y componentes internos por modulo, sin cambiar comportamiento.

Problema: Tipos centralizados grandes.

- Archivo afectado: `expastore-frontend/src/types/index.ts` (~688 lineas).
- Riesgo: acoplamiento amplio y conflictos al crecer dominios.
- Solucion recomendada: mover tipos hacia `entities/<dominio>/model` o `types/<dominio>.ts` cuando se toque cada flujo.

Problema: Warning de hooks. Resuelto durante esta auditoria.

- Archivo afectado: `expastore-frontend/src/pages/Home.tsx`.
- Riesgo: `useMemo` puede usar una version obsoleta de `localizeHeroBanner`.
- Solucion aplicada: `localizeHeroBanner` quedo estabilizada con `useCallback` y agregada a dependencias de `useMemo`.

### Bajo

Problema: Hay entradas duplicadas/legacy de app.

- Archivos afectados: `src/App.tsx`, `src/main.tsx`, `src/app/App.tsx`, `src/app/main.tsx`, `src/app.routes.tsx`, `src/app/router/AppRoutes.tsx`.
- Riesgo: confusion al modificar bootstrap/routing.
- Solucion recomendada: documentar cual es canonica y mantener reexports/compatibilidad hasta remover usos.

## Hallazgos Backend

### Alto

Problema: Servicios admin y controllers grandes.

- Archivos afectados:
  - `expastore-backend/src/modules/admin/services/admin-orders.service.ts` (~1641 lineas)
  - `expastore-backend/src/modules/admin/services/admin-products.service.ts` (~1012 lineas)
  - `expastore-backend/src/modules/admin/services/admin-copilot.service.ts` (~988 lineas)
  - `expastore-backend/src/modules/admin/controllers/admin-orders.controller.ts` (~690 lineas)
- Riesgo: alto costo de cambio en panel admin, mezcla de casos de uso, dificil aislamiento de permisos/auditoria.
- Solucion recomendada: extraer use cases por accion admin, manteniendo `admin.module.ts` como composition root.

Problema: Algunas zonas legacy siguen usando `controller + service + model` plano.

- Archivos/carpetas afectadas: varios modulos en `src/modules/*`, por ejemplo `settings`, `shipping`, `warranty`, `promotions`, `products`.
- Riesgo: nuevos endpoints pueden saltarse puertos/adapters si copian patron legacy.
- Solucion recomendada: no mover todo; cada cambio nuevo debe introducir o reutilizar `application/domain/infrastructure`.

### Medio

Problema: `src/services` backend conserva servicios globales transversales y legacy.

- Archivos afectados: `src/services/*.service.ts`, schedulers y proveedores.
- Riesgo: convertirse en capa de negocio paralela a `modules`.
- Solucion recomendada: servicios globales solo para infraestructura transversal. Negocio de dominio debe vivir en `src/modules/<dominio>`.

Problema: `src/app.ts` concentra mucha configuracion y montaje.

- Archivo afectado: `expastore-backend/src/app.ts`.
- Riesgo: cambios en seguridad/rutas pueden tener efectos colaterales.
- Solucion recomendada: no refactorizar de golpe. Extraer solo configuraciones puras si el archivo se vuelve bloqueante, manteniendo orden de middlewares probado.

Problema: coexistencia de `warranty` y `warranties`.

- Carpetas afectadas:
  - `src/modules/warranty`
  - `src/modules/warranties`
- Riesgo: confusion de ownership y rutas legacy.
- Solucion recomendada: documentar namespace canonico y mantener compatibilidad hasta tener migracion explicita.

### Bajo

Problema: `expastore_db` existe en la raiz.

- Archivo afectado: `expastore_db`.
- Riesgo: posible dump/base de datos en repo, peso innecesario y riesgo de datos sensibles si contiene informacion real.
- Solucion recomendada: revisar contenido y origen antes de eliminar. Si es dump, mover fuera del repo y agregar regla de ignore.

Problema: `git status` no pudo ejecutarse desde la raiz aunque existe `.git` visible.

- Archivo/carpeta afectada: `.git` o metadata de entorno.
- Riesgo: dificultad para auditar cambios pendientes.
- Solucion recomendada: verificar si la carpeta es worktree incompleto, acceso restringido o metadata corrupta antes de operaciones Git.

## Seguridad

Fortalezas:

- CSRF global con pruebas de produccion/dev.
- Rate limiting segmentado para auth, admin, checkout y pagos.
- CORS estricto en produccion.
- Sanitizacion de request y HTML con excepciones controladas.
- Validacion de uploads por magic bytes.
- Tests de IDOR, rutas admin privilegiadas, pagos, transferencias, auditoria y webhooks.

Riesgos a vigilar:

- Archivos grandes de admin y checkout pueden ocultar errores de permisos o auditoria.
- No agregar endpoints admin sin tests de autorizacion.
- No agregar uploads fuera del pipeline oficial.
- Revisar `expastore_db` por posible informacion sensible.

## Rendimiento Y Mantenibilidad

- Backend usa compression, cache/Redis opcional y rate limiters con RedisStore cuando aplica.
- Frontend usa lazy routing y TanStack Query, pero archivos grandes pueden afectar bundle y mantenibilidad.
- Traducciones en `shared/i18n/translations.ts` (~3803 lineas) son aceptables como catalogo, pero deben editarse con cuidado.
- El frontend debe evitar agregar mas estado complejo en pages o hooks globales.

## Prioridades

### Critico

No se detecto un hallazgo critico bloqueante en las verificaciones ejecutadas. Tests, typecheck y guardrails principales pasan.

### Alto

- Reducir archivos frontend grandes empezando por `useProductForm.ts`, `ProductDetail.tsx` y admin content workspace.
- Extraer casos de uso admin backend desde servicios/controllers grandes.
- Consolidar cliente HTTP frontend alrededor de `shared/api/httpClient.ts`.
- Mantener estrictamente los guardrails de backend para nuevos endpoints.

### Medio

- Migrar codigo frontend nuevo a `features/entities/shared`.
- Descomponer `src/types/index.ts`.
- Documentar canonico entre rutas/entradas legacy y nuevas.
- Revisar coexistencia `warranty`/`warranties`.

### Bajo

- Revisar `expastore_db`.
- Mantener `Home.tsx` sin warnings de hooks en futuras ediciones.
- Revisar metadata Git local.
- Mejorar documentacion de comandos Windows con `npm.cmd`.

## Propuesta De Estructura Mejorada

Frontend objetivo gradual:

```text
expastore-frontend/src/
  app/
    providers/
    router/
    routes/
  shared/
    api/
    i18n/
    lib/
    model/
    ui/
  entities/
    product/
    order/
    user/
    cart/
    payment/
  features/
    checkout/
      api/
      model/
      ui/
    products/
      api/
      model/
      ui/
    admin-products/
      api/
      model/
      ui/
    orders/
    payments/
    notifications/
  pages/
```

Backend objetivo gradual:

```text
expastore-backend/src/
  app.ts
  server.ts
  config/
  database/
  middlewares/
  modules/
    <domain>/
      http/
        <domain>.routes.ts
        <domain>.controller.ts
        <domain>.validation.ts
      application/
      domain/
      infrastructure/
      <domain>.module.ts
  services/
    infrastructure-only-or-legacy/
  utils/
```

## Plan De Refactor Gradual

1. Congelar regla: codigo nuevo frontend va a `features/entities/shared`; codigo nuevo backend pasa por use cases.
2. Mantener la correccion de `Home.tsx` y usarla como ejemplo de mejora pequena y segura.
3. Consolidar reexports de HTTP frontend y eliminar duplicados solo cuando no haya imports vivos.
4. Dividir `useProductForm.ts` en estado, validacion, payload, variantes e imagenes.
5. Dividir `ProductDetail.tsx` en presentadores y hooks de carga/seleccion.
6. Extraer `admin-orders.service.ts` por acciones: lectura, status, pagos, delivery, auditoria.
7. Extraer `AdminContentPagesWorkspace.tsx` por tabs, formularios, preview y hooks.
8. Mover tipos por dominio al tocar cada feature.
9. Documentar y eventualmente unificar `warranty`/`warranties`.
10. Subir severidad de reglas frontend solo cuando la deuda existente este migrada.

## Proximos Pasos Concretos

- Revisar `expastore_db` y decidir si debe salir del repo.
- Arreglar el warning de `Home.tsx` y ejecutar `npm run check:architecture`.
- Crear un issue/branch por cada archivo grande, no un refactor unico.
- Agregar tests frontend unitarios para helpers de checkout, pricing, formularios y permisos admin.
- Ejecutar Playwright en flujos criticos cuando se toque auth, checkout, pagos, catalogo o admin.
- Mantener `AGENTS.md` actualizado cada vez que cambien reglas de arquitectura.

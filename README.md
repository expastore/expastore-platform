# ExpaStore Platform

Repositorio liviano para la documentación transversal y las decisiones de plataforma de ExpaStore.

El backend y el frontend conservan repositorios Git independientes. En el workspace local viven como carpetas hermanas, pero este repositorio raíz no versiona su código ni reemplaza sus historiales.

## Repositorios de aplicación

- [expastore-backend](https://github.com/expastore/expastore-backend): API Express + TypeScript.
- [expastore-frontend](https://github.com/expastore/expastore-frontend): storefront y panel administrativo React + Vite.
- [docs](./docs/README.md): documentación funcional y operativa de alcance transversal.

## Fuente de verdad de la API

El contrato HTTP canónico vive en `expastore-backend/openapi/openapi.yaml` y se versiona dentro del repositorio backend:

- [OpenAPI canónico](https://github.com/expastore/expastore-backend/blob/main/openapi/openapi.yaml)
- Swagger UI local: `http://localhost:3000/api-docs`

## Arranque rápido del workspace local

Backend:

```bash
cd expastore-backend
pnpm install
cp .env.example .env
pnpm run dev
```

Frontend:

```bash
cd expastore-frontend
pnpm install
cp .env.example .env
pnpm run dev
```

## Documentación clave

- [Índice de documentación transversal](./docs/README.md)
- [Runbook de despliegue](./docs/operations/DEPLOY_RUNBOOK.md)
- [Estado de seguridad](./SECURITY_STATUS.md)
- [Auditorías históricas](./docs/audits/)

## Regla de documentación

- El OpenAPI del backend define rutas, métodos, autenticación, CSRF, payloads y respuestas.
- `docs/` explica contexto funcional, operación, decisiones transversales y deuda.
- La documentación específica de implementación vive en el repositorio propietario.
- Los archivos Markdown no deben duplicar el contrato HTTP.

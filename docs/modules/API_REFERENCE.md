# API Reference

Este archivo ya no es la fuente de verdad del contrato HTTP de ExpaStore.

## Fuente oficial actual

- Especificacion OpenAPI canonica: [expastore-backend/openapi/openapi.yaml](https://github.com/expastore/expastore-backend/blob/main/openapi/openapi.yaml)
- Swagger UI local del backend: `GET /api-docs`

## Regla del proyecto

La unica fuente de verdad para:

- rutas
- metodos HTTP
- autenticacion
- CSRF
- parametros
- request bodies
- responses
- codigos de error

es `expastore-backend/openapi/openapi.yaml`, versionado en el repositorio backend.

## Que queda en Markdown

La documentacion dentro de `docs/` y `docs/modules/` sigue siendo util para:

- contexto funcional
- decisiones tecnicas
- auditorias
- onboarding
- deuda y prioridades

pero no debe duplicar ni redefinir el contrato de la API.

## Nota de migracion

Este archivo se conserva solo como puntero estable para enlaces historicos y para evitar que reaparezca una segunda referencia manual del API.

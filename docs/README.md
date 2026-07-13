# ExpaStore Platform Docs

Documentación vigente y curada de alcance transversal para ExpaStore.

## Fuente de verdad

El contrato HTTP canónico se versiona en el repositorio backend:

- [expastore-backend/openapi/openapi.yaml](https://github.com/expastore/expastore-backend/blob/main/openapi/openapi.yaml)
- Swagger UI local: `GET /api-docs`

Los archivos Markdown de esta carpeta no deben listar endpoints como contrato oficial.

## Documentación transversal

### Operación

- [`operations/DEPLOY_RUNBOOK.md`](./operations/DEPLOY_RUNBOOK.md)
- [`operations/GITHUB_ACTIONS_SECRETS.md`](./operations/GITHUB_ACTIONS_SECRETS.md)

### Producto y módulos

- [`modules/README.md`](./modules/README.md)
- [`modules/API_REFERENCE.md`](./modules/API_REFERENCE.md)
- Los documentos funcionales de cada dominio se encuentran en [`modules/`](./modules/).

### Planes

- [`plans/AGENT_PLATFORM_MVP_EXECUTION_PLAN.md`](./plans/AGENT_PLATFORM_MVP_EXECUTION_PLAN.md)
- [`plans/AGENT_PLATFORM_PHASE2_IMPROVEMENT_PLAN.md`](./plans/AGENT_PLATFORM_PHASE2_IMPROVEMENT_PLAN.md)

### Auditorías

- [`audits/2026-07-07-project-review.md`](./audits/2026-07-07-project-review.md)

## Documentación propiedad de los repos de aplicación

- [Documentación backend](https://github.com/expastore/expastore-backend/tree/main/docs)
- [Documentación frontend](https://github.com/expastore/expastore-frontend/tree/main/docs)

## Criterio de permanencia

Un documento debe quedarse en `docs/` solo si cumple al menos una de estas funciones:

- define una decision vigente
- guia una operacion actual
- documenta un modulo vivo
- actua como puntero estable a una fuente de verdad actual

Los reportes puntuales, planes ya ejecutados y auditorías históricas se archivan o eliminan cuando dejan de ser útiles para operar o desarrollar el sistema.

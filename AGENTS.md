# AGENTS.md — ExpaStore Platform

Este repositorio raíz versiona exclusivamente documentación transversal de la
plataforma ExpaStore. No contiene ni gobierna el código fuente del backend o del
frontend.

Los repositorios de código son independientes:

- `expastore-backend/` — Express, TypeScript, Sequelize, PostgreSQL y Redis.
- `expastore-frontend/` — React, Vite, TypeScript, Tailwind y React Query.

Cada repositorio hijo tiene su propio `AGENTS.md`. Antes de modificar código,
configuración, pruebas o documentación interna de uno de ellos, se deben seguir
las instrucciones de ese repositorio.

## Alcance del repositorio raíz

Aquí viven:

- `README.md`: visión general y enlaces a los repositorios de la plataforma.
- `SECURITY_STATUS.md`: estado transversal vigente de seguridad.
- `docs/operations/`: runbooks, despliegues, secretos y operación compartida.
- `docs/plans/`: planes transversales y hojas de ruta.
- `docs/modules/`: documentación funcional que cruza backend y frontend.
- `docs/audits/`: fotografías históricas, revisiones y auditorías fechadas.
- `AGENTS.md`: reglas para mantener esta documentación transversal.

## Fuera de alcance

Este repositorio no versiona:

- código del backend;
- código del frontend;
- contratos o configuraciones internos que pertenezcan a un solo repositorio;
- dependencias, builds, cachés o artefactos generados;
- dumps, backups o datos reales;
- configuración local de herramientas.

Las instrucciones sobre arquitectura de código, límites de funciones,
componentes React, servicios backend, tests o convenciones de implementación no
aplican en este repositorio. Esas reglas pertenecen a los `AGENTS.md` de
`expastore-backend/` y `expastore-frontend/`.

La presencia física de los repositorios hijos dentro de esta carpeta no los
convierte en contenido del repositorio raíz. No se deben preparar ni commitear
sus archivos desde el repositorio raíz.

## Fuentes de verdad

La documentación transversal resume o enlaza el estado de los repositorios
hijos, pero no sustituye sus fuentes canónicas.

- El contrato HTTP canónico vive en
  `expastore-backend/openapi/openapi.yaml`.
- La arquitectura y las convenciones de implementación viven en cada
  repositorio hijo.
- Los commits, tests, configuraciones y código reales prevalecen sobre cualquier
  resumen transversal.
- Si un documento raíz contradice una fuente canónica, se debe corregir el
  documento raíz y conservar un enlace a la evidencia.

No se deben copiar contratos técnicos completos hacia la raíz cuando puedan
enlazarse desde su repositorio propietario.

## Mantenimiento de `SECURITY_STATUS.md`

`SECURITY_STATUS.md` es un documento vigente, no una auditoría histórica. Debe
actualizarse cuando un hallazgo transversal cambie de estado.

Antes de mover un hallazgo a “Resuelto”:

1. Identificar el repositorio propietario.
2. Verificar el cambio en código o configuración real.
3. Registrar el hash del commit que contiene la corrección.
4. Confirmar las pruebas o verificaciones relevantes.
5. Registrar la fecha de resolución.

Un hallazgo no se considera resuelto solo porque exista un plan, un comentario o
un cambio local sin commit. Las validaciones que requieran infraestructura real
deben permanecer en “Pendiente” hasta que exista evidencia del entorno
correspondiente.

Al cerrar trabajo de seguridad en un repositorio hijo, revisar también
`SECURITY_STATUS.md` en la misma sesión. Si no se puede actualizar todavía,
dejar explícitamente registrada la necesidad de sincronización.

No mantener documentos paralelos como `SECURITY_PENDING.md`. El estado vivo debe
tener una sola fuente transversal.

## Mantenimiento de auditorías

Los documentos de `docs/audits/` son fotografías fechadas. No deben presentarse
como estado actual indefinidamente.

Cada auditoría debe incluir:

- fecha de realización;
- alcance;
- repositorios o commits revisados;
- hallazgos y evidencia;
- aclaración de que representa una fotografía histórica.

Cuando una auditoría quede desactualizada:

- conservar el documento original como registro histórico;
- añadir una nota visible si sus cifras o conclusiones ya no son vigentes;
- enlazar el estado actual o una auditoría posterior;
- no reescribir silenciosamente los resultados históricos como si siempre
  hubieran sido actuales.

Los estados vivos, como seguridad u operación, deben actualizarse en sus
documentos correspondientes; las auditorías deben conservar la evidencia del
momento en que fueron realizadas.

## Organización documental

Usar estas ubicaciones:

- procedimientos ejecutables y runbooks → `docs/operations/`;
- planes futuros o por fases → `docs/plans/`;
- visión funcional transversal por dominio → `docs/modules/`;
- revisiones fechadas y evidencia histórica → `docs/audits/`.

Antes de crear un documento nuevo, comprobar que no exista ya uno con el mismo
propósito. Si el contenido pertenece exclusivamente al backend o al frontend,
debe vivir en el repositorio hijo correspondiente.

## Convenciones

- Escribir la documentación en español, salvo nombres técnicos o contratos que
  deban conservar su forma original.
- Usar fechas ISO `YYYY-MM-DD`.
- Usar `pnpm` en los comandos actuales de backend y frontend.
- Enlazar archivos o commits propietarios en vez de duplicar detalles.
- Distinguir claramente entre estado vigente, plan futuro y registro histórico.
- No incluir secretos, tokens, datos personales, dumps ni información de
  clientes.
- Verificar enlaces y rutas después de mover documentos.

## Flujo de trabajo

Antes de cerrar un cambio en este repositorio:

1. Confirmar que solo se modificó documentación transversal.
2. Contrastar afirmaciones técnicas con el repositorio hijo correspondiente.
3. Revisar si `SECURITY_STATUS.md` o una auditoría relacionada debe
   sincronizarse.
4. Comprobar que los enlaces Markdown locales resuelven.
5. Revisar `git diff` y `git status`.
6. Confirmar que `expastore-backend/`, `expastore-frontend/` y archivos locales
   excluidos no aparecen en staging.

Los cambios en este repositorio no sustituyen las verificaciones requeridas por
los repositorios hijos cuando también se modifica alguno de ellos.

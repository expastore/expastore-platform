# Estado de seguridad

Fecha de actualización: 2026-07-25

Este documento resume los hallazgos de seguridad resueltos recientemente y las validaciones que todavía requieren un entorno externo o infraestructura real.

## Resuelto

### Rotación atómica de refresh tokens

- Fecha: 2026-07-13.
- Se implementó rotación atómica con semántica compare-and-swap para impedir reutilización concurrente.
- Referencia backend: commit `c7df430`.

### Endurecimiento CORS y CSRF para previews

- Fecha: 2026-07-13.
- Los previews de Vercel dejaron de considerarse orígenes confiables automáticamente.
- Referencia backend: commit `85a83ea`.

### Sanitización de URLs protocol-relative

- Fecha: 2026-07-13.
- Se rechazan URLs que comienzan con `//` para evitar redirecciones abiertas.
- Referencia backend: commit `9140b7e`.

### Sanitización acumulativa de texto plano en CMS pages

- Fecha: 2026-07-25.
- `sanitizeString` no era idempotente: aplicarlo dos veces sobre el mismo valor producía crecimiento acumulativo de entidades HTML, causando corrupción visible desde el primer guardado y truncamiento o pérdida de datos desde el segundo guardado de una página CMS.
- Se corrigió `sanitizeString` para ser realmente idempotente mediante decodificación antes de escapar, como red de seguridad global.
- Se rediseñó la sanitización de `POST /api/v1/admin/settings/pages`: el servicio de dominio (`SitePageService`) es ahora la única frontera de sanitización, con texto plano almacenado canónico, sin escape HTML, y escape aplicado solo en el punto de salida correspondiente.
- Referencia backend: commits `564a4a8` (idempotencia global) y `8ee8f5b` (rediseño de settings/pages).

### TLS estricto de PostgreSQL en producción

- Fecha: 2026-07-13.
- La configuración productiva exige verificación TLS estricta para PostgreSQL.
- Referencia backend: commit `6835939`.

## Pendiente

### Auditoría externa de dependencias backend

El backend no ha sido validado contra el servicio externo de auditoría npm en este entorno. Ejecutar `pnpm audit --audit-level=moderate` en un entorno confiable antes del siguiente rollout productivo y documentar el resultado.

### Release readiness contra producción real

`release-readiness-check.ts` valida condiciones locales y estructurales, pero no ejecuta una comprobación completa contra el despliegue productivo real. Mantener una validación post-deploy autenticada y separada.

### Mismo patrón de doble sanitización en otros módulos

El patrón de “middleware global reescribe texto, luego el servicio de dominio vuelve a procesarlo” también afecta, con distinto grado de severidad, a:

- popups (`title`, `message`, `buttonText`);
- payment-methods (descripción e instrucciones);
- moderación de reseñas (respuestas y motivos);
- admin/settings legado (`update-settings.use-case`);
- blog-posts (acumulativo entre ediciones, aunque sin doble escape en la misma petición).

La idempotencia global de `sanitizeString` (`564a4a8`) actúa como red de seguridad para todos estos módulos: detiene el crecimiento de entidades si se guarda repetidamente. Sin embargo, ninguno recibió todavía el rediseño completo —separación de normalización de almacenamiento y escape de salida— aplicado a settings/pages. Queda pendiente revisar módulo por módulo, priorizando por riesgo de negocio.

### Integración SMTP real

La integración usa `nodemailer` v9 en runtime mientras `@types/nodemailer` permanece en v6 y el paquete se carga con `require()`. Falta validar STARTTLS y autenticación contra un relay SMTP real antes de considerar cerrada esta verificación.

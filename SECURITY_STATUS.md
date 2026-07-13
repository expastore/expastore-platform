# Estado de seguridad

Fecha de actualización: 2026-07-13

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

### TLS estricto de PostgreSQL en producción

- Fecha: 2026-07-13.
- La configuración productiva exige verificación TLS estricta para PostgreSQL.
- Referencia backend: commit `6835939`.

## Pendiente

### Auditoría externa de dependencias backend

El backend no ha sido validado contra el servicio externo de auditoría npm en este entorno. Ejecutar `pnpm audit --audit-level=moderate` en un entorno confiable antes del siguiente rollout productivo y documentar el resultado.

### Release readiness contra producción real

`release-readiness-check.ts` valida condiciones locales y estructurales, pero no ejecuta una comprobación completa contra el despliegue productivo real. Mantener una validación post-deploy autenticada y separada.

### Integración SMTP real

La integración usa `nodemailer` v9 en runtime mientras `@types/nodemailer` permanece en v6 y el paquete se carga con `require()`. Falta validar STARTTLS y autenticación contra un relay SMTP real antes de considerar cerrada esta verificación.

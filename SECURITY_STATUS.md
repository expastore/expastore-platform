# Estado de seguridad

Fecha de actualización: 2026-07-28

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

- Fecha: 2026-07-13 (backend), extendido 2026-07-28 (frontend).
- Se rechazan URLs que comienzan con `//` para evitar redirecciones abiertas.
- Referencia backend: commit `9140b7e`.
- 2026-07-28: el mismo rechazo se extendió al frontend: `useSanitizedHTML.ts` (atributos `href`/`src` dentro de HTML sanitizado en CMS/blog), `PopupManager.tsx` (URL del botón CTA de popups, que antes también aceptaba cualquier esquema no-`http`, incluido `javascript:`, vía `window.location.href`), y `notificationNavigation.ts` (normalización de `actionUrl`, que además ahora preserva query string y hash al convertir una URL absoluta a ruta relativa).
- Referencia frontend: commit `60e8b1b`.

### Sanitización acumulativa de texto plano en CMS pages

- Fecha: 2026-07-25.
- `sanitizeString` no era idempotente: aplicarlo dos veces sobre el mismo valor producía crecimiento acumulativo de entidades HTML, causando corrupción visible desde el primer guardado y truncamiento o pérdida de datos desde el segundo guardado de una página CMS.
- Se corrigió `sanitizeString` para ser realmente idempotente mediante decodificación antes de escapar, como red de seguridad global.
- Se rediseñó la sanitización de `POST /api/v1/admin/settings/pages`: el servicio de dominio (`SitePageService`) es ahora la única frontera de sanitización, con texto plano almacenado canónico, sin escape HTML, y escape aplicado solo en el punto de salida correspondiente.
- Referencia backend: commits `564a4a8` (idempotencia global) y `8ee8f5b` (rediseño de settings/pages).

### Sanitización acumulativa de texto plano en blog-posts

- Fecha: 2026-07-28.
- Mismo patrón y mismo rediseño que CMS pages (ver arriba), aplicado a `POST /api/v1/admin/settings/blog-posts`. El middleware global reescribía el body completo con `sanitizeString`, que elimina TODO tag HTML: el contenido enriquecido del editor (Tiptap `RichTextEditor` → `SafeRichHTML`) perdía su formato en cada guardado, no solo sufría escalado de entidades.
- La ruta queda excluida de la reescritura global del body; `blogPost.service.ts` expone una función pura `normalizeBlogPostPayload` como única frontera de sanitización: `normalizePlainText` para texto plano (título, slug, excerpt, autor, categoría, tags, SEO), `sanitizeHtml` para el contenido enriquecido y `sanitizeUrl` para `coverImage`/`ogImage`.
- Validado con invariantes de idempotencia, un test HTTP de tres guardados consecutivos, un script contra la base de datos real de desarrollo (tres ciclos de guardado + lectura directa a `blog_posts`), y una prueba manual en el navegador (sesión admin real, tres guardados vía UI) confirmando que texto con `&`/`"`/`24/7` y contenido `<strong>` se preservan sin degradación.
- Referencia backend: commit `aa63830`.

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
- admin/settings legado (`update-settings.use-case`).

blog-posts recibió el rediseño completo el 2026-07-28 (ver “Resuelto” arriba, commit `aa63830`) y sale de esta lista.

La idempotencia global de `sanitizeString` (`564a4a8`) actúa como red de seguridad para todos los módulos restantes: detiene el crecimiento de entidades si se guarda repetidamente. Sin embargo, ninguno recibió todavía el rediseño completo —separación de normalización de almacenamiento y escape de salida— aplicado a settings/pages y blog-posts. Queda pendiente revisar módulo por módulo, priorizando por riesgo de negocio.

### Restricción de tags por nivel de sanitización no efectiva en el frontend (DOMPurify)

- Detectado: 2026-07-28, durante la verificación de la Tarea 5 de esta sesión (cambio de `SafeAdminHTML` a `SafeRichHTML` en `ContentPageBody.tsx`, `PageSectionRenderer.tsx`, `BlogPostPage.tsx`).
- `DEFAULT_OPTIONS.USE_PROFILES` en `expastore-frontend/src/shared/lib/useSanitizedHTML.ts` hace que DOMPurify descarte el `ALLOWED_TAGS`/`ALLOWED_ATTR` explícito de cualquier `sanitizeLevel` (`RICH_TEXT`, `BASIC_HTML`, etc.) y aplique en su lugar el perfil completo `"html"` (que sí incluye `img`, `video`, `audio`, etc.), quedando acotado solo por `FORBID_TAGS`/`FORBID_ATTR`. Confirmado instrumentando `useSanitizedHTML`/`SafeHTML` en el navegador: con `ALLOWED_TAGS` sin `img` en la config resuelta, el HTML sanitizado final conserva el `<img>` de todas formas.
- Impacto práctico hoy es bajo porque todo el contenido que pasa por `SafeRichHTML`/`SafeBasicHTML` es autorado por administradores autenticados (CMS, blog, popups), no por usuarios finales. Pero la restricción de tags de estos niveles es, en la práctica, teatro de seguridad: no reduce la superficie de ataque frente a un admin comprometido o datos corruptos más allá de lo que ya bloquea `FORBID_TAGS`.
- No corregido en esta sesión (excede el alcance de la tarea que lo encontró y requiere revisar el comportamiento esperado de cada nivel antes de tocarlo). Pendiente de decisión: lo más probable es eliminar `USE_PROFILES` de `DEFAULT_OPTIONS` y depender solo de `ALLOWED_TAGS`/`ALLOWED_ATTR` explícitos.

### Integración SMTP real

La integración usa `nodemailer` v9 en runtime mientras `@types/nodemailer` permanece en v6 y el paquete se carga con `require()`. Falta validar STARTTLS y autenticación contra un relay SMTP real antes de considerar cerrada esta verificación.

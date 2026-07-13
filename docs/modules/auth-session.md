# Auth y Sesion

## Proposito

Gestiona registro, login por PIN, activacion, sesion por cookies, refresh token, logout y seguridad basica de acceso.

## Frontend involucrado

- `expastore-frontend/src/context/AuthContext.tsx`
- `expastore-frontend/src/pages/Login.tsx`
- `expastore-frontend/src/pages/Register.tsx`
- `expastore-frontend/src/pages/Activate.tsx`
- `expastore-frontend/src/pages/VerifyPIN.tsx`
- `expastore-frontend/src/services/apiClient.ts`

## Backend involucrado

- `expastore-backend/src/modules/auth/*`
- `expastore-backend/src/modules/sessions/*`
- `expastore-backend/src/modules/users/*`
- `expastore-backend/src/middlewares/auth.middleware.ts`
- `expastore-backend/src/middlewares/csrf.middleware.ts`

## Superficie funcional principal

- Frontend: `/login`, `/login/verify`, `/register`, `/activate`
- Backend: auth passwordless, perfil actual y emision/validacion de CSRF

Nota: el detalle contractual exacto de rutas, metodos y payloads vive en `expastore-backend/openapi/openapi.yaml`.

## Dependencias cruzadas

- cookies de auth
- CSRF
- CORS
- settings de frontend domain y cookie domain
- `AuthContext` y refresh automatizado del cliente HTTP

## Automatizaciones o eventos

- refresh de token
- broadcast de cambios de auth entre tabs
- socket connect/disconnect segun sesion

## Deuda conocida

- hay errores TypeScript pendientes en `Login.tsx`
- recientemente se agrego logging extra para fallos de refresh en backend
- la estabilidad de sesion depende de configuracion correcta de Vercel y Railway

## Preguntas abiertas

- conviene consolidar la estrategia de refresh para multi-tab
- faltan docs de ciclo completo de cookies, refresh y expiracion

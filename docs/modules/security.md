# Security

## Proposito

Concentra autenticacion, CSRF, cookies, WAF, rate limiting, dispositivos confiables, incidentes y endurecimiento de produccion.

## Frontend involucrado

- `expastore-frontend/src/context/AuthContext.tsx`
- `expastore-frontend/src/pages/AccountSecurityExperienceV2Page.tsx`
- `expastore-frontend/src/components/security/TurnstileCaptcha.tsx`

## Backend involucrado

- `expastore-backend/src/middlewares/*`
- `expastore-backend/src/services/account-security.service.ts`
- `expastore-backend/src/services/ip-intelligence.service.ts`
- `expastore-backend/src/modules/security/*`
- `expastore-backend/src/modules/users/*`

## Capas principales

- auth y refresh token
- cookies
- CSRF
- CORS
- CAPTCHA
- WAF
- Cloudflare protection
- IP denylist
- rate limiting
- incidentes y dispositivos

## Dependencias cruzadas

- auth
- admin
- email de alertas
- notifications

## Deuda conocida

- la configuracion de auth en produccion requiere documentacion operativa mas detallada
- hay historial reciente de ajustes en CSP, cookies y same-origin

## Preguntas abiertas

- falta un runbook de incidentes y diagnostico de sesiones

# GitHub Actions: secrets y variables

Esta guia refleja los workflows reales que existen hoy en el repo.

## Workflows actuales

### Frontend

- `expastore-frontend/.github/workflows/deploy.yml`
- `expastore-frontend/.github/workflows/manual-deploy.yml`
- `expastore-frontend/.github/workflows/secret-scan.yml`

### Backend

- `expastore-backend/.github/workflows/deploy.yml`
- `expastore-backend/.github/workflows/manual-deploy.yml`
- `expastore-backend/.github/workflows/secret-scan.yml`

## Quality CI

Los workflows `deploy.yml` de frontend y backend son CI de calidad, no deploy real.

No necesitan secrets externos para correr en verde porque:

- el frontend genera `.env` desde `.env.example`
- Playwright corre smoke con mocks
- el backend inyecta variables minimas de desarrollo dentro del workflow

## Secret scan

Los workflows `secret-scan.yml` usan `gitleaks/gitleaks-action@v2`.

No requieren secrets custom. Solo usan:

- `GITHUB_TOKEN`

## Deploy manual

Los workflows `manual-deploy.yml` si dependen de GitHub Environments y secrets reales.

### Frontend

Secrets requeridos:

- `VERCEL_TOKEN`
- `VERCEL_ORG_ID`
- `VERCEL_PROJECT_ID`

Environment usados:

- `staging`
- `Production`

### Backend

Secrets requeridos:

- `RAILWAY_TOKEN`

Environment usados:

- `staging`
- `production`

## Variables de aplicacion

Los workflows de deploy manual autentican contra Vercel o Railway, pero no reemplazan las variables de entorno de la aplicacion.

Esas variables deben existir en:

- Vercel para frontend
- Railway para backend

Referencias:

- [`ENVIRONMENT_VARIABLES.md`](https://github.com/expastore/expastore-backend/blob/main/docs/ENVIRONMENT_VARIABLES.md)
- [`DEPLOY_RUNBOOK.md`](./DEPLOY_RUNBOOK.md)

## Regla operativa

1. Mantener CI de calidad sin dependencia de secrets externos.
2. Mantener secret scanning separado de CI y de deploy.
3. Mantener deploy manual aislado por environment con aprobaciones si aplica.
4. No usar GitHub Actions como sustituto del inventario real de variables de Vercel o Railway.

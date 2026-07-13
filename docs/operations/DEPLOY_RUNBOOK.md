# ExpaStore Runbook

Secuencia operativa vigente para deploy, validacion y rollback.

## Pre-release

### Frontend

```bash
cd expastore-frontend
pnpm run release:check
pnpm run build:check
pnpm run test:e2e
```

### Backend

```bash
cd expastore-backend
pnpm run release:check
pnpm run check:architecture
pnpm run build
pnpm run test:ci
```

## Deploy manual

### Frontend en Vercel

Preview:

```bash
cd expastore-frontend
pnpm run env:pull:preview
pnpm run deploy:preview
```

Produccion:

```bash
cd expastore-frontend
pnpm run env:pull:prod
pnpm run deploy:prod
```

### Backend en Railway

Staging:

```bash
cd expastore-backend
pnpm run deploy:staging
```

Produccion:

```bash
cd expastore-backend
pnpm run deploy:prod
```

## GitHub Actions

Los workflows reales viven en:

- `expastore-frontend/.github/workflows/deploy.yml`
- `expastore-frontend/.github/workflows/manual-deploy.yml`
- `expastore-frontend/.github/workflows/secret-scan.yml`
- `expastore-backend/.github/workflows/deploy.yml`
- `expastore-backend/.github/workflows/manual-deploy.yml`
- `expastore-backend/.github/workflows/secret-scan.yml`

## Health checks

### Backend

- endpoint publico: `GET /health`
- esperado: `200 OK`
- revisar logs de arranque para Redis, cookies, CORS y webhooks

Ejemplo:

```bash
curl -I https://api.expastore.com/health
```

### Frontend

- abrir `https://www.expastore.com`
- confirmar que home carga sin `404/500`
- confirmar que `/api/*` y `/socket.io/*` siguen funcionando same-origin

## Smoke manual post-deploy

1. Login passwordless.
2. Catalogo y PDP.
3. Carrito y checkout.
4. Orden de transferencia.
5. Admin `deliveries` y validacion de PIN.

## Rollback

### Vercel

1. Abrir el dashboard del proyecto.
2. Seleccionar el deployment estable anterior.
3. Promoverlo a produccion.
4. Revalidar home, login y checkout.

### Railway

1. Abrir el servicio `expastore-backend`.
2. Revertir al deployment estable anterior desde el historial.
3. Confirmar que las variables de entorno no cambiaron entre releases.
4. Validar `GET /health` y logs de arranque.

## Incidentes de configuracion frecuentes

- `ALLOWED_ORIGINS` o `CSRF_ALLOWED_ORIGINS` no incluyen el dominio real
- cookies `Secure` o `SameSite` no alineadas con frontend/backend
- `WEB_CONCURRENCY > 1` sin Redis
- `VITE_API_URL` productivo apuntando a localhost
- Turnstile habilitado sin claves validas
- deploy con `RAILWAY_TOKEN` o `VERCEL_*` faltantes en GitHub Environments

## Backup y restore

1. Confirmar que existe un backup reciente antes del deploy.
2. Si el incidente afecta datos, no redeployar antes de revisar restore.
3. Probar restore en entorno seguro antes de tocar produccion cuando sea posible.

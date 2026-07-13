# Modulos

Indice base para el analisis modular de ExpaStore.

## Importante

Los archivos de esta carpeta ya no documentan el contrato HTTP oficial.

Si necesitas confirmar:

- rutas
- metodos
- auth
- CSRF
- requests
- responses

usa [expastore-backend/openapi/openapi.yaml](https://github.com/expastore/expastore-backend/blob/main/openapi/openapi.yaml).

## Convencion

Cada modulo debe documentar:

- proposito
- frontend involucrado
- backend involucrado
- flujos o endpoints relevantes a nivel funcional, sin redefinir el contrato oficial
- dependencias cruzadas
- automatizaciones o eventos
- deuda conocida
- preguntas abiertas

## Modulos iniciales

- [`auth-session.md`](./auth-session.md)
- [`catalog-product.md`](./catalog-product.md)
- [`checkout-payments.md`](./checkout-payments.md)
- [`orders.md`](./orders.md)
- [`shipping-addresses.md`](./shipping-addresses.md)
- [`notifications-alerts.md`](./notifications-alerts.md)
- [`admin-panel.md`](./admin-panel.md)
- [`security.md`](./security.md)
- [`content-popups.md`](./content-popups.md)
- [`email-management.md`](./email-management.md)
- [`warranty-claims.md`](https://github.com/expastore/expastore-backend/blob/main/docs/warranty-claims.md)

## Prioridad sugerida de analisis

1. Auth y sesion
2. Checkout y pagos
3. Orders
4. Shipping y direcciones
5. Notifications y alerts
6. Admin
7. Catalogo y producto
8. Security
9. Content y popups
10. Email management

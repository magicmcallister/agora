# Agora

Backend de e-commerce construido con **DDD + Arquitectura Hexagonal** en
Python/FastAPI. Monorepo con varios *bounded contexts* desplegables de forma
independiente, comunicados por eventos de dominio. Despliegue 100% local con
Docker Compose (sin infraestructura cloud).

## Bounded contexts

| Contexto | Responsabilidad | Agregados |
|---|---|---|
| `identity` | Usuarios, autenticación, perfiles de cliente | `User`, `Customer` |
| `orders` | Catálogo, carrito, pedidos, cupones | `Product`, `Category`, `Cart`, `Order`, `Coupon` |
| `notifications` | Avisos generados a partir de eventos | `Notification`, `NotificationTemplate` |

Librerías compartidas:

- `shared` — kernel: `AggregateRoot`, `DomainEvent`, `EventBus`, `Criteria`,
  value objects base, `SQLAlchemyDatabase`.
- `authorization` — RBAC/JWT stateless.

## Estado

En diseño. El documento de diseño está en
[`docs/superpowers/specs/2026-06-15-agora-backend-design.md`](docs/superpowers/specs/2026-06-15-agora-backend-design.md).

## Stack

Python 3.12 · FastAPI · SQLAlchemy (async) · asyncpg · Alembic ·
pydantic-settings · PyJWT · uv · pytest · Docker Compose.

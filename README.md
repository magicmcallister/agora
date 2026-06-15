# Agora

E-commerce backend built with **DDD + Hexagonal Architecture** in Python/FastAPI.
A monorepo of several independently deployable *bounded contexts* that
communicate through domain events. 100% local deployment with Docker Compose
(no cloud infrastructure).

## Bounded contexts

| Context | Responsibility | Aggregates |
|---|---|---|
| `identity` | Users, authentication, customer profiles | `User`, `Customer` |
| `orders` | Catalog, cart, orders, coupons | `Product`, `Category`, `Cart`, `Order`, `Coupon` |
| `notifications` | Alerts generated from domain events | `Notification`, `NotificationTemplate` |

Shared libraries:

- `shared` — kernel: `AggregateRoot`, `DomainEvent`, `EventBus`, `Criteria`,
  base value objects, `SQLAlchemyDatabase`.
- `authorization` — stateless RBAC/JWT.

## Status

Under development.

## Stack

Python 3.12 · FastAPI · SQLAlchemy (async) · asyncpg · Alembic ·
pydantic-settings · PyJWT · uv · pytest · Docker Compose.

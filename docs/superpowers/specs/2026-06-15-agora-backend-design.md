# Agora — Backend e-commerce (DDD + Hexagonal)

> Spec de diseño. Réplica simplificada de `layers-backend` aplicada al dominio
> e-commerce, **sin infraestructura cloud** (sin Terraform/Terragrunt, AWS,
> LocalStack, Mangum/Lambda). Modelo de dominio **rico** en reglas de negocio.

- **Fecha:** 2026-06-15
- **Estado:** Aprobado para planificar (pendiente revisión final del usuario)
- **Producto/monorepo:** `agora`

---

## 1. Objetivo y alcance

Construir un monorepo Python/FastAPI con **DDD + Arquitectura Hexagonal** que
reproduce los patrones de `layers-backend` pero:

- Aplicado a **e-commerce/pedidos** con nombres propios.
- **Sin nada de infra cloud**: el despliegue es 100% local con Docker Compose.
- Con un **modelo de dominio rico**: value objects con invariantes y
  comportamiento, agregados con máquinas de estado y reglas de negocio reales
  (no CRUD anémico), todo documentado con docstrings que explican invariantes.

### Incluye

1. **Users + autenticación** (contexto `identity`).
2. **Autorización** RBAC por recurso, simplificada respecto a layers (sin
   permisos por organización), como librería stateless (`authorization`).
3. **Alembic** por bounded context (migraciones independientes).
4. **Makefile** + Docker Compose para levantar todo en local.
5. **Eventos de dominio** (internos en proceso + externos entre contextos).
6. Un **segundo bounded context** independiente (`orders`) que se levanta solo.
7. Un **bounded context de notificaciones** (`notifications`) que reacciona a
   eventos de los demás contextos.

### No incluye (fuera de alcance)

- Terraform/Terragrunt, AWS, LocalStack, API Gateway, Lambda/Mangum.
- Frontend.
- Pasarela de pago/email reales (se simulan con stubs que loguean).

---

## 2. Arquitectura general

Monorepo con **3 bounded contexts desplegables de forma independiente** más dos
librerías compartidas:

| Componente | Tipo | Responsabilidad |
|---|---|---|
| `shared` | librería (kernel) | `AggregateRoot`, `DomainEvent`, `EventBus`, `Criteria`, VOs base, `SQLAlchemyDatabase`, manejo de errores |
| `authorization` | librería | RBAC/JWT stateless (`PermissionValidator`, codec de permisos) |
| `identity` | BC desplegable | Usuarios, autenticación, perfiles de cliente |
| `orders` | BC desplegable | Catálogo, carrito, pedidos, cupones |
| `notifications` | BC desplegable | Avisos generados a partir de eventos |

Capas por contexto (interior → exterior):
`domain/` → `application/` → `infrastructure/` → `fastapi_api/`.

### Reglas de arquitectura (no violar, idénticas a layers)

- La capa `domain` no tiene I/O (ni BD, ni HTTP, ni llamadas externas).
- Los agregados se referencian entre sí **por ID**, nunca por objeto.
- Las mutaciones de estado se hacen **solo vía métodos del agregado**.
- Casos de uso de **responsabilidad única** (una operación por clase).
- Interfaces de repositorio en `domain`, implementaciones en `infrastructure`.
- Tras `repo.save()`, **publicar eventos** vía `event_bus`.
- **Un agregado por transacción**.

---

## 3. Estructura de carpetas

```
backend/
├── shared/                      # Kernel compartido (montado como volumen en cada BC)
│   ├── config/                  # logger, settings base, SENTINEL
│   └── context/
│       ├── domain/
│       │   ├── value_objects/   # Uuid, EmailAddress, Datetime, String, Money, ...
│       │   ├── criteria/        # Criteria, Filter, Order (filtros dinámicos)
│       │   ├── event/           # DomainEvent, EventBus, DomainEventSubscriber, failover
│       │   └── aggregate_root.py
│       ├── application/         # excepciones, data_inconsistency, failover consumer
│       └── infrastructure/
│           ├── persistence/     # SQLAlchemyDatabase (async), modelos base, bus models
│           ├── event_bus/       # in_memory, postgres_stream, hybrid, failover, (de)serializers
│           └── criteria/        # criteria → SQLAlchemy, parser de query params
├── authorization/
│   └── context/domain/          # PermissionValidator, VOs de permisos, codec JWT
├── identity/                    # BC: usuarios + auth
│   ├── context/{user,customer,shared}/{domain,application,infrastructure}
│   ├── fastapi_api/{user,customer}/{router,payloads,responses}.py
│   │   ├── main.py
│   │   └── dependencies.py
│   ├── config/{config.py,container.py}
│   ├── migrations/{env.py,versions/}
│   ├── scripts/{db/load_fixtures.py, events/consumer.py, fixtures/}
│   ├── docker/{Dockerfile,docker-compose.yml}
│   ├── tests/{unit,integration,mothers}
│   ├── alembic.ini
│   ├── .env.example
│   └── Makefile
├── orders/                      # BC: catálogo + carrito + pedidos (misma forma)
├── notifications/               # BC: notificaciones (misma forma)
├── docker-compose.yml           # Postgres único + red compartida
├── Makefile                     # Orquestador raíz
├── pyproject.toml               # uv, deps, pytest, coverage
├── uv.lock
└── README.md
```

Cada BC tiene su propio `main.py`, `config`, Alembic, `Makefile` y
`docker-compose.yml` → **se levanta solo**. `shared` y `authorization` se montan
como volúmenes en cada contenedor (hot-reload), igual que en layers.

---

## 4. Modelo de dominio (rico en reglas de negocio)

### 4.1 Value Objects compartidos / de dominio

Todos los VO son `frozen`, validan en construcción y **encapsulan comportamiento**.
Cada uno lleva docstring con sus invariantes.

| Value Object | Contexto | Invariantes / reglas | Comportamiento |
|---|---|---|---|
| `Money` | shared | `Decimal` con escala por divisa (ISO 4217); precios ≥ 0 | `add`/`subtract` (misma divisa → `CurrencyMismatch`), `multiply(qty)`, `apply_percentage`, `allocate` |
| `Currency` | shared | código ISO 4217 válido | escala decimal por divisa |
| `EmailAddress` | shared | formato RFC; normaliza a minúsculas | — |
| `Uuid` | shared | UUID válido (v4) | `generate_random_valid_uuid()` |
| `Datetime` | shared | tz-aware | parse/format |
| `Sku` | orders | patrón `XXX-0000` en mayúsculas | normaliza a mayúsculas |
| `Slug` | orders | URL-safe, minúsculas, sin acentos, longitud máx | `from_name()` |
| `Quantity` | orders | entero ≥ 1, tope máximo por línea | aritmética acotada |
| `StockLevel` | orders | nunca negativo; `reorder_threshold` | `reserve`/`release`/`commit` → `InsufficientStock`; `needs_restock()` |
| `DiscountPercentage` | orders | 0–100 | `applied_to(Money)` |
| `TaxRate` | orders | 0–100, por país | `tax_of(Money)` |
| `CouponCode` | orders | alfanumérico en mayúsculas | — |
| `DateRange` | shared | `start < end` | `contains(now)`, `overlaps()` |
| `OrderStatus` | orders | **máquina de estados** (transiciones permitidas) | `can_transition_to()` |
| `CartStatus` | orders | `active`/`checked_out`/`expired`/`abandoned` | `can_modify()` |
| `PhoneNumber` | shared | E.164 | — |
| `PostalAddress` | identity | VO compuesto; valida CP por país | `is_deliverable_to(zona)` |
| `PasswordPolicy` / `HashedPassword` | identity | longitud, mayús/minús/dígito; hash | `verify()` |
| `LoyaltyTier` | identity | derivado del gasto acumulado (`Money`): bronze/silver/gold | `discount_for_tier()` |
| `NotificationChannel` | notifications | `email`/`sms`/`push` | `supports(content)` |
| `DeliveryStatus` | notifications | **máquina de estados** de entrega | `can_transition_to()` |

### 4.2 Agregados por contexto

#### `identity`

**`User`** (agregado de seguridad — *distinto de lo típico*)
- Campos: `id`, `email: EmailAddress`, `hashed_password: HashedPassword`,
  `status` (`pending_verification`/`active`/`locked`/`disabled`),
  `failed_login_attempts`, `email_verified: bool`, `roles`.
- Reglas:
  - `register()` aplica `PasswordPolicy`; estado inicial `pending_verification`;
    registra `UserRegisteredDomainEvent`.
  - `record_failed_login()` incrementa contador; al llegar a `MAX_FAILED_ATTEMPTS`
    → `lock()` (estado `locked`) y registra `UserLockedDomainEvent`.
  - `verify_email()`: solo desde `pending_verification` → `active`; registra
    `UserEmailVerifiedDomainEvent`.
  - `change_password()` revalida política y resetea intentos fallidos.
  - Transiciones de `status` controladas (`IllegalUserTransition`).

**`Customer`** (fidelización)
- Campos: `id`, `user_id`, `name`, `shipping_address: PostalAddress`,
  `total_spent: Money`, `loyalty_tier: LoyaltyTier`, `credit_limit: Money`.
- Reglas:
  - `loyalty_tier` se **recalcula** a partir de `total_spent` (`record_purchase`).
  - `place_within_credit(total)` → `CreditLimitExceeded` si supera el crédito.
  - `record_purchase(amount)` suma a `total_spent` y reevalúa tier; puede registrar
    `CustomerTierUpgradedDomainEvent`.

#### `orders`

**`Product`** (ciclo de vida + stock)
- Campos: `id`, `sku: Sku`, `name`, `slug: Slug`, `price: Money`,
  `stock: StockLevel`, `category_id`, `status` (`draft`/`published`/`discontinued`).
- Reglas:
  - `publish()` requiere `price > 0` y stock definido → si no, `ProductNotPublishable`.
  - `change_price()` registra `ProductPriceChangedDomainEvent`.
  - `reserve_stock(qty)` / `release_stock(qty)` delegan en `StockLevel`.
  - No se puede vender un producto `discontinued` ni `draft`.

**`Category`** (jerárquica)
- Campos: `id`, `name`, `slug: Slug`, `parent_id` (opcional), `depth`.
- Reglas: profundidad máxima (p.ej. 3) → `CategoryTooDeep`; `slug` único entre
  hermanos.

**`Cart`** (carrito efímero — *distinto y muy rico*)
- Campos: `id`, `customer_id`, `lines: list[CartLine]`, `coupon_code` (opcional),
  `status: CartStatus`, `expires_at`.
- Reglas:
  - Solo modificable si `status == active` y no expirado → si no, `CartNotModifiable`.
  - `add_line(product_id, qty, unit_price)`: tope de items y de cantidad por línea;
    fusiona líneas del mismo producto.
  - `apply_coupon(coupon)`: un solo cupón; valida contra reglas del `Coupon`.
  - `subtotal()` / `total()` calculan con `Money` (+ impuestos − descuento de cupón).
  - `check_out()`: requiere ≥ 1 línea y no expirado; pasa a `checked_out` y registra
    `CartCheckedOutDomainEvent` (lo consume `orders` para crear el `Order`).
  - `expire()` marca `expired` (lo dispara el worker por inactividad).

**`Order`** (máquina de estados real)
- Campos: `id`, `customer_id`, `lines: list[OrderLine]`, `status: OrderStatus`,
  `tax_rate: TaxRate`, `discount: DiscountPercentage`, `total: Money`.
- Estados: `pending_payment → paid → fulfilled → delivered`, con ramas
  `cancelled` y `refunded`.
- Reglas:
  - `add_line()` solo en `pending_payment`; tras `pay()` las líneas son inmutables.
  - `total = Σ líneas + impuestos − descuento`.
  - `pay()` exige `total > 0`; registra `OrderPaidDomainEvent`.
  - `fulfill()` solo desde `paid`; registra `OrderFulfilledDomainEvent`.
  - `cancel()` prohibido si ya `fulfilled`/`delivered` → `IllegalOrderTransition`;
    registra `OrderCancelledDomainEvent`.
  - `OrderPlacedDomainEvent` se registra al crear el pedido.

**`Coupon`** (regla temporal + límite de uso — *distinto*)
- Campos: `code: CouponCode`, `kind` (`percentage`/`fixed`), `value`,
  `validity: DateRange`, `usage_limit`, `times_used`, `min_order_amount: Money`.
- Reglas:
  - `redeem(order_total, now)` → `CouponExpired` (fuera de `validity`),
    `CouponUsageExceeded` (`times_used >= usage_limit`),
    `OrderBelowCouponMinimum` (`order_total < min_order_amount`).
  - No combinable con otros cupones (lo garantiza `Cart`/`Order`).

#### `notifications`

**`Notification`** (máquina de estados de entrega + reintentos)
- Campos: `id`, `recipient`, `channel: NotificationChannel`, `template_code`,
  `payload`, `status: DeliveryStatus`, `attempts`, `max_attempts`.
- Estados: `queued → sent → delivered`, con `failed`.
- Reglas:
  - `mark_sent()` / `mark_delivered()` siguen transiciones válidas.
  - `retry()` incrementa `attempts`; al superar `max_attempts` → `failed` y registra
    `NotificationFailedDomainEvent`.

**`NotificationTemplate`**
- Campos: `code`, `channel`, `subject`, `body` (con variables), `required_vars`.
- Reglas: `render(vars)` exige todas las `required_vars` (`MissingTemplateVariable`);
  valida que el canal soporte el contenido (`UnsupportedChannel`).

---

## 5. Autenticación y autorización

### 5.1 Autenticación (contexto `identity`)

- `POST /identity/v1/user/token`: valida `email` + `password` (hash) y devuelve
  `access_token` + `refresh_token` (JWT firmado con `TOKEN_SECRET_KEY`).
- `POST /identity/v1/user/token/refresh`: renueva el access token.
- El JWT embebe los permisos: `{ sub, is_admin, permissions: [...] }`.
- Registro/verificación de email con cambios de estado del agregado `User`.

### 5.2 Autorización (librería `authorization`, stateless)

- `PermissionValidator.validate(*, user_id, permissions, resource_type, action,
  resource_id=None, allow_only_if_self=False)` → `PermissionCheckResult`.
- Permisos en JWT: `is_admin` (acceso total) + lista de
  `{ resource_type, action, resource: { type: '*'|'list', ids: [...] } }`.
- **Sin permisos por organización** (simplificación frente a layers).
- Roles del dominio:
  - `admin`: todo.
  - `customer`: lee/edita **solo sus** `Order`/`Cart`/`Customer` (vía
    `allow_only_if_self` comparando `resource_id` con su `user_id`/`customer_id`);
    `Product`/`Category` los lee cualquier autenticado, los escribe `admin`.
- `AuthenticatedUserHook` (dependencia FastAPI): decodifica el JWT, expone el
  `authorizer` con `require_permission(...)` y `require_admin()`.

---

## 6. Eventos de dominio y bus

### 6.1 Tipos de evento

- **Internos** (`IS_INTERNAL`): se despachan a subscribers **en el mismo proceso**
  vía `InMemoryEventBus` (p.ej. al crear `Order`, un subscriber interno reserva el
  stock de `Product`).
- **Externos** (`IS_EXTERNAL`): cruzan de contexto; se escriben en la tabla
  `bus.events`.

### 6.2 Bus (Postgres como stream)

- `HybridEventBus` = `InMemoryEventBus` (internos) + `PostgresStreamEventBus`
  (externos).
- El evento externo se **inserta en `bus.events` dentro de la misma transacción**
  que el `save` del agregado (garantía outbox: atomicidad, no se pierden eventos),
  posible porque es una sola BD Postgres.
- Tabla `bus.events`: `id bigserial` (orden total), `event_id uuid`, `event_name`,
  `aggregate_id`, `occurred_on`, `payload jsonb`, `producer_context`.
- Tabla `bus.consumer_offsets`: `consumer_name`, `last_event_id`.
- Tabla `bus.failed_events` (failover): eventos que fallaron tras reintentos.

### 6.3 Consumo

- Cada BC consumidor corre un **worker poller** (contenedor aparte, ≈ el
  `*-infrastructure` de layers) que:
  1. Lee `bus.events` con `id > last_event_id` de su offset.
  2. Deserializa y despacha a sus `DomainEventSubscriber` registrados.
  3. Avanza el offset; reintenta con backoff; ante fallo persistente → `failed_events`.

### 6.4 Mapa de eventos (productor → consumidor)

| Evento | Productor | Consumidor(es) | Reacción |
|---|---|---|---|
| `identity.user.registered` | identity | notifications | Notificación de bienvenida / verificación |
| `identity.user.locked` | identity | notifications | Aviso de cuenta bloqueada |
| `orders.cart.checked_out` | orders | orders (interno) | Crear `Order` a partir del `Cart` |
| `orders.order.placed` | orders | notifications; orders (interno) | Confirmación de pedido; reservar stock |
| `orders.order.paid` | orders | notifications | Recibo de pago |
| `orders.order.fulfilled` | orders | notifications | Aviso de envío |
| `orders.product.price_changed` | orders | (interno/futuro) | Reproyección de catálogo |
| `notifications.notification.failed` | notifications | notifications | Reintento / alerta |

---

## 7. Persistencia y migraciones

- **Una instancia Postgres**, un **schema por BC** (`identity`, `orders`,
  `notifications`) + schema `bus`. Cada BC solo accede a su schema.
- SQLAlchemy **async** (`asyncpg`), `SQLAlchemyDatabase` del kernel.
- **Alembic por BC**: cada uno con su `alembic.ini`, `migrations/env.py` y su
  `alembic_version` dentro de su schema → migraciones independientes.
- El schema `bus` y sus tablas (`events`, `consumer_offsets`, `failed_events`) se
  definen en `shared` y se crean mediante una **migración de bootstrap propia**
  (carpeta `shared/migrations/` con su `alembic_version` en el schema `bus`) que
  `make migrate_all` ejecuta **antes** de las migraciones de cada BC.
- Kernel **fiel a layers**: `Criteria` con filtros dinámicos parseados de query
  params, paginación por cursor, value objects base, failover de eventos.

---

## 8. Desarrollo local (Makefile + Docker Compose)

### Raíz

- `docker-compose.yml`: servicio `postgres` + red externa `agora-net`.
- Targets del `Makefile` raíz:
  - `make up_all` / `make down_all`
  - `make migrate_all` (aplica migraciones de los 3 BC)
  - `make fixtures_all`
  - `make test_all`
  - `make logs`
  - `make build_from_scratch`

### Por BC (`identity/Makefile`, etc.)

- `make -C identity up` (API `uvicorn --reload` + worker poller), `down`
- `make -C identity makemigrations message="..."`
- `make -C identity migrate`
- `make -C identity load_fixtures`
- `make -C identity tests`
- `make -C identity shell`

Cada BC monta su código + `shared` + `authorization` como volúmenes (hot-reload).
Puertos: identity `8001`, orders `8002`, notifications `8003`.

### Stack técnico

- **uv** (gestión de dependencias + `uv.lock`), Python **3.12**.
- FastAPI, SQLAlchemy async, asyncpg, Alembic, pydantic-settings, PyJWT, pytest.

---

## 9. Estrategia de tests (fiel a layers)

- **unit**: dominio (invariantes de VOs, máquinas de estado de agregados) y casos
  de uso con repos/event-bus dobles en memoria.
- **integration**: repos contra Postgres real (docker) y endpoints vía
  `httpx`/`TestClient`, incluyendo auth.
- **mothers**: *Object Mother* para construir agregados y payloads de prueba.
- Cobertura configurada en `pyproject.toml` (omitiendo `shared/*`, `scripts/*`,
  `docker/*`, `tests/*`, `main.py`, `dependencies.py`), igual que layers.

---

## 10. Documentación entregada

- Docstrings en cada VO/agregado explicando **invariantes y eventos registrados**.
- Por contexto: `docs/domain-implementation.md`, `docs/application-implementation.md`,
  `docs/events.md`.
- `README.md` raíz: arquitectura, diagrama de contextos, flujo de eventos y
  workflow local (cómo levantar, migrar, cargar fixtures, probar).
- `authorization/docs/AUTHORIZATION.md`: modelo de permisos.

---

## 11. Orden de implementación sugerido

1. **Andamiaje**: monorepo, `pyproject.toml`/uv, `docker-compose.yml` raíz,
   Makefiles, Postgres + schemas + `bus`.
2. **`shared`** (kernel): `AggregateRoot`, `DomainEvent`, `EventBus`
   (in-memory + postgres-stream + hybrid + failover), `Criteria`, VOs base,
   `SQLAlchemyDatabase`.
3. **`authorization`**: `PermissionValidator`, codec JWT, VOs de permisos.
4. **`identity`**: `User` + `Customer` (dominio → aplicación → infra → API),
   auth (token/refresh), Alembic, fixtures, tests.
5. **`orders`**: `Product`, `Category`, `Cart`, `Order`, `Coupon` (vertical
   completo), Alembic, fixtures, tests; subscriber interno de stock.
6. **`notifications`**: `Notification` + `NotificationTemplate`, worker poller,
   subscribers a eventos de `identity` y `orders`, Alembic, tests.
7. **Integración end-to-end**: levantar los 3 BC, validar el flujo de eventos
   (registro → bienvenida; checkout → pedido → confirmación).
8. **Docs** finales.

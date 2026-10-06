# inventory-service — implementation spec

**Namespace:** `commerce`  
**Database:** Aurora PostgreSQL database `inventory` on cluster `bm-commerce`  
**Cache:** ElastiCache Redis, key prefix `inv:`, TTL 10 seconds. The product-page read may use it. Reserve, release, and commit read Postgres only.  
**Callers:** `customer-bff` for available-to-sell. `seller-bff` for the seller’s on-hand edit. `order-service` for reserve and commit.  
**Calls:** `seller-service` to read approval and suspension. `identity-service` to introspect the seller session.  
**Publishes:** `InventoryChanged`, `StockReserved`, `StockRejected`, `ReservationExpired`  
**Consumes:** `OrderCancelled`, `PaymentFailed`, `PaymentCaptured`, `OrderDelivered`, `ReturnAccepted`  
**Product rules:** [modules/05-inventory.md](../modules/05-inventory.md)

This is the build document for stock on hand, reservations, and available-to-sell. Available-to-sell is `onHand - reserved`. Catalogue owns the price. Order-service owns the order status. This service does not write either database.

Version 1 has one stock pool per variant per seller. A warehouse bin is out of scope until BuyyMart stores goods itself.

Search already accepts `inventory.changed` and stores only the boolean. This service sends that payload and never puts the unit count on the event.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as catalogue-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as catalogue |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Redis | `ioredis` | The `inv:` cache only. Reserve does not read it |
| Ids | UUID v4 from `gen_random_uuid()` | Reservation ids. Pools are keyed by seller and variant |
| Logs | `pino` JSON to stdout | Log the order id, variant id, and seller id. Do not log a customer id |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One reserve across order-service and this API |
| Metrics | `prom-client` on `GET /metrics` | Rejected reserves, held reservations, outbox lag |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as catalogue |
| Container port | 8080 | Service port 80 targets 8080 |

Quantities are integers. There is no fractional unit and no negative stock.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | Availability, seller edits, and synchronous reserve. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Publishes the outbox, expires prepaid holds, and applies consumed events |

The API does not publish to EventBridge itself. A stock change and its outbox rows commit in one transaction. The worker publishes and sets `published_at`. A rejected reserve rolls the stock transaction back, then inserts `stock.rejected` in a second transaction so the failure is still visible.

The reserve path does not call catalogue, seller-service, or Redis. Order-service waits at most 800 ms. A slow dependency inside reserve would make that budget fail closed.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `pool` | One row per seller and variant. `available = on_hand - reserved` |
| `reserve` | Hold units for an order, or reject the whole order |
| `release` | Give a hold back when the order is cancelled, payment fails, or the prepaid timer ends |
| `commit` | On delivery, subtract the units from `on_hand` and clear the hold |
| `restock` | Add units only when a return is accepted as sellable |
| `cache` | `inv:{sellerId}:{variantId}` for the product-page read |
| `outbox` | `inventory.changed`, `stock.reserved`, `stock.rejected`, `reservation.expired` |

This service does not store price, title, GSTIN, the customer id, or the payment row.

---

## 4. How a request is trusted

Browsers never call inventory-service. A BFF or order-service calls:

```text
http://inventory-service.commerce.svc.cluster.local
```

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `inventory-service`. Lifetime at most 60 seconds. HS256. A bad audience or expiry is `401 service_unauthorized` |
| `X-Request-Id` | UUID. Echoed on the response and on every log line |
| `traceparent` | W3C trace context |
| `X-Session-Token` | Present on seller routes only. The BFF has already introspected it with identity. This service checks the family and role again by calling identity introspect. It does not trust a role claim in the body |

`GET /v1/availability` and `POST /v1/reservations` need the service JWT only. They do not need a customer session.

Seller reads and the on-hand edit need a session whose family is `seller` and whose role is `seller_owner` or `seller_catalogue`. `seller_orders` is `403 forbidden`. A staff session is `403 forbidden`. The pool’s `seller_id` must equal the session’s `sellerId`. Another seller’s variant is `404 not_found`, not `403`.

JSON errors:

```json
{ "code": "stock_rejected", "message": "That item is out of stock.", "requestId": "…" }
```

`message` is safe to show on the storefront. `code` is what the caller branches on.

---

## 5. Pool

A pool is created the first time that seller sets `onHand` for that variant. Reserve does not create a pool. A missing pool has available 0.

| Column | Rule |
| --- | --- |
| `seller_id`, `variant_id` | Primary key. One pool |
| `on_hand` | Integer, 0 through 1000000. The seller sets this. They cannot set `reserved` |
| `reserved` | Integer, 0 through `on_hand`. Only reserve, release, expiry, and commit change it |
| Available | `on_hand - reserved`, computed. Never stored as a second source of truth |

A set that would make `on_hand` less than `reserved` is `400 stock_below_reserved`. The seller must wait until those orders are cancelled or delivered.

Until `SELLER_URL` is set, only `OWNED_SELLER_ID` may edit stock, and that id is treated as approved and not suspended. Any other seller is `403 seller_not_approved`. Stage and prod set `SELLER_URL`, and then this shortcut is off. A failed seller-service call is `503 dependency_unavailable` and the pool is unchanged.

This service does not ask catalogue whether the variant exists.

---

## 6. Availability read

`GET /v1/availability?sellerId=&variantId=`

Both query values are required UUIDs. Anything else is `400 invalid_query`.

The handler reads Redis key `inv:{sellerId}:{variantId}`. A hit returns that integer. A miss, or a Redis error, reads Postgres and, when Redis is up, stores the integer with a 10-second TTL. Redis down is not `503` when Postgres answers. Postgres down is `503 stock_unavailable`.

A missing pool is `200` with `available: 0`. It is not `404`.

`200`:

```json
{
  "sellerId": "…",
  "variantId": "…",
  "available": 4,
  "inStock": true
}
```

`inStock` is `available > 0`. The customer BFF waits at most 300 ms for this route. The exact number is for the product page and the cart stepper. It is not written to OpenSearch.

`GET /health/ready` does not ping Redis.

---

## 7. Seller edit

`PUT /v1/seller/stock/:variantId` with `{ "onHand": 12 }`.

`onHand` is an integer from 0 through 1000000. A missing field, a fraction, or a value outside that range is `400 invalid_body`.

The same transaction updates or inserts the pool and, because `on_hand` changed, inserts one `inventory.changed` outbox row. `inStock` on that event is true only when the new available quantity is above zero. The event does not include the quantity.

`200`:

```json
{ "sellerId": "…", "variantId": "…", "onHand": 12, "reserved": 2, "available": 10 }
```

`GET /v1/seller/stock` lists that seller’s pools. It is not cached.

---

## 8. Reserve

`POST /v1/reservations`. Order-service calls this while creating the order. The body is:

```json
{
  "orderId": "…",
  "sellerId": "…",
  "paymentMethod": "prepaid",
  "lines": [{ "variantId": "…", "quantity": 1 }]
}
```

`paymentMethod` is `prepaid` or `cod`. `lines` has from 1 through 20 rows. `quantity` is an integer from 1 through 100. A duplicate `variantId` in `lines` is `400 invalid_body`.

One database transaction:

1. `SELECT … FOR UPDATE` the pools in `variant_id` order, so two reserves cannot deadlock.
2. If any line’s available quantity is below the requested quantity, roll back. No reservation row remains.
3. Otherwise insert one `held` reservation per line and add the quantity to `reserved`.

A prepaid hold sets `expires_at` to 15 minutes after insert. A COD hold sets `expires_at` to null. COD does not use the 15-minute timer.

The second customer to take the last unit gets `409 stock_rejected`. After the rollback, a separate transaction inserts `stock.rejected` for the first line that did not fit:

```json
{
  "orderId": "…",
  "sellerId": "…",
  "variantId": "…",
  "requested": 2,
  "available": 1
}
```

`409`:

```json
{
  "code": "stock_rejected",
  "message": "That item is out of stock.",
  "variantId": "…",
  "available": 1,
  "requestId": "…"
}
```

A successful reserve inserts `stock.reserved`. It inserts `inventory.changed` for a line only when that line’s available quantity crossed to zero or left zero. A hold that leaves available above zero does not emit `inventory.changed`, because `on_hand` did not change and the flag did not flip.

`201`:

```json
{
  "orderId": "…",
  "reservations": [
    { "id": "…", "variantId": "…", "quantity": 1, "expiresAt": "2026-10-06T12:15:00Z" }
  ]
}
```

COD omits `expiresAt` or sends `null`.

The same `orderId` and the same lines, sent again, return `200` with the existing rows and do not increment `reserved`. A repeat with different quantities is `409 reservation_conflict` and does not change the pools.

---

## 9. Release, expiry, and commit

| Cause | What changes |
| --- | --- |
| `order.cancelled` | Every `held` row for that `orderId` becomes `released`. `reserved` decreases by the quantity. `on_hand` stays |
| `payment.failed` | Same release, only for `prepaid` rows that are still `held` and have no `paid_at` |
| Prepaid timer | A `held` prepaid row with no `paid_at` and `expires_at` in the past becomes `expired`. `reserved` decreases. `on_hand` stays |
| `payment.captured` | Sets `paid_at` on `held` prepaid rows for that order. The expiry job then skips them. It does not change quantities |
| `order.delivered` | Every `held` row for that order becomes `committed`. `on_hand` and `reserved` both decrease by the quantity |
| `return.accepted` with `sellable: true` | `on_hand` increases by `quantity`. A hold is not required |
| `return.accepted` with `sellable: false` | No quantity change |

Cancellation after the reservation is already `committed` does not put the units back. Delivery already removed them from `on_hand`. A second delivery event is a no-op.

`payment.captured` that arrives after the row is already `expired` does not recreate the hold. Order-service has to treat a capture against an expired hold as its own problem. This service does not increment `reserved` again.

A COD refusal is `order.cancelled` from order-service, reason `cod_refused`. This service does not consume `CodRefused`. Fulfilment tells order-service, and the cancel releases the hold.

The expiry pass runs in the worker every 60 seconds. For each row it expires, it inserts `reservation.expired`:

```json
{ "orderId": "…", "reservationId": "…", "sellerId": "…", "variantId": "…" }
```

Order-service consumes that event and moves `payment_pending` to `payment_failed` when the order is still `payment_pending`. This service does not update the order row.

`inventory.changed` is inserted when a release, expiry, commit, or restock changes `on_hand`, or when available-to-sell crosses or leaves zero. A release that leaves available above zero, and does not change `on_hand`, does not emit `inventory.changed`.

Commit of a row that is not `held` returns `409 reservation_not_held` on the synchronous route and does not change `on_hand`. The event path records the event id and skips the row.

`POST /v1/reservations/commit` with `{ "orderId": "…" }` is the synchronous form of delivery, for a caller that cannot wait for the bus. It uses the same commit rules. Order-service may call it or emit `order.delivered`. Doing both is safe: the second one finds nothing `held`.

---

## 10. Consumed events

The worker accepts the envelope `{ id, type, source, time, traceId, data }`. Wire `type` values are lowercase. `processed_events.id` is the envelope `id`. A second delivery of the same id does not change quantities.

| Event | `type` | `source` | `data` |
| --- | --- | --- | --- |
| `OrderCancelled` | `order.cancelled` | `order-service` | `{ "orderId": "…" }` |
| `PaymentFailed` | `payment.failed` | `payment-service` | `{ "orderId": "…" }` |
| `PaymentCaptured` | `payment.captured` | `payment-service` | `{ "orderId": "…" }` |
| `OrderDelivered` | `order.delivered` | `order-service` | `{ "orderId": "…" }` |
| `ReturnAccepted` | `return.accepted` | `order-service` | `{ "orderId": "…", "sellerId": "…", "variantId": "…", "quantity": 1, "sellable": true }` |

An unknown type is ignored. A payload missing `orderId` is ignored. The event id is still stored so the bad body is not retried forever.

Empty `EVENTBRIDGE_BUS_NAME` means the worker does not call AWS. In that mode the only ingest is the development hook in section 15.

---

## 11. Events this service publishes

| Name | `type` | When |
| --- | --- | --- |
| `InventoryChanged` | `inventory.changed` | `on_hand` changed, or available-to-sell crossed or left zero |
| `StockReserved` | `stock.reserved` | A hold was created |
| `StockRejected` | `stock.rejected` | A reserve rolled back for lack of units |
| `ReservationExpired` | `reservation.expired` | The worker expired a prepaid hold |

The envelope `source` is `inventory-service`. One outbox row per event. `inventory.changed` matches the payload search-service already applies:

```json
{
  "variantId": "…",
  "sellerId": "…",
  "inStock": true,
  "at": "2026-10-06T12:00:00Z"
}
```

`inStock` is true only when available-to-sell is above zero. The quantity is not in the payload. Search ignores a quantity if one were added, but this service must not add one.

`stock.reserved`:

```json
{
  "orderId": "…",
  "sellerId": "…",
  "paymentMethod": "prepaid",
  "lines": [{ "variantId": "…", "quantity": 1, "reservationId": "…" }]
}
```

---

## 12. HTTP API

Base path `/v1`. Every route except health and metrics requires the service JWT.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| GET | `/v1/availability` | No | `200` available quantity, including 0 for a missing pool |
| GET | `/v1/seller/stock` | Seller owner or catalogue | `200` this seller’s pools |
| PUT | `/v1/seller/stock/:variantId` | Seller owner or catalogue | `200` the pool after the edit |
| POST | `/v1/reservations` | No. Called by order-service | `201` holds, or `200` when the same order is already held |
| POST | `/v1/reservations/commit` | No. Called by order-service | `200` committed rows |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | Cluster scrape only | Prometheus text |

Metrics:

| Name | Meaning |
| --- | --- |
| `inventory_reserve_rejected_total` | Reserves that rolled back |
| `inventory_reservations_held` | Current rows in `held` |
| `inventory_outbox_unpublished` | Outbox rows waiting to be published |

---

## 13. PostgreSQL schema

Database `inventory`. The application role `inventory_app` can `SELECT`, `INSERT`, and `UPDATE` on these tables. It cannot `DELETE`, `DROP`, `TRUNCATE`, or alter schema. Releasing a hold is an update. The migration role `inventory_migrator` runs `migrations/` and is not the runtime role. The API refuses to start if `DATABASE_MIGRATOR_URL` is set or if `DATABASE_URL` uses `inventory_migrator`.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE pools (
  seller_id    uuid NOT NULL,
  variant_id   uuid NOT NULL,
  on_hand      integer NOT NULL,
  reserved     integer NOT NULL DEFAULT 0,
  updated_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (seller_id, variant_id),
  CONSTRAINT pools_bounds CHECK (
    on_hand BETWEEN 0 AND 1000000
    AND reserved BETWEEN 0 AND on_hand
  )
);

CREATE TYPE reservation_status AS ENUM ('held', 'released', 'expired', 'committed');
CREATE TYPE stock_payment_method AS ENUM ('prepaid', 'cod');

CREATE TABLE reservations (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id         uuid NOT NULL,
  seller_id        uuid NOT NULL,
  variant_id       uuid NOT NULL,
  quantity         integer NOT NULL,
  payment_method   stock_payment_method NOT NULL,
  status           reservation_status NOT NULL DEFAULT 'held',
  expires_at       timestamptz,
  paid_at          timestamptz,
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT reservations_quantity CHECK (quantity BETWEEN 1 AND 100),
  CONSTRAINT reservations_expiry CHECK (
    (payment_method = 'prepaid' AND expires_at IS NOT NULL)
    OR (payment_method = 'cod' AND expires_at IS NULL)
  ),
  UNIQUE (order_id, variant_id)
);

CREATE INDEX reservations_held_idx ON reservations (status, expires_at) WHERE status = 'held';

CREATE TABLE processed_events (
  id            text PRIMARY KEY,
  processed_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE outbox (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  type          text NOT NULL,
  payload       jsonb NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  published_at  timestamptz
);

CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;
```

`updated_at` is set by the application on every update. There is no trigger in version 1.

There is no hard delete in version 1. An expired or released reservation stays as a row.

---

## 14. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `inventory_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Used only by the migration job. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `inventory-service` |
| Config | `REDIS_URL` | ElastiCache in stage and prod. Empty skips the cache and uses Postgres |
| Config | `IDENTITY_URL` | Introspect seller sessions |
| Config | `SELLER_URL` | Empty until seller-service is deployed |
| Config | `OWNED_SELLER_ID` | BuyyMart’s internal seller UUID |
| Config | `EVENTBRIDGE_BUS_NAME` | Bus name. Empty means the worker logs the envelope and does not call AWS |

Secrets live in Secrets Manager at `buyymart/{env}/inventory-service`. They are not in the image and not in git.

---

## 15. Local run

Docker Compose for this service is Postgres 16, Redis, the API, and the worker. Identity must already be running for seller routes. Availability and reserve need only the service JWT and Postgres.

Migrations run before the API starts. Host ports are 8089 for the API, 8090 for the worker hook, and 6380 for Redis, so this stack can run beside identity on 8080 and search on 8087. The Redis key prefix is still `inv:`.

Events in dev can be posted to `POST /internal/events` on the worker only when `NODE_ENV=development`. The body is one envelope from section 10. `POST /internal/expire` runs one expiry pass. Both routes require the service JWT and are not registered when `NODE_ENV` is `staging` or `production`.

---

## 16. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Seller sets `onHand` to 5 | Pool created, `reserved` 0, one `inventory.changed` with `inStock` true and no quantity field |
| `onHand` below `reserved` | `400 stock_below_reserved`, pool unchanged |
| `seller_orders` or a staff session | `403 forbidden` |
| Another seller’s variant | `404 not_found` |
| `SELLER_URL` empty and the seller is not `OWNED_SELLER_ID` | `403 seller_not_approved` |
| Availability for a missing pool | `200`, `available` 0, `inStock` false |
| Redis down, Postgres up | Availability still `200`. Ready stays `200` |
| Postgres down | Availability `503 stock_unavailable`. Ready `503`. Live stays `200` |
| Reserve 1 of 1 | `201`, `reserved` 1, available 0, `stock.reserved`, and `inventory.changed` with `inStock` false |
| Reserve 1 when 5 are available | `201`, no `inventory.changed` |
| Two reserves of the last unit | The second is `409 stock_rejected`, `reserved` stays 1, one `stock.rejected` |
| The same reserve body again | `200`, `reserved` does not increase |
| Prepaid hold | `expires_at` about 15 minutes ahead. COD `expires_at` is null |
| Expiry pass after `expires_at` | Status `expired`, `reserved` decreased, `reservation.expired` inserted |
| `payment.captured` before expiry | `paid_at` set. The expiry pass does not release that row |
| `payment.captured` after expiry | The row stays `expired` |
| `order.cancelled` on a held row | Status `released`, `on_hand` unchanged, available increased |
| `order.cancelled` on a committed row | No quantity change |
| `order.delivered` | Status `committed`, `on_hand` and `reserved` both decrease |
| Commit of a row that is not held | `409 reservation_not_held` |
| `return.accepted` with `sellable` true | `on_hand` increases by `quantity` |
| `return.accepted` with `sellable` false | No quantity change |
| The same event id again | No second quantity change |
| `inventory_app` | `DELETE` fails. `UPDATE` of `on_hand` succeeds |
| API process | Refuses to start when `DATABASE_MIGRATOR_URL` is set |
| Development hook when `NODE_ENV` is `production` | The route is not registered |

---

## 17. Outside this service

| Concern | Owner |
| --- | --- |
| Price, title, variant identity | `catalogue-service` |
| The in-stock flag on a search card | `search-service`, from `inventory.changed` |
| Order status, including `payment_failed` after `reservation.expired` | `order-service` |
| Gateway capture and `payment.failed` | `payment-service` |
| COD refusal | `fulfilment-service` publishes `CodRefused`. Order-service cancels. This service only sees `order.cancelled` |
| Whether a returned unit is sellable | Order-service, on `return.accepted` |
| The cart quantity cap shown to the customer | `customer-bff` and `cart-service`, using this availability read |

Not in version 1: warehouse bins, more than one pool per variant per seller, backorders, negative stock, deleting a pool, and auto-cancel when a seller misses the confirm SLA.

# cart-service — implementation spec

**Namespace:** `commerce`  
**Database:** Aurora PostgreSQL database `cart` on cluster `bm-commerce`  
**Cache:** ElastiCache Redis, key `cart:{ownerId}`, TTL 14 days. The key is the live cart. Postgres is the snapshot. A Redis flush must not drop the cart.  
**Callers:** `customer-bff` only. Browsers never call this service.  
**Calls:** `identity-service` to introspect the customer session. `catalogue-service` for the live product and the current price. `inventory-service` for available-to-sell.  
**Publishes:** `CartCheckedOut`  
**Consumes:** nothing in version 1. The BFF calls the checkout handoff after the order exists.  
**Product rules:** [modules/06-cart.md](../modules/06-cart.md)

This is the build document for the set of variants a customer intends to buy, and the quantity. Catalogue owns the price. Inventory owns stock. Promotions own the coupon. Order-service owns the order. This service does not write those databases and does not reserve stock.

A cart with lines from two sellers is still one cart. Checkout turns it into one customer order with two seller groups. That split happens in order-service. This service only groups the lines it returns.

The guest id is a cookie owned by `customer-bff`, not `bm_session`. The cookie name is `bm_guest`. This service never sets a cookie.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as catalogue-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as catalogue |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Redis | `ioredis` | The live cart JSON only |
| Ids | UUID v4 | Owner ids come from identity or from the guest cookie. Line identity is the variant |
| Logs | `pino` JSON to stdout | Log the owner id, variant id, and order id. Do not log a phone, an OTP, or an address |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One add across the BFF, catalogue, and inventory |
| Metrics | `prom-client` on `GET /metrics` | Rejected adds, line count, outbox lag |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as catalogue |
| Container port | 8080 | Service port 80 targets 8080 |

Quantities are integers. The cart does not store a price.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | Read, add, change, remove, merge, and the checkout handoff. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Publishes the outbox |

The API does not publish to EventBridge itself. A cart change and its snapshot commit in one Postgres transaction, then the API writes the Redis key. The worker publishes `cart.checked_out` and sets `published_at`.

A write always updates Postgres first. If Redis then fails, the write still succeeds. The next read loads the snapshot and fills Redis. Redis down is not `503` when Postgres answers. `GET /health/ready` is Postgres `SELECT 1`. It does not ping Redis.

Empty `REDIS_URL` skips Redis and uses the snapshot for every read and write.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `lines` | Add, set quantity, remove. Cap at 30 lines and at `min(10, available)` per variant |
| `read` | Load the snapshot, then join catalogue for price and inventory for available |
| `merge` | Fold a guest cart into the customer cart, then delete the guest cart |
| `checkout` | Clear lines that became an order. Leave rejected lines in the cart |
| `cache` | `cart:{ownerId}` JSON, TTL 14 days |
| `outbox` | `cart.checked_out` |

This service does not store price, title, GSTIN, a coupon, a pin code, or a payment row.

---

## 4. How a request is trusted

Browsers never call cart-service. `customer-bff` calls:

```text
http://cart-service.commerce.svc.cluster.local
```

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `cart-service`. Lifetime at most 60 seconds. HS256. A bad audience or expiry is `401 service_unauthorized` |
| `X-Request-Id` | UUID. Echoed on the response and on every log line |
| `traceparent` | W3C trace context |
| `X-Session-Token` | Present on customer routes. This service checks the family by calling identity introspect. It does not trust an owner id in the body |
| `X-Guest-Id` | UUID. Present on guest routes. The BFF copied it from the `bm_guest` cookie |

Customer routes need a session whose family is `customer`. A seller session or a staff session is `403 forbidden`. The cart owner is the session `subjectId`. A body or query `ownerId` that does not match that subject is `404 not_found`.

Guest routes need `X-Guest-Id` and no session. A missing or non-UUID guest id is `400 invalid_body`. Guest checkout is `403 forbidden`. Payment requires a customer session. That check lives here so a guest cannot clear lines.

`POST /v1/cart/merge` needs the customer session and `X-Guest-Id`.

JSON errors:

```json
{ "code": "not_found", "message": "This item is no longer available.", "requestId": "…" }
```

`message` is safe to show on the storefront. `code` is what the caller branches on.

The BFF sets `bm_guest` on the first guest add. The cookie is `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, and `Max-Age` is 14 days. The value is a UUID. It is not `bm_session`.

---

## 5. What is stored

A line stores only what the cart owns.

| Field | Rule |
| --- | --- |
| `variantId` | What they want. One line per variant per cart |
| `productId` | The catalogue product that contains the variant. Required so a later read can call `GET /v1/products/:id` |
| `sellerId` | Copied from that product. The client cannot set it |
| `quantity` | Integer, 1 through 10, and never above available-to-sell at the time of the write |
| `addedAt` | Set on the first insert. A later add of the same variant does not move it. Display order is `addedAt` ascending |

Price, MRP, title, and image URL are not columns and are not fields in the Redis JSON. A `pricePaise`, `mrpPaise`, or `sellerId` field on an add or update body is ignored.

The Redis value is the same lines, plus `ownerKind` `customer` or `guest`. TTL is 1,209,600 seconds. Every successful read or write refreshes that TTL. The key is `cart:{ownerId}` for both customers and guests.

---

## 6. Add, change, and remove

`POST /v1/cart/lines`

```json
{ "productId": "…", "variantId": "…", "quantity": 1 }
```

`productId` and `variantId` are required UUIDs. `quantity` is an integer from 1 through 100 on the wire so a bad client can be capped rather than rejected for asking for 11. A missing field, a fraction, or a quantity below 1 is `400 invalid_body`.

The handler calls catalogue `GET {CATALOGUE_URL}/v1/products/{productId}` with a service JWT whose audience is `catalogue-service`. Timeout is 1 second.

| Catalogue result | Cart result |
| --- | --- |
| `200` and the variant is in `variants` | Copy `sellerId` from the product. Continue |
| `404`, or `200` without that variant | `404 not_found`, message `This item is no longer available.` The cart is unchanged |
| Timeout or any other status | `503 dependency_unavailable`. The cart is unchanged |

It then calls inventory `GET {INVENTORY_URL}/v1/availability?sellerId=&variantId=` with a service JWT whose audience is `inventory-service`. Timeout is 300 ms. Inventory down, or a timeout, is `503 stock_unavailable`. The cart is unchanged. Available `0` is `409 out_of_stock`, message `Out of stock.` The cart is unchanged.

The stored quantity for a new line, or the sum when the variant is already in the cart, is `min(requested, 10, available)`.

| Result | What the customer sees |
| --- | --- |
| Stored quantity is the requested quantity, and that quantity is at most 10 | `201` for a new line, `200` when the variant was already there. No message |
| Requested quantity is above 10 and available is at least 10 | Quantity stored as 10. No stock message |
| Available is above 0 and below the quantity the cap would otherwise store | Quantity stored as available. `message` is `Only N left.` with N as that integer |

A 31st distinct variant is `400 cart_full`, message `The cart is full.` Existing lines stay.

`PUT /v1/cart/lines/:variantId` with `{ "quantity": 2 }` sets the quantity. It uses the same cap and the same catalogue and inventory calls. The variant must already be in this cart. Another cart’s variant, or a missing line, is `404 not_found`. `addedAt` does not change.

`DELETE /v1/cart/lines/:variantId` removes that line. A missing line is `404 not_found`.

Adding the same variant again adds the quantities, then applies the cap. `addedAt` stays the first time it was added.

---

## 7. Read

`GET /v1/cart`

The handler loads Redis. A miss or a Redis error loads the Postgres snapshot and, when Redis is up, writes the key with a 14-day TTL. Postgres down is `503 cart_unavailable`.

Prices and stock are not taken from Redis. For each distinct `productId` the handler calls catalogue, same route and timeout as section 6. If any of those calls times out or returns a status other than `200` or `404`, the read is `503 dependency_unavailable`. A checkout must not show a remembered price.

`200`:

```json
{
  "ownerId": "…",
  "ownerKind": "customer",
  "lines": [
    {
      "variantId": "…",
      "productId": "…",
      "sellerId": "…",
      "quantity": 2,
      "addedAt": "2026-10-06T12:00:00Z",
      "pricePaise": 49900,
      "mrpPaise": 59900,
      "available": 4,
      "checkoutable": true,
      "message": null
    }
  ],
  "groups": [
    { "sellerId": "…", "variantIds": ["…"] }
  ]
}
```

Lines are ordered by `addedAt` ascending. `groups` splits those lines by `sellerId` without changing the cart.

| Live result | Line |
| --- | --- |
| Catalogue `404`, or the variant is gone from the live product | `checkoutable` false. `pricePaise` and `mrpPaise` are null. `message` is `This item is no longer available.` The line stays |
| Inventory timeout or error | `checkoutable` false. `available` is null. `message` is `Stock is unavailable.` Price is still returned. The line stays |
| `available` is 0 | `checkoutable` false. `message` is `Out of stock.` The line stays |
| `available` is below `quantity` | `checkoutable` false. `message` is `Only N left.` The stored quantity is not changed by a read |
| Otherwise | `checkoutable` true. `message` null |

A read never writes a price into Postgres or Redis.

`checkoutable` false means `customer-bff` excludes the line from checkout. If every line is not checkoutable, or the cart has no lines, checkout does not open. This service still returns `200`.

---

## 8. Guest merge

`POST /v1/cart/merge` with the customer session and `X-Guest-Id`.

No guest header, or a guest cart that does not exist, is `200` with the customer cart unchanged. Login must not fail because there was nothing to merge.

When the guest cart exists, one transaction:

1. Load the customer lines and the guest lines.
2. Same variant: add the quantities, then cap with `min(10, available)` using one inventory read per variant. If that sum is reduced because of stock, `message` on the response line is `Only N left.`
3. Different variants: both lines remain. The customer line keeps its `addedAt`. A guest-only line keeps the guest `addedAt`.
4. If the combined cart would pass 30 lines, keep every customer line, then guest-only lines in `addedAt` order until 30. Drop the rest. Login still succeeds.
5. Delete the guest snapshot and delete the Redis key `cart:{guestId}`.

Inventory down during merge is `503 stock_unavailable`. Neither cart changes, and the guest cart is not deleted, so the BFF can retry.

The response is the customer cart from section 7.

---

## 9. Checkout handoff

`POST /v1/cart/checkout` with the customer session. A guest is `403 forbidden`.

```json
{
  "orderId": "…",
  "clearedVariantIds": ["…"],
  "rejected": [{ "variantId": "…", "available": 1 }]
}
```

`orderId` is a UUID. `clearedVariantIds` and `rejected` may be empty only when the cart itself is empty, which is `409 empty_cart`, message `Your cart is empty.` A variant listed in both arrays is `400 invalid_body`.

The customer BFF calls this after order-service has created the order and inventory has reserved stock. This service does not call order-service and does not reserve stock.

One transaction:

1. If this `orderId` was already applied, return `200` and do not change lines and do not insert another outbox row.
2. Remove each cleared variant that is in this cart. An id that is not in the cart is ignored.
3. For each rejected variant that is in this cart: if `available` is above 0, set `quantity` to `min(quantity, available, 10)` and the next read shows `Only N left.` If `available` is 0, leave `quantity` as stored and the next read shows `Out of stock.` The line stays.
4. When at least one line was removed, insert `cart.checked_out`.

`200` is the cart from section 7 after those changes.

`cart.checked_out` data:

```json
{
  "orderId": "…",
  "customerId": "…",
  "lines": [
    { "variantId": "…", "productId": "…", "sellerId": "…", "quantity": 1 }
  ]
}
```

`lines` are the lines that were removed. There is no price, phone, or address. The envelope `source` is `cart-service`.

---

## 10. Events this service publishes

| Name | `type` | When |
| --- | --- | --- |
| `CartCheckedOut` | `cart.checked_out` | At least one line was removed for an order |

The envelope is `{ id, type, source, time, traceId, data }`. One outbox row per order. A second checkout call with the same `orderId` does not write a second row.

Empty `EVENTBRIDGE_BUS_NAME` means the worker logs the envelope and does not call AWS. It still sets `published_at`. A second poll does not send the row again.

---

## 11. HTTP API

Base path `/v1`. Every route except health and metrics requires the service JWT.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| GET | `/v1/cart` | Customer, or guest id | `200` lines with live price and stock |
| POST | `/v1/cart/lines` | Customer, or guest id | `201` a new line, or `200` when the variant was already in the cart |
| PUT | `/v1/cart/lines/:variantId` | Customer, or guest id | `200` the line after the new quantity |
| DELETE | `/v1/cart/lines/:variantId` | Customer, or guest id | `204` |
| POST | `/v1/cart/merge` | Customer, plus guest id | `200` the customer cart. The guest cart is gone |
| POST | `/v1/cart/checkout` | Customer | `200` the cart after the handoff |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | Cluster scrape only | Prometheus text |

Metrics:

| Name | Meaning |
| --- | --- |
| `cart_add_rejected_total` | Adds and quantity changes that returned `out_of_stock`, `cart_full`, or `not_found` |
| `cart_lines` | Current snapshot lines |
| `cart_outbox_unpublished` | Outbox rows waiting to be published |

---

## 12. PostgreSQL schema

Database `cart`. The application role `cart_app` can `SELECT`, `INSERT`, and `UPDATE` on these tables. It can `DELETE` only on `carts` and `cart_lines`, because a merged guest cart is removed. It cannot `DELETE` the outbox or the checkout record, and it cannot `DROP`, `TRUNCATE`, or alter schema. The migration role `cart_migrator` runs `migrations/` and is not the runtime role. The API refuses to start if `DATABASE_MIGRATOR_URL` is set or if `DATABASE_URL` uses `cart_migrator`.

The migration runner takes advisory lock `2147483005` so it does not share a lock with identity, catalogue, media, or inventory.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE cart_owner_kind AS ENUM ('customer', 'guest');

CREATE TABLE carts (
  owner_id    uuid PRIMARY KEY,
  owner_kind  cart_owner_kind NOT NULL,
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE cart_lines (
  owner_id     uuid NOT NULL REFERENCES carts (owner_id) ON DELETE CASCADE,
  variant_id   uuid NOT NULL,
  product_id   uuid NOT NULL,
  seller_id    uuid NOT NULL,
  quantity     integer NOT NULL,
  added_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (owner_id, variant_id),
  CONSTRAINT cart_lines_quantity CHECK (quantity BETWEEN 1 AND 10)
);

CREATE TABLE checkout_orders (
  order_id      uuid PRIMARY KEY,
  owner_id      uuid NOT NULL,
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

`updated_at` is set by the application. There is no trigger in version 1.

There is no price column. Deleting a guest cart deletes its lines by the foreign key.

---

## 13. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `cart_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Used only by the migration job. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `cart-service` |
| Config | `REDIS_URL` | ElastiCache in stage and prod. Empty skips Redis and uses the snapshot |
| Config | `IDENTITY_URL` | Introspect customer sessions |
| Config | `CATALOGUE_URL` | Live product read. Required in stage and prod |
| Config | `INVENTORY_URL` | Available-to-sell. Required in stage and prod |
| Config | `EVENTBRIDGE_BUS_NAME` | Bus name. Empty means the worker logs the envelope and does not call AWS |

Secrets live in Secrets Manager at `buyymart/{env}/cart-service`. They are not in the image and not in git. Stage and prod files set `CATALOGUE_URL` and `INVENTORY_URL`. They do not contain a database password.

---

## 14. Local run

Docker Compose for this service is Postgres 16, Redis, the API, and the worker. Identity must already be running for customer routes. Guest routes still need catalogue and inventory when those URLs are set. Tests inject those clients and do not start Compose.

Migrations run before the API starts. Host ports are 8091 for the API, 8092 for the worker, and 6381 for Redis, so this stack can run beside inventory on 8089 and identity on 8080. The Redis key is still `cart:{ownerId}`.

There is no development event hook in version 1. This service consumes nothing.

---

## 15. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Add `{ productId, variantId, quantity: 1 }` and a `pricePaise` field | Line stored. The price field is not in Postgres or Redis. `sellerId` is the catalogue product’s seller |
| Add when the product is not live | `404 not_found`, message `This item is no longer available.`, cart unchanged |
| Add when available is 0 | `409 out_of_stock`, message `Out of stock.`, cart unchanged |
| Add quantity 8 when available is 4 | Quantity stored as 4. `message` is `Only 4 left.` |
| Add quantity 15 when available is 20 | Quantity stored as 10 |
| Add the same variant again | Quantities add, then the same cap. `addedAt` is unchanged |
| 31st variant | `400 cart_full`. The first 30 stay |
| Catalogue timeout | `503 dependency_unavailable`. Cart unchanged |
| Inventory timeout on add | `503 stock_unavailable`. Cart unchanged |
| `seller` or `staff` session | `403 forbidden` |
| Guest `POST /v1/cart/checkout` | `403 forbidden` |
| Read after a price change in catalogue | The response price is the new price. The snapshot still has no price column |
| Read when catalogue returns `404` for a stored product | Line stays. `checkoutable` false. Message `This item is no longer available.` |
| Read when inventory is down | `200`. That line has `checkoutable` false and message `Stock is unavailable.` |
| Redis down, Postgres up | Read still `200`. Ready stays `200` |
| Postgres down | Read `503 cart_unavailable`. Ready `503`. Live stays `200` |
| Merge same variant, quantities 6 and 6, available 10 | Customer line quantity 10. Guest key and guest rows are gone |
| Merge two different variants | Both lines remain |
| Merge that would pass 30 lines | Customer lines stay. Guest lines fill up to 30. Guest cart is still deleted |
| Merge when the guest cart is missing | `200`. Customer cart unchanged |
| Checkout with one cleared variant | That line is gone. One `cart.checked_out` with that line and no price. The other line remains |
| Checkout rejected with `available` 2 | That line remains, quantity 2, later read says `Only 2 left.` |
| Checkout rejected with `available` 0 | That line remains. Later read says `Out of stock.` |
| The same `orderId` again | `200`. No second outbox row. Quantities do not change again |
| Empty cart checkout | `409 empty_cart`, message `Your cart is empty.` |
| `cart_app` | `DELETE` on `outbox` fails. `DELETE` on a guest `carts` row succeeds |
| API process | Refuses to start when `DATABASE_MIGRATOR_URL` is set |

---

## 16. Outside this service

| Concern | Owner |
| --- | --- |
| Price, title, variant identity | `catalogue-service` |
| Available-to-sell and reservation | `inventory-service`, called by order-service at order creation |
| Coupon | `promotion-service`, called by `customer-bff` |
| Pin code and delivery fee | `fulfilment-service`, called by `customer-bff` |
| Order status | `order-service` |
| `bm_guest` cookie and `bm_session` | `customer-bff` |
| Whether checkout opens | `customer-bff`, from `checkoutable` on this read |

Not in version 1: saved carts for later, wish lists, price-drop alerts, consuming `UserDeleted`, and deleting a cart when a customer deletes the account.

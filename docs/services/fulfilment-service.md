# fulfilment-service — implementation spec

**Namespace:** `sellers`  
**Database:** Aurora PostgreSQL database `fulfilment` on cluster `bm-ops`  
**Cache:** none. The pin master is Postgres. A Redis flush must not change a fee or an AWB.  
**Callers:** `order-service` for the pin and the delivery fee. `seller-bff` and `admin-bff` for a shipment read and a label reprint, when those routes are added. The courier calls `POST /webhooks/courier` on the public ingress. That route skips the BFF. Browsers never call this service.  
**Calls:** `order-service` for the address snapshot and the group. `seller-service` for the pickup name and city. `catalogue-service` for each line’s declared weight. `identity-service` to introspect a seller or staff session. The courier aggregator for a booking and a label. No other service calls the courier.  
**Publishes:** `ShipmentBooked`, `ShipmentUpdated`, `CodRefused`  
**Consumes:** `OrderConfirmed`, `OrderCancelled`, `ReturnAccepted`  
**Product rules:** [modules/11-fulfilment.md](../modules/11-fulfilment.md), [modules/09-orders.md](../modules/09-orders.md), [modules/12-returns.md](../modules/12-returns.md)

This is the build document for pin-code serviceability, the delivery fee, courier booking, labels, tracking scans, and COD remittance matching. Order-service owns order status. This service does not update the order row. It publishes an event. A delivered COD order can be delivered to the customer and still financially open until the remittance matches.

Version 1 uses one courier aggregator. BuyyMart does not run a rider app and does not store its own warehouse bins. Sellers ship from their own address. A BuyyMart warehouse pickup is config, not a second product.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as seller-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as seller-service |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Redis | none | Webhook duplicates and the outbox live in Postgres |
| Ids | UUID v4 from `gen_random_uuid()` | Shipment ids. The AWB is the courier’s id |
| Logs | `pino` JSON to stdout | Log the order id, seller id, shipment id, and AWB. Do not log a phone, an address line, a label URL, a webhook body, a signature, or a secret |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One booking across this worker, order-service, and the courier |
| Metrics | `prom-client` on `GET /metrics` | Bookings, label-pending rows, duplicate webhooks, outbox lag |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as seller-service |
| Container port | 8080 | Service port 80 targets 8080 |

Money is an integer number of paise. Weight is an integer number of grams. This service does not invent a fee that is not in the pin master or a slab row.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | Serviceability, shipment reads, label reprint, pin edits, remittance match, and the courier webhook. Minimum 3 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Consumes `order.confirmed`, `order.cancelled`, and `return.accepted`. Books the courier. Publishes the outbox. Retries a label that is still pending. Publishes tracking one step at a time |

The API does not publish to EventBridge itself. A scan, a booking, or a remittance match and its outbox row commit in one Postgres transaction. The worker sets `published_at`. A second poll does not send the row again.

`GET /health/ready` is Postgres `SELECT 1`. It does not call the courier. A courier outage must not take this API out of the load balancer. Checkout would then fail closed inside order-service.

The webhook handler must see the exact request bytes. A re-serialized JSON body will not match the signature. The route reads the raw body, checks the signature, and only then parses JSON.

---

## 3. What this build includes

| In this build | Not in this build |
| --- | --- |
| Pin master and an optional weight slab. `GET /v1/serviceability` | A live courier quote on the checkout path. The fee is the table |
| Book one forward shipment per seller group on `order.confirmed` | Booking on `order.paid`. The seller can still cancel before a label is bought |
| Book one reverse shipment on `return.accepted` | A rider app, a warehouse bin, or a second courier |
| Signature-checked tracking webhook, scans, and `shipment.updated` one step at a time | Auto-refund of an RTO. Staff decide cancel or reattempt outside this service |
| COD remittance match by AWB and amount | A settlement ledger line. Settlement reads `codRemitted` later |
| Seller and staff shipment read, and a label reprint | Seller Centre and Admin Console screens. Those BFFs do not call this service yet |

`seller-bff` pack does not book a courier. This service books when it consumes `order.confirmed`. Pack means the seller has printed the label this service already stored.

---

## 4. How a request is trusted

Browsers never call fulfilment-service. A BFF or order-service calls:

```text
http://fulfilment-service.sellers.svc.cluster.local
```

Every call except health, metrics, and the courier webhook carries:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `fulfilment-service`. `exp - iat` is at most 60 seconds. Algorithm HS256. Signed with `SERVICE_JWT_KEY` |
| `X-Request-Id` | UUID. Generated by the caller when missing. Echoed on the response |
| `traceparent` | W3C trace context. Forwarded to order-service, seller-service, catalogue-service, and the courier |
| `X-Session-Token` | Required on seller and staff routes. This service calls identity `POST /v1/sessions/introspect`. Timeout 200 ms. Not sent on serviceability |

A missing, expired, or wrong-audience JWT is `401 service_unauthorized`. A session identity cannot load is `503 dependency_unavailable`. An invalid session is `401 session_invalid`.

| Route | Who |
| --- | --- |
| `GET /v1/serviceability` | Service JWT only. No session. This is the call order-service already makes |
| Shipment read and label reprint | Family `seller`, and `sellerId` on the session equals the shipment. Another seller’s shipment is `404 not_found`. Family `staff` may read any shipment |
| Pin create and edit | Family `staff`, role `admin` or `super_admin` |
| Remittance match | Family `staff`, role `finance`, `admin`, or `super_admin` |
| Webhook | No service JWT and no session. The courier signature is the credential |

`POST /webhooks/courier` is on the public ingress at `https://api.buyymart.com/webhooks/courier`. A service JWT on that route is ignored. A bad signature is `400 signature_invalid` and nothing is written.

Consumed events are not HTTP from the public ingress. The worker takes them from the bus. In development only, `POST /internal/events` on the worker accepts one envelope. That route is absent when `NODE_ENV` is `staging` or `production`.

---

## 5. What is stored

| Shipment field | Meaning |
| --- | --- |
| `id` | This service’s id |
| `orderId`, `sellerId` | The group. One forward shipment per pair |
| `direction` | `forward` or `reverse` |
| `variantId` | Set on a reverse shipment. Null on a forward shipment |
| `status` | Section 7 |
| `awb` | The courier’s waybill. Null while the label is pending |
| `codAmountPaise` | That group’s COD share, or 0 for prepaid and for a reverse shipment |
| `codUtr` | Set when a remittance row matches |
| `lastError` | The last booking or publish failure. Safe to show to staff |
| `orderStatusSent` | The last `shipment.updated` status published for this forward shipment. Null until the first one |

Scans are append-only. A remittance exception is append-only. Nothing in this database is deleted.

Not stored: a phone, the raw webhook body, the webhook secret, the courier key, or a label URL. The reprint route asks the courier again. The URL is not written to Postgres and not written to the log.

The pickup address and the customer address are read at booking time and sent to the courier. They are not copied into a second address book here.

---

## 6. Pin code and fee

Checkout calls this synchronously. The route does not call the courier. order-service waits 1 second and treats any non-200 as `503 dependency_unavailable`. An unknown pin must still be HTTP 200.

`GET /v1/serviceability?pin=560001`

Optional query `weightGrams`, a positive integer. order-service does not send it. When it is absent, `deliveryPaise` is that pin’s `feePaise`. When it is present, `deliveryPaise` is the slab whose gram range contains it. A weight that matches no slab is not serviceable and the fee is 0. This service does not invent a fee.

| Pin | Response |
| --- | --- |
| Six digits, row exists, `serviceable` true | `200` `{ serviceable: true, codServiceable, deliveryPaise }` |
| Six digits, row missing, or `serviceable` false | `200` `{ serviceable: false, codServiceable: false, deliveryPaise: 0 }` |
| `pin` missing or not six digits | `400 invalid_body` |
| `weightGrams` present and not a positive integer | `400 invalid_body` |

`codServiceable` is true only when the row says so. A false value is a normal answer. order-service turns `serviceable: false` into `409 pin_unserviceable` and does not insert an order.

Staff create or replace a pin with `PUT /v1/pins/:pin`. The pin is six digits. Body:

| Field | Rule |
| --- | --- |
| `city` | 1–80 characters |
| `state` | 1–80 characters. The same names seller-service stores. This service does not check the GST table |
| `serviceable` | Boolean |
| `codServiceable` | Boolean. Cannot be true when `serviceable` is false |
| `feePaise` | Integer ≥ 0. Used when checkout omits `weightGrams` |

`201` on create, `200` on replace. A non-admin staff role is `403 forbidden`.

Slabs are `PUT /v1/slabs` by the same roles. The body is `{ slabs: [{ minGrams, maxGrams, feePaise }] }`. `minGrams` ≥ 1, `maxGrams` ≥ `minGrams`, `feePaise` ≥ 0. Ranges must not overlap. An overlap is `409 slab_overlap` and the previous slabs stay. An empty list clears the slabs. Checkout without `weightGrams` does not read this table.

Launch can be a short list of pins. The columns stay the full shape so the next city is a row, not a code change.

---

## 7. Booking

The worker consumes `order.confirmed`. The payload order-service publishes is `{ orderId, sellerId, invoiceNumber }`. There is no phone and no address on the event.

One forward shipment is inserted for that pair, status `label_pending`, in the same transaction as `processed_events`. A second delivery of the same event id does not insert another shipment. The courier call happens after that commit. A timeout does not roll the shipment back.

### Reads before the courier call

| Read | Call | Timeout | If it fails |
| --- | --- | --- | --- |
| Order | `GET {ORDER_URL}/v1/orders/:id`, audience `order-service`, no session | 1 s | `lastError` `order_unreadable`. Stay `label_pending`. Retry |
| Seller | `GET {SELLER_URL}/v1/sellers/:id`, audience `seller-service`, no session | 1 s | `lastError` `pickup_incomplete`. Stay `label_pending`. Retry |
| Weight | `GET {CATALOGUE_URL}/v1/products/:productId` for each line in the group, audience `catalogue-service` | 1 s each | `lastError` `weight_unknown`. Stay `label_pending`. Retry |

The order read this service needs is the staff shape: `payMode`, `payablePaise`, `address` (`name`, `phone`, `line1`, `line2`, `landmark`, `city`, `state`, `pin`), and `groups[].lines[]` of `variantId`, `productId`, `quantity`, `pricePaise`, plus that group’s `status`. order-service today requires `X-Session-Token` on that GET, so a service call with no session is `401 session_invalid`. This service does not invent an address and does not send a session it does not have. The shipment stays `label_pending` until the read is `200` and the address and the seller’s group are present.

The seller read with no session is the operational seller: `legalName`, `city`, `state`. It does not include `address.line1` or `address.pin`. When those two are absent, pickup is incomplete. `PICKUP_MODE=warehouse` uses `WAREHOUSE_LINE1`, `WAREHOUSE_CITY`, `WAREHOUSE_STATE`, and `WAREHOUSE_PIN` instead of the seller street. `PICKUP_MODE=seller` is the default. Stage and prod may set `warehouse` only for stock BuyyMart holds. A partial address is not sent.

Weight is the sum of each live variant’s `weightGrams` times quantity. A missing weight is not replaced with a default. Catalogue already requires weight before a product is live. A product that is no longer readable stays `label_pending`.

`codAmountPaise` is 0 when `payMode` is `prepaid`. When `payMode` is `cod` and the order has one group, it is `payablePaise`. When several groups share the order, each forward shipment gets a share of `payablePaise` in proportion to that group’s `pricePaise * quantity`, seller ids sorted ascending. The last seller receives the remainder so the shares sum to `payablePaise`. This service does not reprice delivery. The fee was fixed at checkout.

### Courier call

Timeout 5 seconds. `COURIER_MODE=fake` is allowed only when `NODE_ENV=development`. It does not open a socket. It sets `awb` to `fake-` plus the shipment id, status `booked`, and publishes `shipment.booked`. Stage and prod refuse to start when `COURIER_MODE` is `fake` or `COURIER_WEBHOOK_SECRET` is empty.

A live call sends the pickup, the customer snapshot, the weight, the COD amount or zero, and `orderId` as the merchant reference. Dimensions are omitted. This service does not invent them.

| Result | Shipment |
| --- | --- |
| Courier returns an AWB | Status `booked`. Publish `shipment.booked` with `{ orderId, sellerId, awb }`. No phone and no address on the event |
| Timeout, 5xx, or a body with no AWB | Stay `label_pending`. Set `lastError` to `courier_unavailable`. Set `nextAttemptAt` 1 minute later, then 2, then 4, capped at 15 minutes. The order stays `confirmed` |

`shipment.booked` is what notification-service will use to tell the customer the AWB. This service does not send SMS.

### Cancel before pickup

`order.cancelled` data is `{ orderId, reason, sellerIds }`. For each listed seller, a forward shipment still `label_pending` or `booked` with no scan becomes `cancelled` and is not retried. A shipment that already has a scan is left as it is. This service does not call a courier cancel API in version 1.

### Reverse pickup

`return.accepted` data is `{ orderId, sellerId, variantId, quantity, sellable }`. Insert one reverse shipment for that order, seller, and variant. `codAmountPaise` is 0. The same address reads apply. The customer snapshot is the pickup and the seller address is the destination. A failed read stays `label_pending` with the same error codes. Success publishes `shipment.booked` with `{ orderId, sellerId, variantId, awb, direction: "reverse" }`.

Reverse scans do not publish `shipment.updated`. That event moves a forward group along `packed → shipped → out_for_delivery → delivered`. A return is already `delivered`. The return AWB stays on this shipment. order-service does not read it back from `shipment.updated` today.

---

## 8. Tracking

`POST /webhooks/courier`. No JWT.

1. Read the raw body. Check `X-Courier-Signature`. The value must be the hex HMAC-SHA256 of those exact bytes with `COURIER_WEBHOOK_SECRET`. Compare in constant time. A missing or wrong signature is `400 signature_invalid`. Nothing is written.
2. Parse JSON only after the signature check. The body this service accepts is `{ eventId, awb, status, scannedAt }`. `eventId` is a non-empty string. `status` is `picked_up`, `in_transit`, `out_for_delivery`, `delivered`, `rto`, or `cod_refused`. `scannedAt` is an ISO-8601 timestamp. Anything else is `400 invalid_body` after the signature has already passed. The event id is still stored so a retry of the same bad body does not keep failing the signature path. A duplicate `eventId` returns `200` `{ duplicate: true }` and does no other work.
3. Unknown AWB stores the event id and an audit row, and returns `200`. No shipment change and no outbox row.

| Courier `status` | Shipment | Event |
| --- | --- | --- |
| `picked_up`, `in_transit` | `in_transit` | `shipment.updated` `shipped`, and only as below |
| `out_for_delivery` | `out_for_delivery` | `shipment.updated` `out_for_delivery` after `shipped` was published |
| `delivered` | `delivered` | `shipment.updated` `delivered` after `out_for_delivery` was published |
| `rto` | `rto` | None. No refund. Staff cancel or reattempt outside this service |
| `cod_refused` | `cod_refused` | `cod.refused` with `{ orderId, sellerId }` |

`shipment.updated` data is `{ orderId, sellerId, status }` and nothing else. `status` is `shipped`, `out_for_delivery`, or `delivered`. order-service applies a step only when the group is already the previous one: `packed` then `shipped`, `shipped` then `out_for_delivery`, `out_for_delivery` then `delivered`. A skip is ignored, and that event id is consumed.

This service therefore publishes one step per outbox row, in order. It does not publish `shipped` until a read of the order shows that group is `packed`. If the read fails, the scan is kept and `orderStatusSent` stays where it is. The worker retries the publish. It does not reuse an event id that was already sent.

A reverse shipment stores the scan and does not publish `shipment.updated` or `cod.refused`.

---

## 9. COD remittance

`POST /v1/remittances`. Staff finance, admin, or super_admin. Body `{ rows: [{ awb, amountPaise, utr, paidOn }] }`. `amountPaise` is a positive integer. `utr` and `awb` are non-empty. `paidOn` is `YYYY-MM-DD`. At least one row. More than 500 rows is `400 invalid_body`.

| Match | Result |
| --- | --- |
| AWB exists, `amountPaise` equals `codAmountPaise`, direction `forward` | Shipment `cod_remitted`, `codUtr` set. No outbox row. Settlement will read this status later |
| AWB exists, amount differs | Exception row `amount_differs`. The shipment is not marked remitted |
| AWB unknown | Exception row `awb_unknown` |

A second post of the same AWB and UTR does not insert a second exception and does not change a shipment already `cod_remitted`. The response is `200` `{ matched, exceptions }`.

Until the match, a delivered COD shipment stays `delivered`. This service does not pay the seller.

---

## 10. What a seller or a staff member can read

`GET /v1/orders/:orderId/shipments`

A seller sees only shipments whose `sellerId` is theirs. Staff see every shipment on that order. `200` `{ shipments }`. Each shipment is `id`, `orderId`, `sellerId`, `direction`, `variantId`, `status`, `awb`, `codAmountPaise`, `codUtr`, `lastError`, and `scans` of `{ status, scannedAt }`. No address, no phone, no label URL.

`GET /v1/shipments/:id` is the same object. Another seller is `404 not_found`.

`GET /v1/shipments/:id/label` asks the courier for a reprint. Timeout 3 seconds. Success is `200` `{ url, expiresAt }`. The URL is not logged and not stored. `COURIER_MODE=fake` returns `200` `{ awb, mode: "fake" }` and no URL. A shipment with no AWB is `409 label_pending` and the body includes `lastError`. A courier failure is `503 dependency_unavailable`. The caller does not mark the group packed. Pack stays on order-service.

---

## 11. Events

The envelope is `{ id, type, source, time, traceId, data }`. Wire `type` values are lowercase. `source` on publish is `fulfilment-service`.

| Name | `type` | When |
| --- | --- | --- |
| `ShipmentBooked` | `shipment.booked` | An AWB was stored |
| `ShipmentUpdated` | `shipment.updated` | The next forward step is ready to tell order-service |
| `CodRefused` | `cod.refused` | The courier said the customer refused COD |

`shipment.booked` has no phone and no address. `shipment.updated` is only `{ orderId, sellerId, status }`.

### Consumed

The worker stores `processed_events.id` as the envelope `id`. A second delivery of the same id does not book a second shipment.

| Event | `type` | `source` | Effect |
| --- | --- | --- | --- |
| `OrderConfirmed` | `order.confirmed` | `order-service` | Insert `label_pending`, then book |
| `OrderCancelled` | `order.cancelled` | `order-service` | Cancel a forward shipment that has no scan |
| `ReturnAccepted` | `return.accepted` | `order-service` | Insert a reverse `label_pending`, then book |

An unknown type is ignored. A payload missing `orderId` is ignored. The event id is still stored.

order-service does not consume `cod.refused` or `shipment.booked` today. This service still publishes them. It does not cancel the order itself.

Empty `EVENTBRIDGE_BUS_NAME` means the worker logs the envelope and does not call AWS. It still sets `published_at`. The development hook is the only ingest in that mode.

---

## 12. HTTP API

Base path `/v1` except the webhook. Every `/v1` route requires the service JWT.

| Method | Path | Who | Success |
| --- | --- | --- | --- |
| GET | `/v1/serviceability` | Service | `200` section 6 |
| PUT | `/v1/pins/:pin` | Admin staff | `201` or `200` |
| PUT | `/v1/slabs` | Admin staff | `200` |
| GET | `/v1/orders/:orderId/shipments` | That seller, or staff | `200` |
| GET | `/v1/shipments/:id` | That seller, or staff | `200` |
| GET | `/v1/shipments/:id/label` | That seller, or staff | `200` |
| POST | `/v1/remittances` | Finance staff | `200` |
| POST | `/webhooks/courier` | Courier signature | `200` after the event id is stored, including a duplicate |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | None. No JWT | Prometheus text |

JSON errors use `{ "code", "message", "requestId" }`. `message` is safe to show. The webhook’s `400` body uses the same shape. A duplicate webhook is `200` with `{ "duplicate": true }`.

| Code | HTTP | When |
| --- | --- | --- |
| `invalid_body` | 400 | Bad pin, bad slab, bad remittance row, bad webhook JSON |
| `signature_invalid` | 400 | Webhook signature does not match the raw body |
| `session_invalid` | 401 | No session, or identity rejected it |
| `service_unauthorized` | 401 | Bad service JWT |
| `forbidden` | 403 | A seller on a staff route, or a staff role that cannot edit pins or post a remittance |
| `not_found` | 404 | Another seller’s shipment, or an unknown shipment id |
| `slab_overlap` | 409 | Two slabs cover the same gram |
| `label_pending` | 409 | Reprint before an AWB exists |
| `dependency_unavailable` | 503 | Identity or the courier reprint failed |
| `fulfilment_unavailable` | 503 | Postgres is down |

`fulfilment_booked_total` counts a transition to `booked`. `fulfilment_label_pending` is the gauge of forward rows still `label_pending`. `fulfilment_webhook_duplicate_total` counts a repeated `eventId`. `fulfilment_outbox_unpublished` is the gauge of outbox rows with `published_at` null. Set the gauges on `GET /metrics`.

---

## 13. PostgreSQL schema

Database `fulfilment`. The application role `fulfilment_app` can `SELECT`, `INSERT`, and `UPDATE` these tables. It cannot `DELETE`, `DROP`, `TRUNCATE`, or alter schema. A cancel is a status change, not a delete. The migration role `fulfilment_migrator` runs `migrations/` and is not the runtime role. The API refuses to start if `DATABASE_MIGRATOR_URL` is set or if `DATABASE_URL` uses `fulfilment_migrator`.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE pins (
  pin              text PRIMARY KEY CHECK (pin ~ '^[0-9]{6}$'),
  city             text NOT NULL,
  state            text NOT NULL,
  serviceable      boolean NOT NULL,
  cod_serviceable  boolean NOT NULL,
  fee_paise        integer NOT NULL CHECK (fee_paise >= 0),
  updated_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT pins_cod CHECK (serviceable OR NOT cod_serviceable)
);

CREATE TABLE fee_slabs (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  min_grams  integer NOT NULL CHECK (min_grams >= 1),
  max_grams  integer NOT NULL CHECK (max_grams >= min_grams),
  fee_paise  integer NOT NULL CHECK (fee_paise >= 0)
);

CREATE TABLE shipments (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id           uuid NOT NULL,
  seller_id          uuid NOT NULL,
  direction          text NOT NULL CHECK (direction IN ('forward', 'reverse')),
  variant_id         uuid,
  status             text NOT NULL CHECK (status IN (
    'label_pending', 'booked', 'in_transit', 'out_for_delivery', 'delivered',
    'rto', 'cod_refused', 'cancelled', 'cod_remitted'
  )),
  awb                text,
  cod_amount_paise   integer NOT NULL DEFAULT 0 CHECK (cod_amount_paise >= 0),
  cod_utr            text,
  last_error         text,
  attempts           integer NOT NULL DEFAULT 0,
  next_attempt_at    timestamptz,
  order_status_sent  text,
  invoice_number     text,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT shipments_reverse_variant CHECK (direction = 'forward' OR variant_id IS NOT NULL)
);

CREATE UNIQUE INDEX shipments_forward_idx ON shipments (order_id, seller_id) WHERE direction = 'forward';
CREATE UNIQUE INDEX shipments_reverse_idx ON shipments (order_id, seller_id, variant_id) WHERE direction = 'reverse';

CREATE TABLE scans (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  shipment_id  uuid NOT NULL REFERENCES shipments (id),
  event_id     text NOT NULL UNIQUE,
  status       text NOT NULL,
  scanned_at   timestamptz NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE remittance_exceptions (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  awb           text NOT NULL,
  amount_paise  integer NOT NULL CHECK (amount_paise > 0),
  utr           text NOT NULL,
  reason        text NOT NULL CHECK (reason IN ('amount_differs', 'awb_unknown')),
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (awb, utr, reason)
);

CREATE TABLE webhook_events (
  id           text PRIMARY KEY,
  received_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE outbox (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  type         text NOT NULL,
  payload      jsonb NOT NULL,
  trace_id     text NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz
);

CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;

CREATE TABLE processed_events (
  id          text PRIMARY KEY,
  type        text NOT NULL,
  received_at timestamptz NOT NULL DEFAULT now()
);
```

The advisory lock for migrations is `2147483009`.

---

## 14. Configuration

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `fulfilment_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Migration job only. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `fulfilment-service` |
| Secret | `COURIER_WEBHOOK_SECRET` | HMAC key for `X-Courier-Signature`. Required in stage and prod |
| Secret | `COURIER_API_KEY` | The aggregator key. Empty when `COURIER_MODE=fake` |
| Config | `IDENTITY_URL` | Introspect seller and staff sessions |
| Config | `ORDER_URL` | Address snapshot. Required in stage and prod |
| Config | `SELLER_URL` | Pickup name and city. Required in stage and prod |
| Config | `CATALOGUE_URL` | Declared weight. Required in stage and prod |
| Config | `COURIER_MODE` | `fake` only in development. `live` in stage and prod |
| Config | `COURIER_URL` | Aggregator base URL. Required when mode is `live` |
| Config | `PICKUP_MODE` | `seller` or `warehouse`. Default `seller` |
| Config | `WAREHOUSE_LINE1`, `WAREHOUSE_CITY`, `WAREHOUSE_STATE`, `WAREHOUSE_PIN` | Used only when `PICKUP_MODE=warehouse` |
| Config | `EVENTBRIDGE_BUS_NAME` | Empty means the worker logs the envelope and does not call AWS |

Secrets live in Secrets Manager at `buyymart/{env}/fulfilment-service`. They are not in the image and not in git. `COURIER_WEBHOOK_SECRET` in Compose is a dev value and is not copied into `deploy/stage.env` or `deploy/prod.env`.

---

## 15. Local run

Docker Compose for this service is Postgres 16, the API, and the worker. Order, seller, catalogue, and identity must already be running for a real booking. Tests inject those clients and do not start Compose.

Migrations run before the API starts. Host ports are 8101 for the API and 8102 for the worker, so this stack can run beside admin-bff on 8100 and seller-service on 8097.

`POST /internal/events` on the worker accepts one envelope from section 11 only when `NODE_ENV=development`. It requires the service JWT and is not registered in staging or production.

`COURIER_MODE=fake`. A confirmed event still inserts `label_pending`. The fake booker sets an AWB without a network call once the order, seller, and weight reads succeed. A pin that is not in `pins` is `serviceable: false` with `deliveryPaise` 0.

---

## 16. Tests

Tests call the route functions with an injected pool and injected order, seller, catalogue, and courier clients. They do not start Postgres, identity, or the courier.

| Check | Expected |
| --- | --- |
| Known serviceable pin, no weight | `200`. `deliveryPaise` is the pin’s `feePaise`. The courier client is not called |
| Unknown pin | `200`. `serviceable` false. `deliveryPaise` 0. Not a 409 |
| `weightGrams` inside a slab | `deliveryPaise` is that slab, not the pin fee |
| `weightGrams` outside every slab | `serviceable` false. `deliveryPaise` 0 |
| Overlapping slabs | `409 slab_overlap`. The stored slabs are unchanged |
| `order.confirmed` twice | One forward shipment |
| Order read returns 401 | Status stays `label_pending`. `lastError` is `order_unreadable`. The courier client is not called |
| Seller operational body has no street | `pickup_incomplete`. No courier call |
| Fake mode after a complete read | Status `booked`. Outbox `shipment.booked` has `awb` and no phone |
| Courier timeout | Stays `label_pending`. `nextAttemptAt` is set |
| `order.cancelled` while `label_pending` | Status `cancelled`. A later retry does not book |
| Webhook bad signature | `400 signature_invalid`. No `webhook_events` row |
| Webhook duplicate `eventId` | `200` `{ duplicate: true }`. One scan |
| In-transit scan before the group is `packed` | Scan stored. No `shipment.updated` |
| In-transit scan after the group is `packed` | Outbox `shipment.updated` with `status` `shipped` only |
| Delivered scan with no earlier publish | The next publish is `shipped`, not `delivered` |
| `rto` | Shipment `rto`. No outbox row |
| `cod_refused` | Outbox `cod.refused`. No `shipment.updated` |
| Remittance amount matches | `cod_remitted` and `codUtr` set |
| Remittance amount differs | Exception `amount_differs`. Shipment unchanged |
| Unknown AWB | Exception `awb_unknown` |
| Seller reads another seller’s shipment | `404 not_found` |
| Label reprint with no AWB | `409 label_pending` |
| Support staff edits a pin | `403 forbidden` |

---

## 17. What stays in other services

| Concern | Owner |
| --- | --- |
| Order status, including `packed`, `shipped`, `delivered`, and `cod_refused` | `order-service` |
| The delivery fee copied onto the order | `order-service`, from section 6 |
| Invoice number | `order-service`, at confirm |
| Declared weight | `catalogue-service` |
| Pickup street, once the operational read includes it | `seller-service` |
| SMS with the AWB | `notification-service`, from `shipment.booked` |
| COD cash on the seller statement | `settlement-service`, after `cod_remitted` |
| Pack and confirm buttons | Seller Centre, through `seller-bff` |

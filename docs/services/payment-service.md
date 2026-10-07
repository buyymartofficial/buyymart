# payment-service — implementation spec

**Namespace:** `money`  
**Database:** Aurora PostgreSQL database `payment` on cluster `bm-commerce`.  
**Cache:** none. Idempotency is a Postgres row. A Redis flush must not be required to stop a second capture.  
**Callers:** `customer-bff` to open a gateway session and to read payment status. `admin-bff` for a staff refund. The payment aggregator calls `POST /webhooks/payments` on the public ingress. That route skips the BFF. Browsers never call this service for a capture.  
**Calls:** `identity-service` to introspect the session. `order-service` to read the order that is about to be paid. The gateway HTTPS API to create an order, fetch a payment, and refund.  
**Publishes:** `PaymentCaptured`, `PaymentFailed`, `RefundCompleted`  
**Consumes:** `OrderPlaced`, `OrderCancelled`, `ReturnAccepted`  
**Product rules:** [modules/10-payments.md](../modules/10-payments.md), [modules/08-checkout.md](../modules/08-checkout.md), [modules/12-returns.md](../modules/12-returns.md)

This is the build document for the gateway session, the verified webhook, capture, refund, and idempotency. Order-service owns order status. This service does not update the order row. It publishes an event. Order-service is the only writer that moves `payment_pending` to `paid` or `payment_failed`, and the only writer that moves `return_accepted` to `refunded`.

BuyyMart is a merchant of a payment aggregator. It is not a bank. Version 1 uses Razorpay test mode in dev and stage, and live keys only in production. The domain code talks to a gateway port. The Razorpay adapter is the only adapter in version 1. Tests inject the port and do not call Razorpay.

One checkout is one order and one payment row. COD does not create a gateway order.

The browser return URL is not a payment. Nothing in this service marks a payment captured because a browser called it.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as order-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as order-service |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Ids | UUID v4 | The payment id and the refund id |
| Logs | `pino` JSON to stdout | Log the order id, payment id, refund id, and gateway event id. Do not log a raw webhook body, a signature, a secret, a card, a VPA, an email, or a phone |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One checkout across the BFF, order, and this service |
| Metrics | `prom-client` on `GET /metrics` | Captures, failures, refunds, amount mismatches, outbox lag |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as order-service |
| Container port | 8080 | Service port 80 targets 8080 |

Money is an integer number of paise. The only currency is `INR`. A client amount is ignored. The amount sent to the gateway is the order’s `payablePaise`.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | Open a session, read a payment, accept the webhook, accept a staff refund. Minimum 3 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Publishes the outbox. Applies consumed order events. Reconciles a pending payment or a pending refund with the gateway |

The API does not publish to EventBridge itself. A payment change and its outbox row commit in one Postgres transaction. The worker sets `published_at`. A second poll does not send the row again.

`GET /health/ready` is Postgres `SELECT 1`. It does not call the gateway and it does not call order-service.

The webhook handler must see the exact request bytes. A re-serialized JSON body will not match the gateway signature. The route reads the raw body, checks the signature, and only then parses JSON.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `session` | Read the order, create one gateway order, return the public checkout fields |
| `webhook` | Verify the signature, store the event id, apply capture, failure, or refund |
| `reconcile` | Ask the gateway about a payment or refund the webhook may have missed |
| `refund` | Create a refund row and call the gateway. Publish only after the gateway confirms |
| `read` | The customer’s payment, or a staff read |
| `gateway` | The Razorpay adapter behind the port in section 8 |
| `outbox` | The events in section 10 |

This service does not store a card number, CVV, expiry, UPI PIN, netbanking password, or the raw webhook body.

---

## 4. How a request is trusted

`customer-bff` and `admin-bff` call:

```text
http://payment-service.money.svc.cluster.local
```

Every `/v1` call except health and metrics carries:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `payment-service`. `exp - iat` is at most 60 seconds. Algorithm HS256. Signed with `SERVICE_JWT_KEY` |
| `X-Request-Id` | UUID. Generated by the BFF when the caller did not send one. Echoed on the response |
| `traceparent` | W3C trace context. Forwarded to identity, order-service, and the gateway |
| `X-Session-Token` | The caller’s session on session create, payment read, and staff refund. This service calls identity `POST /v1/sessions/introspect`. Timeout 200 ms |
| `Idempotency-Key` | UUID. Required on `POST /v1/payments/sessions` and on `POST /v1/payments/:orderId/refunds` |

A missing, expired, or wrong-audience JWT is `401 service_unauthorized`. A session identity cannot load is `503 dependency_unavailable`. An invalid session is `401 session_invalid`.

| Route | Session |
| --- | --- |
| Open a session, customer read | Family `customer`. A seller, guest, or staff token is `403 forbidden` |
| Staff read, staff refund | Family `staff` |
| Webhook | No service JWT and no session. The gateway signature is the credential |

`POST /webhooks/payments` is on the public ingress at `https://api.buyymart.com/webhooks/payments`. It is not behind customer-bff. A service JWT on that route is ignored. A bad signature is `400 signature_invalid` and nothing is written.

A body `customerId` or `amountPaise` is ignored. The customer is the session subject. The amount is the order’s `payablePaise`.

Consumed events are not HTTP from the public ingress. The worker takes them from the bus. In development only, `POST /internal/events` on the worker accepts one envelope. That route is absent when `NODE_ENV` is `staging` or `production`.

---

## 5. What is stored

Stored fields are the order id, the gateway order id, the gateway payment id, status, method, amount in paise, and refund ids.

| Payment field | Meaning |
| --- | --- |
| `id` | This service’s payment id |
| `orderId` | The order-service id. One payment row per order |
| `customerId` | The session subject at session create |
| `status` | Section 7 |
| `amountPaise` | Copied from the order. This is what the gateway is asked to capture |
| `capturedPaise` | Set only by a verified capture whose amount matches |
| `refundedPaise` | Sum of refunds the gateway has confirmed |
| `method` | `upi`, `card`, `netbanking`, or `wallet`, copied from a verified webhook. Null until then |
| `gatewayOrderId` | The aggregator’s order id |
| `gatewayPaymentId` | The aggregator’s payment id. Null until a webhook or reconcile names it |
| `currency` | Always `INR` |

| Line copy | Meaning |
| --- | --- |
| `variantId`, `quantity`, `pricePaise`, `discountPaise` | Copied at session create so a later catalogue edit cannot change the refund |

A refund row stores `id`, `paymentId`, `orderId`, optional `variantId`, `amountPaise`, `reason` (`cancel`, `return`, `duplicate`, `staff`), `status`, and `gatewayRefundId`.

Not stored: PAN, card number, CVV, expiry, UPI PIN, VPA, customer email, customer phone, the webhook secret, the gateway key secret, and the raw webhook body. If saved cards are added later, the gateway holds the token. This service would store the token reference only. Version 1 does not store a token reference.

---

## 6. Open a session

`POST /v1/payments/sessions`. Customer session. Header `Idempotency-Key` is a UUID. A missing or non-UUID key is `400 invalid_body`.

```json
{ "orderId": "…" }
```

`orderId` is a UUID. Any `amountPaise` on the body is ignored and is not sent to the gateway.

This service calls `GET {ORDER_URL}/v1/orders/{orderId}` with a service JWT whose audience is `order-service`, and forwards `X-Session-Token`. Timeout 1 second.

| Order read | This service |
| --- | --- |
| `404` | `404 not_found`. No payment row |
| Timeout or any other failure | `503 dependency_unavailable`. No payment row |
| `payMode` `cod`, or status `cod_pending` | `409 not_prepaid`. Message `Cash on delivery does not use the payment gateway.` No payment row |
| Status `paid` | `409 already_paid`. Message `This order is already paid.` |
| Status other than `payment_pending` | `409 not_payable`. Message `This order cannot be paid.` No new gateway order |
| `payablePaise` missing, or a line missing `variantId`, `quantity`, `pricePaise`, or `discountPaise` | `503 dependency_unavailable`. No payment row. This service does not invent a discount |

The order customer read must include `discountPaise` on each line. Order-service already stores that column. A read that omits it is a broken dependency, not a zero discount.

The gateway is then asked for one order of `payablePaise` in `INR`. The receipt is the order id. Timeout 3 seconds. A gateway failure rolls the payment row back. The order stays `payment_pending`. The customer sees that payment could not be started. There is no second local row.

`201` body. `keyId` is the public gateway key. The key secret and the webhook secret are not in this body.

```json
{
  "paymentId": "…",
  "orderId": "…",
  "status": "pending",
  "amountPaise": 49900,
  "currency": "INR",
  "gatewayOrderId": "…",
  "keyId": "…"
}
```

The same order while the payment is still `pending` returns `200` and the same `gatewayOrderId`. It does not create another gateway order. The same `Idempotency-Key` and the same `orderId` return that payment. The same key with a different `orderId` is `409 idempotency_conflict`. A key already used by another customer is `404 not_found`.

A `failed` payment is not reopened. The customer places a new order. Inventory has already released the hold on its own clock.

---

## 7. Status

```text
pending → captured → refund_pending → partially_refunded
              ↓            ↓
           (terminal)   refunded
pending → failed
```

| From | To | Who |
| --- | --- | --- |
| no row | `pending` | Session create, after the gateway order exists |
| `pending` | `captured` | A verified `payment.captured` webhook, or reconcile, and only when the amount equals `amountPaise` |
| `pending` | `failed` | A verified `payment.failed` webhook, or reconcile |
| `captured` | `refund_pending` | A refund row is inserted and the gateway refund is requested |
| `refund_pending` | `partially_refunded` or `refunded` | A verified `refund.processed` webhook, or reconcile. `refunded` when `refundedPaise` equals `capturedPaise` |

`payment.captured` is published only in the transaction that moves `pending` to `captured`. The update is `WHERE status = 'pending'`. If no row changes, a second delivery does not publish again.

A capture whose amount is not `amountPaise` does not change status and does not publish. The event id is still stored. An audit row `amount_mismatch` is written for finance. The order stays `payment_pending` because this service never publishes `payment.captured` for that event.

Partial capture is not used. The gateway is asked for the full `payablePaise`.

COD never enters this table through session create. An `order.placed` with `payMode` `cod` is stored in `processed_events` and does not create a payment.

---

## 8. Gateway port

The domain calls this port. The Razorpay adapter is the version 1 implementation. Tests pass a fake.

| Method | Gateway | Timeout | Effect |
| --- | --- | --- | --- |
| `createOrder` | `POST /v1/orders` with `amount`, `currency: INR`, `receipt` = order id | 3 s | Returns `gatewayOrderId` |
| `fetchOrder` | `GET /v1/orders/{gatewayOrderId}/payments` | 3 s | Returns the latest payment status, `gatewayPaymentId`, `amountPaise`, and `method` |
| `refund` | `POST /v1/payments/{gatewayPaymentId}/refund` with `amount` and the refund id as the idempotency key | 3 s | Returns `gatewayRefundId`. Does not by itself publish `refund.completed` |

`GATEWAY_MODE=fake` is allowed only when `NODE_ENV=development`. `createOrder` returns `gatewayOrderId` `fake_` plus the order id and `keyId` `fake_key`. It does not call the network. Stage and prod refuse to start unless `GATEWAY_MODE=razorpay` and `GATEWAY_KEY_ID`, `GATEWAY_KEY_SECRET`, and `WEBHOOK_SECRET` are all non-empty.

The adapter sends `Authorization: Basic` of `key_id:key_secret`. That header is not logged.

### Webhook

`POST /webhooks/payments`. No JWT.

1. Read the raw body. Check `X-Razorpay-Signature`. The value must be the hex HMAC-SHA256 of those exact bytes with `WEBHOOK_SECRET`. Compare in constant time. A missing or wrong signature is `400 signature_invalid`. Nothing is written.
2. Read `x-razorpay-event-id`. A missing or empty id is `400 invalid_body`. Nothing is written.
3. Insert that id into `webhook_events`. If the id already exists, return `200` and do no work. A gateway retry for about 24 hours must not capture twice and must not refund twice.
4. Parse JSON only after the signature check.

| Gateway `event` | Effect |
| --- | --- |
| `payment.captured` | Section 7 capture, when the payload amount and `order_id` match a `pending` row |
| `payment.failed` | Section 7 failure for that gateway order |
| `refund.processed` | Mark that refund `completed`, add its amount to `refundedPaise`, publish `refund.completed` |
| Anything else | Store the event id, write nothing else, return `200` |

The payload amount is paise. Method is copied only when it is `upi`, `card`, `netbanking`, or `wallet`. Any other method is stored as null. Card network fields, VPA, email, and contact are dropped before the write.

A gateway order this service does not know is stored as the event id, an audit row `unknown_order`, and `200`. No event is published.

A second captured payment for an order that is already `captured`, with a different `gatewayPaymentId`, inserts a refund of reason `duplicate` for that second payment and calls the gateway. It does not publish a second `payment.captured`. The customer who paid twice is refunded on the second instrument. The first capture stays.

Return `200` after the transaction commits. Work that can wait, such as mail and the ledger, is not done on this request.

---

## 9. Refunds

A refund exists only for a captured prepaid payment, and only for an amount up to `capturedPaise - refundedPaise`. The gateway call uses the refund id as its idempotency key. `refund.completed` is published from `refund.processed` or from reconcile, not from the moment the refund row is inserted.

Version 1 does not refund delivery on a delivered return. The line amount is `pricePaise * quantity - discountPaise` from the copied line. A negative or zero result does not call the gateway.

| Source | Amount | `variantId` on `refund.completed` |
| --- | --- | --- |
| `order.cancelled` and every group of that order is in `sellerIds`, and a captured payment exists | The remaining captured paise, which includes delivery | Omitted. The order is already `cancelled`. This event records the money |
| `order.cancelled` for some sellers only | The sum of those sellers’ copied lines. Delivery stays | One `refund.completed` per line, each with that `variantId` |
| `return.accepted` | That line’s copied amount. Delivery is not included | That `variantId`, so order-service can move the `return_accepted` line to `refunded` |
| Second gateway payment on an order already captured | The second payment’s amount | Omitted |
| Staff `POST /v1/payments/:orderId/refunds` | The body amount, capped by the remaining capture | The body `variantId` when present |

`order.cancelled` with reason `stock_rejected` usually has no payment row. Store the event id and do nothing else. COD has no payment row. Store the event id and do not call the gateway. A missing payment for any of these events is not an error.

A refund that would exceed the remaining capture is not sent. An audit row `refund_exceeds` is written. The event id is still stored.

Two workers must not create two refund rows for the same return. The unique key is `(order_id, variant_id, reason)` for a return, and one `cancel` refund per payment for a full cancel. A second delivery of the same order event id does nothing because `processed_events` already has it.

### Staff refund

`POST /v1/payments/:orderId/refunds`. Staff session.

```json
{ "amountPaise": 49900, "variantId": null, "reason": "staff", "confirmedBy": null }
```

`amountPaise` is a positive integer. `variantId` is a UUID or null. When `amountPaise` is greater than `REFUND_MAKER_CHECKER_PAISE`, `confirmedBy` must be a different staff subject. The same staff member confirming their own refund is `403 forbidden`. A missing confirmer above the threshold is `409 confirm_required`. Automatic refunds from `order.cancelled` and `return.accepted` are already authorized by those events and do not wait for a second person.

This route inserts the refund row and calls the gateway. It does not publish `refund.completed` itself.

---

## 10. Events

The envelope is `{ id, type, source, time, traceId, data }`. Wire `type` values are lowercase. `source` on publish is `payment-service`.

| Name | `type` | When |
| --- | --- | --- |
| `PaymentCaptured` | `payment.captured` | A pending payment became `captured` for the full `amountPaise` |
| `PaymentFailed` | `payment.failed` | A pending payment became `failed` |
| `RefundCompleted` | `refund.completed` | The gateway confirmed a refund |

`payment.captured` data. `amountPaise` equals the order’s `payablePaise`. Order-service ignores a capture whose amount does not match, so this service must not publish a mismatched amount.

```json
{ "orderId": "…", "amountPaise": 49900, "currency": "INR" }
```

`payment.failed` data:

```json
{ "orderId": "…" }
```

`refund.completed` data. `variantId` is present for a return line and omitted for a full-order cancel or a duplicate-payment refund.

```json
{ "orderId": "…", "refundId": "…", "amountPaise": 49900, "variantId": "…" }
```

There is no phone, no address, and no gateway secret on these events.

### Consumed

The worker stores `processed_events.id` as the envelope `id`. A second delivery of the same id does not create a second refund and does not call the gateway again.

| Event | `type` | `source` | Effect |
| --- | --- | --- | --- |
| `OrderPlaced` | `order.placed` | `order-service` | Remember `payMode`. `cod` does not create a payment. `prepaid` waits for session create |
| `OrderCancelled` | `order.cancelled` | `order-service` | Section 9, when a captured payment exists |
| `ReturnAccepted` | `return.accepted` | `order-service` | Section 9 for that `variantId` |

`return.accepted` data this service reads is `{ orderId, sellerId, variantId, quantity, sellable }`. The price comes from the copied line, not from the event. `sellable` does not change the refund. Inventory puts a sellable unit back. This service still refunds the prepaid line.

An unknown type is ignored. A payload missing `orderId` is ignored. The event id is still stored.

Empty `EVENTBRIDGE_BUS_NAME` means the worker logs the envelope and does not call AWS. The development hook is the only ingest in that mode.

---

## 11. Reconcile

The browser return URL is not used here.

Every 2 minutes the worker asks the gateway about each `pending` payment whose `created_at` is older than 2 minutes and younger than 30 minutes. A capture or failure from that read uses the same status transition as the webhook. The idempotency key for the reconcile attempt is `reconcile:{gatewayOrderId}:{status}`. A later webhook with the gateway’s own event id finds the payment already `captured` or `failed` and does not publish a second time.

The same loop asks the gateway about each refund still `pending` and older than 2 minutes. A processed refund publishes one `refund.completed`.

The worker does not invent a failure when the gateway has no payment yet. The row stays `pending` until 30 minutes have passed, then it is left `pending` for a person to read. It is not marked `failed` only because the webhook was late. Order-service’s own 15-minute sweep moves the order to `payment_failed` if it is still `payment_pending`. This service does not publish `payment.failed` for that sweep.

---

## 12. HTTP API

`/v1` routes require the service JWT. The webhook does not.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| POST | `/v1/payments/sessions` | Customer | `201` a new pending payment, or `200` when that order already has one |
| GET | `/v1/payments/:orderId` | That customer, or staff | `200` the payment. Another customer is `404` |
| POST | `/v1/payments/:orderId/refunds` | Staff | `202` refund requested. Not `refunded` yet |
| POST | `/webhooks/payments` | Gateway signature | `200` after the event id is stored, including a duplicate |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | None. No JWT | Prometheus text |

JSON errors use `{ "code", "message", "requestId" }`. `message` is safe to show. The webhook’s `400` body uses the same shape. A duplicate webhook is `200` with `{ "duplicate": true }`.

| Code | HTTP | When |
| --- | --- | --- |
| `invalid_body` | 400 | Bad JSON, a bad order id, a missing idempotency key |
| `signature_invalid` | 400 | Webhook signature does not match the raw body |
| `session_invalid` | 401 | No session, or identity rejected it |
| `service_unauthorized` | 401 | Bad service JWT on `/v1` |
| `forbidden` | 403 | Seller or guest on a customer route, or a customer on a staff route, or a staff member confirming their own refund |
| `not_found` | 404 | Unknown order, another customer’s payment, no payment to read |
| `not_prepaid` | 409 | COD order |
| `already_paid` | 409 | The order read is already `paid`, or this payment is `captured` and the caller asked for a new session |
| `not_payable` | 409 | The order is not `payment_pending` |
| `idempotency_conflict` | 409 | The same key was reused for a different order or a different refund body |
| `confirm_required` | 409 | A staff refund above the maker-checker amount has no second staff id |
| `refund_exceeds` | 409 | Staff refund larger than the remaining capture |
| `dependency_unavailable` | 503 | Identity, order-service, or the gateway failed on a caller who is waiting |
| `payments_unavailable` | 503 | Postgres is down |

`payments_captured_total` counts the transition to `captured`. `payments_failed_total` counts the transition to `failed`. `refunds_completed_total` counts a confirmed refund. `payments_amount_mismatch_total` counts a capture whose amount did not match. `payments_outbox_unpublished` is the gauge of outbox rows with `published_at` null. Set the gauge on `GET /metrics`.

Customer read:

```json
{
  "paymentId": "…",
  "orderId": "…",
  "status": "pending",
  "amountPaise": 49900,
  "currency": "INR",
  "method": null
}
```

The customer read does not include `gatewayPaymentId`, the key secret, or the webhook secret. Staff read may include `gatewayOrderId` and `gatewayPaymentId`.

---

## 13. PostgreSQL schema

Database `payment`. The application role `payment_app` can `SELECT`, `INSERT`, and `UPDATE` these tables. It cannot `DELETE`, `DROP`, `TRUNCATE`, or alter schema. A failure is a status change, not a delete. The migration role `payment_migrator` runs `migrations/` and is not the runtime role. The API refuses to start if `DATABASE_MIGRATOR_URL` is set or if `DATABASE_URL` uses `payment_migrator`.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE payment_status AS ENUM (
  'pending', 'captured', 'failed', 'refund_pending', 'partially_refunded', 'refunded'
);

CREATE TYPE refund_status AS ENUM ('pending', 'completed', 'failed');

CREATE TYPE payment_method AS ENUM ('upi', 'card', 'netbanking', 'wallet');

CREATE TABLE payments (
  id                   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id             uuid NOT NULL UNIQUE,
  customer_id          uuid NOT NULL,
  status               payment_status NOT NULL,
  amount_paise         integer NOT NULL CHECK (amount_paise > 0),
  captured_paise       integer NOT NULL DEFAULT 0 CHECK (captured_paise >= 0),
  refunded_paise       integer NOT NULL DEFAULT 0 CHECK (refunded_paise >= 0),
  currency             text NOT NULL DEFAULT 'INR',
  method               payment_method,
  gateway_order_id     text NOT NULL UNIQUE,
  gateway_payment_id   text UNIQUE,
  created_at           timestamptz NOT NULL DEFAULT now(),
  captured_at          timestamptz,
  CHECK (refunded_paise <= captured_paise)
);

CREATE TABLE payment_lines (
  payment_id      uuid NOT NULL REFERENCES payments (id),
  variant_id      uuid NOT NULL,
  quantity        integer NOT NULL CHECK (quantity BETWEEN 1 AND 10),
  price_paise     integer NOT NULL CHECK (price_paise >= 0),
  discount_paise  integer NOT NULL CHECK (discount_paise >= 0),
  PRIMARY KEY (payment_id, variant_id)
);

CREATE TABLE refunds (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id         uuid NOT NULL REFERENCES payments (id),
  order_id           uuid NOT NULL,
  variant_id         uuid,
  amount_paise       integer NOT NULL CHECK (amount_paise > 0),
  reason             text NOT NULL,
  status             refund_status NOT NULL,
  gateway_refund_id  text UNIQUE,
  created_at         timestamptz NOT NULL DEFAULT now(),
  completed_at       timestamptz
);

CREATE UNIQUE INDEX refunds_return_once
  ON refunds (order_id, variant_id, reason)
  WHERE variant_id IS NOT NULL;

CREATE UNIQUE INDEX refunds_full_cancel_once
  ON refunds (payment_id)
  WHERE reason = 'cancel' AND variant_id IS NULL;

CREATE TABLE session_keys (
  idempotency_key uuid PRIMARY KEY,
  customer_id     uuid NOT NULL,
  order_id        uuid NOT NULL,
  payment_id      uuid NOT NULL REFERENCES payments (id),
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE webhook_events (
  id           text PRIMARY KEY,
  event_type   text NOT NULL,
  received_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE processed_events (
  id           uuid PRIMARY KEY,
  type         text NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE audit (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id   uuid,
  payment_id uuid,
  action     text NOT NULL,
  at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE outbox (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  type         text NOT NULL,
  payload      jsonb NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz
);

CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;
```

The advisory lock for migrations is `2147483007`.

---

## 14. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `payment_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Migration job only. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `payment-service` |
| Secret | `GATEWAY_KEY_ID` | Razorpay key id. Public. Still not committed |
| Secret | `GATEWAY_KEY_SECRET` | Razorpay key secret |
| Secret | `WEBHOOK_SECRET` | HMAC secret for `X-Razorpay-Signature` |
| Config | `GATEWAY_MODE` | `fake` in development only. `razorpay` in stage and prod |
| Config | `ORDER_URL` | Order read. Required in stage and prod |
| Config | `IDENTITY_URL` | Introspect sessions |
| Config | `REFUND_MAKER_CHECKER_PAISE` | Staff refunds above this need a second staff id. Default `10000000` (₹1,00,000) |
| Config | `EVENTBRIDGE_BUS_NAME` | Empty means the worker logs the envelope and does not call AWS |

Secrets live in Secrets Manager at `buyymart/{env}/payment-service`. They are not in the image and not in git. Stage uses gateway test mode. Prod uses live keys in the production account only. Stage and prod env files set `ORDER_URL` to `http://order-service.commerce.svc.cluster.local` and do not contain a database password, `payment_app_dev`, or `payment_migrator_dev`.

---

## 15. Local run

Docker Compose for this service is Postgres 16, the API, and the worker. There is no Redis. Identity and order-service must already be running for a real session. Tests inject the order read and the gateway and do not start Compose.

Migrations run before the API starts. Host ports are 8095 for the API and 8096 for the worker, so this stack can run beside order-service on 8093. `IDENTITY_URL` is `http://host.docker.internal:8080`. `ORDER_URL` is `http://host.docker.internal:8093`. `GATEWAY_MODE` is `fake`. `WEBHOOK_SECRET` in Compose is a dev value and is not copied into `deploy/stage.env` or `deploy/prod.env`.

`POST /internal/events` on the worker accepts one envelope from section 10 only when `NODE_ENV=development`. It requires the service JWT and is not registered in staging or production.

---

## 16. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Customer opens a session for a `payment_pending` prepaid order | `201` `pending`. Gateway `createOrder` is called once with `payablePaise`. A client `amountPaise` is not what was sent |
| Same order, second session call | `200` and the same `gatewayOrderId`. `createOrder` is not called again |
| Same idempotency key, different `orderId` | `409 idempotency_conflict` |
| COD order | `409 not_prepaid`. No payment row |
| Order read `404` | `404 not_found`. No payment row |
| Order read missing `discountPaise` | `503 dependency_unavailable`. No payment row |
| Seller session on `POST /v1/payments/sessions` | `403 forbidden` |
| Webhook with a bad signature | `400 signature_invalid`. No `webhook_events` row and no outbox row |
| `payment.captured` for the same amount | Status `captured`. One `payment.captured` whose `amountPaise` equals the order |
| The same `x-razorpay-event-id` again | `200` duplicate. No second outbox row |
| `payment.captured` whose amount is off by one | Stays `pending`. No `payment.captured`. One `amount_mismatch` audit row |
| `payment.failed` while `pending` | Status `failed`. One `payment.failed` |
| A second gateway payment after capture | No second `payment.captured`. A duplicate refund is requested |
| `return.accepted` | One refund for `pricePaise * quantity - discountPaise`. Delivery is not included. `refund.completed` includes `variantId` only after the gateway confirms |
| `order.cancelled` on a captured payment | One refund of the remaining capture. A second delivery of that event id does not call the gateway again |
| `order.cancelled` with no payment row | Event id stored. No gateway call |
| Staff refund above the maker-checker amount with no second staff id | `409 confirm_required`. No refund row |
| Reconcile finds a capture the webhook missed | One `payment.captured`. A later webhook does not publish another |
| Postgres down | Session create is `503 payments_unavailable`. Ready is `503`. Live stays `200` |
| Worker publish | `published_at` is set. A second poll does not send it |
| `NODE_ENV=production` | `POST /internal/events` is not registered. `GATEWAY_MODE=fake` refuses to start |

---

## 17. Outside this service

| Concern | Owner |
| --- | --- |
| Order status, including `paid` and `refunded` | `order-service`. It consumes the events in section 10 |
| The live price and the payable total | `order-service`. This service copies them |
| The stock hold and the 15-minute release | `inventory-service` |
| Opening this session after `201` from order-service | `customer-bff` |
| Showing the browser return URL | `customer-bff`. The URL does not capture |
| COD bank refunds | `order-service` staff reference, and later `settlement-service`. This service does not call the gateway for COD |
| Ledger, TCS, commission | `settlement-service`, from `payment.captured` and `refund.completed` |
| SMS | `notification-service` |
| A ticket that is not a refund | `support-service` |

# notification-service — implementation spec

**Namespace:** `engagement`  
**Database:** Aurora PostgreSQL database `notification` on cluster `bm-ops`  
**Cache:** none. A Redis flush must not drop a queued message or a template.  
**Callers:** `identity-service` for the OTP SMS and the password-reset email. The worker takes customer and seller messages from the bus. Staff read a message through `admin-bff`, when that route is added. Browsers never call this service.  
**Calls:** `order-service` for the customer phone on an order event. `identity-service` to introspect a staff session on the read and template routes. The SMS provider and the email provider. No other service calls either provider.  
**Publishes:** nothing. A send does not change an order, a payment, or a seller.  
**Consumes:** `OrderConfirmed`, `ShipmentBooked`, `ShipmentUpdated`, `RefundCompleted`, `SellerApproved`  
**Product rules:** [modules/15-notifications.md](../modules/15-notifications.md)

This is the build document for SMS and email. Push and WhatsApp are not in version 1. Marketing SMS is not in version 1.

The service does not decide that an order is paid, confirmed, or delivered. It sends after another service has already decided. A failed send does not roll back that decision.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as fulfilment-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as fulfilment-service |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Redis | none | Duplicate events and the send queue live in Postgres |
| Ids | UUID v4 from `gen_random_uuid()` | Message ids. The provider message id is the provider’s id |
| Logs | `pino` JSON to stdout | Log the template, message id, and provider message id. Do not log an OTP, a reset link, a full phone, a full email, a PAN, or a bank account |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One OTP across identity and this API |
| Metrics | `prom-client` on `GET /metrics` | Sends, queued rows, failed rows |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as fulfilment-service |
| Container port | 8080 | Service port 80 targets 8080 |

Money in a template is formatted from integer paise. `49900` is the text `499.00`. The paise value is not stored as a float.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | OTP SMS, password-reset email, template replace, and staff message read. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Consumes the events in section 7. Sends queued messages. Retries a provider failure |

The OTP and the reset email are answered on the API. Identity waits 2 seconds and does not poll. Those two routes must not wait on the worker.

`GET /health/ready` is Postgres `SELECT 1`. It does not call the SMS provider. A provider outage must not take this API out of the load balancer. Identity would then fail every login.

---

## 3. What this build includes

| In this build | Not in this build |
| --- | --- |
| `POST /v1/sms/otp` and `POST /v1/email/password-reset`, the calls identity already makes | A new identity client. This spec does not change identity-service |
| Customer SMS for confirm, AWB, delivered, and refund, once the order read returns a phone | Inventing a phone that is not on the order |
| Seller approval and new-order email rows that stay queued until a recipient email exists on the event | A seller email lookup that no service exposes today |
| DLT template rows and a staff read of a message | Admin Console screens, and an admin-bff route. Those come later |
| Dev log of the OTP so an engineer can sign in | Push, WhatsApp, and marketing SMS |

`seller.kyc_rejected` is not an event. seller-service updates the row and does not publish. This service does not poll for it. A rejection email waits until that event exists.

Staff invite email is not a call identity makes. The admin API returns the temporary password to the creator. Password reset is the email this service sends.

---

## 4. How a request is trusted

Browsers never call notification-service. Cluster DNS:

```text
http://notification-service.engagement.svc.cluster.local
```

These paths are not on the public ingress.

| Route | Credential |
| --- | --- |
| `POST /v1/sms/otp` and `POST /v1/email/password-reset` | The body identity already sends. Identity’s client sends `Content-Type: application/json` and no `Authorization` header. A missing JWT is accepted on these two routes only. A JWT that is present and fails the check below is `401 service_unauthorized` |
| Template replace and message read | `Authorization: Bearer` service JWT, audience `notification-service`, `exp - iat` at most 60 seconds, HS256, `SERVICE_JWT_KEY`. Plus `X-Session-Token`. This service calls identity `POST /v1/sessions/introspect`. Timeout 200 ms |
| Health and metrics | None |
| `POST /internal/events` | Service JWT. The route exists only when `NODE_ENV=development` |

`X-Request-Id` is a UUID, generated when missing, and echoed. `traceparent` is forwarded to order-service and the providers.

An invalid session is `401 session_invalid`. Identity being down on a staff route is `503 dependency_unavailable`.

| Route | Who |
| --- | --- |
| OTP and password reset | identity-service, as the client works today |
| Template replace | Family `staff`, role `admin` or `super_admin` |
| Message read | Family `staff`. A seller or a customer is `403 forbidden` |

---

## 5. What is stored

| Message field | Meaning |
| --- | --- |
| `id` | This service’s id |
| `template` | Which message |
| `channel` | `sms` or `email` |
| `eventId` | The domain event, when the send came from the bus. Null for OTP and password reset. Unique when set |
| `status` | `queued`, `sent`, or `failed` |
| `providerMessageId` | For a support lookup. Null until the provider accepts |
| `phoneTail` | Last two digits, or null for email |
| `destinationHash` | HMAC-SHA256 of the E.164 phone or the normalised email, with `NOTIFICATION_HMAC_PEPPER`. The raw phone and the raw email are not columns |
| `lastError` | Safe to show to staff |
| `attempts`, `nextAttemptAt` | Provider retries for a queued event message |

Not stored: the OTP, the reset link, the raw phone, the raw email, a PAN, or a bank account. A retry of an event message reads the phone from order-service again. It does not read it back from this table.

A duplicate `eventId` does not insert a second message and does not send a second SMS.

---

## 6. OTP and password reset

These are the routes identity-service already calls. Timeout on that client is 2 seconds. One retry on a network error. No retry on 4xx or 5xx. If the call fails, identity deletes the OTP key and returns `503 otp_delivery_failed`. A password-reset failure does not change the `202` identity already returned to the browser.

### `POST /v1/sms/otp`

Body: `{ "phone", "code" }`. `phone` is E.164 `+91` plus 10 digits, first digit 6–9. `code` is 6 digits. Anything else is `400 invalid_body`. There is no event id. A second call sends again. Identity only retries when the first attempt never got a response.

| Environment | Result |
| --- | --- |
| Development | Do not call the provider. Write one log line that includes the code, so an engineer can sign in. `202` `{ messageId }` |
| Staging | If `phone` is on `SMS_ALLOWLIST`, call the provider. Any other number does not call the provider and still returns `202` with a fake message id. The log line has the last two digits and does not include the code |
| Production | Call the provider. The log has `messageId` and the last two digits. It does not include the code |

`messageId` is a non-empty string. Development and a stage number that is not on the list use `fake-` plus the message id. The row is `sent`. The code is not written to Postgres.

The provider call, when it happens, must finish inside the request. Timeout 1.5 seconds. A timeout or a 5xx from the provider is `503 dependency_unavailable`. No `sent` row is kept for that attempt. Identity then fails the OTP.

### `POST /v1/email/password-reset`

Body: `{ "email", "link" }`. Both non-empty after trim. Email is stored only as a hash. The link is the one identity generated (`https://seller.buyymart.com/reset#token=…` or `https://admin.buyymart.com/reset#token=…`). This service does not generate a token and does not check that the token exists.

Same environment rules as OTP, with `EMAIL_ALLOWLIST` in staging. Development logs the link. Staging and production do not. Success is `202` `{ messageId }`. A provider failure is `503 dependency_unavailable`.

---

## 7. Messages from events

The worker stores `processed_events.id` as the envelope `id`. A second delivery of the same id does not send again.

The envelope is `{ id, type, source, time, traceId, data }`. Wire `type` values are lowercase.

| Event | `type` | `source` | Payload today | Message |
| --- | --- | --- | --- | --- |
| `OrderConfirmed` | `order.confirmed` | `order-service` | `{ orderId, sellerId, invoiceNumber }` | Customer SMS `order_confirmed`. One event per seller group, so a multi-seller order can send more than one SMS |
| `ShipmentBooked` | `shipment.booked` | `fulfilment-service` | `{ orderId, sellerId, awb }` for forward. Reverse adds `variantId` and `direction: "reverse"` | Customer SMS `shipment_booked`, including `awb`. Reverse is the same SMS |
| `ShipmentUpdated` | `shipment.updated` | `fulfilment-service` | `{ orderId, sellerId, status }` | Customer SMS `shipment_delivered` only when `status` is `delivered`. `shipped` and `out_for_delivery` are stored and not sent |
| `RefundCompleted` | `refund.completed` | `payment-service` | `{ orderId, refundId, amountPaise, variantId? }` | Customer SMS `refund_completed`. `amount` in the template is paise rendered as rupees |
| `SellerApproved` | `seller.approved` | `seller-service` | `{ sellerId, kind }` | Email `seller_approved`. There is no email on the event |

None of these payloads include a phone or an email. `order.confirmed` does not include an address. Fulfilment’s spec says the same thing.

### Customer phone

`GET {ORDER_URL}/v1/orders/:id`, audience `order-service`, no session, timeout 1 second. The phone this service needs is `address.phone` on a `200` body.

order-service today requires `X-Session-Token` on that GET, so a service call with no session is `401 session_invalid`. This service does not invent a phone and does not send a session it does not have. The message stays `queued` with `lastError` `order_unreadable`. The provider is not called. The worker retries. The order stays whatever status it already had.

A `200` without `address.phone` is the same error. A phone that is not E.164 `+91` is `order_unreadable`. Do not send it.

`order_sms` consent is collected at registration and is not on the order event. Identity does not expose a service read of that consent. This service does not guess. OTP does not check consent. Order SMS is sent when the phone read succeeds. A later withdrawal is not visible here until identity publishes it.

### Seller email

`seller.approved` and the seller copy of a new order need an email. seller-service’s no-session read does not include one, and identity has no service read of the owner address. The message is inserted `queued` with `lastError` `recipient_unknown`. The provider is not called. The event id is still stored, so a redelivery does not insert a second row.

`order.confirmed` still sends the customer SMS when the phone read works. The missing seller email does not block that SMS. They are two message rows. The seller row uses event id `{envelope id}:seller` so the unique index can hold both.

### Ignored

An unknown type is ignored. A payload missing `orderId` on an order or shipment or refund event is ignored. A `seller.approved` payload missing `sellerId` is ignored. The event id is still stored.

`shipment.updated` with a status other than `delivered` stores the event and does not create a message.

This service does not consume `order.delivered`. The delivered SMS is `shipment.updated` only, so the customer does not get two.

---

## 8. Templates

Templates are rows, not a free-text body from the caller. The OTP route does not accept a body. The event worker does not accept a body.

Seeded codes:

| Code | Channel | Variables |
| --- | --- | --- |
| `otp` | sms | `code` |
| `order_confirmed` | sms | `orderId` |
| `shipment_booked` | sms | `orderId`, `awb` |
| `shipment_delivered` | sms | `orderId` |
| `refund_completed` | sms | `orderId`, `amount` |
| `password_reset` | email | `link` |
| `seller_approved` | email | `sellerId` |
| `seller_new_order` | email | `orderId`, `sellerId` |

`PUT /v1/templates/:code` replaces `dltTemplateId`, `subject`, and `body`. Admin or super_admin only. `body` is 1–500 characters. `subject` is required for email and must be null for SMS. `dltTemplateId` is required for an SMS template when `PROVIDER_MODE=live`, and may be empty in development. An unknown code is `404 not_found`. Support staff are `403 forbidden`.

A variable in the body must be one of that code’s variables, wrapped as `{orderId}`. Any other brace is `400 invalid_body` and the previous row stays. The sender fills only those variables. It does not append a free-text suffix.

In production and staging, an SMS template with an empty `dltTemplateId` does not call the provider. The message stays `queued` with `lastError` `template_unregistered` until a staff member sets the id.

---

## 9. Provider

`PROVIDER_MODE=fake` is allowed only when `NODE_ENV=development`. It does not open a socket. Stage and prod refuse to start when the mode is `fake`, when `SMS_PROVIDER_URL` or `EMAIL_PROVIDER_URL` is empty, or when `NOTIFICATION_HMAC_PEPPER` is empty.

A live SMS call sends the DLT template id, the variable map, and the phone. A live email call sends the subject, the filled body, and the address. Timeout 1.5 seconds on the API routes and 5 seconds on the worker.

| Result | Message |
| --- | --- |
| Provider returns a message id | `sent`. Store `providerMessageId` |
| Timeout, 5xx, or a body with no message id | Stay `queued` for an event message. Set `lastError` `provider_unavailable`. Set `nextAttemptAt` 1 minute later, then 2, then 4, capped at 15 minutes. OTP and password reset return `503` instead of queueing, because identity is waiting |

Version 1 does not register a provider webhook. `delivered` from the module is not a status in this schema until that callback exists. `sent` means the provider accepted the message.

---

## 10. HTTP API

| Method | Path | Who | Success |
| --- | --- | --- | --- |
| POST | `/v1/sms/otp` | identity, section 6 | `202` `{ messageId }` |
| POST | `/v1/email/password-reset` | identity, section 6 | `202` `{ messageId }` |
| PUT | `/v1/templates/:code` | Admin staff | `200` |
| GET | `/v1/messages/:id` | Staff | `200` |
| GET | `/health/live` | None | `200` if the process is up |
| GET | `/health/ready` | None | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | None | Prometheus text |

`GET /v1/messages/:id` returns `id`, `template`, `channel`, `status`, `providerMessageId`, `phoneTail`, `lastError`, `eventId`. No phone, no email, no OTP, no link. An unknown id is `404 not_found`.

JSON errors use `{ "code", "message", "requestId" }`. `message` is safe to show.

| Code | HTTP | When |
| --- | --- | --- |
| `invalid_body` | 400 | Bad phone, bad code, bad email, bad template body |
| `session_invalid` | 401 | No session on a staff route, or identity rejected it |
| `service_unauthorized` | 401 | A JWT was sent and it is not valid for `notification-service` |
| `forbidden` | 403 | A seller or customer on a staff route, or support editing a template |
| `not_found` | 404 | Unknown template code or message id |
| `dependency_unavailable` | 503 | Identity introspect failed, or the provider failed an OTP or reset send |
| `notification_unavailable` | 503 | Postgres is down |

`notification_sent_total` counts a transition to `sent`. `notification_queued` is the gauge of rows still `queued`. `notification_failed` is the gauge of rows `failed`. Set the gauges on `GET /metrics`. A row becomes `failed` when `attempts` reaches 8. It stays in the table. This service does not delete it.

---

## 11. PostgreSQL schema

Database `notification`. The application role `notification_app` can `SELECT`, `INSERT`, and `UPDATE`. It cannot `DELETE`. The migration role `notification_migrator` runs `migrations/` and is not the runtime role. The API refuses to start if `DATABASE_MIGRATOR_URL` is set or if `DATABASE_URL` uses `notification_migrator`.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE templates (
  code              text PRIMARY KEY,
  channel           text NOT NULL CHECK (channel IN ('sms', 'email')),
  dlt_template_id   text,
  subject           text,
  body              text NOT NULL,
  updated_at        timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE messages (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  template              text NOT NULL,
  channel               text NOT NULL CHECK (channel IN ('sms', 'email')),
  event_id              text,
  status                text NOT NULL CHECK (status IN ('queued', 'sent', 'failed')),
  provider_message_id   text,
  phone_tail            text,
  destination_hash      text,
  last_error            text,
  attempts              integer NOT NULL DEFAULT 0,
  next_attempt_at       timestamptz,
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX messages_event_idx ON messages (event_id) WHERE event_id IS NOT NULL;

CREATE TABLE processed_events (
  id          text PRIMARY KEY,
  type        text NOT NULL,
  received_at timestamptz NOT NULL DEFAULT now()
);
```

Seed `templates` in the same migration for the eight codes in section 8. SMS `dlt_template_id` starts null. Email `subject` is a short fixed subject. Bodies use only that code’s variables.

The advisory lock for migrations is `2147483010`.

---

## 12. Configuration

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `notification_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Migration job only. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `notification-service` |
| Secret | `NOTIFICATION_HMAC_PEPPER` | HMAC key for `destination_hash`. Required in stage and prod. Not the OTP pepper |
| Config | `IDENTITY_URL` | Staff session introspect |
| Config | `ORDER_URL` | Customer phone. Required in stage and prod |
| Config | `PROVIDER_MODE` | `fake` only in development. `live` in stage and prod |
| Config | `SMS_PROVIDER_URL` | Required when mode is `live` |
| Config | `EMAIL_PROVIDER_URL` | Required when mode is `live` |
| Config | `SMS_ALLOWLIST` | Comma-separated E.164 numbers. Used only when `NODE_ENV=staging` |
| Config | `EMAIL_ALLOWLIST` | Comma-separated emails. Used only when `NODE_ENV=staging` |
| Config | `EVENTBRIDGE_BUS_NAME` | Empty means the worker does not call AWS. The development hook is the only ingest |

Secrets live in Secrets Manager at `buyymart/{env}/notification-service`. They are not in the image and not in git. The Compose pepper is a dev value and is not copied into `deploy/stage.env` or `deploy/prod.env`.

---

## 13. Local run

Docker Compose for this service is Postgres 16, the API, and the worker. Order-service and identity must already be running for a real event send. Tests inject those clients and do not start Compose.

Migrations run before the API starts. Host ports are 8103 for the API and 8104 for the worker, so this stack can run beside fulfilment-service on 8101 and 8102.

`POST /internal/events` on the worker accepts one envelope from section 7 only when `NODE_ENV=development`. It requires the service JWT and is not registered in staging or production.

`PROVIDER_MODE=fake`. An OTP still returns `202` and writes the code to the log. A confirmed event still inserts `queued` when the order read fails.

Identity’s dev stack can point `NOTIFICATION_URL` at this API instead of the stub in identity-service. The request body does not change.

---

## 14. Tests

Tests call the route functions with an injected store and injected order and provider clients. They do not start Postgres, identity, or a provider.

| Check | Expected |
| --- | --- |
| OTP in development | `202` with a non-empty `messageId`. The provider client is not called. The code is not on the stored row |
| OTP with a bad phone | `400 invalid_body`. No message row |
| OTP provider timeout | `503 dependency_unavailable`. No `sent` row |
| Stage number not on the allow-list | `202`. The provider client is not called |
| Password reset in development | `202`. The link is not on the stored row |
| `order.confirmed` twice | One customer message |
| Order read returns 401 | Status stays `queued`. `lastError` is `order_unreadable`. The provider client is not called |
| Order read returns a phone, fake mode | Status `sent`. Provider client is not called |
| `shipment.updated` with `shipped` | No message row |
| `shipment.updated` with `delivered` and a phone | One `shipment_delivered` message |
| `shipment.booked` | The template variables include `awb` and do not include a phone |
| `refund.completed` with `amountPaise` 49900 | The amount text is `499.00` |
| `seller.approved` | A queued email. `lastError` is `recipient_unknown`. The provider client is not called |
| Support staff replaces a template | `403 forbidden`. The stored body is unchanged |
| A template body with an unknown variable | `400 invalid_body`. The previous body stays |
| Staff reads a message | `200`. The body has `phoneTail` and no phone |

---

## 15. What stays in other services

| Concern | Owner |
| --- | --- |
| The OTP value, the send cap, and the lock | `identity-service` |
| The reset token and the link | `identity-service` |
| Order status | `order-service` |
| The AWB | `fulfilment-service` |
| The refund amount | `payment-service` |
| Seller approval | `seller-service`. Rejection is not an event yet |
| Consent rows | `identity-service` |
| Showing a failed-send count on Admin home | `admin-bff`, from `GET /metrics` or the message read, later |

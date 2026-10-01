# customer-bff — implementation spec

**Namespace:** `edge`  
**Database:** none  
**Cache:** ElastiCache Redis, key prefix `bff:`, used only when product-page fragments exist. The identity slice does not read or write Redis.  
**Callers:** the customer storefront, and later Android and iOS  
**Calls:** `identity-service` in this build. Later: search, catalogue, inventory, cart, promotion, order, payment, fulfilment, support  
**Publishes:** nothing  
**Consumes:** nothing  
**Product rules:** [modules/01-identity.md](../modules/01-identity.md), [modules/18-applications.md](../modules/18-applications.md)  
**Identity contract:** [identity-service.md](./identity-service.md)

This is the build document for the shopper edge. Browsers never call identity-service. They call this BFF. This file is the public routes, the cookie, and the call this process makes to identity.

The first build is the identity slice: OTP login, the session cookie, profile, addresses, consent, logout, and account deletion. Catalogue, cart, and checkout are listed in section 14 so the later routes have a place. They are not part of this build. Those domain services do not exist yet.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as identity-service and the storefront |
| Runtime | Node.js 22 LTS | One process. This service has no worker |
| HTTP | Fastify 5 | Same server as identity. Public routes and the health routes are explicit |
| Validation | Zod | Reject a bad body before the identity call |
| SQL | none | No database. No migrations |
| Redis | `ioredis`, only for the later `bff:` cache | Login does not use it |
| Service JWT | HMAC-SHA256, `crypto` | The BFF signs. Identity verifies. No extra JWT library is required |
| Logs | `pino` JSON to stdout | Same redaction rules as identity for codes, tokens, and the cookie |
| Traces | OpenTelemetry SDK, W3C `traceparent` | The storefront trace continues into identity |
| Metrics | `prom-client` on `GET /metrics` | Identity latency, identity errors, cookie rejects |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as identity |
| Container port | 8080 | Service port 80 targets 8080 |

---

## 2. Processes

One container.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | HTTP. Minimum 3 replicas in production, on the `apps` node group. HorizontalPodAutoscaler max is 20 |

There is no outbox and no worker. This service does not publish events.

---

## 3. What this build includes

| In this build | Not in this build |
| --- | --- |
| OTP request and verify | Product, search, cart, checkout, payment |
| `bm_session` cookie | Guest cart cookie |
| Profile, addresses, consent | Seller or staff login |
| Session list, logout, account deletion | `GET /seller/me` and every admin route |
| Forward identity error `code` and `message` | A second copy of the OTP, session, or address rules |

Identity owns the phone rules, the send cap, the lock, consent, sessions, and deletion. This service does not reimplement them. If identity returns `429 otp_locked`, the BFF returns `429 otp_locked` with the same message.

---

## 4. How a browser request is trusted

Public host: `api.buyymart.com`, path prefix `/v1`. The storefront origin is `https://buyymart.com`. Stage and dev use the same path on their own hosts.

| Header from the browser | Rule |
| --- | --- |
| `Origin` | Must be on the allow-list or the response is `403 origin_rejected`. No `Access-Control-Allow-Origin: *` |
| `X-Request-Id` | Optional. If missing or not a UUID, the BFF generates one. It is forwarded to identity and echoed on the response |
| `traceparent` | Forwarded when present. Otherwise the BFF starts a trace |
| `Cookie` | `bm_session` on authenticated routes. The raw token is never logged and never put in a response body |

CORS, on the allow-listed origins only:

| Response header | Value |
| --- | --- |
| `Access-Control-Allow-Origin` | The request origin, echoed, not `*` |
| `Access-Control-Allow-Credentials` | `true` |
| `Access-Control-Allow-Headers` | `content-type, x-request-id, traceparent` |
| `Access-Control-Allow-Methods` | The methods that route actually allows |
| `Vary` | `Origin` |

The storefront calls `fetch` with `credentials: "include"`.

Authenticated routes read the cookie, then call identity `POST /v1/sessions/introspect` with that token. The BFF caches nothing. A Redis flush in identity logs everyone out on the next call. Identity’s target for that call is 20 ms at p99 on a Redis hit. The BFF’s client timeout for introspect is 200 ms. On timeout or a connection error the BFF returns `503 dependency_unavailable` and does not clear the cookie.

If introspect returns `401 session_invalid` or `401 session_expired`, the BFF clears `bm_session` and returns that same status and code. The family must be `customer`. A seller or staff token presented to this BFF is `401 session_invalid` and the cookie is cleared. This BFF never forwards a non-customer session to a customer route.

---

## 5. Service JWT

Every call to identity carries:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `identity-service`. `exp - iat` is at most 60 seconds. Algorithm HS256. Signed with `SERVICE_JWT_KEY`, the same secret identity uses to verify |
| `X-Request-Id` | The id from section 4 |
| `traceparent` | The trace from section 4 |
| `X-Session-Token` | The raw `bm_session` value, only on routes that act as the customer. Never on OTP request |

The browser never sees this JWT. A missing or rejected service JWT is identity’s `401 service_unauthorized`. The BFF maps that to `503 dependency_unavailable` with message “A required dependency is unavailable.” It does not tell the shopper that an internal key was wrong.

The JWT is minted per identity call. It is not reused across requests.

---

## 6. OTP and the cookie

```text
Storefront
  → POST /v1/auth/otp/request
       → POST /v1/customer/otp/request    identity-service
  ← 202 { "accepted": true }

Storefront
  → POST /v1/auth/otp/verify
       → POST /v1/customer/otp/verify
       ← { sessionToken, customer }
  → Set-Cookie bm_session
  ← 200 { customer }          the sessionToken is not in this body
```

### Request bodies

The BFF forwards these bodies to identity without adding fields. Phone shape, the 6-digit code, consent purposes, and `policyVersion` are identity’s rules. See identity-service section 5 and section 10.

OTP request:

```json
{ "phone": "9876543210", "channel": "web" }
```

OTP verify:

```json
{
  "phone": "9876543210",
  "code": "123456",
  "channel": "web",
  "policyVersion": "2026-09-28",
  "consent": [
    { "purpose": "account", "granted": true },
    { "purpose": "order_sms", "granted": true },
    { "purpose": "marketing", "granted": false }
  ]
}
```

`channel` for the storefront is `web`. Android and iOS, when they exist, send `android` or `ios`. The BFF does not rewrite it.

The BFF does not retry verify. The code is single use. A retry of a successful verify is `401 otp_invalid` from identity, and the BFF returns that.

Client timeout for OTP request and verify is 5 seconds. Identity’s notification call can take about 4 seconds when it retries once.

### Cookie

Set only on a `200` from verify.

| Attribute | Value |
| --- | --- |
| Name | `bm_session` |
| Value | The raw `sessionToken` from identity |
| HttpOnly | yes |
| Secure | yes |
| SameSite | `Lax` |
| Path | `/` |
| Domain | omitted. Host-only on the API host, so `seller.buyymart.com` and `admin.buyymart.com` do not receive it |
| Max-Age | 7776000 seconds (90 days), the customer absolute cap |

The cookie can outlive the login. Identity’s Redis TTL slides 30 days and stops at the 90-day absolute cap. When the token is dead, introspect fails and this BFF clears the cookie.

Clearing the cookie is `Set-Cookie` with the same name, `Max-Age=0`, and an empty value. Same `Path`, `HttpOnly`, `Secure`, and `SameSite`.

Logout is `POST /v1/auth/logout`. The BFF reads the cookie, calls identity `POST /v1/sessions/revoke` for that token, then clears the cookie. Revoke failure still clears the cookie. The response is `204`.

### What the browser receives on verify

```json
{
  "customer": {
    "id": "…",
    "name": null,
    "email": null,
    "phone": "+919876543210",
    "status": "active"
  }
}
```

No `sessionToken`. No `Set-Cookie` on the error responses `otp_invalid`, `otp_locked`, `phone_banned`, `phone_cooldown`, or `consent_required`.

---

## 7. Authenticated customer routes

These routes require a customer session. The BFF introspects, then calls identity with `X-Session-Token`. Response bodies are identity’s bodies. The BFF does not rename fields.

| Browser | Identity | Notes |
| --- | --- | --- |
| `GET /v1/me` | `GET /v1/customer/me` | Profile |
| `PATCH /v1/me` | `PATCH /v1/customer/me` | `name` and `email` only |
| `GET /v1/me/addresses` | `GET /v1/customer/addresses` | |
| `POST /v1/me/addresses` | `POST /v1/customer/addresses` | |
| `PATCH /v1/me/addresses/:id` | `PATCH /v1/customer/addresses/:id` | |
| `DELETE /v1/me/addresses/:id` | `DELETE /v1/customer/addresses/:id` | No JSON body |
| `POST /v1/me/consent` | `POST /v1/customer/consent` | |
| `GET /v1/me/sessions` | `GET /v1/sessions` | |
| `DELETE /v1/me` | `DELETE /v1/customer/me` | Then clear the cookie on `202` |

Client timeout for these calls is 1 second.

Account deletion: the storefront must show, before this call, that paid and later orders will finish. This BFF does not call order-service. The body is `{ "confirm": true }`. Without that, identity returns `400 confirm_required` and the cookie stays. On `202` the BFF clears `bm_session`.

`POST /v1/me/consent` with `purpose: "account"` and `granted: false` is identity’s `409 use_delete_endpoint`. The BFF returns that. It does not delete the account on that call.

Address limits, pin rules, and the single default address are identity’s rules. The BFF forwards `400`, `404`, and `409` as identity sent them.

---

## 8. HTTP API

Public base path: `/v1`. Health and metrics are on port 8080 and are not routed by the public `/v1/*` ingress rule. The ALB health check calls `/health/ready` on the pod directly.

| Method | Path | Cookie | Success |
| --- | --- | --- | --- |
| POST | `/v1/auth/otp/request` | No | `202 { "accepted": true }` |
| POST | `/v1/auth/otp/verify` | Set on success | `200 { customer }` |
| POST | `/v1/auth/logout` | Cleared | `204` |
| GET | `/v1/me` | Required | `200` customer |
| PATCH | `/v1/me` | Required | `200` customer |
| GET | `/v1/me/addresses` | Required | `200 { addresses }` |
| POST | `/v1/me/addresses` | Required | `201` address |
| PATCH | `/v1/me/addresses/:id` | Required | `200` address |
| DELETE | `/v1/me/addresses/:id` | Required | `204` |
| POST | `/v1/me/consent` | Required | `201` |
| GET | `/v1/me/sessions` | Required | `200 { sessions }` |
| DELETE | `/v1/me` | Cleared on `202` | `202 { "status": "deleted" }` |
| GET | `/health/live` | No | `200` if the process is up |
| GET | `/health/ready` | No | `200` only when `SERVICE_JWT_KEY` and `IDENTITY_URL` are non-empty |
| GET | `/metrics` | No | Prometheus text. Cluster scrape only |

`/health/ready` does not call identity. A blip in identity must not take every BFF pod out of the load balancer. A missing key or URL means this pod cannot do its job, so it stays unready.

JSON errors, same shape as identity:

```json
{ "code": "otp_invalid", "message": "That code is wrong or expired.", "requestId": "…" }
```

`message` is safe to show. `code` is what the storefront branches on. The BFF does not rewrite identity’s `message`.

| Identity status | BFF status | BFF `code` |
| --- | --- | --- |
| The identity `code` below | Same status | Same `code` |
| `401 service_unauthorized` | `503` | `dependency_unavailable` |
| Timeout or connection reset | `503` | `dependency_unavailable` |
| Identity `5xx` | `503` | `dependency_unavailable` |

Codes passed through unchanged: `phone_invalid`, `otp_invalid`, `otp_locked`, `otp_delivery_failed`, `phone_banned`, `phone_cooldown`, `consent_required`, `confirm_required`, `use_delete_endpoint`, `session_invalid`, `session_expired`, `invalid_body`, `not_found`, and the address codes identity already returns (`pin_invalid`, `address_limit`, `default_required`).

Logs redact `code` when it is an OTP, `authorization`, `cookie`, `bm_session`, `sessionToken`, and `x-session-token`. The phone in an OTP request log is the last two digits only.

---

## 9. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `SERVICE_JWT_KEY` | Same HMAC key as identity. The BFF signs, identity verifies |
| Config | `IDENTITY_URL` | `http://identity-service.identity.svc.cluster.local` |
| Config | `CUSTOMER_WEB_ORIGINS` | Comma-separated allow-list. Prod is `https://buyymart.com` |
| Config | `PORT` | `8080` |

No `DATABASE_URL`. No `REDIS_URL` in this build.

Secrets live in Secrets Manager at `buyymart/{env}/customer-bff` and arrive as a Kubernetes Secret through External Secrets. They are not in the image and not in git. Stage and prod do not log OTP codes. Dev still receives the code only in the notification stub log, which is identity’s dependency, not this service’s.

---

## 10. Local run

Docker Compose for this slice is the identity-service stack (Postgres 16, Redis 7, identity API, notification stub) plus this API.

| Variable | Local value |
| --- | --- |
| `IDENTITY_URL` | `http://identity:8080` inside Compose, or `http://127.0.0.1:8080` when identity is already running on the host |
| `SERVICE_JWT_KEY` | The same value identity was started with |
| `CUSTOMER_WEB_ORIGINS` | `http://localhost:5173` |
| `PORT` | `8082`, so it does not collide with identity on 8080 |

The storefront is not part of this Compose file. A browser or `curl` against port 8082 is enough to prove the cookie and the identity call.

`curl` without a browser must send the `Cookie` header it stored from verify. The `Set-Cookie` on verify is the session. The JSON body is not.

---

## 11. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| OTP request and verify with a valid phone | `202`, then `200` and `Set-Cookie: bm_session`. The JSON body has `customer` and no `sessionToken` |
| Same phone again | Same customer id |
| Wrong code | `401 otp_invalid`, no cookie |
| Five wrong codes, then the right code | `429 otp_locked` |
| Banned phone | `403 phone_banned`, message does not say banned, no cookie |
| Missing `account` consent | `400 consent_required`, no cookie |
| `GET /v1/me` with the cookie | `200` profile. Identity was called with a service JWT and `X-Session-Token` |
| `GET /v1/me` with no cookie | `401 session_invalid` |
| Seller or staff token in `bm_session` | `401 session_invalid`, cookie cleared |
| Logout | `204`, cookie cleared, introspect of the old token is `401` |
| Delete without `confirm: true` | `400 confirm_required`, cookie remains |
| Delete with `confirm: true` | `202`, cookie cleared |
| Identity down | `503 dependency_unavailable` |
| Origin not on the allow-list | `403 origin_rejected` |
| `/health/ready` with an empty `SERVICE_JWT_KEY` | `503` |
| `/metrics` | Not served on the public `/v1` path |

---

## 12. Outside this service

| Concern | Owner |
| --- | --- |
| OTP, session row, profile, addresses, consent, deletion | `identity-service` |
| SMS | `notification-service`, called by identity |
| Cookie copy and the login screen | The storefront |
| Edge rate limit in front of OTP | WAF, in addition to identity’s Redis lock |
| Seller Centre and Admin Console | `seller-bff` and `admin-bff` |

---

## 13. Later routes, not this build

These stay in the product docs. Do not add them until the domain service they call is deployed. Timeouts already fixed:

| Caller | Callee | Timeout | If it fails |
| --- | --- | --- | --- |
| customer-bff | search-service | 400 ms | Fall back to a catalogue keyword query |
| customer-bff | inventory-service | 300 ms | Show “stock unavailable”, block add-to-cart |
| customer-bff | payment-service | 3 s | Show a payment error. The order stays `payment_pending` |

When those routes exist:

- Product-page fragments may be cached in Redis under `bff:` for 30–60 seconds. `/health/ready` then also requires Redis `PING`.
- Checkout runs in this BFF: read the cart, price the coupon, ask fulfilment for the pin-code fee, create the order, then open a payment session for prepaid. The browser return URL only displays status. It does not mark an order paid. The steps are [modules/08-checkout.md](../modules/08-checkout.md).
- A guest cart merges on OTP success. The guest id is a cookie owned by that later cart work, not by `bm_session`.
- Payment webhooks and courier webhooks never enter this BFF.

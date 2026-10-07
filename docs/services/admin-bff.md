# admin-bff — implementation spec

**Namespace:** `edge`  
**Database:** none  
**Cache:** none in this build  
**Callers:** Admin Console at `https://admin.buyymart.com`  
**Calls:** `identity-service`, `seller-service`, `catalogue-service`, `order-service`, `payment-service`, `search-service`  
**Publishes:** nothing  
**Consumes:** nothing  
**Product rules:** [modules/01-identity.md](../modules/01-identity.md), [modules/13-sellers.md](../modules/13-sellers.md), [modules/18-applications.md](../modules/18-applications.md)  
**Domain contracts:** [identity-service.md](./identity-service.md), [seller-service.md](./seller-service.md), [catalogue-service.md](./catalogue-service.md), [order-service.md](./order-service.md), [payment-service.md](./payment-service.md), [search-service.md](./search-service.md)

This is the build document for the Admin Console edge. Browsers never call a domain service. They call this BFF. This file is the public routes, the cookie, the staff role gate, and the calls this process makes.

A seller login cannot open these routes. A customer login cannot open them either. Staff actions are audited in the domain service that owns the change. This process does not write that audit row itself.

The payment webhook stays on `payment-service`. This BFF does not register it.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as the other BFFs |
| Runtime | Node.js 22 LTS | One process. This service has no worker |
| HTTP | Fastify 5 | Public routes and the health routes are explicit |
| Validation | Zod | Reject a bad body before any domain call |
| SQL | none | No database. No migrations |
| Redis | none | Login does not cache the session |
| Service JWT | HMAC-SHA256, `crypto` | The BFF signs. Each domain service verifies its own audience |
| Logs | `pino` JSON to stdout | Redact passwords, TOTP, the cookie, PAN, bank fields, and presigned URLs |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One Admin Console action continues into identity and the domain service |
| Metrics | `prom-client` on `GET /metrics` | Upstream latency, upstream errors, cookie rejects |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as seller-bff |
| Container port | 8080 | Service port 80 targets 8080 |

---

## 2. Processes

One container.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | HTTP. Minimum 3 replicas in production, on the `apps` node group. HorizontalPodAutoscaler max is 20 |

There is no outbox and no worker. This service does not publish events.

`GET /health/live` is the process. `GET /health/ready` is `200` only when `SERVICE_JWT_KEY`, `IDENTITY_URL`, `SELLER_URL`, `CATALOGUE_URL`, `ORDER_URL`, `PAYMENT_URL`, `SEARCH_URL`, and `ADMIN_WEB_ORIGINS` are non-empty. It does not call those services. A blip in one of them must not take every BFF pod out of the load balancer.

---

## 3. What this build includes

| In this build | Not in this build |
| --- | --- |
| Staff login, TOTP enrol, TOTP verify, logout, password reset | Seller or customer login |
| Create a staff user and change a role | A staff profile route. Identity does not expose one |
| Seller review, suspension, close, KYC link, bank decision, payout read | The seller’s own profile edit |
| Catalogue approve, reject, hide, and category edits | Seller catalogue writes |
| Order read, staff cancel, return override, COD refund reference | Confirm and pack. Those stay on `seller-bff` |
| Payment read and a staff refund | The payment webhook |
| Search inspect for one variant | A home queue. No domain service has that list |
| Ban a customer by id | Customer search. Identity has no lookup-by-phone for staff |

Identity owns the password, TOTP, and the staff role. Seller-service owns KYC status and the bank ciphertext. Catalogue owns publish. Order-service owns cancel and the return override. Payment-service owns the gateway refund. Search-service owns the index row. This BFF does not reimplement them.

---

## 4. How a browser request is trusted

Public host: `api.buyymart.com`, path prefix `/admin`. Admin Console’s origin is `https://admin.buyymart.com`. Stage and dev use the same path on their own hosts.

| Header from the browser | Rule |
| --- | --- |
| `Origin` | Must be on `ADMIN_WEB_ORIGINS` or the response is `403 origin_rejected`, message `That origin is not allowed.` No `Access-Control-Allow-Origin: *` |
| `X-Request-Id` | Optional. If missing or not a UUID, the BFF generates one. It is forwarded and echoed |
| `traceparent` | Forwarded when present. Otherwise the BFF starts a trace |
| `Idempotency-Key` | Required on the staff refund. A missing or non-UUID key is `400 invalid_body` and payment-service is not called |
| `Cookie` | `bm_staff` on authenticated routes. The raw token is never logged and never put in a response body |

CORS, on the allow-listed origins only:

| Response header | Value |
| --- | --- |
| `Access-Control-Allow-Origin` | The request origin, echoed, not `*` |
| `Access-Control-Allow-Credentials` | `true` |
| `Access-Control-Allow-Headers` | `content-type, x-request-id, traceparent, idempotency-key` |
| `Access-Control-Allow-Methods` | The methods that route actually allows |
| `Vary` | `Origin` |

Admin Console calls `fetch` with `credentials: "include"`.

Authenticated routes read `bm_staff`, then call identity `POST /v1/sessions/introspect`. The BFF caches nothing. The client timeout is 200 ms. On timeout or a connection error the BFF returns `503 dependency_unavailable` and does not clear the cookie.

The family must be `staff`. A customer or seller token is `401 session_invalid`, message `That session is not valid.`, and the cookie is cleared. A missing cookie on an authenticated route is the same `401` and does not call identity. `session_expired` from identity is forwarded and the cookie is cleared.

A body field `role` or `family` is ignored. The role comes from introspect. The seller id, order id, and customer id come from the path. A body `sellerId` that disagrees with the path is ignored.

---

## 5. Service JWT

Every domain call carries:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. `exp - iat` is at most 60 seconds. Algorithm HS256. Signed with `SERVICE_JWT_KEY` |
| `X-Request-Id` | The id from section 4 |
| `traceparent` | The trace from section 4 |
| `X-Session-Token` | The raw `bm_staff` value, on routes that act as the staff user. Never on login, MFA, forgot, or reset |
| `Idempotency-Key` | The browser’s key, only on the staff refund |

Audience is the service being called:

| Call | Audience |
| --- | --- |
| identity-service | `identity-service` |
| seller-service | `seller-service` |
| catalogue-service | `catalogue-service` |
| order-service | `order-service` |
| payment-service | `payment-service` |
| search-service | `search-service` |

The browser never sees this JWT. A domain `401 service_unauthorized`, a domain `5xx`, a timeout, or a connection reset becomes `503 dependency_unavailable`, message `A required dependency is unavailable.` The JWT is minted per call. It is not reused across requests.

---

## 6. Login, MFA, and the cookie

```text
Admin Console
  → POST /admin/auth/login
       → POST /v1/staff/login
  ← 200 { mfaToken }     or { mfaEnrollmentToken, otpauthUrl }
        no cookie

Admin Console
  → POST /admin/auth/mfa/verify     or /admin/auth/mfa/confirm
       → POST /v1/staff/mfa/verify  or /v1/staff/mfa/confirm
       ← { sessionToken, … }
  → Set-Cookie bm_staff
  ← 200 without sessionToken
```

Login does not set a cookie. An enrol token and an MFA token are not sessions. `otpauthUrl` is returned to the browser so it can show the QR. It is not logged. The TOTP code is not logged.

Forgot is `{ "email" }`. Reset is `{ "token", "password" }`. Both call `/v1/staff/password/forgot` and `/v1/staff/password/reset`. The link host is identity’s `https://admin.buyymart.com`. This BFF does not send the email.

Client timeout for login, MFA, forgot, and reset is 2 seconds. The BFF does not retry login. Unknown email and a wrong password are both identity’s `401 credentials_invalid`. A wrong TOTP is identity’s error, forwarded. At five wrong codes identity deletes the MFA token. This BFF does not count those attempts.

### Cookie

Set only when identity’s success body contains `sessionToken`. That is MFA confirm and MFA verify, not login.

| Attribute | Value |
| --- | --- |
| Name | `bm_staff` |
| Value | The raw `sessionToken` from identity |
| HttpOnly | yes |
| Secure | yes |
| SameSite | `Lax` |
| Path | `/admin` |
| Domain | omitted. Host-only on the API host |
| Max-Age | 28800 seconds (8 hours), the staff absolute cap |

`Path=/admin` keeps this cookie off `/v1` and `/seller`. This BFF never reads `bm_session` or `bm_seller`.

Clearing the cookie is `Set-Cookie` with the same name, `Path=/admin`, `Max-Age=0`, and an empty value. Same `HttpOnly`, `Secure`, and `SameSite`.

Logout is `POST /admin/auth/logout`. The BFF reads the cookie, calls identity `POST /v1/sessions/revoke`, then clears the cookie. Revoke failure still clears the cookie. The response is `204`. No cookie is `401 session_invalid`.

`GET /admin/me` returns `{ subjectId, role, sessionId }` from introspect. It does not call another identity route.

---

## 7. Role gate

Introspect returns `role` of `support`, `catalogue`, `trust`, `finance`, `admin`, or `super_admin`. The BFF checks the role before the domain call. The domain service checks it again. A wrong role is `403 forbidden`, message `Forbidden.`, and the domain service is not called.

| Route group | Roles |
| --- | --- |
| `GET /admin/me`, logout | Any staff role |
| Create staff, change a role | `admin`, `super_admin` |
| Ban a customer | `trust`, `admin`, `super_admin` |
| Seller review, KYC link, suspend, reinstate, close, auto-approve | `trust`, `admin`, `super_admin` |
| Bank decision, payout read | `finance`, `admin`, `super_admin` |
| Catalogue category and product moderation | `catalogue`, `trust` |
| Search inspect | `catalogue`, `trust` |
| Order read | Any staff role |
| Order cancel, return override | `support`, `trust`, `admin`, `super_admin` |
| COD refund reference | `finance`, `admin`, `super_admin` |
| Payment read | Any staff role |
| Staff refund | `finance`, `admin`, `super_admin` |

`admin` and `super_admin` are not catalogue reviewers in version 1. Catalogue-service allows `catalogue` or `trust` only. This BFF uses that same pair so those two roles are not sent a call catalogue will reject.

A person cannot change their own role. Identity enforces that, and the BFF forwards `403`. Granting `super_admin` is also identity’s rule. The reason on a role change is at least 10 characters. A shorter reason is `400 invalid_body` and identity is not called.

---

## 8. Sellers

These routes require the review roles in section 7, except the bank and payout routes, which require finance. The seller id is the path id.

| Browser | seller-service |
| --- | --- |
| `GET /admin/sellers/:id` | `GET /v1/sellers/:id` |
| `POST /admin/sellers/:id/approve` | `POST /v1/sellers/:id/approve` |
| `POST /admin/sellers/:id/reject` | `POST /v1/sellers/:id/reject` |
| `POST /admin/sellers/:id/suspend` | `POST /v1/sellers/:id/suspend` |
| `POST /admin/sellers/:id/reinstate` | `POST /v1/sellers/:id/reinstate` |
| `POST /admin/sellers/:id/close` | `POST /v1/sellers/:id/close` |
| `PATCH /admin/sellers/:id/auto-approve` | `PATCH /v1/sellers/:id/auto-approve` |
| `POST /admin/sellers/:id/kyc/:document/link` | `POST /v1/sellers/:id/kyc/:document/link` |
| `POST /admin/sellers/:id/bank/approve` | `POST /v1/sellers/:id/bank/approve` |
| `POST /admin/sellers/:id/bank/reject` | `POST /v1/sellers/:id/bank/reject` |
| `GET /admin/sellers/:id/payout` | `GET /v1/sellers/:id/payout` |

Client timeout is 1 second. Notes and reasons shorter than 10 characters are `400 invalid_body`. Reject `fields` must be a non-empty list from the seller-service set. The BFF forwards `409 not_reviewable`, `409 bank_pending`, and `404 not_found` as sent.

The KYC link response is `{ url, expiresAt }`. The URL lasts 60 seconds. It is not logged. This BFF does not call media-service. Trust does not get a permanent KYC URL.

The payout body contains PAN and the live bank account. It is not logged. A catalogue or trust session on the payout route is `403 forbidden` and seller-service is not called. `POST /admin/sellers/:id/bank` is not a route here. The seller requests that change on `seller-bff`. This path answers `404`.

---

## 9. Catalogue and search

Catalogue moderation requires `catalogue` or `trust`.

| Browser | Domain |
| --- | --- |
| `POST /admin/categories` | catalogue `POST /v1/staff/categories` |
| `PATCH /admin/categories/:id` | catalogue `PATCH /v1/staff/categories/:id` |
| `POST /admin/products/:id/approve` | catalogue `POST /v1/staff/products/:id/approve` |
| `POST /admin/products/:id/reject` | catalogue `POST /v1/staff/products/:id/reject` |
| `POST /admin/products/:id/hide` | catalogue `POST /v1/staff/products/:id/hide` |
| `GET /admin/variants/:id` | search `GET /v1/staff/variants/:id` |

Client timeout is 1 second. Approve, reject, and hide require a reason of at least 10 characters. A seller session never reaches these routes. A finance session is `403 forbidden` and catalogue-service is not called.

Search inspect returns the index row, including `inIndex: false` for a variant that was never indexed. Staff cannot edit the index from this BFF. There is no write route.

---

## 10. Orders and payments

Order read allows any staff role. Cancel and the return override allow `support`, `trust`, `admin`, or `super_admin`. The COD reference allows finance.

| Browser | order-service |
| --- | --- |
| `GET /admin/orders/:id` | `GET /v1/orders/:id` |
| `POST /admin/orders/:id/cancel` | `POST /v1/orders/:id/cancel` |
| `POST /admin/orders/:id/returns/:returnId/override` | `POST /v1/orders/:id/returns/:returnId/override` |
| `POST /admin/orders/:id/returns/:returnId/cod-refund` | `POST /v1/orders/:id/returns/:returnId/cod-refund` |

Cancel reason is `staff`. A missing reason is `400 invalid_body`. The BFF does not confirm or pack. Those paths answer `404`.

Client timeout is 1 second for the read and 2 seconds for cancel, override, and the COD reference.

| Browser | payment-service |
| --- | --- |
| `GET /admin/payments/:orderId` | `GET /v1/payments/:orderId` |
| `POST /admin/payments/:orderId/refunds` | `POST /v1/payments/:orderId/refunds` |

The refund body is `{ amountPaise, variantId, reason, confirmedBy }`. `reason` for a staff refund is `staff`. `variantId` is a UUID or null. `confirmedBy` is the second staff id, or null. This BFF does not fill `confirmedBy` from the session. Payment-service returns `403 forbidden` when the confirmer is the caller, and `409 confirm_required` when the amount is above the maker-checker threshold and `confirmedBy` is missing. Both are forwarded. The success is `202`. The order is not marked refunded here.

The refund timeout is 3 seconds because payment-service calls the gateway. The read timeout is 1 second. The browser’s `Idempotency-Key` is forwarded. A second click with the same key is payment-service’s `409 idempotency_conflict` or the original `202`. This BFF does not store the key.

`POST /webhooks/payments` is not registered. A request to it on this process is `404`.

---

## 11. Staff accounts and bans

| Browser | identity-service | Who |
| --- | --- | --- |
| `POST /admin/staff` | `POST /v1/staff` | `admin` or `super_admin` |
| `PATCH /admin/staff/:id` | `PATCH /v1/staff/:id` | `admin` or `super_admin` |
| `POST /admin/customers/:id/ban` | `POST /v1/customer/:id/ban` | `trust`, `admin`, or `super_admin` |

The create body is identity’s body: email and role. Email must be on the staff domain. Identity rejects anything else. The temporary password is not logged.

The role-change body includes `role` and `reason`. Reason is at least 10 characters. The caller’s own id is still sent to identity, which returns `403`. This BFF does not special-case it into a different code.

Ban requires `{ reason }` of at least 10 characters. Identity sets the customer banned, revokes their sessions, and writes `auth_audit`. This BFF does not look the customer up. There is no `GET /admin/customers`.

Client timeout for these calls is 1 second.

---

## 12. HTTP API

Public base path: `/admin`. Health and metrics are on port 8080 and are not routed by the public `/admin/*` ingress rule. The ALB health check calls `/health/ready` on the pod directly.

| Method | Path | Cookie | Success |
| --- | --- | --- | --- |
| POST | `/admin/auth/login` | No | `200` MFA token or enrol token. No `sessionToken` |
| POST | `/admin/auth/mfa/confirm` | Set on success | `200`, no `sessionToken` |
| POST | `/admin/auth/mfa/verify` | Set on success | `200`, no `sessionToken` |
| POST | `/admin/auth/password/forgot` | No | `202` |
| POST | `/admin/auth/password/reset` | No | `204` |
| POST | `/admin/auth/logout` | Cleared | `204` |
| GET | `/admin/me` | Any staff | `200` `{ subjectId, role, sessionId }` |
| POST | `/admin/staff` | Admin | `201` |
| PATCH | `/admin/staff/:id` | Admin | `200` |
| POST | `/admin/customers/:id/ban` | Trust | `200` |
| GET | `/admin/sellers/:id` | Trust | `200` |
| POST | `/admin/sellers/:id/approve` | Trust | `200` |
| POST | `/admin/sellers/:id/reject` | Trust | `200` |
| POST | `/admin/sellers/:id/suspend` | Trust | `200` |
| POST | `/admin/sellers/:id/reinstate` | Trust | `200` |
| POST | `/admin/sellers/:id/close` | Trust | `200` |
| PATCH | `/admin/sellers/:id/auto-approve` | Trust | `200` |
| POST | `/admin/sellers/:id/kyc/:document/link` | Trust | `200` `{ url, expiresAt }` |
| POST | `/admin/sellers/:id/bank/approve` | Finance | `200` |
| POST | `/admin/sellers/:id/bank/reject` | Finance | `200` |
| GET | `/admin/sellers/:id/payout` | Finance | `200` decrypted live account |
| POST | `/admin/categories` | Catalogue or trust | `201` |
| PATCH | `/admin/categories/:id` | Catalogue or trust | `200` |
| POST | `/admin/products/:id/approve` | Catalogue or trust | `200` |
| POST | `/admin/products/:id/reject` | Catalogue or trust | `200` |
| POST | `/admin/products/:id/hide` | Catalogue or trust | `200` |
| GET | `/admin/variants/:id` | Catalogue or trust | `200` index state |
| GET | `/admin/orders/:id` | Any staff | `200` |
| POST | `/admin/orders/:id/cancel` | Support or trust | `200` |
| POST | `/admin/orders/:id/returns/:returnId/override` | Support or trust | `200` |
| POST | `/admin/orders/:id/returns/:returnId/cod-refund` | Finance | `200` |
| GET | `/admin/payments/:orderId` | Any staff | `200` |
| POST | `/admin/payments/:orderId/refunds` | Finance | `202` |
| GET | `/health/live` | No | `200` if the process is up |
| GET | `/health/ready` | No | `200` only when the config in section 2 is set |
| GET | `/metrics` | No | Prometheus text. Cluster scrape only |

Trust in that table means `trust`, `admin`, or `super_admin`. Finance means `finance`, `admin`, or `super_admin`. Admin means `admin` or `super_admin`. Catalogue or trust does not include `admin`.

JSON errors:

```json
{ "code": "forbidden", "message": "Forbidden.", "requestId": "…" }
```

`message` is safe to show. `code` is what Admin Console branches on. The BFF does not rewrite a domain `message`.

| Upstream | BFF |
| --- | --- |
| A domain 4xx with `code` and `message` | Same status, same `code`, same `message` |
| `401 service_unauthorized` | `503 dependency_unavailable` |
| Timeout, connection reset, or domain 5xx | `503 dependency_unavailable` |

Codes this BFF generates itself: `origin_rejected`, `session_invalid`, `forbidden`, `invalid_body`, `dependency_unavailable`.

Logs redact `authorization`, `cookie`, `bm_staff`, `sessionToken`, `x-session-token`, `password`, `token`, `passwordChangeToken`, `mfaToken`, `mfaEnrollmentToken`, `otpauthUrl`, `code`, `pan`, `bankAccount`, `ifsc`, `accountName`, and `url`.

---

## 13. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `SERVICE_JWT_KEY` | Same HMAC key the domain services use to verify |
| Config | `IDENTITY_URL` | `http://identity-service.identity.svc.cluster.local` |
| Config | `SELLER_URL` | `http://seller-service.sellers.svc.cluster.local` |
| Config | `CATALOGUE_URL` | `http://catalogue-service.catalogue.svc.cluster.local` |
| Config | `ORDER_URL` | `http://order-service.commerce.svc.cluster.local` |
| Config | `PAYMENT_URL` | `http://payment-service.money.svc.cluster.local` |
| Config | `SEARCH_URL` | `http://search-service.catalogue.svc.cluster.local` |
| Config | `ADMIN_WEB_ORIGINS` | Comma-separated. Prod is `https://admin.buyymart.com` |
| Config | `PORT` | `8080` |

No `DATABASE_URL`. No `REDIS_URL`. No `DATA_KEY`. No gateway secret. This process does not verify a webhook and does not decrypt PAN except by returning seller-service’s payout body to finance.

Secrets live in Secrets Manager at `buyymart/{env}/admin-bff` and arrive as a Kubernetes Secret through External Secrets. They are not in the image and not in git.

---

## 14. Local run

This Compose file is the API only. There is no Postgres and no Redis. Identity, seller, catalogue, order, payment, and search must already be running when a test follows a real call. Unit tests inject those clients and do not start Compose.

| Variable | Local value |
| --- | --- |
| `IDENTITY_URL` | `http://host.docker.internal:8080` |
| `SELLER_URL` | `http://host.docker.internal:8097` |
| `CATALOGUE_URL` | `http://host.docker.internal:8083` |
| `ORDER_URL` | `http://host.docker.internal:8093` |
| `PAYMENT_URL` | `http://host.docker.internal:8095` |
| `SEARCH_URL` | `http://host.docker.internal:8087` |
| `SERVICE_JWT_KEY` | The same value those services were started with |
| `ADMIN_WEB_ORIGINS` | `http://localhost:5175` |
| `PORT` | `8100`, so it does not collide with seller-bff on 8099 |

The container still listens on 8080. Compose publishes `8100:8080`.

Admin Console is not part of this Compose file. A browser or `curl` against port 8100 is enough to prove the cookie and one domain call. `curl` must send the `Cookie` header stored from MFA verify. The JSON body is not the session.

---

## 15. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Login with a password | `200` with `mfaToken` or `mfaEnrollmentToken`. No `Set-Cookie` |
| MFA verify | `200`, `Set-Cookie: bm_staff`, `Path=/admin`, `Max-Age=28800`. The JSON body has no `sessionToken` |
| Wrong password | `401 credentials_invalid`, no cookie |
| Logout | `204`, cookie cleared |
| Seller or customer token in `bm_staff` | `401 session_invalid`, cookie cleared |
| `GET /admin/sellers/:id` with no cookie | `401 session_invalid`. seller-service is not called |
| `finance` calls approve | `403 forbidden`. seller-service is not called |
| `trust` calls `GET /admin/sellers/:id/payout` | `403 forbidden`. The body has no account number |
| `trust` approves a seller | `200`. seller-service was called with the service JWT and `X-Session-Token` |
| KYC link | `200` `{ url, expiresAt }`. The log line has no URL |
| `finance` approves a bank change | `200`. `POST /admin/sellers/:id/bank` is `404` |
| `finance` hides a product | `403 forbidden`. catalogue-service is not called |
| `catalogue` approves a product | `200`. A reason shorter than 10 characters is `400 invalid_body` |
| Search inspect | `200` including `inIndex: false` |
| Staff cancel | order-service is called with reason `staff` |
| `POST /admin/orders/:id/confirm` | `404` |
| Refund without `Idempotency-Key` | `400 invalid_body`. payment-service is not called |
| Refund above the maker-checker amount with no `confirmedBy` | `409 confirm_required` |
| `confirmedBy` equal to the caller | `403 forbidden` from payment-service, forwarded |
| `POST /webhooks/payments` | `404` |
| Identity down on login | `503 dependency_unavailable`, no cookie |
| Origin not on the allow-list | `403 origin_rejected` |
| `/health/ready` with an empty `PAYMENT_URL` | `503` |
| Logs for a payout read | No PAN, bank account, password, cookie, TOTP, or presigned URL |

---

## 16. Outside this service

| Concern | Owner |
| --- | --- |
| Password, TOTP, staff role, customer ban | `identity-service` |
| KYC status, suspension, bank ciphertext, payout decrypt | `seller-service` |
| Product approve, reject, hide | `catalogue-service` |
| Cancel, return override, COD reference | `order-service` |
| Gateway refund and the webhook | `payment-service` |
| Whether a variant is in the index | `search-service` |
| Seller Centre | `seller-bff` |
| Admin Console screens | The Admin Console web app |

---

## 17. Later routes, not this build

Do not add these until the domain service they call exists.

| Route | Callee | Why it waits |
| --- | --- | --- |
| Shipment and label read | `fulfilment-service` | Not deployed |
| Settlement exceptions | `settlement-service` | Not deployed |
| Coupons | `promotion-service` | Not deployed |
| Tickets near 48 hours | `support-service` | Not deployed |
| Home queue | Several of the above | No service has the combined list. Do not scan their databases from this BFF |
| Customer search | `identity-service` | Staff can ban by id. There is no phone lookup |

When the settlement read exists, the timeout is 1 second and the body never includes a decrypted bank account. Finance still reads that account from the payout route in section 8.

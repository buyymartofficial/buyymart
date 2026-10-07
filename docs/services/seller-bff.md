# seller-bff — implementation spec

**Namespace:** `edge`  
**Database:** none  
**Cache:** none in this build  
**Callers:** Seller Centre at `https://seller.buyymart.com`  
**Calls:** `identity-service`, `seller-service`, `media-service`, `catalogue-service`, `inventory-service`, `order-service`  
**Publishes:** nothing  
**Consumes:** nothing  
**Product rules:** [modules/01-identity.md](../modules/01-identity.md), [modules/13-sellers.md](../modules/13-sellers.md), [modules/18-applications.md](../modules/18-applications.md)  
**Domain contracts:** [identity-service.md](./identity-service.md), [seller-service.md](./seller-service.md), [media-service.md](./media-service.md), [catalogue-service.md](./catalogue-service.md), [inventory-service.md](./inventory-service.md), [order-service.md](./order-service.md)

This is the build document for the Seller Centre edge. Browsers never call a domain service. They call this BFF. This file is the public routes, the cookie, the role and status gate, and the calls this process makes.

A seller login cannot reach admin routes. Approve, reject, suspend, reinstate, close, the KYC review link, the payout read, and catalogue moderation stay on `admin-bff`. This process does not register those paths.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as the other BFFs and domain services |
| Runtime | Node.js 22 LTS | One process. This service has no worker |
| HTTP | Fastify 5 | Public routes and the health routes are explicit |
| Validation | Zod | Reject a bad body before any domain call |
| SQL | none | No database. No migrations |
| Redis | none | Login does not cache the session |
| Service JWT | HMAC-SHA256, `crypto` | The BFF signs. Each domain service verifies its own audience |
| Logs | `pino` JSON to stdout | Redact passwords, the cookie, PAN, bank fields, and presigned URLs |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One Seller Centre action continues into identity, seller, media, catalogue, inventory, and order |
| Metrics | `prom-client` on `GET /metrics` | Upstream latency, upstream errors, cookie rejects |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as customer-bff |
| Container port | 8080 | Service port 80 targets 8080 |

---

## 2. Processes

One container.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | HTTP. Minimum 3 replicas in production, on the `apps` node group. HorizontalPodAutoscaler max is 20 |

There is no outbox and no worker. This service does not publish events.

`GET /health/live` is the process. `GET /health/ready` is `200` only when `SERVICE_JWT_KEY`, `IDENTITY_URL`, `SELLER_URL`, `MEDIA_URL`, `CATALOGUE_URL`, `INVENTORY_URL`, `ORDER_URL`, and `SELLER_WEB_ORIGINS` are non-empty. It does not call those services. A blip in one of them must not take every BFF pod out of the load balancer.

---

## 3. What this build includes

| In this build | Not in this build |
| --- | --- |
| Seller register, login, logout, password reset | Staff login and MFA |
| Owner profile, agreement, GSTIN, KYC attach, submit | Trust approve, reject, suspend, close |
| Bank-change request after approval | Finance approve of that change, and the payout read |
| Product upload, catalogue draft, submit, on-hand stock | Courier labels, AWB, tracking |
| Confirm, pack, cancel, and return decisions for one order | A seller order index. `order-service` has no list-by-seller route |
| Forward the domain `code` and `message` | A second copy of GSTIN, stock, price, or invoice rules |

Identity owns the password, the session, and seller staff accounts. Seller-service owns the profile, KYC status, and the bank ciphertext. Media-service owns the bytes. Catalogue owns the product. Inventory owns `onHand`. Order-service owns confirm, pack, and the invoice number. This BFF does not reimplement them.

---

## 4. How a browser request is trusted

Public host: `api.buyymart.com`, path prefix `/seller`. Seller Centre’s origin is `https://seller.buyymart.com`. Stage and dev use the same path on their own hosts.

| Header from the browser | Rule |
| --- | --- |
| `Origin` | Must be on `SELLER_WEB_ORIGINS` or the response is `403 origin_rejected`, message `That origin is not allowed.` No `Access-Control-Allow-Origin: *` |
| `X-Request-Id` | Optional. If missing or not a UUID, the BFF generates one. It is forwarded and echoed |
| `traceparent` | Forwarded when present. Otherwise the BFF starts a trace |
| `Cookie` | `bm_seller` on authenticated routes. The raw token is never logged and never put in a response body |

CORS, on the allow-listed origins only:

| Response header | Value |
| --- | --- |
| `Access-Control-Allow-Origin` | The request origin, echoed, not `*` |
| `Access-Control-Allow-Credentials` | `true` |
| `Access-Control-Allow-Headers` | `content-type, x-request-id, traceparent` |
| `Access-Control-Allow-Methods` | The methods that route actually allows |
| `Vary` | `Origin` |

Seller Centre calls `fetch` with `credentials: "include"`.

Authenticated routes read `bm_seller`, then call identity `POST /v1/sessions/introspect`. The BFF caches nothing. The client timeout is 200 ms. On timeout or a connection error the BFF returns `503 dependency_unavailable` and does not clear the cookie.

The family must be `seller`, and `sellerId` must be a UUID. A customer or staff token is `401 session_invalid`, message `That session is not valid.`, and the cookie is cleared. A missing cookie on an authenticated route is the same `401` and does not call identity. `session_expired` from identity is forwarded and the cookie is cleared.

A body field `sellerId`, `role`, or `family` is ignored. The id and role come from introspect.

---

## 5. Service JWT

Every domain call carries:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. `exp - iat` is at most 60 seconds. Algorithm HS256. Signed with `SERVICE_JWT_KEY` |
| `X-Request-Id` | The id from section 4 |
| `traceparent` | The trace from section 4 |
| `X-Session-Token` | The raw `bm_seller` value, on routes that act as the seller. Never on register, login, forgot, or reset |

Audience is the service being called:

| Call | Audience |
| --- | --- |
| identity-service | `identity-service` |
| seller-service | `seller-service` |
| media-service | `media-service` |
| catalogue-service | `catalogue-service` |
| inventory-service | `inventory-service` |
| order-service | `order-service` |

The browser never sees this JWT. A domain `401 service_unauthorized`, a domain `5xx`, a timeout, or a connection reset becomes `503 dependency_unavailable`, message `A required dependency is unavailable.` The BFF does not tell the seller that an internal key was wrong. The JWT is minted per call. It is not reused across requests.

---

## 6. Register, login, and the cookie

```text
Seller Centre
  → POST /seller/auth/register
       → POST /v1/seller/register          identity-service
       ← { sessionToken, … }
  → Set-Cookie bm_seller
  ← 201 { … }          sessionToken is not in this body

Seller Centre
  → POST /seller/auth/login
       → POST /v1/seller/login
  → Set-Cookie bm_seller on 200
  ← 200 without sessionToken
```

The BFF forwards the JSON body and does not add fields. Email shape, the 12-character password, and `policyVersion` are identity’s rules.

Register:

```json
{
  "email": "owner@example.com",
  "password": "a-long-password",
  "policyVersion": "2026-10"
}
```

Login:

```json
{ "email": "owner@example.com", "password": "a-long-password" }
```

Forgot is `{ "email": "owner@example.com" }`. Reset is `{ "token": "…", "password": "a-new-long-password" }`. The same reset route accepts the `passwordChangeToken` from a forced change. This BFF does not add a second identity path.

Client timeout for register and login is 2 seconds. Argon2id is slow on purpose. Forgot and reset use 2 seconds because identity may email a link. The BFF does not retry login. Unknown email and a wrong password are both identity’s `401 credentials_invalid`. The BFF returns that status and code and sets no cookie.

`403 password_change_required` returns identity’s body, including `passwordChangeToken`, and sets no cookie. The token is not a session. Logs redact `password`, `token`, and `passwordChangeToken`.

### Cookie

Set only when identity’s success body contains `sessionToken`.

| Attribute | Value |
| --- | --- |
| Name | `bm_seller` |
| Value | The raw `sessionToken` from identity |
| HttpOnly | yes |
| Secure | yes |
| SameSite | `Lax` |
| Path | `/seller` |
| Domain | omitted. Host-only on the API host |
| Max-Age | 43200 seconds (12 hours), the seller absolute cap |

`Path=/seller` keeps this cookie off `/v1` customer routes. Customer-bff’s `bm_session` is a different name. This BFF never reads `bm_session`.

Clearing the cookie is `Set-Cookie` with the same name, `Path=/seller`, `Max-Age=0`, and an empty value. Same `HttpOnly`, `Secure`, and `SameSite`.

Logout is `POST /seller/auth/logout`. The BFF reads the cookie, calls identity `POST /v1/sessions/revoke`, then clears the cookie. Revoke failure still clears the cookie. The response is `204`. No cookie is `401 session_invalid`.

The JSON success body is identity’s body with `sessionToken` removed. Seller Centre then opens the profile. The seller row is created when seller-service consumes `user.registered`. Until that row exists, section 8 returns `503 account_pending`.

---

## 7. Role and status gate

Introspect returns `role` of `seller_owner`, `seller_catalogue`, or `seller_orders`, and `sellerId`. The BFF checks the role before the domain call. The domain service checks it again. A wrong role is `403 forbidden`, message `Forbidden.`, and the domain service is not called.

| Route group | `seller_owner` | `seller_catalogue` | `seller_orders` |
| --- | --- | --- | --- |
| Profile, agreement, KYC, submit, bank | yes | no | no |
| Staff invite | yes | no | no |
| Product upload, catalogue, stock | yes | yes | no |
| Order read, confirm, pack, cancel, returns | yes | no | yes |
| `GET /seller/me` | yes | yes | yes |

Listing and order routes then read `GET {SELLER_URL}/v1/sellers/{sellerId}` with the seller session. Timeout 1 second. The BFF uses `status` only. It does not log the body.

| Seller status | Profile and KYC | Catalogue and stock writes | Catalogue and stock reads | Orders |
| --- | --- | --- | --- | --- |
| Row missing (`404`) | `503 account_pending` | `503 account_pending` | `503 account_pending` | `503 account_pending` |
| `applied`, `kyc_pending`, `kyc_rejected` | owner, as in section 8 | `403 not_approved` | `403 not_approved` | `403 not_approved` |
| `approved`, `active` | owner | allowed | allowed | allowed |
| `suspended` | `409` from seller-service on edit | `403 listing_blocked` | allowed | allowed |
| `closed` | `409` from seller-service on edit | `403 listing_blocked` | `403 listing_blocked` | reads allowed, mutations `403 listing_blocked` |

`not_approved` message is `This seller is not approved yet.` `listing_blocked` message is `This seller cannot change listings.` `account_pending` message is `Setting up your account.` Seller Centre retries that last one. A seller who is not `approved` does not get an empty catalogue they can publish.

Suspended sellers can still confirm and pack an in-flight order. They cannot add stock or submit a listing. Closed sellers can read an order and cannot change it.

---

## 8. Profile, KYC, and the bank change

These routes require `seller_owner`. The path does not contain a seller id. The BFF calls `/v1/sellers/{sellerId}` with the id from introspect.

| Browser | seller-service | Notes |
| --- | --- | --- |
| `GET /seller/profile` | `GET /v1/sellers/:id` | Owner view. Masked PAN and account. `missing` when incomplete |
| `PUT /seller/profile` | `PUT /v1/sellers/:id` | Forward the body. Editable only while `applied` or `kyc_rejected` |
| `POST /seller/agreement` | `POST /v1/sellers/:id/agreement` | `{ agreementVersion }` |
| `POST /seller/kyc` | `POST /v1/sellers/:id/kyc` | `{ assetId, document }` after the asset is `ready` |
| `POST /seller/submit` | `POST /v1/sellers/:id/submit` | `409 profile_incomplete` includes `missing` |
| `POST /seller/bank` | `POST /v1/sellers/:id/bank` | `202` while finance has not approved. The live account is unchanged |

Client timeout is 1 second. GSTIN, PAN, IFSC, and the incomplete-profile rules are seller-service’s. This BFF forwards `400`, `404`, and `409` as sent, including `gstin_taken`, `kyc_not_ready`, `profile_incomplete`, `not_editable`, and `bank_pending`.

KYC bytes never hit this process.

```text
POST /seller/uploads     purpose kyc
  → POST /v1/uploads     media-service
  ← { url, … }           browser PUTs the file to that URL
GET /seller/uploads/:id
  → GET /v1/images/:id   until status is ready
POST /seller/kyc
  → POST /v1/sellers/:id/kyc
```

`POST /seller/uploads` for `purpose: "kyc"` is owner-only. Product uploads are in section 9. The BFF returns media’s body, which includes the presigned PUT. That URL is not logged. This BFF never returns a KYC download URL. The 60-second review GET belongs to admin-bff.

A bank change is the owner’s request only. `POST /seller/bank/approve` and `GET /seller/payout` are not routes here. They answer `404`.

`GET /seller/me` calls identity `GET /v1/seller/me` and does not call seller-service. `POST /seller/staff` forwards `{ email, password, role }` to identity `POST /v1/seller/staff`. `role` is `seller_catalogue` or `seller_orders`. The temporary password is not logged.

---

## 9. Catalogue, images, and stock

These routes require `seller_owner` or `seller_catalogue`, then the status gate in section 7. The BFF does not send a seller id in the body. Catalogue and inventory take it from the session.

| Browser | Domain | Notes |
| --- | --- | --- |
| `GET /seller/categories` | catalogue `GET /v1/categories` | The tree, so the form can pick a leaf |
| `GET /seller/products` | catalogue `GET /v1/seller/products` | This seller’s rows, any status |
| `POST /seller/products` | catalogue `POST /v1/seller/products` | `201` draft |
| `PATCH /seller/products/:id` | catalogue `PATCH /v1/seller/products/:id` | |
| `POST /seller/products/:id/variants` | catalogue `POST /v1/seller/products/:id/variants` | |
| `PATCH /seller/variants/:id` | catalogue `PATCH /v1/seller/variants/:id` | |
| `POST /seller/variants/:id/images` | catalogue `POST /v1/seller/variants/:id/images` | Attaches a ready product asset |
| `POST /seller/products/:id/submit` | catalogue `POST /v1/seller/products/:id/submit` | `pending_review` or `live`. Publish checks stay in catalogue |
| `POST /seller/uploads` | media `POST /v1/uploads` | `purpose: "product"` for catalogue staff. `purpose: "kyc"` is owner-only |
| `GET /seller/uploads/:id` | media `GET /v1/images/:id` | Ready product rows may include CDN URLs. KYC rows do not include a download URL |
| `GET /seller/stock` | inventory `GET /v1/seller/stock` | This seller’s pools |
| `PUT /seller/stock/:variantId` | inventory `PUT /v1/seller/stock/:variantId` | Body `{ onHand }`. `reserved` in the body is ignored |

Client timeout is 1 second for each of these calls. The browser uploads bytes to the presigned URL. This BFF does not proxy the file.

Catalogue’s `409 publish_blocked`, inventory’s `400 stock_below_reserved`, and media’s upload errors pass through with the same status and code. Price, MRP, HSN, and tax are catalogue’s rules. Available-to-sell is inventory’s rule. This BFF does not recompute either.

---

## 10. Orders

These routes require `seller_owner` or `seller_orders`, then the status gate. The browser never supplies `sellerId`. Confirm and pack always use the session id.

| Browser | order-service |
| --- | --- |
| `GET /seller/orders/:id` | `GET /v1/orders/:id` |
| `POST /seller/orders/:id/confirm` | `POST /v1/orders/:id/groups/{sellerId}/confirm` |
| `POST /seller/orders/:id/pack` | `POST /v1/orders/:id/groups/{sellerId}/pack` |
| `POST /seller/orders/:id/cancel` | `POST /v1/orders/:id/cancel` |
| `POST /seller/orders/:id/returns/:returnId/accept` | `POST /v1/orders/:id/returns/:returnId/accept` |
| `POST /seller/orders/:id/returns/:returnId/reject` | `POST /v1/orders/:id/returns/:returnId/reject` |

Client timeout is 1 second for the read and the return decisions, and 2 seconds for confirm, pack, and cancel. Confirm allocates an invoice number. The BFF does not invent that number.

There is no `GET /seller/orders`. Order-service does not list by seller. Seller Centre opens an order it already has an id for. Do not scan another service’s database to fake the queue.

A seller sees only their lines and the ship-to snapshot, because order-service strips the rest. Another seller’s order is `404 not_found` from order-service. This BFF forwards that. It does not turn it into `403`.

Confirm and pack do not book a courier. The label is fulfilment’s, and that service is not in this build. Pack means the seller has printed whatever label they already have. It does not mean this BFF created one.

---

## 11. HTTP API

Public base path: `/seller`. Health and metrics are on port 8080 and are not routed by the public `/seller/*` ingress rule. The ALB health check calls `/health/ready` on the pod directly.

| Method | Path | Cookie | Success |
| --- | --- | --- | --- |
| POST | `/seller/auth/register` | Set on success | `201`, no `sessionToken` |
| POST | `/seller/auth/login` | Set on success | `200`, no `sessionToken` |
| POST | `/seller/auth/password/forgot` | No | `202` |
| POST | `/seller/auth/password/reset` | No | `204` |
| POST | `/seller/auth/logout` | Cleared | `204` |
| GET | `/seller/me` | Required | `200` |
| POST | `/seller/staff` | Owner | `201` |
| GET | `/seller/profile` | Owner | `200` |
| PUT | `/seller/profile` | Owner | `200` |
| POST | `/seller/agreement` | Owner | `200` |
| POST | `/seller/uploads` | Owner for KYC, owner or catalogue for product | `201` presigned PUT |
| GET | `/seller/uploads/:id` | Same as the upload | `200` status |
| POST | `/seller/kyc` | Owner | `200` |
| POST | `/seller/submit` | Owner | `200`, or `409 profile_incomplete` |
| POST | `/seller/bank` | Owner | `202` |
| GET | `/seller/categories` | Catalogue role, and approved | `200` |
| GET | `/seller/products` | Catalogue role, and approved | `200` |
| POST | `/seller/products` | Catalogue role, and approved | `201` |
| PATCH | `/seller/products/:id` | Catalogue role, and approved | `200` |
| POST | `/seller/products/:id/variants` | Catalogue role, and approved | `201` |
| PATCH | `/seller/variants/:id` | Catalogue role, and approved | `200` |
| POST | `/seller/variants/:id/images` | Catalogue role, and approved | `201` |
| POST | `/seller/products/:id/submit` | Catalogue role, and approved | `200` |
| GET | `/seller/stock` | Catalogue role, and approved or suspended | `200` |
| PUT | `/seller/stock/:variantId` | Catalogue role, and approved | `200` |
| GET | `/seller/orders/:id` | Orders role, and approved, suspended, or closed | `200` |
| POST | `/seller/orders/:id/confirm` | Orders role, and approved or suspended | `200` |
| POST | `/seller/orders/:id/pack` | Orders role, and approved or suspended | `200` |
| POST | `/seller/orders/:id/cancel` | Orders role, and approved or suspended | `200` |
| POST | `/seller/orders/:id/returns/:returnId/accept` | Orders role, and approved or suspended | `200` |
| POST | `/seller/orders/:id/returns/:returnId/reject` | Orders role, and approved or suspended | `200` |
| GET | `/health/live` | No | `200` if the process is up |
| GET | `/health/ready` | No | `200` only when the config in section 2 is set |
| GET | `/metrics` | No | Prometheus text. Cluster scrape only |

JSON errors:

```json
{ "code": "not_approved", "message": "This seller is not approved yet.", "requestId": "…" }
```

`message` is safe to show. `code` is what Seller Centre branches on. The BFF does not rewrite a domain `message`. `profile_incomplete` still includes `missing` when seller-service sent it.

| Upstream | BFF |
| --- | --- |
| A domain 4xx with `code` and `message` | Same status, same `code`, same `message` |
| `401 service_unauthorized` | `503 dependency_unavailable` |
| Timeout, connection reset, or domain 5xx | `503 dependency_unavailable` |

Codes this BFF generates itself: `origin_rejected`, `session_invalid`, `forbidden`, `not_approved`, `listing_blocked`, `account_pending`, `dependency_unavailable`, `invalid_body`.

Logs redact `authorization`, `cookie`, `bm_seller`, `sessionToken`, `x-session-token`, `password`, `token`, `passwordChangeToken`, `pan`, `bankAccount`, `ifsc`, `accountName`, and `url`.

---

## 12. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `SERVICE_JWT_KEY` | Same HMAC key the domain services use to verify |
| Config | `IDENTITY_URL` | `http://identity-service.identity.svc.cluster.local` |
| Config | `SELLER_URL` | `http://seller-service.sellers.svc.cluster.local` |
| Config | `MEDIA_URL` | `http://media-service.catalogue.svc.cluster.local` |
| Config | `CATALOGUE_URL` | `http://catalogue-service.catalogue.svc.cluster.local` |
| Config | `INVENTORY_URL` | `http://inventory-service.commerce.svc.cluster.local` |
| Config | `ORDER_URL` | `http://order-service.commerce.svc.cluster.local` |
| Config | `SELLER_WEB_ORIGINS` | Comma-separated. Prod is `https://seller.buyymart.com` |
| Config | `PORT` | `8080` |

No `DATABASE_URL`. No `REDIS_URL`. No `DATA_KEY`. This process does not decrypt PAN or a bank account.

Secrets live in Secrets Manager at `buyymart/{env}/seller-bff` and arrive as a Kubernetes Secret through External Secrets. They are not in the image and not in git.

---

## 13. Local run

This Compose file is the API only. There is no Postgres and no Redis. Identity, seller, media, catalogue, inventory, and order must already be running when a test follows a real call. Unit tests inject those clients and do not start Compose.

| Variable | Local value |
| --- | --- |
| `IDENTITY_URL` | `http://host.docker.internal:8080` |
| `SELLER_URL` | `http://host.docker.internal:8097` |
| `MEDIA_URL` | `http://host.docker.internal:8085` |
| `CATALOGUE_URL` | `http://host.docker.internal:8083` |
| `INVENTORY_URL` | `http://host.docker.internal:8089` |
| `ORDER_URL` | `http://host.docker.internal:8093` |
| `SERVICE_JWT_KEY` | The same value those services were started with |
| `SELLER_WEB_ORIGINS` | `http://localhost:5174` |
| `PORT` | `8099`, so it does not collide with seller-service on 8097 |

The container still listens on 8080. Compose publishes `8099:8080`.

Seller Centre is not part of this Compose file. A browser or `curl` against port 8099 is enough to prove the cookie and one domain call. `curl` must send the `Cookie` header stored from login. The JSON body is not the session.

---

## 14. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Register with a valid email and password | `201`, `Set-Cookie: bm_seller`, `Path=/seller`. The JSON body has no `sessionToken` |
| Login with the wrong password | `401 credentials_invalid`, no cookie |
| Login that must change the password | `403 password_change_required`, `passwordChangeToken` present, no cookie |
| Logout | `204`, cookie cleared |
| Customer or staff token in `bm_seller` | `401 session_invalid`, cookie cleared |
| `GET /seller/profile` with no cookie | `401 session_invalid` |
| `seller_catalogue` calls `PUT /seller/profile` | `403 forbidden`. seller-service is not called |
| `seller_orders` calls `POST /seller/products` | `403 forbidden`. catalogue-service is not called |
| Profile while the seller row is missing | `503 account_pending` |
| Owner submits a complete profile | `200` and seller-service was called with the service JWT and `X-Session-Token` |
| Submit with no agreement | `409 profile_incomplete` and `missing` contains `agreement` |
| KYC asset not ready | `409 kyc_not_ready` |
| `POST /seller/products` while `applied` | `403 not_approved`. catalogue-service is not called |
| `PUT /seller/stock/:variantId` while `suspended` | `403 listing_blocked` |
| `POST /seller/orders/:id/confirm` while `suspended` | order-service is called at `/groups/{sessionSellerId}/confirm` |
| Confirm body that includes a different `sellerId` | The path still uses the session id |
| `GET /seller/orders/:id` for another seller’s order | `404 not_found` |
| `POST /seller/kyc/pan/link` and `POST /seller/bank/approve` | `404`. Those routes are not registered |
| Identity down on login | `503 dependency_unavailable`, no cookie |
| Origin not on the allow-list | `403 origin_rejected` |
| `/health/ready` with an empty `ORDER_URL` | `503` |
| Logs for a profile save | No PAN, bank account, password, cookie, or presigned URL |

---

## 15. Outside this service

| Concern | Owner |
| --- | --- |
| Password, session, seller staff | `identity-service` |
| Profile, GSTIN, KYC status, bank ciphertext | `seller-service` |
| File bytes and the presigned PUT | `media-service` |
| Product, price, tax, publish checks | `catalogue-service` |
| `onHand` and available-to-sell | `inventory-service` |
| Confirm, pack, invoice number, return decision | `order-service` |
| Trust review and the 60-second KYC link | `admin-bff` |
| Seller Centre screens | The Seller Centre web app |

---

## 16. Later routes, not this build

Do not add these until the domain service they call exists.

| Route | Callee | Why it waits |
| --- | --- | --- |
| Label, AWB, and tracking | `fulfilment-service` | Not deployed. Pack does not book a courier |
| Statement download | `settlement-service` | Not deployed. Version 1 pays by a transfer outside the app |
| `GET /seller/orders` | `order-service` | That service has no seller index. Do not invent one in this BFF |

When the label route exists, the timeout is 3 seconds. A failure shows the error and does not mark the group packed. The statement route is a read, timeout 1 second, and it never returns a decrypted bank account. Finance still approves a bank change in admin-bff.

# seller-service — implementation spec

**Namespace:** `sellers`  
**Database:** Aurora PostgreSQL database `seller` on cluster `bm-commerce`.  
**Cache:** none. A Redis flush must not be required to read approval or GSTIN.  
**Callers:** `seller-bff` for the application, KYC attach, and the owner’s own profile. `admin-bff` for review, suspension, bank change, and the auto-approve flag. `customer-bff` for the public card on a product page. `catalogue-service`, `inventory-service`, `media-service`, and `order-service` for approval, suspension, state, and GSTIN. Browsers never call this service.  
**Calls:** `identity-service` to introspect the session. `media-service` to read a KYC asset’s status. AWS S3 to sign a 60-second GET of a KYC object.  
**Publishes:** `SellerApproved`, `SellerSuspended`  
**Consumes:** `UserRegistered`, `ProductChanged`  
**Product rules:** [modules/13-sellers.md](../modules/13-sellers.md)

This is the build document for the seller profile, KYC status, suspension, and the public card. Identity owns the login, the password, and the session role. Catalogue owns the product. Inventory owns stock. Settlement owns the ledger and the payout. This service stores the bank account and the tax ids that settlement will read. It does not calculate TCS, TDS, or commission.

BuyyMart’s own inventory is one seller row with `kind = owned`. Every other seller is `kind = marketplace`. Settlement uses `kind` to skip marketplace TCS and TDS for the owned row. This service does not compute those amounts.

`approved` is the gate for the first listing. `active` is set when the first product goes live. A suspended or closed seller cannot add stock or receive a new order. Past orders stay. Catalogue consumes `SellerSuspended` and hides that seller’s products. This service does not update the catalogue row.

Version 1 is still reviewed by trust staff whether the application was invited or open. Nothing goes live without that review, except the owned seller row created from configuration.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as payment-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as payment-service |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Ids | UUID | The seller id comes from identity. It is not generated here |
| Logs | `pino` JSON to stdout | Log the seller id and the staff id. Do not log PAN, a bank account, IFSC, an account name, a KYC object body, a presigned URL, an email, or a phone |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One Seller Centre action across the BFF, identity, media, and this service |
| Metrics | `prom-client` on `GET /metrics` | Approvals, rejections, suspensions, outbox lag |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as payment-service |
| Container port | 8080 | Service port 80 targets 8080 |

PAN, bank account number, IFSC, and account name are encrypted before insert. GSTIN is stored in plaintext because the product page and the invoice show it. Customers do not see PAN, the bank account, or KYC images.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | Profile, KYC attach, review, public card, operational read. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Publishes the outbox. Creates the seller row from `UserRegistered`. Sets `active` from the first live `ProductChanged` |

The API does not publish to EventBridge itself. A status change and its outbox row commit in one Postgres transaction. The worker sets `published_at`. A second poll does not send the row again.

`GET /health/ready` is Postgres `SELECT 1`. It does not call identity, media, or S3.

There is no Redis.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `profile` | The application fields, the agreement, and submit |
| `kyc` | Attach a ready media asset and sign a 60-second review GET |
| `review` | Approve, reject, suspend, reinstate, close, and the auto-approve flag |
| `bank` | A bank change after approval, without replacing the payout account early |
| `read` | Operational read, public card, and the owner’s own view |
| `crypto` | Encrypt and decrypt PAN and bank fields. Tests use a local key |
| `outbox` | The events in section 10 |

This service does not store a password, a session token, a product, or a stock count. Seller staff logins stay in identity. This service reads `family`, `role`, and `sellerId` from introspect and does not keep a second role table.

---

## 4. How a request is trusted

BFFs and domain services call:

```text
http://seller-service.sellers.svc.cluster.local
```

Every `/v1` call except health and metrics carries:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `seller-service`. `exp - iat` is at most 60 seconds. Algorithm HS256. Signed with `SERVICE_JWT_KEY` |
| `X-Request-Id` | UUID. Generated by the BFF when the caller did not send one. Echoed on the response |
| `traceparent` | W3C trace context. Forwarded to identity and media |
| `X-Session-Token` | Present on seller and staff routes. This service calls identity `POST /v1/sessions/introspect`. Timeout 200 ms. A body `sellerId` or `role` is ignored |

A missing, expired, or wrong-audience JWT is `401 service_unauthorized`. A session identity cannot load is `503 dependency_unavailable`. An invalid session is `401 session_invalid`.

| Route | Who |
| --- | --- |
| Operational `GET /v1/sellers/:id` | Service JWT and no session. Catalogue, inventory, media, order, and the BFFs |
| `GET /v1/sellers/:id/public` | Service JWT and no session. `customer-bff` only forwards this shape |
| Own profile, submit, KYC attach, agreement | Family `seller`, role `seller_owner`, and `sellerId` equal to the path. `seller_catalogue` and `seller_orders` are `403 forbidden`. Another seller’s id is `404 not_found` |
| Review link, approve, reject, suspend, reinstate, close, auto-approve | Family `staff`, role `trust`, `admin`, or `super_admin` |
| Bank decision and payout read | Family `staff`, role `finance`, `admin`, or `super_admin` |
| A customer or guest session | `403 forbidden` on seller and staff routes |

JSON errors:

```json
{ "code": "not_found", "message": "Not found.", "requestId": "…" }
```

`message` is one sentence a BFF may show. `code` is what the BFF branches on.

Consumed events are not HTTP from the public ingress. The worker takes them from the bus. In development only, `POST /internal/events` on the worker accepts one envelope. That route is absent when `NODE_ENV` is `staging` or `production`.

---

## 5. Status

```text
applied → kyc_pending → kyc_rejected
                      → approved → active
active → suspended
active → closed
suspended → active
suspended → closed
```

| Status | Meaning |
| --- | --- |
| `applied` | The row exists from `UserRegistered`. The profile is not submitted |
| `kyc_pending` | Trust can approve or reject. The seller cannot edit the profile until a decision |
| `kyc_rejected` | The seller may edit the named fields and submit again |
| `approved` | Listing is allowed. No live product yet |
| `active` | At least one product has been live. Listing stays allowed |
| `suspended` | No new listing and no new order. Catalogue hides live products |
| `closed` | Returns and payouts are finished. Not a delete. Ledger rows stay |

`listingAllowed` is not a column. Callers treat `approved` and `active` as allowed. Every other status is not allowed. Suspended and closed are not allowed even if an older cache said otherwise. There is no cache in this service.

A seller cannot approve, suspend, or close their own row.

`closed` is not a hard delete. The application role cannot `DELETE` the seller row.

---

## 6. Profile

`UserRegistered` with `family: seller` and a `sellerId` inserts one row: `kind = marketplace`, `status = applied`, and the owner email from the event. A customer registration stores the event id and does not insert a seller. A second delivery of the same event id does nothing. If the seller id already exists, including the owned row, the event is stored and the row is not downgraded to `applied`.

The owner then fills the profile with `PUT /v1/sellers/:id`. Allowed only in `applied` or `kyc_rejected`. `kyc_pending`, `approved`, `active`, `suspended`, and `closed` return `409 not_editable`, message `This seller profile cannot be edited.`

| Field | Rule |
| --- | --- |
| `legalName` | 1–200 characters. Shown on the product page and the invoice |
| `brandName` | Optional. 1–200 characters, or null |
| `registered` | Boolean. A registered business is taxable in version 1 |
| `address.line1` | 1–200 characters |
| `address.city` | 1–80 characters |
| `address.state` | One name from the GST table below. This is the string order-service compares with the delivery state |
| `address.pin` | Six digits |
| `customerCarePhone` | Ten digits, starting with 6–9 |
| `customerCareEmail` | Trimmed and lowercased. Must contain `@`. Not shown on the public card |
| `gstin` | Required when `registered` is true. Null when `registered` is false. Format below |
| `pan` | Required. Five letters, four digits, one letter. Stored encrypted. A SHA-256 of the normalised PAN is stored for uniqueness. Two sellers cannot share a PAN |
| `bankAccount` | Required. 9–18 digits. Stored encrypted |
| `ifsc` | Required. Four letters, a zero, six letters or digits. Stored encrypted |
| `accountName` | Required. 1–200 characters. Stored encrypted |
| `grievanceOfficerName` | Required before submit. 1–200 characters |
| `grievancePhone` | Required before submit. Same phone rule |
| `grievanceEmail` | Required before submit. Same email rule |
| `licence` | Null in version 1. The column exists so a later FSSAI or BIS value has a place |

GSTIN is 15 characters: two digits, five letters, four digits, one letter, one letter or digit, `Z`, one letter or digit. The first two digits must be the code for `address.state`. A GSTIN whose state code does not match is `400 invalid_body`. A duplicate GSTIN is `409 gstin_taken`, message `That GSTIN is already registered.`

| Code | `address.state` |
| --- | --- |
| 01 | Jammu and Kashmir |
| 02 | Himachal Pradesh |
| 03 | Punjab |
| 04 | Chandigarh |
| 05 | Uttarakhand |
| 06 | Haryana |
| 07 | Delhi |
| 08 | Rajasthan |
| 09 | Uttar Pradesh |
| 10 | Bihar |
| 11 | Sikkim |
| 12 | Arunachal Pradesh |
| 13 | Nagaland |
| 14 | Manipur |
| 15 | Mizoram |
| 16 | Tripura |
| 17 | Meghalaya |
| 18 | Assam |
| 19 | West Bengal |
| 20 | Jharkhand |
| 21 | Odisha |
| 22 | Chhattisgarh |
| 23 | Madhya Pradesh |
| 24 | Gujarat |
| 26 | Dadra and Nagar Haveli and Daman and Diu |
| 27 | Maharashtra |
| 29 | Karnataka |
| 30 | Goa |
| 31 | Lakshadweep |
| 32 | Kerala |
| 33 | Tamil Nadu |
| 34 | Puducherry |
| 35 | Andaman and Nicobar Islands |
| 36 | Telangana |
| 37 | Andhra Pradesh |
| 38 | Ladakh |

`POST /v1/sellers/:id/agreement` with `{ agreementVersion }` sets `agreementAcceptedAt` to now. `agreementVersion` is 1–40 characters. Only `seller_owner` in `applied` or `kyc_rejected`. Submit is blocked until this has been called for the current version.

`POST /v1/sellers/:id/submit` moves `applied` or `kyc_rejected` to `kyc_pending` when every required field is set, the agreement is accepted, PAN and bank proof assets are attached, and a GST certificate is attached when `registered` is true. Anything missing is `409 profile_incomplete`. The body is `{ missing: ["pan", "agreement"] }` and the message is `The seller profile is incomplete.` A submit that is already `kyc_pending` returns `200` and does not write a second audit row.

---

## 7. KYC documents

The owner uploads the file through media-service. That service signs the PUT and marks the asset `ready`. It does not publish an event for KYC. This service does not accept the bytes.

`POST /v1/sellers/:id/kyc` body `{ assetId, document }` where `document` is `pan`, `gst_certificate`, or `bank_proof`.

1. The caller is `seller_owner` and the status is `applied` or `kyc_rejected`. Otherwise `409 not_editable`.
2. `GET {MEDIA_URL}/v1/images/{assetId}` with a service JWT for audience `media-service`, forwarding `X-Session-Token`. Timeout 1 second.
3. A media failure or timeout is `503 dependency_unavailable`. No document row changes.
4. `purpose` must be `kyc`, `document` must match the body, and `status` must be `ready`. Otherwise `409 kyc_not_ready`, message `That document is not ready.`
5. Store `asset_id` and `object_key = kyc/{sellerId}/{assetId}`. Replace the previous row for that document. Do not download the object.

Trust staff do not get a KYC URL from media-service. `POST /v1/sellers/:id/kyc/{document}/link` returns a presigned GET of 60 seconds for a stored key. The response is `{ url, expiresAt }`. Only `trust`, `admin`, or `super_admin`. The URL is not logged. A missing document is `404 not_found`.

`SIGNING_MODE=fake` is allowed only when `NODE_ENV=development`. The fake URL is `https://kyc.local/{objectKey}` and it does not call AWS. Stage and prod refuse to start unless `SIGNING_MODE=s3` and `KYC_BUCKET` is set. Tests inject the signer.

---

## 8. Review, suspension, and the owned seller

Approve: `POST /v1/sellers/:id/approve` while `kyc_pending`. Body `{ note }` of at least 10 characters. Status becomes `approved`. One outbox `seller.approved`. An audit row stores the staff id and the note. A second approve of an `approved` or `active` seller returns `200` and does not publish again. Any other status is `409 not_reviewable`, message `This seller cannot be reviewed.`

Reject: `POST /v1/sellers/:id/reject` while `kyc_pending`. Body `{ note, fields }` where `note` is at least 10 characters and `fields` is a non-empty list drawn from `legalName`, `address`, `gstin`, `pan`, `bank`, `grievance`, `agreement`, `gst_certificate`. Status becomes `kyc_rejected`. The field list is stored. No `seller.approved` event. The seller may edit and submit again.

Suspend: `POST /v1/sellers/:id/suspend` while `approved` or `active`. Body `{ reason }` of at least 10 characters. Status becomes `suspended`. One outbox `seller.suspended`. Catalogue hides that seller’s live products. In-flight orders are not cancelled here.

Reinstate: `POST /v1/sellers/:id/reinstate` while `suspended`. Body `{ note }` of at least 10 characters. If `activated_at` is set, status returns to `active`. Otherwise it returns to `approved`. This does not publish an event and does not unhide products. The seller submits listings again. Catalogue keeps them hidden until they pass publish.

Close: `POST /v1/sellers/:id/close` while `active` or `suspended`. Body `{ reason }` of at least 10 characters. Status becomes `closed`. One outbox `seller.suspended` whose `reason` is the staff reason and whose `status` is `closed`, so catalogue hides any product still live. The row stays.

Auto-approve: `PATCH /v1/sellers/:id/auto-approve` body `{ enabled: boolean }`. Staff `trust`, `admin`, or `super_admin`. Default is false. Catalogue reads this flag. This service does not count live products. After the first ten live listings, catalogue may skip its own review queue when the flag is on. A marketplace seller is not auto-approved by an environment variable.

### Owned seller

On API startup, if `OWNED_SELLER_ID` is set, upsert that id with `kind = owned`, `status = active`, `legal_name` from `OWNED_LEGAL_NAME`, `state` from `OWNED_STATE`, and `gstin` from `OWNED_GSTIN`. `activated_at` is set. The upsert does not publish `seller.approved`. Stage and prod refuse to start when `OWNED_SELLER_ID` or `OWNED_GSTIN` is empty. Development may omit them, and then no owned row is created.

`OWNED_SELLER_AUTO_APPROVE=true` sets `auto_approve` on that row only when `NODE_ENV=development`. Stage and prod ignore that variable and leave the column false unless staff set it.

### Bank change

After `approved` or `active`, the owner submits `POST /v1/sellers/:id/bank` with a new account, IFSC, account name, and a ready `bank_proof` asset id. The live ciphertext is not replaced. A `bank_changes` row is `pending`. Payouts stay on the old account until finance approves.

`POST /v1/sellers/:id/bank/approve` by finance, admin, or super_admin copies the pending ciphertext onto the seller and marks the change `approved`. `POST /v1/sellers/:id/bank/reject` with a note of at least 10 characters marks it `rejected` and leaves the live account. A second pending change while one is pending is `409 bank_pending`, message `A bank change is already waiting.`

`GET /v1/sellers/:id/payout` is finance, admin, or super_admin. It returns decrypted `pan`, `bankAccount`, `ifsc`, and `accountName` of the live account, plus `gstin` and `legalName`. It is not the operational read. Catalogue and order-service must not call it. The response is not logged.

---

## 9. Reads

Operational `GET /v1/sellers/:id` with a service JWT and no session. `200`:

```json
{
  "id": "…",
  "kind": "marketplace",
  "status": "approved",
  "legalName": "Example Traders",
  "brandName": null,
  "registered": true,
  "city": "Bengaluru",
  "state": "Karnataka",
  "gstin": "29ABCDE1234F1Z5",
  "customerCarePhone": "9876543210",
  "autoApprove": false,
  "licence": null
}
```

No PAN, no bank fields, no email, no KYC key. Unknown id is `404 not_found`. This is the body catalogue, inventory, media, and order-service already expect. Order-service compares `state` with the delivery address state and copies `gstin` onto the invoice. A failed call is their `503`, not a guess.

Public `GET /v1/sellers/:id/public`. `200` only when status is `approved` or `active`. Every other status, and an unknown id, is `404 not_found`.

```json
{
  "id": "…",
  "legalName": "Example Traders",
  "registered": true,
  "city": "Bengaluru",
  "customerCarePhone": "9876543210",
  "gstin": "29ABCDE1234F1Z5",
  "rating": null
}
```

`rating` is null in version 1. This service does not store ratings. The public body does not include `autoApprove`, `state`, PAN, or bank fields. `customer-bff` uses this route for the product page.

The owner’s `GET /v1/sellers/:id` with a seller session returns the application, the status, `missing` when the profile is incomplete, masked PAN and account (last four characters), and each document as `{ document, attached }`. It does not return a presigned URL or the full account.

A staff review `GET` returns the same application plus `fieldsToFix` and whether a bank change is pending. Full PAN and the full account are only on the payout route.

Another seller’s id is `404 not_found`, not `403`.

---

## 10. Events

Wire `type` values are lowercase, matching catalogue:

| Name | `type` | When |
| --- | --- | --- |
| `SellerApproved` | `seller.approved` | Trust approves a pending application. Not sent for the owned upsert and not sent again on a repeat approve |
| `SellerSuspended` | `seller.suspended` | Trust suspends, or staff close the seller |

`seller.approved`:

```json
{ "sellerId": "…", "kind": "marketplace" }
```

`seller.suspended`:

```json
{ "sellerId": "…", "status": "suspended", "reason": "…" }
```

On close, `status` is `closed` and `reason` is the staff reason. Catalogue already hides products from `seller.suspended`. It does not need a new event name for close.

Envelope:

```json
{
  "id": "…",
  "type": "seller.approved",
  "source": "seller-service",
  "time": "2026-10-07T08:00:00Z",
  "traceId": "…",
  "data": {}
}
```

`id` is the outbox row id. The worker publishes at most 50 unpublished rows per poll. An empty `EVENTBRIDGE_BUS_NAME` logs the envelope and does not call AWS, and still sets `published_at`.

### Consumed

| Event | What this service does |
| --- | --- |
| `user.registered` | Seller family with `sellerId`: insert `applied` if missing. Customer family, or a seller id that already exists: store the event id and do not change the row |
| `product.changed` | When `status` is `live` and this seller is `approved`, set `active` and `activated_at`. Already `active` is a no-op. `suspended`, `closed`, and `kyc_*` do not become `active` |

The worker stores `processed_events.id` from the envelope. A second delivery does nothing. An unknown type, or a payload without the id this service needs, stores the event id and does not change a seller.

`product.changed` does not publish `seller.approved`. Approval already happened.

---

## 11. HTTP API

| Method | Path | Who | Success |
| --- | --- | --- | --- |
| GET | `/health/live` | Public | `200` `{ status: "live" }` |
| GET | `/health/ready` | Public | `200` or `503` |
| GET | `/metrics` | Public | Prometheus text |
| GET | `/v1/sellers/:id` | Service JWT, no session | `200` operational read |
| GET | `/v1/sellers/:id/public` | Service JWT, no session | `200` public card, or `404` |
| GET | `/v1/sellers/:id` | Owner, or review staff | `200` the view in section 9 |
| PUT | `/v1/sellers/:id` | Owner, while editable | `200` |
| POST | `/v1/sellers/:id/agreement` | Owner, while editable | `200` |
| POST | `/v1/sellers/:id/kyc` | Owner, while editable | `200` |
| POST | `/v1/sellers/:id/submit` | Owner | `200` `kyc_pending`, or `409 profile_incomplete` |
| POST | `/v1/sellers/:id/kyc/:document/link` | Trust, admin, super_admin | `200` `{ url, expiresAt }` |
| POST | `/v1/sellers/:id/approve` | Trust, admin, super_admin | `200` |
| POST | `/v1/sellers/:id/reject` | Trust, admin, super_admin | `200` |
| POST | `/v1/sellers/:id/suspend` | Trust, admin, super_admin | `200` |
| POST | `/v1/sellers/:id/reinstate` | Trust, admin, super_admin | `200` |
| POST | `/v1/sellers/:id/close` | Trust, admin, super_admin | `200` |
| PATCH | `/v1/sellers/:id/auto-approve` | Trust, admin, super_admin | `200` |
| POST | `/v1/sellers/:id/bank` | Owner, after approval | `202` pending |
| POST | `/v1/sellers/:id/bank/approve` | Finance, admin, super_admin | `200` |
| POST | `/v1/sellers/:id/bank/reject` | Finance, admin, super_admin | `200` |
| GET | `/v1/sellers/:id/payout` | Finance, admin, super_admin | `200` decrypted live account |

There is no browser route that approves a seller. There is no route that returns a permanent KYC URL.

| Code | Message |
| --- | --- |
| `service_unauthorized` | The caller is not authorized. |
| `session_invalid` | That session is not valid. |
| `dependency_unavailable` | A required dependency is unavailable. |
| `sellers_unavailable` | Sellers are unavailable. |
| `forbidden` | Forbidden. |
| `not_found` | Not found. |
| `invalid_body` | The request body is invalid. |
| `not_editable` | This seller profile cannot be edited. |
| `profile_incomplete` | The seller profile is incomplete. |
| `kyc_not_ready` | That document is not ready. |
| `not_reviewable` | This seller cannot be reviewed. |
| `gstin_taken` | That GSTIN is already registered. |
| `bank_pending` | A bank change is already waiting. |

Postgres down on a write or a read is `503 sellers_unavailable`. Live stays `200`. Ready is `503`.

---

## 12. PostgreSQL schema

Database `seller`. The application role `seller_app` can `SELECT`, `INSERT`, and `UPDATE` on these tables. It cannot `DELETE`, `DROP`, `TRUNCATE`, or alter schema. The migration role `seller_migrator` runs `migrations/` and is not the runtime role. The API refuses to start if `DATABASE_URL` uses `seller_migrator` or if `DATABASE_MIGRATOR_URL` is set.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE seller_kind AS ENUM ('owned', 'marketplace');

CREATE TYPE seller_status AS ENUM (
  'applied', 'kyc_pending', 'kyc_rejected', 'approved', 'active', 'suspended', 'closed'
);

CREATE TYPE kyc_document AS ENUM ('pan', 'gst_certificate', 'bank_proof');

CREATE TYPE bank_change_status AS ENUM ('pending', 'approved', 'rejected');

CREATE TABLE sellers (
  id                       uuid PRIMARY KEY,
  kind                     seller_kind NOT NULL,
  status                   seller_status NOT NULL,
  email                    text,
  legal_name               text,
  brand_name               text,
  registered               boolean NOT NULL DEFAULT false,
  address_line1            text,
  city                     text,
  state                    text,
  pin                      text,
  customer_care_phone      text,
  customer_care_email      text,
  gstin                    text UNIQUE,
  pan_ciphertext           bytea,
  pan_fingerprint          text UNIQUE,
  bank_account_ciphertext  bytea,
  ifsc_ciphertext          bytea,
  account_name_ciphertext  bytea,
  grievance_name           text,
  grievance_phone          text,
  grievance_email          text,
  agreement_version        text,
  agreement_accepted_at    timestamptz,
  auto_approve             boolean NOT NULL DEFAULT false,
  licence                  text,
  fields_to_fix            text[] NOT NULL DEFAULT '{}',
  suspension_reason        text,
  activated_at             timestamptz,
  closed_at                timestamptz,
  created_at               timestamptz NOT NULL DEFAULT now(),
  updated_at               timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE kyc_documents (
  seller_id   uuid NOT NULL REFERENCES sellers (id),
  document    kyc_document NOT NULL,
  asset_id    uuid NOT NULL,
  object_key  text NOT NULL,
  attached_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (seller_id, document)
);

CREATE TABLE bank_changes (
  id                       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id                uuid NOT NULL REFERENCES sellers (id),
  bank_account_ciphertext  bytea NOT NULL,
  ifsc_ciphertext          bytea NOT NULL,
  account_name_ciphertext  bytea NOT NULL,
  asset_id                 uuid NOT NULL,
  status                   bank_change_status NOT NULL,
  note                     text,
  decided_by               uuid,
  created_at               timestamptz NOT NULL DEFAULT now(),
  decided_at               timestamptz
);

CREATE UNIQUE INDEX bank_changes_one_pending
  ON bank_changes (seller_id)
  WHERE status = 'pending';

CREATE TABLE processed_events (
  id           uuid PRIMARY KEY,
  type         text NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE audit (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id  uuid,
  actor_id   uuid,
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

The advisory lock for migrations is `2147483008`.

Ciphertext is AES-256-GCM. `DATA_KEY` is 32 bytes, base64. The nonce and the auth tag are stored with the ciphertext. Development may use a local key. Stage and prod refuse to start when `DATA_KEY` is empty. Tests that need a payout read use the local key. Logs and the operational read never contain the plaintext.

---

## 13. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `seller_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Migration job only. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `seller-service` |
| Secret | `DATA_KEY` | 32-byte key for PAN and bank fields |
| Config | `IDENTITY_URL` | Introspect sessions |
| Config | `MEDIA_URL` | KYC asset status. Required in stage and prod |
| Config | `KYC_BUCKET` | `buyymart-prod-kyc` in prod. The task role is the only role that can `GetObject` |
| Config | `SIGNING_MODE` | `fake` in development only. `s3` in stage and prod |
| Config | `OWNED_SELLER_ID` | Required in stage and prod |
| Config | `OWNED_LEGAL_NAME` | Printed for BuyyMart’s own goods |
| Config | `OWNED_STATE` | A name from the GST table |
| Config | `OWNED_GSTIN` | Required in stage and prod |
| Config | `EVENTBRIDGE_BUS_NAME` | Empty means the worker logs the envelope and does not call AWS |

`OWNED_SELLER_AUTO_APPROVE` is read only when `NODE_ENV=development`.

Secrets live in Secrets Manager at `buyymart/{env}/seller-service`. They are not in the image and not in git. Stage and prod env files set `MEDIA_URL` to `http://media-service.catalogue.svc.cluster.local`, `SIGNING_MODE=s3`, and `KYC_BUCKET`. They do not contain a database password, `seller_app_dev`, `seller_migrator_dev`, or a development `DATA_KEY`.

---

## 14. Local run

Docker Compose for this service is Postgres 16, the API, and the worker. There is no Redis. Identity and media-service must already be running for a real upload. Tests inject introspect, the media read, and the signer, and do not start Compose.

Migrations run before the API starts. Host ports are 8097 for the API and 8098 for the worker, so this stack can run beside payment-service on 8095. `IDENTITY_URL` is `http://host.docker.internal:8080`. `MEDIA_URL` is `http://host.docker.internal:8085`. `SIGNING_MODE` is `fake`. The Compose `DATA_KEY` is a dev value and is not copied into `deploy/stage.env` or `deploy/prod.env`.

`POST /internal/events` on the worker accepts one envelope from section 10 only when `NODE_ENV=development`. It requires the service JWT and is not registered in staging or production.

---

## 15. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| `user.registered` for a seller | One `applied` marketplace row and the owner email. A second delivery does not insert another row |
| `user.registered` for a customer | Event id stored. No seller row |
| Owner submits a complete profile | `200` `kyc_pending`. GSTIN state code matches `address.state` |
| GSTIN whose first two digits are a different state | `400 invalid_body`. Status stays `applied` |
| Submit with no agreement | `409 profile_incomplete` and `missing` contains `agreement`. Status stays `applied` |
| Media asset not `ready` | `409 kyc_not_ready`. No document row |
| `seller_catalogue` attaches KYC | `403 forbidden` |
| Another seller reads this id | `404 not_found` |
| Trust approves | Status `approved`. One `seller.approved`. A second approve does not write a second outbox row |
| Trust rejects | Status `kyc_rejected`. No `seller.approved`. `fieldsToFix` is stored |
| `product.changed` with `status: live` while `approved` | Status `active`. No `seller.approved` |
| `product.changed` while `kyc_pending` | Status stays `kyc_pending` |
| Suspend an active seller | Status `suspended`. One `seller.suspended`. Reinstate does not publish an unhide event |
| Close | Status `closed`. One `seller.suspended` with `status: closed`. The row remains |
| Public read of an `applied` seller | `404 not_found` |
| Public read of an `approved` seller | Legal name, city, phone, GSTIN, and `rating: null`. No PAN and no `autoApprove` |
| Operational read | Includes `state`, `gstin`, `status`, and `autoApprove`. No bank ciphertext |
| Bank change | Live account unchanged until finance approves. A second pending change is `409 bank_pending` |
| Payout read by catalogue’s service JWT with no finance session | `401` or `403`, and the body has no account number |
| Postgres down | Profile write is `503 sellers_unavailable`. Ready is `503`. Live stays `200` |
| Worker publish | `published_at` is set. A second poll does not send it |
| `NODE_ENV=production` | `POST /internal/events` is not registered. `SIGNING_MODE=fake` refuses to start. Empty `DATA_KEY` refuses to start |
| `seller_app` | `INSERT` and `UPDATE` succeed. `DELETE` on `sellers` and `outbox` is rejected |

---

## 16. Outside this service

| Concern | Owner |
| --- | --- |
| Seller login, password, and staff invite | `identity-service`. Roles are `seller_owner`, `seller_catalogue`, and `seller_orders` |
| KYC bytes and the presigned PUT | `media-service`. This service only signs the review GET |
| Whether a product may go live | `catalogue-service`, using the operational read and `autoApprove` |
| Hiding products after suspension | `catalogue-service`, from `seller.suspended` |
| Stock edits for a seller who is not allowed | `inventory-service` |
| Place of supply and the invoice GSTIN | `order-service`, from `state` and `gstin` on the operational read |
| TCS, TDS, commission, and the bank transfer | `settlement-service`. It will read `kind` and, later, the payout account |
| The product page card | `customer-bff`, from the public route |
| Seller Centre screens | `seller-bff`. A seller who is not `approved` sees registration, KYC, and status only |
| KYC queue in Admin Console | `admin-bff` |

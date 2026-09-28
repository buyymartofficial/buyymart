# identity-service — implementation spec

**Namespace:** `identity`  
**Database:** Aurora PostgreSQL database `identity` on cluster `bm-identity`  
**Cache:** ElastiCache Redis, key prefix `id:`  
**Callers:** `customer-bff`, `seller-bff`, `admin-bff`  
**Calls:** `notification-service` for OTP and password-reset email  
**Publishes:** `UserRegistered`, `UserDeleted`  
**Consumes:** nothing  
**Product rules:** [modules/01-identity.md](../modules/01-identity.md)

This is the build document for the first service. Product behaviour is fixed in the module doc. This file is the stack, the tables, the Redis keys, and the request path from the browser to the row.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as the three web apps |
| Runtime | Node.js 22 LTS | Current LTS. One process for the API, one for the worker |
| HTTP | Fastify 5 | Schema validation, low overhead, `/health` is trivial |
| Validation | Zod, shared types in `buyymart-contracts` | Reject a bad body before it touches Redis or Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration. No ORM-generated drift |
| Redis | `ioredis` | OTP, rate limits, and the live session |
| Password hash | Argon2id via `@node-rs/argon2` | Memory-hard. Parameters below |
| Staff MFA | RFC 6238 TOTP, 6 digits, 30-second step, SHA-1, window ±1 | Works with standard authenticator apps |
| OTP | 6 digits from `crypto.randomInt(0, 1000000)` | Not `Math.random` |
| Ids | UUID v4 from `gen_random_uuid()` | Available on Aurora PostgreSQL 16 |
| Logs | `pino` JSON to stdout | Fluent Bit ships them. OTP, tokens, and TOTP secrets are redacted |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One login across BFF, identity, notification |
| Metrics | `prom-client` on `GET /metrics` | OTP send failures, login latency, Redis errors |
| Image | Distroless or `node:22-bookworm-slim`, non-root uid 1000 | Matches the pod standard in the architecture doc |
| Container port | 8080 | Service port 80 targets 8080 |

Argon2id parameters, from the OWASP password-storage guidance:

| Parameter | Value |
| --- | --- |
| Memory | 19456 KiB (19 MiB) |
| Iterations | 2 |
| Parallelism | 1 |
| Hash length | 32 bytes |
| Salt length | 16 bytes, random per password |

The encoded hash string (algorithm, salt, parameters, hash) is what is stored. A later parameter change rehashes on the next successful login.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | HTTP. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Reads the outbox and publishes to EventBridge. Minimum 2 replicas |

The API does not publish to EventBridge itself. It inserts an outbox row in the same transaction as the business row. The worker publishes and sets `published_at`.

---

## 3. Modules inside the service

Each module is a folder under `src/`. It owns its SQL and does not import another module’s tables except through a function that module exports.

| Module | Responsibility |
| --- | --- |
| `customer-otp` | Request and verify the SMS code. Create the customer on first success |
| `session` | Issue, introspect, slide, and revoke opaque sessions |
| `seller-auth` | Seller register, login, password reset, seller staff |
| `staff-auth` | Staff login, TOTP enrol, TOTP verify, role changes |
| `profile` | Customer name, email, and the address book |
| `consent` | Append-only consent rows |
| `deletion` | Customer account deletion and the ban list |
| `outbox` | Insert and publish `UserRegistered` and `UserDeleted` |
| `audit` | Staff role changes, bans, and KYC-unrelated auth events this service owns |

Seller KYC, GSTIN, and bank accounts are not modules here. `seller_id` is generated at owner registration and handed to `seller-service` on `UserRegistered`.

---

## 4. How a request is trusted

Browsers never call identity-service. They call a BFF. The BFF calls:

```text
http://identity-service.identity.svc.cluster.local
```

Headers on every internal call:

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `identity-service`. Lifetime 60 seconds. Signed with the key in Secrets Manager. A bad audience or expiry is 401 `service_unauthorized` |
| `X-Request-Id` | UUID. Echoed on the response and on every log line |
| `traceparent` | W3C trace context |
| `X-Session-Token` | Present only when the call is on behalf of a logged-in person. The raw opaque token. Identity hashes it and looks it up. Never log this header |

The service JWT proves the caller is a BFF. The session token proves which person. Both are required on `/me` routes. OTP request needs only the service JWT, because the person is not logged in yet.

JSON errors:

```json
{ "code": "otp_invalid", "message": "That code is wrong or expired.", "requestId": "…" }
```

`message` is safe to show in the UI. `code` is what the BFF branches on.

---

## 5. Customer OTP, end to end

```text
Storefront
  → POST /v1/auth/otp/request     customer-bff
       → POST /v1/customer/otp/request    identity-service
            1. Validate phone
            2. Refuse if locked or over the send cap
            3. Generate 6-digit code
            4. Store HMAC in Redis, TTL 5 minutes
            5. POST notification-service /v1/sms/otp
            6. If SMS fails, delete the Redis key and return 503
  → SMS arrives
Storefront
  → POST /v1/auth/otp/verify
       → POST /v1/customer/otp/verify
            1. Same error if the code is wrong, expired, or the phone is unknown to Redis
            2. On match, delete the OTP key
            3. Insert customer if this phone has no active row
            4. Insert consent rows
            5. Insert session in Postgres and Redis
            6. Insert outbox UserRegistered on first create
       ← { sessionToken, customer }
  → BFF sets cookie bm_session, HttpOnly, Secure, SameSite=Lax, Path=/
```

### Phone

Input is 10 digits, first digit 6–9, or already in `+91` form. Stored as E.164: `+91` plus those 10 digits. Anything else is `400 phone_invalid` with message “Enter a valid Indian mobile number.”

`phone_hash` is hex HMAC-SHA256 of the E.164 number with pepper `PHONE_HMAC_PEPPER`. The pepper is in Secrets Manager. The hash is stored for every customer, including after the phone column is cleared.

### OTP Redis value

Key `id:otp:{e164}` TTL 300 seconds. Value is a hash:

| Field | Value |
| --- | --- |
| `mac` | Hex HMAC-SHA256 of the 6-digit code with pepper `OTP_HMAC_PEPPER` |
| `attempts` | Integer, starts at 0 |

Compare with `crypto.timingSafeEqual`. On a mismatch, `HINCRBY` attempts. At 5, delete the OTP key and set `id:otp:lock:{e164}` for 900 seconds.

Send cap: `id:otp:send:{e164}` incremented on each request, TTL 900 seconds. At 3, set the same lock and do not send. A locked number gets `429 otp_locked`, message “Try again in 15 minutes.” The lock key’s remaining TTL is not returned, so the response does not help an attacker tune the wait.

Wrong, expired, and missing codes all return `401 otp_invalid`, message “That code is wrong or expired.” The handler waits until 150 ms have passed since the handler started, so those three cases take the same time.

### First verify

In one Postgres transaction:

1. If a row exists with this `phone` and `status = active`, use it.
2. If a row exists with this `phone_hash` and `status = banned`, return `403 phone_banned`. Do not create a session. Do not say “banned” in the message. Message: “This number cannot be used.”
3. If the latest `deleted` row for this `phone_hash` has `deleted_at` within 24 hours, return `429 phone_cooldown`, message “Try again tomorrow.” This stops delete-and-recreate SMS abuse. After 24 hours the same number may register again as a new customer id.
4. Otherwise insert `customers` with `status = active`.
5. Insert consent rows for the purposes in the request. `account` and `order_sms` must be `granted: true` or the transaction rolls back with `400 consent_required`.
6. Insert the session.
7. If step 4 inserted a customer, insert outbox `UserRegistered`.

Then set Redis `id:session:{tokenHash}`.

The raw session token is 32 bytes from `crypto.randomBytes`, base64url. Only `sha256(token)` is stored. The raw token is returned once.

### Cookie

The BFF sets the cookie. Identity does not. Customer cookie lifetime matches the session absolute cap (90 days) and the Redis TTL is what actually expires the login (sliding 30 days). See sessions.

### Environments and the SMS

| Environment | notification-service behaviour identity relies on |
| --- | --- |
| Dev | Does not send. Logs the code. Identity still calls it. |
| Stage | Sends only to an allow-list. Other numbers return a successful fake send so the flow can be tested without a real SMS. |
| Prod | DLT template send. Identity logs the provider message id and the last two digits of the phone. Never the code. |

Timeout on the notification call: 2 seconds. One retry on a network error. No retry on 4xx. If both attempts fail, delete `id:otp:{e164}` and return `503 otp_delivery_failed`.

---

## 6. Sessions

| Family | Absolute life | Idle / sliding | Issued only after |
| --- | --- | --- | --- |
| `customer` | 90 days from `issued_at` | Redis TTL and `expires_at` slide to now + 30 days, never past the absolute cap | Correct OTP |
| `seller` | 12 hours from `issued_at` | Slide to now + 2 hours, never past 12 hours | Correct password |
| `staff` | 8 hours from `issued_at` | No slide | Password and TOTP |

Introspect (`POST /v1/sessions/introspect`):

1. Hash the token.
2. `GET id:session:{hash}`. Miss means `401 session_invalid`, even if Postgres still has a row. A Redis flush logs everyone out.
3. If Postgres `revoked_at` is set, delete the Redis key and return `401 session_invalid`.
4. If `now` is past `absolute_expires_at`, revoke and return `401 session_expired`.
5. For customer and seller, extend `expires_at` and the Redis TTL as in the table, in one update.
6. Return `{ family, subjectId, role, sellerId | null, sessionId }`.

The BFF calls introspect on every authenticated request and caches nothing. Identity p99 target for introspect is 20 ms with a Redis hit.

Revoke deletes the Redis key and sets `revoked_at`. Logout revokes the current session. Password change and account deletion revoke every session for that subject.

`GET /v1/sessions` for the current subject lists Postgres rows (id, issued_at, last_seen_at, user_agent) and does not include token hashes.

---

## 7. Seller login, end to end

Registration (`POST /v1/seller/register`) creates the owner only.

1. Email is normalised: trim, lowercase.
2. Password must be 12–128 characters. No composition rules beyond length. Check it is not the email.
3. Argon2id hash.
4. Insert `seller_users` with `role = seller_owner`, `seller_id = gen_random_uuid()`, `status = active`.
5. Insert consent `account` granted, `policyVersion` from the request.
6. Outbox `UserRegistered` with `family: seller`, `sellerId`, `email`.
7. Return a seller session. Seller Centre then sends the user to KYC. The seller profile row is created by `seller-service` when it consumes the event. Until that consumer runs, Seller Centre shows “Setting up your account” and retries.

Login compares the hash with Argon2id verify. Unknown email and wrong password are both `401 credentials_invalid`, message “Email or password is wrong.”, and both take at least 150 ms. A `disabled` user gets the same response so the endpoint is not a user-enumeration oracle.

Seller staff are created by the owner: `POST /v1/seller/staff` with email, temporary password, and role `seller_catalogue` or `seller_orders`. The temporary password is marked `must_change_password`. The next login returns `403 password_change_required` plus a 10-minute `passwordChangeToken` (same shape as a reset token) and does not issue a session.

### Password reset

1. `POST /v1/seller/password/forgot` with email. Always `202` and the same message, whether or not the email exists.
2. If the email exists, store `id:reset:{sha256(token)}` for 30 minutes with the user id, and call notification-service to email the link `https://seller.buyymart.com/reset#token=…`. The token is 32 random bytes. It is not stored in Postgres.
3. `POST /v1/seller/password/reset` with the token and the new password. `GETDEL` the Redis key so it is single use. Update `password_hash`, set `password_changed_at`, revoke all seller sessions for that user.

Staff reset is the same path under `/v1/staff/password/*` and the link host is `https://admin.buyymart.com`.

---

## 8. Staff login and MFA, end to end

Staff are not self-serve. `POST /v1/staff` is allowed only when the caller’s introspected role is `admin` or `super_admin`. Email must end with `@` plus `STAFF_EMAIL_DOMAIN`. The creator’s action is an `auth_audit` row.

First login:

1. `POST /v1/staff/login` with email and password. On success, if `totp_confirmed_at` is null, return `{ mfaEnrollmentToken, otpauthUrl }`. The token is Redis `id:mfa-enroll:{hash}` for 10 minutes. `otpauthUrl` is `otpauth://totp/BuyyMart:{email}?secret=…&issuer=BuyyMart`. The secret is generated now, encrypted, and stored. It is not confirmed until the next step.
2. The UI shows the QR from `otpauthUrl`. `POST /v1/staff/mfa/confirm` with the enrol token and a 6-digit code. On match, set `totp_confirmed_at` and issue a staff session.
3. Later logins: password success returns `{ mfaToken }` in Redis `id:mfa:{hash}` for 5 minutes. `POST /v1/staff/mfa/verify` with that token and the TOTP code issues the session. A wrong code increments attempts. At 5, delete the mfa token and require the password again.

A staff user with `totp_confirmed_at` null cannot introspect as staff. The enrol token is not a session. Admin BFF rejects any session whose family is not `staff`.

TOTP secret at rest: AES-256-GCM. Key `TOTP_ENCRYPTION_KEY` is 32 bytes in Secrets Manager. The column stores `nonce || ciphertext || tag`. Decrypt only inside the verify function. Logs never print the secret or the code.

Changing a staff role (`PATCH /v1/staff/:id`) requires `admin` or `super_admin`. A person cannot change their own role. `super_admin` can only be granted by an existing `super_admin`. The row in `auth_audit` stores before, after, and a reason of at least 10 characters. All sessions for that staff user are revoked.

---

## 9. Profile, consent, deletion

### Profile

`GET /v1/customer/me` returns id, name, email, phone (E.164), status, and default address id. `PATCH` may set `name` (1–80 chars) and `email`. Email is not unique across customers in version 1, because it is not a login. It is checked as a basic email shape.

### Addresses

Owned here because deletion must remove them, and no other service has an address book. Checkout copies a snapshot into the order. Later edits do not change old orders.

| Rule | Detail |
| --- | --- |
| Maximum | 10 addresses per customer |
| Pin | 6 digits, first digit not 0 |
| Default | Exactly one `is_default` when any address exists. Setting a new default clears the old one in the same transaction |
| Phone on the address | Indian mobile, may differ from the login phone. This is the delivery contact |

### Consent changes

`POST /v1/customer/consent` appends a row. It does not update the old row. The current value of a purpose is the latest row for that subject and purpose. Withdrawing `marketing` does not delete anything else. Withdrawing `account` does not delete by itself. It returns `409 use_delete_endpoint`. Deletion is only `DELETE /v1/customer/me`.

### Deletion

`DELETE /v1/customer/me` body `{ "confirm": true }`. Without `confirm: true`, `400 confirm_required`.

The handler tells the BFF, before this call, to show that paid and later orders will finish. Identity does not cancel orders. It does not call order-service.

In one transaction:

1. Revoke all sessions for this customer (Postgres `revoked_at`, and delete matching Redis keys).
2. Delete address rows.
3. Set `name` and `email` to null, `phone` to null, `status = deleted`, `deleted_at = now()`.
4. Keep `phone_hash` and the consent log.
5. Insert outbox `UserDeleted` with `customerId` and `deletedAt`. Do not include the phone.

`UserDeleted` is how other services drop profile copies. They must not delete orders, payments, invoices, or ledger lines.

Ban is separate: `POST /v1/customer/:id/ban` for staff role `trust` or above, with a reason. Sets `status = banned`, revokes sessions, writes `auth_audit`. A banned hash cannot register again.

---

## 10. HTTP API

Base path inside the cluster: `/v1`. All routes require the service JWT unless noted.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| POST | `/customer/otp/request` | No | `202 { "accepted": true }` |
| POST | `/customer/otp/verify` | No | `200 { sessionToken, customer }` |
| POST | `/sessions/introspect` | Token in body | `200 { family, subjectId, role, sellerId, sessionId }` |
| POST | `/sessions/revoke` | Current | `204` |
| GET | `/sessions` | Current | `200 { sessions: [...] }` |
| GET | `/customer/me` | Customer | `200` customer |
| PATCH | `/customer/me` | Customer | `200` customer |
| GET | `/customer/addresses` | Customer | `200` |
| POST | `/customer/addresses` | Customer | `201` |
| PATCH | `/customer/addresses/:id` | Customer | `200` |
| DELETE | `/customer/addresses/:id` | Customer | `204` |
| POST | `/customer/consent` | Customer | `201` |
| DELETE | `/customer/me` | Customer | `202 { "status": "deleted" }` |
| POST | `/seller/register` | No | `201` plus session |
| POST | `/seller/login` | No | `200` plus session |
| POST | `/seller/password/forgot` | No | `202` |
| POST | `/seller/password/reset` | Reset token | `204` |
| GET | `/seller/me` | Seller | `200` |
| POST | `/seller/staff` | `seller_owner` | `201` |
| POST | `/staff/login` | No | `200` mfa token or enrol token |
| POST | `/staff/mfa/confirm` | Enrol token | `200` session |
| POST | `/staff/mfa/verify` | MFA token | `200` session |
| POST | `/staff` | `admin` or `super_admin` | `201` |
| PATCH | `/staff/:id` | `admin` or `super_admin` | `200` |
| POST | `/customer/:id/ban` | `trust`, `admin`, or `super_admin` | `200` |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only if Postgres and Redis answer `PING` / `SELECT 1` |
| GET | `/metrics` | Cluster scrape only | Prometheus text |

`/health/*` is the only unauthenticated surface, and the network policy does not expose it outside the cluster and the kubelet.

### OTP request body

```json
{
  "phone": "9876543210",
  "channel": "web"
}
```

### OTP verify body

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

`code` must match `^[0-9]{6}$`. It is redacted before logging.

---

## 11. Redis

| Key | TTL | Contents |
| --- | --- | --- |
| `id:otp:{e164}` | 300 s | `mac`, `attempts` |
| `id:otp:send:{e164}` | 900 s | Integer send count |
| `id:otp:lock:{e164}` | 900 s | `1` |
| `id:session:{sha256}` | Remaining session life | `{ sessionId, family, subjectId, role, sellerId }` |
| `id:reset:{sha256}` | 1800 s | `{ userId, family }` |
| `id:mfa:{sha256}` | 300 s | `{ staffId, attempts }` |
| `id:mfa-enroll:{sha256}` | 600 s | `{ staffId, attempts }` |

If Redis is down, OTP request, login, and introspect return `503 dependency_unavailable`. They do not fall open. Postgres remains the record of customers and of revoked sessions. It is not a second place to accept a token after Redis has lost it.

---

## 12. PostgreSQL schema

Database `identity`. The application role `identity_app` can `SELECT`, `INSERT`, `UPDATE` on these tables and cannot `DROP`, `TRUNCATE`, or alter schema. The migration role `identity_migrator` runs the files in `migrations/` as a Helm pre-upgrade job and is not the runtime role.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TYPE subject_family AS ENUM ('customer', 'seller', 'staff');
CREATE TYPE customer_status AS ENUM ('active', 'deleted', 'banned');
CREATE TYPE account_status AS ENUM ('active', 'disabled');
CREATE TYPE seller_role AS ENUM ('seller_owner', 'seller_catalogue', 'seller_orders');
CREATE TYPE staff_role AS ENUM ('support', 'catalogue', 'trust', 'finance', 'admin', 'super_admin');
CREATE TYPE consent_purpose AS ENUM ('account', 'order_sms', 'marketing');
CREATE TYPE consent_channel AS ENUM ('web', 'android', 'ios');

CREATE TABLE customers (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  phone         text,
  phone_hash    text NOT NULL,
  name          text,
  email         citext,
  status        customer_status NOT NULL DEFAULT 'active',
  deleted_at    timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT customers_phone_e164 CHECK (
    phone IS NULL OR phone ~ '^\+91[6-9][0-9]{9}$'
  ),
  CONSTRAINT customers_name_len CHECK (
    name IS NULL OR char_length(name) BETWEEN 1 AND 80
  ),
  CONSTRAINT customers_deleted_pair CHECK (
    (status = 'active' AND phone IS NOT NULL AND deleted_at IS NULL)
    OR (status = 'deleted' AND phone IS NULL AND deleted_at IS NOT NULL)
    OR (status = 'banned' AND deleted_at IS NULL)
  )
);

CREATE UNIQUE INDEX customers_phone_active_uidx
  ON customers (phone) WHERE status = 'active';

CREATE UNIQUE INDEX customers_phone_hash_banned_uidx
  ON customers (phone_hash) WHERE status = 'banned';

CREATE INDEX customers_phone_hash_idx ON customers (phone_hash);

CREATE TABLE customer_addresses (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id  uuid NOT NULL REFERENCES customers (id),
  contact_name text NOT NULL,
  phone        text NOT NULL,
  line1        text NOT NULL,
  line2        text,
  landmark     text,
  city         text NOT NULL,
  state        text NOT NULL,
  pin          text NOT NULL,
  is_default   boolean NOT NULL DEFAULT false,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT address_phone CHECK (phone ~ '^\+91[6-9][0-9]{9}$'),
  CONSTRAINT address_pin CHECK (pin ~ '^[1-9][0-9]{5}$'),
  CONSTRAINT address_line1 CHECK (char_length(line1) BETWEEN 1 AND 200)
);

CREATE INDEX customer_addresses_customer_idx
  ON customer_addresses (customer_id);

CREATE UNIQUE INDEX customer_addresses_one_default_uidx
  ON customer_addresses (customer_id) WHERE is_default;

CREATE TABLE seller_users (
  id                     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id              uuid NOT NULL,
  email                  citext NOT NULL UNIQUE,
  password_hash          text NOT NULL,
  role                   seller_role NOT NULL,
  status                 account_status NOT NULL DEFAULT 'active',
  must_change_password   boolean NOT NULL DEFAULT false,
  password_changed_at    timestamptz,
  created_at             timestamptz NOT NULL DEFAULT now(),
  updated_at             timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX seller_users_seller_idx ON seller_users (seller_id);

CREATE TABLE staff_users (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email               citext NOT NULL UNIQUE,
  password_hash       text NOT NULL,
  totp_secret         bytea,
  totp_confirmed_at   timestamptz,
  role                staff_role NOT NULL,
  status              account_status NOT NULL DEFAULT 'active',
  must_change_password boolean NOT NULL DEFAULT true,
  password_changed_at timestamptz,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT staff_totp_pair CHECK (
    (totp_confirmed_at IS NULL) OR (totp_secret IS NOT NULL)
  )
);

CREATE TABLE sessions (
  id                   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  family               subject_family NOT NULL,
  subject_id           uuid NOT NULL,
  token_hash           text NOT NULL UNIQUE,
  role                 text NOT NULL,
  seller_id            uuid,
  issued_at            timestamptz NOT NULL DEFAULT now(),
  expires_at           timestamptz NOT NULL,
  absolute_expires_at  timestamptz NOT NULL,
  last_seen_at         timestamptz NOT NULL DEFAULT now(),
  revoked_at           timestamptz,
  revoke_reason        text,
  user_agent           text,
  ip                   inet,
  CONSTRAINT sessions_window CHECK (expires_at <= absolute_expires_at),
  CONSTRAINT sessions_seller_scope CHECK (
    (family = 'seller' AND seller_id IS NOT NULL)
    OR (family <> 'seller' AND seller_id IS NULL)
  )
);

CREATE INDEX sessions_subject_idx
  ON sessions (family, subject_id) WHERE revoked_at IS NULL;

CREATE TABLE consent_events (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  family          subject_family NOT NULL,
  subject_id      uuid NOT NULL,
  purpose         consent_purpose NOT NULL,
  policy_version  text NOT NULL,
  granted         boolean NOT NULL,
  channel         consent_channel NOT NULL,
  request_id      text NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX consent_events_latest_idx
  ON consent_events (family, subject_id, purpose, created_at DESC);

CREATE TABLE auth_audit (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  at            timestamptz NOT NULL DEFAULT now(),
  actor_id      uuid NOT NULL,
  actor_role    text NOT NULL,
  action        text NOT NULL,
  subject_type  text NOT NULL,
  subject_id    uuid NOT NULL,
  reason        text NOT NULL,
  before        jsonb,
  after         jsonb,
  request_id    text NOT NULL,
  CONSTRAINT auth_audit_reason CHECK (char_length(reason) >= 10)
);

CREATE INDEX auth_audit_subject_idx ON auth_audit (subject_type, subject_id, at DESC);
CREATE INDEX auth_audit_actor_idx ON auth_audit (actor_id, at DESC);

CREATE TABLE outbox (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  type          text NOT NULL,
  payload       jsonb NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  published_at  timestamptz
);

CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;
```

`updated_at` is set by the application on every update. There is no trigger in version 1, so a forgotten update is a code bug the tests catch.

Addresses are removed with `DELETE`, not soft-deleted. A deleted customer’s address book must be gone. Consent, audit, and outbox rows are never updated except `outbox.published_at`.

### Event payload

`UserRegistered`:

```json
{
  "family": "customer",
  "subjectId": "…",
  "sellerId": null,
  "at": "2026-09-28T15:30:00Z"
}
```

For a seller owner, `family` is `seller` and `sellerId` is set. Email is included for the seller so `seller-service` can show it on the application. Email is not included for a customer.

`UserDeleted`:

```json
{
  "family": "customer",
  "subjectId": "…",
  "at": "2026-09-28T15:30:00Z"
}
```

The envelope around that payload is the platform event (`id`, `type`, `source`, `time`, `traceId`, `data`) from the architecture doc. `source` is `identity-service`. `type` is `user.registered` or `user.deleted`.

---

## 13. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | App role connection string |
| Secret | `REDIS_URL` | TLS URL |
| Secret | `SERVICE_JWT_KEY` | HMAC key for verifying caller JWTs |
| Secret | `OTP_HMAC_PEPPER` | 32 bytes |
| Secret | `PHONE_HMAC_PEPPER` | 32 bytes, distinct from the OTP pepper |
| Secret | `TOTP_ENCRYPTION_KEY` | 32 bytes |
| Config | `STAFF_EMAIL_DOMAIN` | Set before the first staff user. Not guessed in code |
| Config | `NOTIFICATION_URL` | `http://notification-service.engagement.svc.cluster.local` |
| Config | `POLICY_VERSION_MIN` | Oldest privacy version still accepted at OTP verify |

Secrets live in Secrets Manager at `buyymart/{env}/identity-service` and arrive as a Kubernetes Secret through External Secrets. They are not in the image and not in git.

---

## 14. Local run

Docker Compose for this service is Postgres 16, Redis 7, and the API. Notification is a stub that prints the OTP. Migrations run before the API starts. A seed command creates one staff user only when `SEED_STAFF_EMAIL` and `SEED_STAFF_PASSWORD` are set, and only when `NODE_ENV=development`. Those variables are absent in stage and prod.

---

## 15. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Valid 10-digit phone, then the code | Customer row, session in Postgres and Redis, cookie from the BFF |
| Same phone again | Same customer id, no second `UserRegistered` |
| Five wrong codes | Lock, sixth attempt is `otp_locked` even with the right code |
| Fourth send inside 15 minutes | `otp_locked`, no SMS |
| Expired code | `otp_invalid` |
| Notification 500 | No OTP key left in Redis, `503` |
| Banned hash | `phone_banned`, no session |
| Delete, then same phone inside 24 hours | `phone_cooldown` |
| Delete, then same phone after 24 hours | New customer id |
| Redis flushed | Old session token is `session_invalid` |
| Seller wrong password and unknown email | Same status, same message |
| Staff password without TOTP | No session that introspect accepts |
| Staff changes own role | `403` |
| Address default flip | Only one `is_default` |
| Outbox row | Worker sets `published_at` and a second poll does not send it again |

---

## 16. Outside this service

| Concern | Owner |
| --- | --- |
| Seller legal name, GSTIN, PAN, bank, KYC files | `seller-service` |
| SMS template and DLT | `notification-service` |
| Order history and the checkout consent checkbox | `order-service` |
| Banners, cookies copy on the marketing site | The web app |
| Rate limit at the edge of the OTP route | WAF, in addition to the Redis lock here |

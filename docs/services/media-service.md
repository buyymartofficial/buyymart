# media-service — implementation spec

**Namespace:** `catalogue`  
**Database:** Aurora PostgreSQL database `media` on cluster `bm-commerce`  
**Object store:** S3 bucket `buyymart-prod-images` for product images. S3 bucket `buyymart-prod-kyc` for KYC documents.  
**Cache:** none.  
**Callers:** `seller-bff`, `admin-bff`  
**Calls:** `seller-service` to read approval and suspension. `identity-service` to introspect the session.  
**Publishes:** `MediaReady`  
**Consumes:** S3 `ObjectCreated` for the two buckets above. It does not consume catalogue events.  
**Product rules:** [modules/04-media.md](../modules/04-media.md)

This is the build document for upload permission and image processing. Catalogue stores the image id on the variant. This service stores the object keys and whether the upload became ready. The API never accepts the file body. The browser uploads straight to S3 with a presigned PUT.

Invoice PDFs are not this service. `order-service` and `settlement-service` write `buyymart-prod-invoices`.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as catalogue-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as catalogue |
| Validation | Zod | Reject a bad body before it touches Postgres or S3 |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Objects | AWS SDK v3 S3 client | Presigned PUT. The worker reads product originals and writes renditions |
| Images | `sharp` | Re-encode to WebP and drop metadata. The package’s prebuild supplies libvips |
| Redis | none | No cache and no session store in this service |
| Ids | UUID v4 from `gen_random_uuid()` | The same id catalogue stores as `image_id` |
| Logs | `pino` JSON to stdout | Log the object key and seller id. Do not log bytes, EXIF, or the presigned query string |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One upload across the BFF, this API, and the worker |
| Metrics | `prom-client` on `GET /metrics` | Presign failures, rejected uploads, outbox lag |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as catalogue |
| Container port | 8080 | Service port 80 targets 8080 |

The maximum upload is 8 MiB, which is 8388608 bytes. The API does not accept a larger declared size.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | HTTP. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Validates new objects, writes product renditions, publishes the outbox |

The API does not publish to EventBridge itself. A status change and its outbox row commit in one transaction. The worker publishes and sets `published_at`.

The task role can `PutObject` and `GetObject` on `buyymart-prod-images`. It can `PutObject` on `buyymart-prod-kyc` so it can sign an upload. It cannot `GetObject` on the KYC bucket. `seller-service` is the only role that reads KYC bytes.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `upload` | Decide whether this seller may upload, and return a presigned PUT |
| `product-image` | Check the original, strip metadata, write the page and card renditions |
| `kyc` | Sign a private upload and record that the object exists. Do not open the bytes |
| `outbox` | `media.ready` |

This service does not store the product, the variant, the seller legal name, or the GSTIN. It does not attach an image to a variant. Seller Centre calls catalogue with the id this service returned.

---

## 4. How a request is trusted

Browsers never call media-service. A BFF calls:

```text
http://media-service.catalogue.svc.cluster.local
```

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `media-service`. Lifetime at most 60 seconds. HS256. A bad audience or expiry is `401 service_unauthorized` |
| `X-Request-Id` | UUID. Echoed on the response and on every log line |
| `traceparent` | W3C trace context |
| `X-Session-Token` | Present on upload and image reads. The BFF has already introspected it with identity. This service checks the family and role again by calling identity introspect. It does not trust a role claim in the body |

Product uploads need a session whose family is `seller` and whose role is `seller_owner` or `seller_catalogue`. `seller_orders` is `403 forbidden`. A staff session is `403 forbidden`. Sellers upload their own images.

KYC uploads need `seller_owner`. `seller_catalogue` is `403 forbidden` on a KYC purpose. Trust staff do not get a KYC download URL from this service. `seller-service` issues the 60-second presigned GET.

A read of another seller’s id is `404 not_found`, not `403`.

JSON errors:

```json
{ "code": "too_large", "message": "That file is too large.", "requestId": "…" }
```

`message` is safe to show in Seller Centre. `code` is what the BFF branches on.

---

## 5. Product upload

The seller asks for an upload before the browser sends bytes.

| Check | Rule |
| --- | --- |
| Seller | Status `approved` or `active`, and not `suspended` or `closed`. Otherwise `403 seller_not_approved` |
| Content type | `image/jpeg`, `image/png`, or `image/webp`. Anything else is `400 content_type_invalid` |
| Size | Integer from 1 through 8388608. Larger is `400 too_large` |
| Key | `products/{id}/original`. The browser filename is ignored |

The presigned PUT lasts 10 minutes. The signature fixes `Content-Type` and `Content-Length` to the declared values, so a different type or a larger body is rejected by S3.

The response is `201`:

```json
{
  "id": "…",
  "uploadUrl": "https://…",
  "method": "PUT",
  "expiresAt": "2026-10-01T12:10:00Z",
  "headers": { "content-type": "image/jpeg" }
}
```

`id` is the value catalogue stores as `image_id`. The row is `pending`. Catalogue may attach that id immediately. Pending does not count as a ready image.

Until `SELLER_URL` is set, only `OWNED_SELLER_ID` may request a product upload, and that id is treated as approved and not suspended. Any other seller is `403 seller_not_approved`. Stage and prod set `SELLER_URL`, and then this shortcut is off. A failed seller-service call is `503 dependency_unavailable` and no row is inserted.

If signing the URL fails, the insert is rolled back and the response is `503 dependency_unavailable`.

---

## 6. Product processing

S3 sends `ObjectCreated` for `products/{id}/original`. The worker loads the object only when the key matches a `pending` product row.

| Check | Result |
| --- | --- |
| Object missing, or larger than 8388608 bytes | `rejected`, code `too_large` or `missing_object` |
| Magic bytes are not JPEG, PNG, or WebP, or they disagree with the declared type | `rejected`, code `not_an_image` |
| `sharp` cannot decode the image | `rejected`, code `not_an_image` |

Magic bytes are `FF D8` for JPEG, `89 50 4E 47` for PNG, and `RIFF` plus `WEBP` for WebP.

On success the worker writes two WebP objects and drops metadata, including GPS EXIF:

| Rendition | Key | Width |
| --- | --- | --- |
| Product page | `products/{id}/page.webp` | 1200 px, aspect ratio kept, no enlargement |
| Listing card | `products/{id}/card.webp` | 400 px, aspect ratio kept, no enlargement |

WebP quality is 80. The original stays in the bucket and is not given a CloudFront behavior. Customers read only the two renditions, and only through CloudFront. The bucket is not a public website.

The row becomes `ready` in the same transaction as one outbox row. A second `ObjectCreated` for a row that is already `ready` or `rejected` does not write another outbox row and does not overwrite the renditions.

A rejected upload never becomes `ready`. Seller Centre reads the row and shows “Upload failed.” The seller asks for a new id and tries again.

---

## 7. KYC upload

KYC uses `buyymart-prod-kyc`. The bucket blocks public access and uses its own KMS key. Objects are not re-encoded, not turned into WebP, and not served from the image CDN.

| Field | Rule |
| --- | --- |
| `document` | `pan`, `gst_certificate`, or `bank_proof` |
| Content type | `application/pdf`, `image/jpeg`, or `image/png` |
| Size | Same 8388608 byte cap |
| Key | `kyc/{sellerId}/{id}` |

The caller is `seller_owner` and must pass the same seller-status check as a product upload. The presigned PUT lasts 10 minutes and pins content type and length.

The worker receives `ObjectCreated` and does not download the object. It records `ready` when the event’s size is within the cap and the key matches a pending KYC row. A size above the cap, or a key this service did not issue, becomes `rejected` with code `too_large` or `missing_object`. No KYC event is published. `seller-service` reads the object and stores the key on its own KYC row. A reviewer URL is a presigned GET of 60 seconds from `seller-service`, not from this service.

This service never returns a KYC download URL.

---

## 8. Events

The wire `type` is lowercase, matching catalogue:

| Name | `type` | When |
| --- | --- | --- |
| `MediaReady` | `media.ready` | A product original has been accepted or rejected |

KYC does not publish this event. Catalogue marks a variant image `ready` only when `accepted` is true. A rejected upload stays pending on the variant forever unless the seller attaches a different id.

`media.ready` payload when the renditions exist:

```json
{
  "imageId": "…",
  "accepted": true,
  "status": "ready"
}
```

`media.ready` payload when the original failed the checks:

```json
{
  "imageId": "…",
  "accepted": false,
  "status": "rejected"
}
```

The envelope is `{ id, type, source, time, traceId, data }` with `source` `media-service`. `data` is the payload above. One outbox row per asset. The payload has no pixels, no seller document, and no presigned URL.

Catalogue already accepts `type` `media.ready` or `MediaReady`, and treats `accepted: false` or `status: "rejected"` as not ready. This service sends `media.ready` and always sets both `accepted` and `status`.

---

## 9. HTTP API

Base path `/v1`. Every route requires the service JWT. Session rules are in the table.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| POST | `/v1/uploads` | Seller owner or catalogue for `product`. Seller owner for `kyc` | `201` presigned PUT |
| GET | `/v1/images/:id` | Owning seller owner or catalogue. KYC rows are owner-only | `200` status. Ready product rows include CDN URLs |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | Cluster scrape only | Prometheus text |

### Upload body

Product:

```json
{ "purpose": "product", "contentType": "image/jpeg", "byteSize": 245000 }
```

KYC:

```json
{ "purpose": "kyc", "document": "pan", "contentType": "application/pdf", "byteSize": 80000 }
```

`document` is required for `kyc` and forbidden for `product`. A missing or extra field is `400 invalid_body`.

### Image read

Pending:

```json
{ "id": "…", "purpose": "product", "status": "pending" }
```

Ready product:

```json
{
  "id": "…",
  "purpose": "product",
  "status": "ready",
  "pageUrl": "https://images.buyymart.com/products/…/page.webp",
  "cardUrl": "https://images.buyymart.com/products/…/card.webp"
}
```

The host is `IMAGES_CDN_HOST`. When that variable is empty, `pageUrl` and `cardUrl` are omitted and the body still says `ready`. Stage and prod set the CloudFront host. The original key is never returned.

Rejected:

```json
{ "id": "…", "purpose": "product", "status": "rejected", "code": "not_an_image", "message": "Upload failed." }
```

A KYC read returns `id`, `purpose`, `document`, and `status` only. A rejected KYC row uses the same “Upload failed.” message.

---

## 10. PostgreSQL schema

Database `media`. The application role `media_app` can `SELECT`, `INSERT`, and `UPDATE` on these tables. It cannot `DELETE`, `DROP`, `TRUNCATE`, or alter schema. The migration role `media_migrator` runs `migrations/` and is not the runtime role.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE media_purpose AS ENUM ('product', 'kyc');
CREATE TYPE media_status AS ENUM ('pending', 'ready', 'rejected');
CREATE TYPE kyc_document AS ENUM ('pan', 'gst_certificate', 'bank_proof');

CREATE TABLE assets (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id        uuid NOT NULL,
  purpose          media_purpose NOT NULL,
  document         kyc_document,
  status           media_status NOT NULL DEFAULT 'pending',
  content_type     text NOT NULL,
  declared_bytes   integer NOT NULL,
  original_key     text NOT NULL,
  page_key         text,
  card_key         text,
  rejection_code   text,
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT assets_bytes CHECK (declared_bytes BETWEEN 1 AND 8388608),
  CONSTRAINT assets_key CHECK (char_length(original_key) BETWEEN 1 AND 200),
  CONSTRAINT assets_kyc_document CHECK (
    (purpose = 'product' AND document IS NULL)
    OR (purpose = 'kyc' AND document IS NOT NULL)
  ),
  CONSTRAINT assets_renditions CHECK (
    (status <> 'ready' AND page_key IS NULL AND card_key IS NULL)
    OR (purpose = 'kyc' AND page_key IS NULL AND card_key IS NULL)
    OR (purpose = 'product' AND status = 'ready' AND page_key IS NOT NULL AND card_key IS NOT NULL)
  ),
  UNIQUE (original_key)
);

CREATE INDEX assets_seller_idx ON assets (seller_id, purpose, status);

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

There is no hard delete in version 1. Hiding a product does not delete these rows. A later job may delete a product object only when catalogue reports no live or draft reference and no order snapshot still points at the id. That job is not this build. KYC objects are not deleted here.

---

## 11. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `media_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Used only by the migration job. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `media-service` |
| Secret | `S3_ACCESS_KEY_ID` | Local MinIO only. Stage and prod use the task role and leave this unset |
| Secret | `S3_SECRET_ACCESS_KEY` | Local MinIO only |
| Config | `IDENTITY_URL` | Introspect seller sessions |
| Config | `SELLER_URL` | Empty until seller-service is deployed |
| Config | `OWNED_SELLER_ID` | BuyyMart’s internal seller UUID |
| Config | `IMAGES_BUCKET` | `buyymart-prod-images` in prod |
| Config | `KYC_BUCKET` | `buyymart-prod-kyc` in prod |
| Config | `IMAGES_CDN_HOST` | `images.buyymart.com`. Empty in local dev |
| Config | `S3_ENDPOINT` | Empty in stage and prod. Local MinIO URL for the worker |
| Config | `S3_PUBLIC_ENDPOINT` | Empty in stage and prod. Browser-reachable MinIO URL used when signing |
| Config | `EVENTBRIDGE_BUS_NAME` | Bus name. Empty means the worker logs the envelope and does not call AWS |

Secrets live in Secrets Manager at `buyymart/{env}/media-service`. They are not in the image and not in git.

Stage and prod do not set `S3_ENDPOINT`, `S3_PUBLIC_ENDPOINT`, `S3_ACCESS_KEY_ID`, or `S3_SECRET_ACCESS_KEY`. The SDK then uses the regional S3 endpoint and the task role.

---

## 12. Local run

Docker Compose for this service is Postgres 16, MinIO, the API, and the worker. Identity must already be running, because upload and image reads introspect. Catalogue must be running only when the tester wants to see a variant image leave `pending`.

Migrations run before the API starts. MinIO is the local S3 API. The API signs with `S3_PUBLIC_ENDPOINT` so the browser on the host can PUT. The worker reads and writes through `S3_ENDPOINT` on the Compose network. Host ports are 8085 for the API and 9000 for MinIO, so this stack can run beside identity on 8080 and catalogue on 8083.

`ObjectCreated` in dev can be posted to the worker’s test hook `POST /internal/objects` only when `NODE_ENV=development`. The body is `{ "key": "products/{id}/original" }`. The hook runs the same checks as a real event. Stage and prod receive events from S3. An asset does not become `ready` because the presign was issued.

The hook requires the service JWT. It is not registered when `NODE_ENV` is `staging` or `production`.

---

## 13. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| JPEG under 8 MiB for an approved seller | `201`, row `pending`, PUT URL expires in 10 minutes |
| PNG and WebP | `201` |
| `image/gif` or `text/plain` | `400 content_type_invalid`, no row |
| `byteSize` 8388609 | `400 too_large`, no row |
| `seller_orders` or a staff session | `403 forbidden` |
| Another seller’s id on GET | `404 not_found` |
| Seller not approved, or `SELLER_URL` empty and the seller is not `OWNED_SELLER_ID` | `403 seller_not_approved`, no row |
| Seller-service timeout | `503 dependency_unavailable`, no row |
| Original is a JPEG, worker runs | `ready`, `page.webp` width 1200, `card.webp` width 400, no EXIF in the WebP, one `media.ready` with `accepted: true` |
| Original is not an image, or `sharp` throws | `rejected`, `not_an_image`, `media.ready` with `accepted: false` and `status: "rejected"`. Catalogue would leave the variant image pending |
| Second `ObjectCreated` for a ready id | No second outbox row, rendition keys unchanged |
| KYC PDF by `seller_owner` | `201`, key under `kyc/{sellerId}/`. Event size within the cap marks `ready`. No `media.ready` row. Response has no download URL |
| `seller_catalogue` asks for KYC | `403 forbidden` |
| Worker KYC path | Does not call `GetObject` on the KYC bucket |
| Outbox | Worker sets `published_at`. A second poll does not send it again |
| `media_app` | `DELETE` fails. `UPDATE` of `status` succeeds |
| Stage and prod env files | Do not set `S3_ENDPOINT` or MinIO keys |
| API process | Refuses to start when `DATABASE_MIGRATOR_URL` is set |

---

## 14. Outside this service

| Concern | Owner |
| --- | --- |
| Variant image id, pending until this event, live-image rule | `catalogue-service` |
| Seller legal name, GSTIN, approval, suspension, KYC row, 60-second review GET | `seller-service` |
| Browser session and the call to this service | `seller-bff` |
| Public product page and card image tags | `customer-bff`, using the CDN URLs catalogue already has after `MediaReady` |
| Search cards | `search-service`. It does not read this database |
| Invoice PDFs and object lock | `order-service` and `settlement-service` |

Not in version 1: video, more than two renditions, image cropping in the browser, bulk upload, and deleting originals after a product is hidden.

# catalogue-service — implementation spec

**Namespace:** `catalogue`  
**Database:** Aurora PostgreSQL database `catalogue` on cluster `bm-commerce`  
**Cache:** none. The product page cache is `bff:` in customer-bff. Stock is `inv:` in inventory-service. Search is OpenSearch, written only by search-service.  
**Callers:** `customer-bff`, `seller-bff`, `admin-bff`  
**Calls:** `seller-service` to read approval, suspension, and the auto-approve flag. `media-service` is not called synchronously. This service consumes `MediaReady`.  
**Publishes:** `ProductChanged`, `ProductHidden`  
**Consumes:** `SellerSuspended`, `MediaReady`  
**Product rules:** [modules/02-catalogue.md](../modules/02-catalogue.md)

This is the build document for the product record. Search, image bytes, stock, and the order snapshot are not stored here. A listing is not public until the publish checks in this file pass. Hiding a product does not delete orders that already copied its fields.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as identity-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as identity |
| Validation | Zod | Reject a bad body before it touches Postgres |
| SQL | `pg`, SQL migrations in `migrations/*.sql` | The schema in this file is the migration |
| Redis | none | No cache and no session store in this service |
| Ids | UUID v4 from `gen_random_uuid()` | Same as identity |
| Logs | `pino` JSON to stdout | Do not log image bytes or seller KYC |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One product save across BFF, catalogue, and the outbox worker |
| Metrics | `prom-client` on `GET /metrics` | Publish rejects, outbox lag, seller-service timeouts |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as identity |
| Container port | 8080 | Service port 80 targets 8080 |

Prices are integers in paise. The API does not accept rupees. Display in rupees is the storefront’s job.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | HTTP. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Publishes the outbox, and applies `SellerSuspended` and `MediaReady` |

The API does not publish to EventBridge itself. A status change and its outbox row commit in one transaction. The worker publishes and sets `published_at`.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `category` | The department, category, and subcategory tree. Admin only |
| `product` | The listing: title, brand, origin, HSN, GST, return terms, status |
| `variant` | SKU, at most two attributes, MRP, price, weight, best-before |
| `image` | Image ids from media-service, and whether `MediaReady` has arrived |
| `publish` | The checks that allow `live` |
| `outbox` | `product.changed` and `product.hidden` |
| `audit` | Staff approve, reject, and hide. Seller price edits on a live variant |

This service does not store seller legal name, GSTIN, or KYC files. The product page reads those from seller-service. It does not store stock. Inventory owns the in-stock flag.

---

## 4. How a request is trusted

Browsers never call catalogue-service. A BFF calls:

```text
http://catalogue-service.catalogue.svc.cluster.local
```

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `catalogue-service`. Lifetime at most 60 seconds. HS256. A bad audience or expiry is `401 service_unauthorized` |
| `X-Request-Id` | UUID. Echoed on the response and on every log line |
| `traceparent` | W3C trace context |
| `X-Session-Token` | Present on seller and staff routes. The BFF has already introspected it with identity. This service checks the family and role again by calling identity introspect. It does not trust a role claim in the body |

Public product reads need the service JWT only. They do not need a customer session. Guests can browse.

Seller writes need a session whose family is `seller` and whose role is `seller_owner` or `seller_catalogue`. `seller_orders` is `403 forbidden`. The product’s `seller_id` must equal the session’s `sellerId`. Another seller’s id is `404 not_found`, not `403`, so the route is not a catalogue probe.

Staff writes need a session whose family is `staff` and whose role is `catalogue` or `trust`. Other staff roles are `403 forbidden`.

JSON errors:

```json
{ "code": "publish_blocked", "message": "This listing cannot go live yet.", "requestId": "…" }
```

`message` is safe to show in Seller Centre. `code` is what the BFF branches on.

---

## 5. Category

Categories are a tree of three levels: department, category, subcategory. A seller picks a leaf. Only staff `catalogue` or `trust` create or edit the tree.

| Column | Rule |
| --- | --- |
| `name` | 1–80 characters, plain text |
| `slug` | Lowercase, unique among siblings. Used in the storefront URL |
| `parent_id` | Null for a department. A subcategory’s parent is a category, whose parent is a department. Deeper than that is `400 category_depth` |
| `hsn_hint` | Optional. 4, 6, or 8 digits. The seller still confirms `hsn` on the product |
| `gst_rate_options` | Non-empty set of integers, for example 5, 12, 18. The product’s `gst_rate` must be one of these |
| `return_window_days` | Integer, 0 through 30. The default for products in this leaf |
| `return_shipping_paid_by` | `seller` or `customer` |
| `requires_best_before` | When true, every variant needs a date before the product can go live |
| `licence` | `none` in version 1. `fssai` or `bis` blocks publish until that licence is on the seller. Food stays out of version 1 |

A leaf is a category whose id is not anyone’s `parent_id`. Products reference leaves only. A department or mid-level category as `category_id` is `400 category_not_leaf`.

---

## 6. Product and variant

A product is the listing. A variant is what the customer adds to the cart. MRP, price, weight, and best-before sit on the variant so two sizes can differ.

### Product

| Field | Rule |
| --- | --- |
| `seller_id` | The seller who fulfils it. BuyyMart’s own goods use `OWNED_SELLER_ID` |
| `category_id` | A leaf |
| `title` | 1–160 characters. HTML is stripped. Required before review |
| `brand` | 1–80 characters. HTML is stripped |
| `description` | Up to 4000 characters. HTML is stripped. Plain text |
| `country_of_origin` | Required. “India” is allowed. Stored as the country name, not a code |
| `importer_name` | Required when origin is not India. 1–160 characters |
| `hsn` | 4, 6, or 8 digits. Required. The category hint is not copied automatically |
| `gst_rate` | One of the leaf’s `gst_rate_options` |
| `return_window_days` | Null means use the category default. Otherwise 0 through 30 |
| `return_shipping_paid_by` | Null means use the category default |
| `status` | `draft`, `pending_review`, `live`, `hidden`, `rejected` |

### Variant

| Field | Rule |
| --- | --- |
| `sku` | Unique per seller. 1–64 characters |
| `attributes` | At most two axes, for example size and colour. Each value is 1–40 characters. A third axis is `400 attribute_limit` |
| `mrp_paise` | Integer. Greater than or equal to `price_paise` |
| `price_paise` | Integer. What the customer pays for one unit, tax inclusive. Greater than 0 |
| `weight_grams` | Integer greater than 0. Required before the product can go live. Fulfilment prices delivery from this declared weight |
| `best_before` | A date. Required when the category has `requires_best_before` |

Tax is computed, not stored as an input:

```text
taxPaise = round(pricePaise * gstRate / (100 + gstRate))
taxablePaise = pricePaise - taxPaise
```

`round` is half away from zero, to the nearest paise. The order copies these numbers at purchase time. Later edits do not change an old order.

### Images

Catalogue stores the image id, not the bytes. Media-service owns the object.

| Column | Rule |
| --- | --- |
| `image_id` | The id from media-service |
| `variant_id` | The variant it is attached to |
| `position` | 0-based order on that variant |
| `status` | `pending` until `MediaReady`, then `ready`. A rejected upload never becomes `ready` |

A product cannot go live while any variant has zero `ready` images. Pending images do not count.

### Status

```text
draft → pending_review → live
                       → rejected → draft
live → hidden → live
```

`hidden` is reversible when the publish checks still pass. `rejected` stores the staff reason. The seller edits the draft and submits again. There is no hard delete in version 1. Hiding a product does not delete the row.

---

## 7. Publish rules

A product moves to `live` only when all of these are true:

- Seller status is `approved` or `active`, and not `suspended` or `closed`.
- Title, leaf category, HSN, GST rate, and country of origin are set.
- Importer name is set when origin is not India.
- The category `licence` is `none`, or the seller holds that licence. Version 1 categories use `none`.
- Every variant has MRP, price, and weight, and MRP is at least the price.
- Best-before is set on every variant when the category requires it.
- Every variant has at least one `ready` image.
- Staff have approved it, unless the seller’s auto-approve flag applies.

If any check fails, the response is `409 publish_blocked`. The body includes `missing`, an array of short tokens such as `image`, `weight`, `importer_name`, `seller_not_approved`. The `message` stays one sentence.

### Seller status

Publish calls seller-service `GET /v1/sellers/:id` with the service JWT. Timeout 1 second. If that call fails, publish returns `503 dependency_unavailable` and the product stays where it was. Catalogue does not guess that the seller is approved.

Until seller-service is deployed, `SELLER_URL` is empty. In that mode only `OWNED_SELLER_ID` may be submitted, and that id is treated as approved and not suspended. Any other `seller_id` is `409 publish_blocked` with `seller_not_approved`. Stage and prod set `SELLER_URL` once seller-service is up, and then the owned-seller shortcut is off.

### Auto-approve

Seller-service holds an admin flag: after the seller’s first ten live listings, further submissions may skip the queue. Catalogue counts its own `live` rows for that `seller_id`. When the flag is on and the count is at least 10, a submit that passes the other checks goes straight to `live` and writes `product.changed`.

Dev may set `OWNED_SELLER_AUTO_APPROVE=true` so the internal seller can reach `live` without staff. Stage and prod leave it unset or `false`. Marketplace sellers never use that variable.

### Staff actions

| Action | Who | Effect |
| --- | --- | --- |
| Approve | Staff `catalogue` or `trust` | `pending_review` becomes `live` if the checks pass. Reason at least 10 characters. Audit row |
| Reject | Staff `catalogue` or `trust` | `pending_review` becomes `rejected`. Reason at least 10 characters. Audit row. `product.hidden` if it had been live, which it has not |
| Hide | Staff `catalogue` or `trust` | `live` becomes `hidden`. Reason at least 10 characters. Audit row. `product.hidden` |

A seller cannot approve their own listing.

### Live price edit

A seller with `seller_owner` or `seller_catalogue` may change `mrp_paise` and `price_paise` on a live variant. MRP must stay at least the price. The audit row stores the previous price and the new price. No reason is required. The same transaction inserts `product.changed` for that variant. Other fields on a live product go through edit-and-submit, back to `pending_review`, unless auto-approve applies.

### Seller suspended

The worker consumes `SellerSuspended`. Every `live` product for that `seller_id` becomes `hidden` in one transaction per product, with an audit row whose actor is the event and whose reason is the suspension reason, and an outbox `product.hidden`. In-flight orders are not cancelled here.

---

## 8. Events

Wire `type` values are lowercase, matching identity:

| Name | `type` | When |
| --- | --- | --- |
| `ProductChanged` | `product.changed` | A variant is now `live`, or a live variant’s customer-visible fields change |
| `ProductHidden` | `product.hidden` | A live variant becomes `hidden` or `rejected`, or the seller is suspended |

One outbox row per variant. Search upserts from the payload and does not read this database.

`product.changed` payload:

```json
{
  "productId": "…",
  "variantId": "…",
  "sellerId": "…",
  "title": "…",
  "brand": "…",
  "categoryPath": ["Department", "Category", "Leaf"],
  "pricePaise": 49900,
  "mrpPaise": 59900,
  "countryOfOrigin": "India",
  "status": "live",
  "updatedAt": "2026-09-28T15:30:00Z"
}
```

`product.hidden` payload:

```json
{
  "productId": "…",
  "variantId": "…",
  "sellerId": "…",
  "status": "hidden",
  "at": "2026-09-28T15:30:00Z"
}
```

The envelope is `{ id, type, source, time, traceId, data }` with `source` `catalogue-service`. `data` is the payload above. Rating is not included. Version 1 does not invent a rating. `inStock` is not included. Search sets that from `InventoryChanged`.

Checkout does not trust the search index or the BFF cache for price. The product page and checkout read price from this service.

---

## 9. HTTP API

Base path `/v1`. Every route requires the service JWT. Session rules are in the table.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| GET | `/v1/categories` | No | `200` the tree |
| GET | `/v1/products/:id` | No | `200` live product, variants, ready images, computed tax |
| GET | `/v1/products?q=` | No | `200` live keyword match on title and brand. Limit 20. No filters. This is the search fallback |
| GET | `/v1/seller/products` | Seller owner or catalogue | `200` this seller’s rows, any status |
| POST | `/v1/seller/products` | Seller owner or catalogue | `201` draft |
| PATCH | `/v1/seller/products/:id` | Seller owner or catalogue | `200` |
| POST | `/v1/seller/products/:id/variants` | Seller owner or catalogue | `201` |
| PATCH | `/v1/seller/variants/:id` | Seller owner or catalogue | `200` |
| POST | `/v1/seller/variants/:id/images` | Seller owner or catalogue | `201` pending image |
| POST | `/v1/seller/products/:id/submit` | Seller owner or catalogue | `200` `pending_review` or `live` |
| POST | `/v1/staff/categories` | Staff catalogue or trust | `201` |
| PATCH | `/v1/staff/categories/:id` | Staff catalogue or trust | `200` |
| POST | `/v1/staff/products/:id/approve` | Staff catalogue or trust | `200` `live` |
| POST | `/v1/staff/products/:id/reject` | Staff catalogue or trust | `200` `rejected` |
| POST | `/v1/staff/products/:id/hide` | Staff catalogue or trust | `200` `hidden` |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only after Postgres `SELECT 1` |
| GET | `/metrics` | Cluster scrape only | Prometheus text |

`GET /v1/products/:id` for a draft, pending, hidden, or rejected product is `404 not_found`. The seller route returns those statuses to the owning seller.

`GET /v1/products?q=` matches `ILIKE` on title and brand, status `live` only, ordered by `updated_at` descending. It does not apply category, price, brand, discount, or in-stock filters. Those belong to search-service. The customer BFF uses this route only when search does not answer within 400 ms, and it must not present the result as a search-index response.

A customer read of a live product includes, per variant, `pricePaise`, `mrpPaise`, `taxPaise`, `taxablePaise`, `weightGrams`, `bestBefore` when set, attributes, and ready image ids in `position` order. It does not include the seller’s legal name or GSTIN. Customer-bff reads those from seller-service.

### Submit body

No body. The checks use the stored row.

### Reject and hide body

```json
{ "reason": "Title does not match the images." }
```

`reason` is at least 10 characters.

### Image body

```json
{ "imageId": "…", "position": 0 }
```

The row is `pending` until `MediaReady`.

---

## 10. PostgreSQL schema

Database `catalogue`. The application role `catalogue_app` can `SELECT`, `INSERT`, and `UPDATE` on these tables. It cannot `DELETE`, `DROP`, `TRUNCATE`, or alter schema. Hiding a product is an update. The migration role `catalogue_migrator` runs `migrations/` and is not the runtime role.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE product_status AS ENUM (
  'draft', 'pending_review', 'live', 'hidden', 'rejected'
);
CREATE TYPE return_payer AS ENUM ('seller', 'customer');
CREATE TYPE category_licence AS ENUM ('none', 'fssai', 'bis');
CREATE TYPE image_status AS ENUM ('pending', 'ready');

CREATE TABLE categories (
  id                        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id                 uuid REFERENCES categories (id),
  name                      text NOT NULL,
  slug                      text NOT NULL,
  hsn_hint                  text,
  gst_rate_options          smallint[] NOT NULL,
  return_window_days        smallint NOT NULL,
  return_shipping_paid_by   return_payer NOT NULL,
  requires_best_before      boolean NOT NULL DEFAULT false,
  licence                   category_licence NOT NULL DEFAULT 'none',
  created_at                timestamptz NOT NULL DEFAULT now(),
  updated_at                timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT categories_name_len CHECK (char_length(name) BETWEEN 1 AND 80),
  CONSTRAINT categories_slug CHECK (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'),
  CONSTRAINT categories_hsn_hint CHECK (
    hsn_hint IS NULL OR hsn_hint ~ '^[0-9]{4}$' OR hsn_hint ~ '^[0-9]{6}$' OR hsn_hint ~ '^[0-9]{8}$'
  ),
  CONSTRAINT categories_gst_options CHECK (cardinality(gst_rate_options) >= 1),
  CONSTRAINT categories_return_window CHECK (return_window_days BETWEEN 0 AND 30),
  UNIQUE (parent_id, slug)
);

CREATE TABLE products (
  id                         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  seller_id                  uuid NOT NULL,
  category_id                uuid NOT NULL REFERENCES categories (id),
  title                      text,
  brand                      text,
  description                text,
  country_of_origin          text,
  importer_name              text,
  hsn                        text,
  gst_rate                   smallint,
  return_window_days         smallint,
  return_shipping_paid_by    return_payer,
  status                     product_status NOT NULL DEFAULT 'draft',
  rejected_reason            text,
  created_at                 timestamptz NOT NULL DEFAULT now(),
  updated_at                 timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT products_title_len CHECK (title IS NULL OR char_length(title) BETWEEN 1 AND 160),
  CONSTRAINT products_brand_len CHECK (brand IS NULL OR char_length(brand) BETWEEN 1 AND 80),
  CONSTRAINT products_description_len CHECK (
    description IS NULL OR char_length(description) BETWEEN 1 AND 4000
  ),
  CONSTRAINT products_hsn CHECK (
    hsn IS NULL OR hsn ~ '^[0-9]{4}$' OR hsn ~ '^[0-9]{6}$' OR hsn ~ '^[0-9]{8}$'
  ),
  CONSTRAINT products_return_window CHECK (
    return_window_days IS NULL OR return_window_days BETWEEN 0 AND 30
  )
);

CREATE INDEX products_seller_idx ON products (seller_id, status);
CREATE INDEX products_live_title_idx ON products (title) WHERE status = 'live';

CREATE TABLE variants (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id     uuid NOT NULL REFERENCES products (id),
  seller_id      uuid NOT NULL,
  sku            text NOT NULL,
  attributes     jsonb NOT NULL DEFAULT '[]',
  mrp_paise      integer,
  price_paise    integer,
  weight_grams   integer,
  best_before    date,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT variants_sku_len CHECK (char_length(sku) BETWEEN 1 AND 64),
  CONSTRAINT variants_price CHECK (
    (mrp_paise IS NULL AND price_paise IS NULL)
    OR (mrp_paise >= price_paise AND price_paise > 0)
  ),
  CONSTRAINT variants_weight CHECK (weight_grams IS NULL OR weight_grams > 0),
  UNIQUE (seller_id, sku)
);

CREATE INDEX variants_product_idx ON variants (product_id);

CREATE TABLE variant_images (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  variant_id  uuid NOT NULL REFERENCES variants (id),
  image_id    uuid NOT NULL,
  position    smallint NOT NULL,
  status      image_status NOT NULL DEFAULT 'pending',
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (variant_id, image_id),
  UNIQUE (variant_id, position)
);

CREATE TABLE catalogue_audit (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  at            timestamptz NOT NULL DEFAULT now(),
  actor_id      uuid,
  actor_role    text NOT NULL,
  action        text NOT NULL,
  product_id    uuid NOT NULL,
  variant_id    uuid,
  reason        text,
  before        jsonb,
  after         jsonb,
  request_id    text NOT NULL
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

`updated_at` is set by the application on every update. There is no trigger in version 1.

`attributes` is a JSON array of `{ "name", "value" }` with length 0, 1, or 2. The length check is in the application. A third element is `400 attribute_limit`.

Staff approve, reject, and hide require `reason` of at least 10 characters. That check is in the application, because seller price-edit audit rows have a null reason and still store `before` and `after` prices.

---

## 11. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `DATABASE_URL` | `catalogue_app` connection string |
| Secret | `DATABASE_MIGRATOR_URL` | Used only by the migration job. The API refuses to start if this is set |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `catalogue-service` |
| Config | `IDENTITY_URL` | Introspect seller and staff sessions |
| Config | `SELLER_URL` | Empty until seller-service is deployed |
| Config | `OWNED_SELLER_ID` | BuyyMart’s internal seller UUID |
| Config | `OWNED_SELLER_AUTO_APPROVE` | `true` only in dev. Stage and prod unset or `false` |
| Config | `EVENTBRIDGE_BUS_NAME` | Bus name. Empty means the worker logs the envelope and does not call AWS |

Secrets live in Secrets Manager at `buyymart/{env}/catalogue-service`. They are not in the image and not in git.

---

## 12. Local run

Docker Compose for this service is Postgres 16 and the API. The worker starts in the same Compose file. Identity must already be running if seller or staff routes are called, because those routes introspect. Customer reads of a live product need only the service JWT.

Migrations run before the API starts. A seed command inserts the owned seller’s first department, category, and leaf only when `SEED_CATEGORY_SLUG` is set and `NODE_ENV=development`. That variable is absent in stage and prod.

`MediaReady` in dev can be posted to the worker’s test hook only when `NODE_ENV=development`. Stage and prod receive it from the bus. An image does not become `ready` because the seller attached an id.

---

## 13. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| Create a draft, submit with no image | `409 publish_blocked`, `missing` contains `image`, status stays `draft` or returns to it. Nothing in the outbox |
| Submit with a pending image | `publish_blocked`, `image` |
| `MediaReady`, then submit with price, MRP, weight, origin India, HSN, GST | `pending_review`, or `live` when auto-approve is on |
| MRP less than price | `400 price_invalid` |
| Origin not India and no importer | `publish_blocked`, `importer_name` |
| Category `requires_best_before` and a variant with no date | `publish_blocked`, `best_before` |
| Third attribute | `400 attribute_limit` |
| SKU reused by the same seller | `409 sku_taken` |
| Another seller’s product id | `404 not_found` |
| `seller_orders` session | `403 forbidden` |
| Staff `support` approves | `403 forbidden` |
| Approve without a 10-character reason | `400 reason_required` |
| Approve a complete pending product | `live`, one `product.changed` per variant, audit row |
| Hide | `hidden`, `product.hidden`, public GET is `404` |
| Seller suspended | That seller’s live products become `hidden` |
| Public keyword query | Live title and brand only. Drafts are absent |
| GST rate not in the leaf’s options | `400 gst_rate_invalid` |
| Outbox | Worker sets `published_at`. A second poll does not send it again |
| `catalogue_app` | `DELETE` fails. `UPDATE` of `status` succeeds |

---

## 14. Outside this service

| Concern | Owner |
| --- | --- |
| Search index and filters | `search-service` |
| Image bytes, WebP renditions, presigned upload | `media-service` |
| Stock count and the in-stock flag | `inventory-service` |
| Seller legal name, GSTIN, approval, suspension | `seller-service` |
| Price on an order already placed | `order-service` snapshot |
| Delivery fee | `fulfilment-service`, using `weight_grams` |
| Product-page cache | `customer-bff`, prefix `bff:` |

Not in version 1: bulk CSV, rich video, more than two variant attributes, and customer questions on the product page. Ratings are phase 2 and are not stored here.

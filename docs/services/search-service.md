# search-service — implementation spec

**Namespace:** `catalogue`  
**Database:** none. This service does not use Aurora.  
**Index:** OpenSearch index `products` for customer hits. OpenSearch index `index_state` for the last event and the last error. This service is the only writer of both.  
**Cache:** none.  
**Callers:** `customer-bff` for search. `admin-bff` for the staff inspect route.  
**Calls:** `identity-service` to introspect a staff session. `catalogue-service` only from the nightly repair, and only when `CATALOGUE_URL` is set.  
**Publishes:** nothing.  
**Consumes:** `ProductChanged`, `ProductHidden`, `InventoryChanged`.  
**Product rules:** [modules/03-search.md](../modules/03-search.md)

This is the build document for the product index. Catalogue is the source of truth for title, brand, category, and price. Inventory is the source of truth for stock. This service stores a copy so the storefront can filter and sort. The product page reloads price from catalogue and stock from inventory. Checkout does not read this index.

The customer BFF waits at most 400 ms for this service. If it does not answer, the BFF calls catalogue `GET /v1/products?q=` and must not present that keyword list as an index result. This service does not call catalogue on the search path.

---

## 1. Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | TypeScript 5, strict | Same language as catalogue-service |
| Runtime | Node.js 22 LTS | One process for the API, one for the worker |
| HTTP | Fastify 5 | Same server as catalogue |
| Validation | Zod | Reject a bad query before it touches OpenSearch |
| Index | `@opensearch-project/opensearch` | The only store. There is no `pg` dependency |
| SQL | none | No migrations, no roles, no outbox table |
| Redis | none | No cache and no session store in this service |
| Ids | The variant UUID is the document id | A second write updates the same document |
| Logs | `pino` JSON to stdout | Log the filter names, the hit count, and the event id. Do not log the query text |
| Traces | OpenTelemetry SDK, W3C `traceparent` | One search across the BFF and this API |
| Metrics | `prom-client` on `GET /metrics` | Query failures, events applied, events ignored, document count |
| Image | `node:22-bookworm-slim`, non-root uid 1000 | Same image base as catalogue |
| Container port | 8080 | Service port 80 targets 8080 |

Prices in the index are integers in paise. This service does not compute GST.

---

## 2. Processes

Two containers, same image.

| Process | Command | Job |
| --- | --- | --- |
| API | `node dist/api.js` | Search and the staff inspect route. Minimum 2 replicas in production, on the `apps` node group |
| Worker | `node dist/worker.js` | Applies catalogue and inventory events. Runs the nightly repair |

The API does not write the customer index except by refusing to. All upserts and deletes happen in the worker. A search that finds OpenSearch down is `503 search_unavailable`. It does not fall back to catalogue. That fallback belongs to `customer-bff`.

Both processes create the two indexes if they are missing. Creating an index that already exists is a no-op. Neither process deletes an index.

The task role can `es:ESHttpGet`, `es:ESHttpPost`, `es:ESHttpPut`, and `es:ESHttpDelete` on this domain only. No other service has that access.

---

## 3. Modules inside the service

| Module | Responsibility |
| --- | --- |
| `query` | Text, filters, and sort over `products` |
| `index` | Upsert, delete, and the in-stock flag |
| `repair` | Once a day, drop documents catalogue no longer lists as live |
| `inspect` | Tell staff whether a variant is in the index |

This service does not store stock counts, seller legal name, GSTIN, phone, address, or image bytes. Version 1 does not store a rating and does not store an image URL. `product.changed` does not carry either field, and this service does not invent them.

---

## 4. How a request is trusted

Browsers never call search-service. A BFF calls:

```text
http://search-service.catalogue.svc.cluster.local
```

| Header | Rule |
| --- | --- |
| `Authorization` | `Bearer` service JWT. Audience `search-service`. Lifetime at most 60 seconds. HS256. A bad audience or expiry is `401 service_unauthorized` |
| `X-Request-Id` | UUID. Echoed on the response and on every log line |
| `traceparent` | W3C trace context |
| `X-Session-Token` | Present on the staff route only. The BFF has already introspected it with identity. This service checks the family and role again by calling identity introspect. It does not trust a role claim in the body |

`GET /v1/search` needs the service JWT only. It does not need a customer session. Guests can search.

The staff route needs a session whose family is `staff` and whose role is `catalogue` or `trust`. A seller session, including `seller_owner`, is `403 forbidden`. Other staff roles are `403 forbidden`.

JSON errors:

```json
{ "code": "invalid_query", "message": "That search is not valid.", "requestId": "…" }
```

`message` is safe to show on the storefront. `code` is what the BFF branches on.

---

## 5. Document

The customer index is `products`. The document id is `variantId`. One live variant is one document. A replay cannot create a second document for the same variant.

| Field | Source | Rule |
| --- | --- | --- |
| `productId`, `variantId`, `sellerId` | `product.changed` | UUIDs. The card opens the catalogue product |
| `title` | `product.changed` | Text search. 1–200 characters |
| `brand` | `product.changed` | Text search and an exact filter |
| `categoryPath` | `product.changed` | One to three non-empty strings, department then category then leaf |
| `pricePaise` | `product.changed` | Integer. Filter and sort |
| `mrpPaise` | `product.changed` | Integer, greater than or equal to `pricePaise`. Discount is `(mrpPaise - pricePaise) / mrpPaise` |
| `inStock` | `InventoryChanged`, or `false` on the first insert | Out-of-stock variants stay in the index and are labelled |
| `countryOfOrigin` | `product.changed` | Exact filter. Required, non-empty |
| `status` | `product.changed` | Only `live` is stored in `products` |
| `updatedAt` | `product.changed` | Tie-break. Newer wins |
| `lastEventId` | Envelope `id` | The last event that changed this document |

`index_state` uses the same document id. Customers never query it.

| Field | Meaning |
| --- | --- |
| `inIndex` | `true` when a `products` document exists |
| `lastEventId` | Last event id accepted for this variant, including a delete |
| `lastEventAt` | Envelope `time` of that event |
| `lastError` | `null`, `invalid_event`, or `seller_mismatch` |
| `inStock` | Latest stock flag, kept even after the product document is removed |
| `stockAt` | `at` from the inventory event that set `inStock` |
| `updatedAt` | Catalogue `updatedAt` last applied, if any |

There is no `rating`. A query that asks for one is `400 invalid_query`. There is no sponsored or boost field. A query that sends `boost` or `sponsored` is `400 invalid_query`.

---

## 6. Customer query

`GET /v1/search`. Unknown query keys are `400 invalid_query`.

| Query | Rule |
| --- | --- |
| `q` | Optional. Trimmed. Empty means no text clause. Longer than 80 characters is `400 invalid_query`. The text is a match, not Lucene syntax |
| `category` | Optional. Exact match on one element of `categoryPath` |
| `brand` | Optional. Exact match on the brand keyword |
| `country` | Optional. Exact match on `countryOfOrigin` |
| `priceMinPaise`, `priceMaxPaise` | Optional integers. `priceMinPaise` greater than `priceMaxPaise` is `400 invalid_query` |
| `minDiscountPercent` | Optional integer from 1 through 90. Compared as `(mrpPaise - pricePaise) * 100 >= minDiscountPercent * mrpPaise`. A zero MRP does not match |
| `inStock` | Optional `true` or `false`. Omitted returns both. Any other value is `400 invalid_query` |
| `sort` | `relevance`, `price_asc`, `price_desc`, or `newest`. Default `relevance` |
| `limit` | Default 20. Maximum 20. Anything else is `400 invalid_query` |
| `page` | Default 1. Maximum 5. Page 5 is the last page, 100 hits |

Text uses an OpenSearch `multi_match` of type `cross_fields` with operator `and`. Field boosts are title 3, brand 2, `categoryPath` 1. User input is not passed to `query_string`.

Version 1 ranking for `relevance`, in order:

1. The text score, when `q` is present. With no `q`, this step is skipped.
2. `inStock: true` above `inStock: false`.
3. Newer `updatedAt`.

`price_asc` and `price_desc` sort by `pricePaise`, then in-stock, then newer `updatedAt`. `newest` sorts by `updatedAt` descending, then in-stock. There is no paid placement and no seller boost.

The query also filters `status: live`. Hidden and rejected variants are not in this index.

The OpenSearch request timeout is 300 ms, inside the BFF’s 400 ms budget. A timeout or a dead cluster is `503 search_unavailable`.

`200`:

```json
{
  "source": "search",
  "total": 1,
  "hits": [
    {
      "productId": "…",
      "variantId": "…",
      "sellerId": "…",
      "title": "Cotton kurta",
      "brand": "BuyyMart",
      "categoryPath": ["Women", "Clothing", "Kurtas"],
      "pricePaise": 49900,
      "mrpPaise": 59900,
      "inStock": false,
      "countryOfOrigin": "India",
      "status": "live",
      "updatedAt": "2026-09-28T15:30:00Z"
    }
  ]
}
```

`source` is always `search`. The hit has no rating, no image URL, no GSTIN, and no stock count. `total` is the number of matches, not the page size. An empty index returns `200` with `total: 0` and `hits: []`.

Example: MRP 59900 and price 49900 is a 16 percent discount. `minDiscountPercent=10` matches. `minDiscountPercent=17` does not.

---

## 7. How the index stays current

The worker accepts the envelope `{ id, type, source, time, traceId, data }`. Wire `type` values are lowercase.

| Event | `type` | `source` | Effect |
| --- | --- | --- | --- |
| `ProductChanged` | `product.changed` | `catalogue-service` | Upsert the live variant |
| `ProductHidden` | `product.hidden` | `catalogue-service` | Delete the `products` document |
| `InventoryChanged` | `inventory.changed` | `inventory-service` | Set `inStock` only |

`product.changed` data is the catalogue payload: `productId`, `variantId`, `sellerId`, `title`, `brand`, `categoryPath`, `pricePaise`, `mrpPaise`, `countryOfOrigin`, `status`, `updatedAt`. Rating and `inStock` are not in that payload.

`product.hidden` data is `productId`, `variantId`, `sellerId`, `status`, `at`. `status` is `hidden` or `rejected`. Both delete the document.

`inventory.changed` data is the contract this service requires. Inventory-service is not built yet. When it is, it sends:

```json
{
  "variantId": "…",
  "sellerId": "…",
  "inStock": true,
  "at": "2026-09-28T15:30:00Z"
}
```

`inStock` is true only when available-to-sell is above zero. The quantity is not in the event and is not stored.

Apply rules, in order:

| Rule | Result |
| --- | --- |
| Envelope `id` equals `index_state.lastEventId` for that variant | No write. The same event does not change the document again |
| `product.changed` whose `status` is not `live` | Same as `product.hidden` |
| `product.changed` missing a required field, with a non-integer price, with MRP below price, or with a `categoryPath` outside one to three strings | `lastError` `invalid_event`. The `products` document is left as it was. `lastEventId` becomes this event id |
| `product.changed` with `updatedAt` older than or equal to the stored `updatedAt` | Ignored. Not an error. A missing document is still inserted |
| `product.changed` that applies | Upsert `products`. Keep the stored `inStock` when `index_state.stockAt` is set. Otherwise set `inStock` to `false`. Clear `lastError` |
| `product.hidden` | Delete `products` if present. `inIndex` becomes `false`. The stock flag in `index_state` stays |
| `inventory.changed` whose `sellerId` does not match the product document | `lastError` `seller_mismatch`. `inStock` is unchanged |
| `inventory.changed` whose `at` is older than or equal to `stockAt` | Ignored |
| `inventory.changed` and the product document exists | Update `inStock` on both indexes |
| `inventory.changed` and the product document does not exist | Store the flag on `index_state` only. Do not create a customer hit |

A hidden variant returns to the index only when a newer `product.changed` with `status: live` arrives. The worker does not read the catalogue database.

Empty `EVENTBRIDGE_BUS_NAME` means the worker does not call AWS. In that mode the only ingest is the development hook in section 12.

---

## 8. Nightly repair

The repair is a worker loop every 24 hours. It does not run inside `GET /v1/search`.

`CATALOGUE_URL` empty, which is the stage and prod value until catalogue grows the route below, means the loop logs and deletes nothing.

When `CATALOGUE_URL` is set, the worker calls:

```text
GET {CATALOGUE_URL}/v1/internal/live-variants?after={variantId}
```

The service JWT audience for that call is `catalogue-service`. The response search expects is:

```json
{
  "variants": [{ "variantId": "…", "productId": "…" }],
  "next": null
}
```

`variants` is at most 100 rows, ordered by `variantId`. `next` is the last id on the page, or `null` when the scan is finished. This route is not in the current catalogue spec. A non-200, a timeout, or a short page that never returns `next: null` aborts the repair and deletes nothing.

A finished scan deletes `products` documents whose `variantId` is absent from the live set, and sets `inIndex` to `false`. It does not create documents and does not fill title or price. A missing live variant waits for a replay of `product.changed`.

---

## 9. Staff inspect

`GET /v1/staff/variants/:id`

A malformed id is `404 not_found`. An id that has never been seen is `200` with `inIndex: false`. Staff cannot edit the index. There is no write route for them.

`200` when the variant was indexed and then hidden:

```json
{
  "variantId": "…",
  "inIndex": false,
  "lastEventId": "…",
  "lastEventAt": "2026-09-28T16:00:00Z",
  "lastError": null,
  "updatedAt": "2026-09-28T15:30:00Z"
}
```

`inIndex: true` means a customer search can return that variant. `lastError` is the last ingest error, or `null`.

Identity down on this route is `503 dependency_unavailable`. The customer search route does not call identity.

---

## 10. HTTP API

Base path `/v1`. Every route except health and metrics requires the service JWT.

| Method | Path | Session | Success |
| --- | --- | --- | --- |
| GET | `/v1/search` | No | `200` the page of hits |
| GET | `/v1/staff/variants/:id` | Staff `catalogue` or `trust` | `200` index state, including a variant that is absent |
| GET | `/health/live` | None. No JWT | `200` if the process is up |
| GET | `/health/ready` | None. No JWT | `200` only when the `products` index health is `green` or `yellow` |
| GET | `/metrics` | Cluster scrape only | Prometheus text |

`GET /health/ready` does not ping catalogue or identity. A `red` index, a missing index, or a connection error is `503`. A local single-node cluster is `yellow` when replicas are 1, so local Compose sets replicas to 0 and ready accepts `yellow` as well as `green`.

Metrics:

| Name | Meaning |
| --- | --- |
| `search_query_failures_total` | Search routes that returned `503` |
| `search_events_applied_total` | Events that upserted or deleted a document, or stored a stock flag |
| `search_events_ignored_total` | Replays, older catalogue events, and older stock events |
| `search_index_documents` | Current `products` count |

---

## 11. Index mapping

Both indexes are created at startup. Production uses one primary shard and one replica, on two data nodes. Staging uses one node and `OPENSEARCH_REPLICAS=0`. Local Compose uses the same.

`products`:

| Field | OpenSearch type |
| --- | --- |
| `productId`, `variantId`, `sellerId`, `countryOfOrigin`, `status`, `lastEventId` | `keyword` |
| `title`, `brand`, `categoryPath` | `text`, plus a `keyword` subfield for exact filters |
| `pricePaise`, `mrpPaise` | `long` |
| `inStock` | `boolean` |
| `updatedAt` | `date` |

`index_state` fields are keywords, one `boolean` for `inIndex`, one `boolean` for `inStock`, and dates for `lastEventAt`, `stockAt`, and `updatedAt`. `lastError` is a keyword.

The customer search never sets `index` to `index_state`.

---

## 12. Config and secrets

| Kind | Name | Example |
| --- | --- | --- |
| Secret | `SERVICE_JWT_KEY` | HMAC key. Audience `search-service` |
| Config | `OPENSEARCH_URL` | VPC domain in stage and prod. `http://opensearch:9200` in Compose |
| Config | `OPENSEARCH_REPLICAS` | `1` in prod. `0` in staging and local |
| Config | `IDENTITY_URL` | Introspect staff sessions |
| Config | `CATALOGUE_URL` | Empty until the live-variant route exists |
| Config | `EVENTBRIDGE_BUS_NAME` | Bus name. Empty means the worker does not call AWS |
| Secret | `OPENSEARCH_USERNAME` | Local only, and only if the security plugin is on |
| Secret | `OPENSEARCH_PASSWORD` | Local only, and only if the security plugin is on |

Secrets live in Secrets Manager at `buyymart/{env}/search-service`. They are not in the image and not in git.

Stage and prod do not set `OPENSEARCH_USERNAME` or `OPENSEARCH_PASSWORD`. The client signs with the task role. They also leave `CATALOGUE_URL` empty until catalogue exposes `GET /v1/internal/live-variants`.

---

## 13. Local run

Docker Compose for this service is OpenSearch 2.19 and the two processes. The security plugin is off in Compose. Identity must already be running for the staff route. Customer search needs only the service JWT and OpenSearch.

Host ports are 8087 for the API, 8088 for the worker hook, and 9200 for OpenSearch, so this stack can run beside identity on 8080, catalogue on 8083, and media on 8085.

Events in dev can be posted to `POST /internal/events` on the worker only when `NODE_ENV=development`. The body is one envelope from section 7. The hook runs the same apply rules as the bus. Stage and prod do not register the hook. A document does not appear because a test called the API.

The hook requires the service JWT.

---

## 14. What has to be tested before the first deploy

| Test | Expected |
| --- | --- |
| `product.changed` for a live variant | One `products` document, id equals `variantId`, `inStock` false, `index_state.inIndex` true |
| The same event id again | No second document. `search_events_ignored_total` increases |
| An older `updatedAt` after a newer one | The newer title and price remain |
| `product.changed` with MRP below price, or `status` other than `live` | No customer hit for a bad body. A non-live status deletes the document |
| `product.hidden` | The customer document is gone. `inIndex` is false. A later live `product.changed` inserts it again |
| `inventory.changed` with `inStock: true` | The existing document flips to true. The quantity is not stored |
| `inventory.changed` before the product event | `index_state.inStock` is true and `inIndex` is false. The following `product.changed` copies that flag |
| `inventory.changed` with a different `sellerId` | `lastError` is `seller_mismatch` and `inStock` does not change |
| Search `q` matching the title | `200`, `source` is `search`, the variant is in `hits` |
| `minDiscountPercent=10` on MRP 59900 and price 49900 | The hit is included |
| `minDiscountPercent=17` on those prices | The hit is excluded |
| `inStock=true` while the flag is false | The hit is excluded, and a search with the filter omitted still returns it |
| Sort `price_asc` | Lower `pricePaise` first |
| `q` longer than 80, `limit` above 20, or `boost=1` | `400 invalid_query` |
| OpenSearch down | `503 search_unavailable` on search. `GET /health/ready` is `503`. `GET /health/live` stays `200` |
| Staff `catalogue` inspects a known variant | `200` with `lastEventId` |
| A seller session on the staff route | `403 forbidden` |
| Missing service JWT | `401 service_unauthorized` |
| Repair when `CATALOGUE_URL` is empty | No delete |
| Repair when the catalogue call fails | No delete |
| Stage and prod env files | Do not set `OPENSEARCH_USERNAME`, `OPENSEARCH_PASSWORD`, or `CATALOGUE_URL` |
| Development hook when `NODE_ENV` is `production` | The route is not registered |

---

## 15. Outside this service

| Concern | Owner |
| --- | --- |
| Title, brand, category, MRP, price, country of origin, live or hidden | `catalogue-service` |
| Image URLs on the product page and on a card | `customer-bff`, from catalogue after `MediaReady` |
| Available-to-sell count, reservations | `inventory-service` |
| The 400 ms budget and the keyword fallback | `customer-bff` |
| Ranking-page copy that names text match, availability, and recency | Customer web |
| Sponsored labels, if a later version adds paid rank | Customer web, together with a new field in this index |

Not in version 1: ratings, image URLs in the hit, paid placement, suggestions, facets with counts, deep pages past 100 hits, and writing the index from the API.

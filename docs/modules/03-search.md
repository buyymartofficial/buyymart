# Search

**Owns:** the product index used for search, filters, and sort.  
**Service:** `search-service`.  
**Store:** OpenSearch index `products`. This service is the only writer.

Search is a copy of catalogue plus an in-stock flag from inventory. It is not the source of truth for price or stock. The product page reloads price from catalogue and stock from inventory.

---

## What is in a document

| Field | Why it is there |
| --- | --- |
| `productId`, `variantId`, `sellerId` | Open the real record |
| `title`, `brand`, `categoryPath` | Text search and breadcrumbs |
| `pricePaise` | Filter and sort |
| `mrpPaise` | Discount filter: `(mrp - price) / mrp` |
| `rating` | Phase 2. Version 1 indexes nothing and hides the filter. |
| `inStock` | From `InventoryChanged`. Out-of-stock items stay searchable and are labelled. |
| `countryOfOrigin` | Filter, and a check that the field was not dropped |
| `status` | Only `live` is returned to customers |

The index has no phone, address, or other customer data.

---

## Customer query

The storefront sends text plus optional filters: category, price range, brand, discount, country of origin, and in-stock only. Sort is relevance, price ascending, price descending, or newest.

Version 1 ranking, in order:

1. Text match on title, then brand, then category.
2. In-stock above out-of-stock.
3. Newer `updatedAt` as a tie-break.

There is no paid placement in version 1. The ranking page on the site says that the main parameters are text match, availability, and recency. When sponsored results are added later, each sponsored card is labelled and the ranking page is updated. The 2026 amendment briefings treat unlabelled paid rank as a disclosure failure. Do not add a boost flag without that label.

If search does not answer within 400 ms, the customer BFF falls back to a catalogue keyword query on title and brand. The page shows results without the richer filters and does not pretend they came from the index.

---

## How the index stays current

| Event | Change |
| --- | --- |
| `ProductChanged` | Upsert the document from the event payload |
| `ProductHidden` | Remove the document or mark it not live |
| `InventoryChanged` | Set `inStock` |

Consumers are idempotent. A replay of the same event id does not create a second document. A nightly job compares live catalogue ids with index ids and repairs drift. It does not run inside the customer request.

---

## Admin

Staff can open a product and see whether it is in the index, the last event id applied, and the last error. They cannot edit the index by hand. They edit the catalogue, and the event updates the index.

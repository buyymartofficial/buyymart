# Catalogue

**Owns:** categories, products, variants, commercial attributes, and the product record customers are allowed to see.  
**Service:** `catalogue-service`.  
**Does not own:** the search index (search), image bytes (media), stock counts (inventory), or the order snapshot (orders).

A listing is not public until it passes the checks in this document. Hiding a product does not delete orders that already copied its fields.

---

## What a customer must be able to see before paying

The Consumer Protection (E-Commerce) Rules, 2020 require a single total price and a breakup, seller identity, country of origin, and return terms before purchase. The 2026 amendment briefings add best-before where the goods need it, and a government identifier such as GSTIN. The product page therefore shows:

| Shown | Source |
| --- | --- |
| Title, brand, images | Product |
| Variant (size, colour, other) | Variant the customer selected |
| MRP and selling price in rupees | Variant, stored in paise |
| Tax-inclusive item price as one number | Computed from taxable value and GST rate |
| Delivery fee | Empty until a pin code is set. Checkout fills it. |
| Seller legal name, registered or not, city, customer-care phone | Seller profile |
| GSTIN, when the seller has one | Seller profile |
| Country of origin | Product. Required. “India” is allowed. Imported goods also store the importer’s name. |
| Return window in days, and who pays return shipping | Category default, overridable per product |
| Stock state | Inventory: in stock or not. The exact count is not shown in version 1. |
| Best-before or use-before | Required only when the category says the goods expire. Food stays out of version 1 until FSSAI is cleared. |

Ratings from verified buyers are phase 2. Version 1 does not show a fabricated rating.

---

## Category

Categories are a tree: department, category, subcategory. Admin owns the tree. A seller picks a leaf.

| Field | Use |
| --- | --- |
| `name`, `slug`, `parentId` | Navigation and URL |
| `hsnHint` | Suggested HSN. The seller still confirms the code on the product. |
| `gstRateOptions` | Allowed rates for that leaf, for example 5, 12, 18 |
| `returnWindowDays` | Default return window |
| `returnShippingPaidBy` | `seller` or `customer`. Shown on the product page. |
| `requiresBestBefore` | Forces a date on each variant |
| `licence` | `none` in version 1. `fssai` or `bis` blocks publish until that licence is on the seller. |

---

## Product and variant

A product is the listing. A variant is what the customer adds to cart. Stock, MRP, and price sit on the variant so two sizes can differ.

| Product field | Rule |
| --- | --- |
| `sellerId` | The seller who fulfils it. BuyyMart’s own goods use the internal seller id. |
| `categoryId` | A leaf |
| `title`, `brand`, `description` | Plain text. HTML is stripped. |
| `countryOfOrigin` | ISO country name. Required. |
| `importerName` | Required when origin is not India |
| `hsn` | 4, 6, or 8 digits. Required. |
| `gstRate` | One of the category’s allowed rates |
| `status` | `draft`, `pending_review`, `live`, `hidden`, `rejected` |

| Variant field | Rule |
| --- | --- |
| `sku` | Unique per seller |
| `attributes` | Size, colour, or other axis |
| `mrpPaise` | Greater than or equal to selling price |
| `pricePaise` | What the customer pays for the item, tax inclusive |
| `bestBefore` | Required when the category says so |
| `imageIds` | At least one image before the product can go live |

Taxable value and tax are computed, not typed:

```text
taxPaise = pricePaise * gstRate / (100 + gstRate)   rounded to the nearest paise
taxablePaise = pricePaise - taxPaise
```

The order copies these numbers at purchase time. Later edits to the listing do not change an old order.

---

## Publish rules

A product can move to `live` only when all of these are true:

- Seller status is `approved` and not `suspended`.
- Title, leaf category, HSN, GST rate, country of origin, and at least one image exist.
- Every variant has MRP and price, and MRP is at least the price.
- Importer name exists when origin is not India.
- Best-before exists when the category requires it.
- Admin has approved it when the category or the seller is still on manual review. Invite-only sellers in month one can be set to auto-approve after the first ten live listings, as an admin flag on the seller.

`hidden` is reversible. `rejected` stores the staff reason and the seller can edit and resubmit.

`SellerSuspended` moves that seller’s live products to `hidden` and publishes `ProductHidden`.

---

## Events

| Event | When |
| --- | --- |
| `ProductChanged` | A live product or variant is created or edited |
| `ProductHidden` | Hidden, rejected, or seller suspended |

Search consumes both. The storefront cache (`bff:`, 30–60 seconds) may show the previous page until it expires. Checkout never trusts that cache for price.

---

## Admin and seller actions

| Action | Who | Audit |
| --- | --- | --- |
| Create or edit a draft | Seller catalogue role | No |
| Submit for review | Seller | No |
| Approve, reject, hide | Staff catalogue or trust | Yes, with reason |
| Edit a live price | Seller | Yes. The previous price is kept on the audit row. |

---

## Out of version 1

Bulk CSV, rich video, variant matrix beyond two attributes, and customer questions on the product page.

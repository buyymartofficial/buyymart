# Seller Centre web app

**URL:** `https://seller.buyymart.com`  
**Local origin:** `http://localhost:5174`  
**Users:** seller owner, catalogue staff, and orders staff, at a desk  
**Calls:** `seller-bff` only, at `https://api.buyymart.com/seller`  
**Does not call:** identity, seller, media, catalogue, inventory, order, or the payment gateway  
**Product rules:** [modules/18-applications.md](../modules/18-applications.md), [modules/13-sellers.md](../modules/13-sellers.md)  
**API contract:** [seller-bff.md](../services/seller-bff.md)

This is the build document for the seller web app. It lists the pages, the fields on each page, and the path from register to a listing and an order. Business rules stay in the services. This app renders what `seller-bff` returns and sends the next legal action.

A customer login cannot open this app. A staff login cannot open it. The build does not contain an admin screen.

---

## 1. What this build includes

| In this build | Not in this build |
| --- | --- |
| Register, sign in, forced password change, forgot, reset, logout | Phone OTP. Sellers use email and a password |
| Owner application: profile, agreement, KYC upload, submit, status | A reviewer download of the KYC file. That link is Admin Console |
| Product form, images, and on-hand stock after approval | Bulk CSV catalogue |
| Open one order by id, confirm, pack, cancel | An order queue. `seller-bff` has no `GET /seller/orders` |
| Accept or reject a return when the order body includes that return | Courier label, AWB, and tracking |
| Invite catalogue or orders staff | Statement download and payout amounts |
| A home page that follows role and seller status | A dashboard of late confirms or low stock counts |

Version 1 is English. Money on screen is rupees with two decimals. The API stores integer paise. A price of ₹499.00 is sent as `49900`. The app does not send a rupee float.

---

## 2. How the browser talks to the API

The dev server origin must be `http://localhost:5174`. Production origin is `https://seller.buyymart.com`. Any other origin is `403 origin_rejected`. Show `message`.

Every call is `fetch` with `credentials: "include"`. The session is the `bm_seller` cookie. It is `HttpOnly`, so this app cannot read it and must not copy it into `localStorage` or a query string. The JSON body is not the session.

| Response | What the app does |
| --- | --- |
| `401 session_invalid` or `401 session_expired` | Go to Sign in. Do not keep the failed page filled with another seller’s data |
| `403 forbidden` | This role cannot open the page. Go to Home |
| `403 not_approved` | Show “This seller is not approved yet.” Stay on Application or Home |
| `403 listing_blocked` | Show “This seller cannot change listings.” Leave the read-only page up |
| `503 account_pending` | Show “Setting up your account.” Retry `GET /seller/profile` for an owner. Do not send the seller back to Register |
| `503 dependency_unavailable` | Show `message`. Do not clear the form |
| `409 profile_incomplete` | Show `message` and mark each name in `missing` |
| Any other 4xx with `code` and `message` | Show `message`. Branch on `code` |

The app does not recompute GST, available stock, or the invoice number. It shows the numbers the API returned.

A file never goes to `seller-bff`. The app asks for an upload, then `PUT`s the bytes to `uploadUrl` with the same `Content-Type` and `Content-Length` it declared. That URL lasts 10 minutes and is not written to the console.

---

## 3. App flow

```text
Register or Sign in
  → cookie set by seller-bff
  → GET /seller/me
       role seller_owner
         → GET /seller/profile
              503 account_pending     wait and retry
              applied, kyc_rejected   Application, editable
              kyc_pending             Application, read only
              approved, active        Home, then Products, Stock, Orders, Staff, Bank
              suspended               Home. Stock is read only. Orders can still be confirmed
              closed                  Home. Order read only
       role seller_catalogue
         → Products, if the seller is approved or active
         → Stock read, if suspended
         → Home with not_approved, otherwise
       role seller_orders
         → Order lookup, if the seller is approved, active, suspended, or closed
         → Home with not_approved, otherwise
```

Sign out is `POST /seller/auth/logout`, then Sign in. The app clears its in-memory screens even when logout fails. The cookie is cleared by the API.

A seller who is not `approved` or `active` does not see an empty catalogue they can publish. The nav hides Products, Stock, and Orders until a call would be allowed. Catalogue staff and orders staff do not call `GET /seller/profile`.

---

## 4. Shell

Desktop first. A tablet can use it. It is not a phone storefront.

The header shows the email from `GET /seller/me` and Sign out. The owner header also shows `legalName` and `status` from the profile. Catalogue and orders staff see their role and no legal name, because profile is owner-only.

| Nav item | Who | When it is shown |
| --- | --- | --- |
| Application | Owner | Always after sign-in, including after approval, as a read-only record |
| Products | Owner, catalogue | `approved` or `active`. Hidden while `applied`, `kyc_pending`, or `kyc_rejected` |
| Stock | Owner, catalogue | `approved`, `active`, or `suspended`. Writes hidden while `suspended` |
| Orders | Owner, orders | `approved`, `active`, `suspended`, or `closed`. Confirm, pack, and cancel hidden while `closed` |
| Staff | Owner | `approved` or `active` |
| Bank | Owner | `approved`, `active`, or `suspended` |

`seller_catalogue` never sees Application, Orders, Staff, or Bank. `seller_orders` never sees Application, Products, Stock, Staff, or Bank. A direct URL to a hidden page calls the API, and `403 forbidden` returns to Home.

---

## 5. Register

`POST /seller/auth/register`

| Field | Sent | Rule on the form |
| --- | --- | --- |
| Email | `email` | Required. The API lowercases and trims it |
| Password | `password` | 12 to 128 characters. A second box checks the same value and is not sent |
| Policy | `policyVersion` | Hidden. Version 1 sends `2026-10` |

Success is `201` and no `sessionToken` in the JSON. The app then follows section 3. Wrong shape is `400 invalid_body`. An email already registered is identity’s error. Show `message`.

There is no phone field and no OTP step.

---

## 6. Sign in

`POST /seller/auth/login`

| Field | Sent |
| --- | --- |
| Email | `email` |
| Password | `password` |

Success is `200` with no `sessionToken`. Unknown email and a wrong password are both `401 credentials_invalid`. The copy is the API `message`. Do not say which of the two was wrong.

`403 password_change_required` includes `passwordChangeToken` and sets no cookie. Go to Reset password with that token held in memory for the form. Do not put the token in the address bar.

Forgot password is a link to section 7. It does not reveal whether the email exists.

---

## 7. Reset password

Forgot: `POST /seller/auth/password/forgot` with `{ email }`. Success is `202`. Show “If that email is registered, a reset link is on its way.” The link host is `https://seller.buyymart.com`. This app does not send the email.

Reset: `POST /seller/auth/password/reset` with `{ token, password }`.

| Field | Sent |
| --- | --- |
| Token | From the query string on the email link, or from `passwordChangeToken` after a forced change |
| New password | `password`, 12 to 128 characters |

Success is `204` and no cookie. Go to Sign in. The token is not a session.

---

## 8. Application

Owner only. `GET /seller/profile` fills the page. While the seller row is still being created, the page shows “Setting up your account.” and retries. It does not show an empty form as if the seller had no status.

### Status

| `status` | Page |
| --- | --- |
| `applied` | Editable. Submit is available |
| `kyc_rejected` | Editable. Show `missing`. The owner can change the profile and submit again |
| `kyc_pending` | Read only. Copy: “BuyyMart is reviewing this application.” |
| `approved`, `active` | Read only. Catalogue and orders nav appear |
| `suspended` | Read only. Copy: “This seller cannot change listings.” Orders stay available to the orders role |
| `closed` | Read only. Copy: “This seller account is closed.” |

`PUT /seller/profile` is offered only for `applied` and `kyc_rejected`. Other statuses return `409 not_editable`. Do not show Save then.

The owner read does not include `fieldsToFix`, a rejection note, the full PAN, the full account, IFSC, or a file URL. Masked PAN and account show the last four characters from `panMasked` and `bankAccountMasked`. Documents are `{ document, attached }`.

### Profile fields

`PUT /seller/profile`. The API checks these. The form blocks an obvious empty required field before the call, and still shows the API `message` when the check is richer.

| Field | JSON | Rule |
| --- | --- | --- |
| Legal name | `legalName` | Required. 1–200 characters. Shown on the invoice |
| Brand name | `brandName` | Optional. Empty sends null |
| Registered business | `registered` | Yes or no. Yes means GSTIN is required |
| Address line | `address.line1` | Required. 1–200 characters |
| City | `address.city` | Required. 1–80 characters |
| State | `address.state` | Required. One GST state name from seller-service. The GSTIN’s first two digits must match that state |
| PIN | `address.pin` | Required. Six digits |
| Customer care phone | `customerCarePhone` | Required. Ten digits, starting with 6–9 |
| Customer care email | `customerCareEmail` | Required. Must contain `@` |
| GSTIN | `gstin` | Required when registered. Empty sends null when not registered. A duplicate is `409 gstin_taken` |
| PAN | `pan` | Required on save. Five letters, four digits, one letter. The API stores it encrypted. After a reload the form shows `panMasked`, not the full value |
| Account number | `bankAccount` | Required. 9–18 digits. Encrypted. After a reload the form shows `bankAccountMasked` |
| IFSC | `ifsc` | Required. Four letters, a zero, six letters or digits |
| Account name | `accountName` | Required. 1–200 characters |
| Grievance officer | `grievanceOfficerName` | Required before submit. 1–200 characters |
| Grievance phone | `grievancePhone` | Required before submit. Same phone rule |
| Grievance email | `grievanceEmail` | Required before submit. Same email rule |

`licence` stays null in version 1. Do not show an FSSAI or BIS control.

### Agreement

`POST /seller/agreement` with `{ agreementVersion }`. Version 1 sends `2026-10`. The control is a checkbox, “I accept the seller agreement,” and it is not pre-checked. Success stores `agreementVersion` on the next profile read. Submit stays disabled until `missing` no longer contains `agreement`.

### KYC files

Three slots: PAN, GST certificate, bank proof. GST certificate is required only when `registered` is true. The slot shows `attached` from the profile.

Upload, for the owner, while the profile is still editable:

1. `POST /seller/uploads` with `{ purpose: "kyc", document, contentType, byteSize }`. `document` is `pan`, `gst_certificate`, or `bank_proof`. Content type is `application/pdf`, `image/jpeg`, or `image/png`. Size is 1 through 8388608 bytes.
2. `PUT` the file to `uploadUrl`.
3. `GET /seller/uploads/:id` until `status` is `ready` or `rejected`. Rejected shows “Upload failed.”
4. `POST /seller/kyc` with `{ assetId, document }` using the upload id. `409 kyc_not_ready` means the file is not ready yet. Wait and retry this step. Do not upload a second object for the same attempt.

The page never offers a download of the KYC object.

### Submit

`POST /seller/submit`. Enabled when the form has been saved, the agreement is accepted, and each required document is `attached`. The API still decides. `409 profile_incomplete` returns `missing`. Mark those fields. Names are `legalName`, `address`, `gstin`, `pan`, `bank`, `grievance`, `agreement`, and `gst_certificate`.

Success moves the page to read only with status `kyc_pending`.

---

## 9. Home

Home is the router in section 3. It is not a queue.

| Role | Home shows |
| --- | --- |
| Owner, not yet approved | Status, and a link to Application |
| Owner, approved or active | Links to Products, Stock, Orders, Staff, and Bank |
| Owner, suspended | The listing-blocked sentence, a link to Stock (read only), and a link to Orders |
| Owner, closed | The closed sentence, and a link to Orders (read only) |
| Catalogue | A link to Products, or the not-approved sentence |
| Orders | A link to Orders, or the not-approved sentence |

Do not count unconfirmed orders, late dispatch, or low stock on this page. Those counts need a list the API does not have. Stock levels are on the Stock page, one row at a time from `GET /seller/stock`.

---

## 10. Products

`GET /seller/products`. Owner or catalogue, and the seller is `approved` or `active`. Suspended sellers do not get this list, because catalogue reads are `403 listing_blocked` while suspended. They can still open Stock.

Each row shows title, status, and a link to the form. Status is `draft`, `pending_review`, `live`, `hidden`, or `rejected`. There is no delete.

`rejected` shows the reason when the product body includes it. The seller edits and submits again. `hidden` is read only until staff publish it again. The seller does not see an Approve button.

New product is `POST /seller/products` and opens the form on the new id.

---

## 11. Product form

Owner or catalogue. Writes require `approved` or `active`.

The category control is a leaf from `GET /seller/categories`. A parent category cannot be saved as the product category. The leaf’s GST options fill the GST control. The leaf’s HSN hint is shown as hint text and is not copied into `hsn`.

### Product

`POST /seller/products` creates a draft. Later edits are `PATCH /seller/products/:id`.

| Field | JSON | Rule |
| --- | --- | --- |
| Category | `categoryId` | A leaf |
| Title | `title` | 1–160 characters. Required before submit |
| Brand | `brand` | 1–80 characters |
| Description | `description` | Up to 4000 characters. Plain text |
| Country of origin | `countryOfOrigin` | Required. “India” is allowed |
| Importer | `importerName` | Required when origin is not India. 1–160 characters |
| HSN | `hsn` | 4, 6, or 8 digits. Required. The seller types it |
| GST rate | `gstRate` | One of the leaf’s options |
| Return window | `returnWindowDays` | Empty uses the category default. Otherwise 0 through 30 |
| Return shipping | `returnShippingPaidBy` | Empty uses the category default. Otherwise `seller` or `customer` |

The app does not compute `taxPaise`. Catalogue does that at publish.

### Variant

`POST /seller/products/:id/variants` and `PATCH /seller/variants/:id`.

| Field | JSON | Rule |
| --- | --- | --- |
| SKU | `sku` | 1–64 characters. Unique for this seller |
| Attributes | `attributes` | At most two, for example size and colour. Each value is 1–40 characters. A third is `400 attribute_limit` |
| MRP | `mrpPaise` | Integer paise. The form shows rupees. MRP is at least the price |
| Price | `pricePaise` | Integer paise, greater than 0. Tax inclusive |
| Weight | `weightGrams` | Integer greater than 0. Required before the listing can go live |
| Best before | `bestBefore` | Required when the category says so. A date |

A live listing can change MRP and price only. Other live fields go back through edit and submit, to `pending_review`, unless the seller is auto-approved. If the API returns `409`, show `message` and do not pretend the edit was saved.

### Images

One or more images on each variant. A listing cannot go live with zero ready images.

1. `POST /seller/uploads` with `{ purpose: "product", contentType, byteSize }`. Content type is `image/jpeg`, `image/png`, or `image/webp`. The same size cap as KYC. Do not send `document`.
2. `PUT` the file to `uploadUrl`.
3. Poll `GET /seller/uploads/:id` until `ready` or `rejected`. A ready product row may include a CDN URL. Show that image. A KYC-style download URL is not used here.
4. `POST /seller/variants/:id/images` with `{ imageId, position }`. `position` starts at 0.

### Submit listing

`POST /seller/products/:id/submit`. Success is `pending_review`, or `live` when catalogue auto-approves. `409 publish_blocked` lists what is still wrong. Show `message`. The seller cannot approve their own listing.

---

## 12. Stock

`GET /seller/stock` for owner or catalogue while `approved`, `active`, or `suspended`.

| Column | Source | Editable |
| --- | --- | --- |
| Variant | The row’s variant id | No |
| On hand | `onHand` | Yes, except while suspended or closed |
| Reserved | `reserved` | No. The form does not send it |
| Available | `onHand - reserved` when both are present. Otherwise show the API’s available | No |

Save is `PUT /seller/stock/:variantId` with `{ onHand }`. `onHand` is an integer from 0 through 1000000. A value below `reserved` is `400 stock_below_reserved`. Show `message`. Do not retry with a lower number.

Suspended sellers see the table and no save control. The API would answer `403 listing_blocked`.

---

## 13. Order

There is no list. The page is one field, Order id, and `GET /seller/orders/:id`. Owner or orders staff. Another seller’s order is `404 not_found`. Show “Not found.” Do not say it belongs to someone else.

The read shows:

| Field | Use |
| --- | --- |
| `id` | The order id |
| `status` | The order status. The actions below use the seller group’s `status` |
| `payMode` | `prepaid` or COD |
| `address` | Ship-to snapshot: name, phone, line, city, state, PIN. Later edits to the customer’s address book do not change it |
| `groups` | This seller’s group only |
| `groups[].status` | What confirm, pack, and cancel look at |
| `groups[].lines[]` | `variantId`, `productId`, `quantity`, `pricePaise` shown as rupees |

The page does not show the customer’s email, their other addresses, or another seller’s lines. `payablePaise` on the order is the customer’s order total. Label it as the order total, not as this seller’s settlement.

### Actions

Show only the next legal action for that group.

| Group status | Control | Call |
| --- | --- | --- |
| `paid` or `cod_pending` | Confirm | `POST /seller/orders/:id/confirm` |
| `confirmed` | Mark packed | `POST /seller/orders/:id/pack` |
| Before `shipped` | Cancel | `POST /seller/orders/:id/cancel` with `{ reason: "seller_unavailable" }` |
| `packed` and later, until a label exists | None for shipping | Pack does not book a courier |

Confirm returns `invoiceNumber`. Show it. The shape is `{year}-{n}`. The app does not invent the next number.

Cancel is one button, not a list of staff or customer reasons. The body reason is `seller_unavailable`.

Mark packed copy: “Marked packed. A courier label is not available in this version.” Do not draw a fake label or AWB.

While `suspended`, confirm, pack, and cancel stay available. While `closed`, the page is read only.

### Returns

If the order body includes return rows with a `returnId`, each row shows the line, the quantity, and the customer’s reason. Accept and reject are shown only for a return that is still open.

| Control | Body |
| --- | --- |
| Accept | `POST …/returns/:returnId/accept` with `{ qc }` of `sellable`, `damaged`, or `wrong_item` |
| Reject | `POST …/returns/:returnId/reject` with `reason` of `used`, `wrong_photos`, or `not_our_item`, and `note` of at least 10 characters |

The current order read returns the group and lines. It does not include return rows. Until it does, this block says there is nothing to decide. Do not call another service to find return ids. The seller cannot override a rejection. That is Admin Console.

---

## 14. Staff

Owner only, after approval. `POST /seller/staff`.

| Field | JSON | Rule |
| --- | --- | --- |
| Email | `email` | Required |
| Temporary password | `password` | 12 to 128 characters. Shown once on this form. Not shown again after success |
| Role | `role` | `seller_catalogue` or `seller_orders` |

There is no owner role in the list. The owner cannot create a second owner here.

Success is `201`. Tell the owner that the new person must change the password at first sign-in. `403 password_change_required` on that first sign-in is section 6.

Catalogue staff cannot open this page. Orders staff cannot open it.

---

## 15. Bank change

Owner only, when the seller is `approved`, `active`, or `suspended`. This is not the application form. The live account stays in use until finance approves the change in Admin Console.

`POST /seller/bank`

| Field | JSON | Rule |
| --- | --- | --- |
| Account number | `bankAccount` | 9–18 digits |
| IFSC | `ifsc` | Four letters, a zero, six letters or digits |
| Account name | `accountName` | 1–200 characters |
| Bank proof | `assetId` | A ready KYC upload with `document: "bank_proof"` |

The proof uses the same upload steps as section 8, then this call. Success is `202`. Copy: “Finance has not approved this change. Payouts stay on the current account.” The masked account on Application does not change on this response.

`409 bank_pending` means a change is already waiting. Do not send a second one. `POST /seller/bank/approve` is not a seller page. It is `404` here.

---

## 16. Sign out and errors on every page

Sign out calls `POST /seller/auth/logout` and then shows Sign in.

Pages share one error line. `message` is safe to show. `requestId` can sit in a details disclosure for support. It is not the headline.

The app does not log PAN, account number, IFSC, account name, password, reset token, or `uploadUrl`.

---

## 17. Local run

Run the app at `http://localhost:5174` so `seller-bff` allows the origin. Point API calls at `http://localhost:8099`, which is seller-bff’s published port. Identity, seller, media, catalogue, inventory, and order must already be running for a real sign-in. This app has no database and no `.env` secret. The service JWT stays on seller-bff.

A browser walk-through is the test. Automated checks cover the routing and the request bodies. They do not need a live seller-bff when the client is injected.

| Check | Expected |
| --- | --- |
| Register | `POST /seller/auth/register` with email, password, and `policyVersion`. No token stored in `localStorage` |
| Wrong password | Stay on Sign in. `message` is shown. No catalogue request |
| Forced change | Reset opens with the token in memory. After `204`, Sign in is shown |
| Owner, profile missing | “Setting up your account.” Retry. No product form |
| Owner, `applied` | Application is shown. Products, Stock, and Orders are not in the nav |
| Submit with `missing: ["agreement"]` | Agreement is marked. Status does not become `kyc_pending` |
| KYC file | Bytes go to `uploadUrl`, then `POST /seller/kyc` after `ready` |
| Catalogue staff opens Application | `403 forbidden` returns to Home. `PUT /seller/profile` is not called |
| `applied` seller opens Products | The not-approved sentence. `POST /seller/products` is not called |
| Suspended seller opens Stock | Rows are visible. Save is hidden |
| Confirm | `POST /seller/orders/:id/confirm` with no `sellerId` in the body. `invoiceNumber` is shown |
| Cancel | Body is `{ reason: "seller_unavailable" }` |
| Unknown order id | “Not found.” |
| Pack | No label and no AWB control |
| Staff invite | Role is only `seller_catalogue` or `seller_orders` |
| Bank change `202` | The waiting copy. The masked account is not replaced |
| Sign out | Login is shown |

---

## 18. Later pages

Do not build these until `seller-bff` has the route.

| Page | What it waits on |
| --- | --- |
| Order queue, late confirm | `GET /seller/orders` |
| Label, AWB, tracking | Fulfilment on seller-bff |
| Statement and net payout | Settlement on seller-bff. Version 1 pays by a transfer outside this app |
| Return list independent of one order | Return rows on the order read |
| Bulk catalogue CSV | Not version 1 |

When the label exists, a failure shows the error and does not mark the group packed by itself. The statement is read only and never shows a full account number. The bank account on file stays masked.

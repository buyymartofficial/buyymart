# Admin Console web app

**URL:** `https://admin.buyymart.com`  
**Local origin:** `http://localhost:5175`  
**Users:** staff only, at a desk. Roles are `support`, `catalogue`, `trust`, `finance`, `admin`, and `super_admin`  
**Calls:** `admin-bff` only, at `https://api.buyymart.com/admin`  
**Does not call:** identity, seller, catalogue, order, payment, search, or the payment gateway  
**Product rules:** [modules/18-applications.md](../modules/18-applications.md), [modules/01-identity.md](../modules/01-identity.md), [modules/13-sellers.md](../modules/13-sellers.md)  
**API contract:** [admin-bff.md](../services/admin-bff.md)

This is the build document for the staff web app. It lists the pages, the fields on each page, and the path from sign-in through MFA to a seller review, a listing decision, and an order. Business rules stay in the services. This app renders what `admin-bff` returns and sends the next legal action.

A customer login cannot open this app. A seller login cannot open it. The build does not contain a Seller Centre screen.

---

## 1. What this build includes

| In this build | Not in this build |
| --- | --- |
| Sign in, authenticator enrol, authenticator verify, forgot, reset, logout | A session cookie on the password step. The cookie is set only after MFA |
| Home with the links this role may open | A home queue of late orders, KYC waiting, payment mismatches, tickets, or settlement exceptions. No service has that combined list |
| Open one seller by id. Review, suspend, close, KYC link, bank decision, payout read | A seller search. There is no `GET /admin/sellers` |
| Create and edit a category. Approve, reject, or hide one product by id. Inspect one variant in search | Seller catalogue writes, and a product queue |
| Open one order by id. Staff cancel, return override, COD refund reference | Confirm and pack. Those stay on Seller Centre. `POST /admin/orders/:id/confirm` is `404` |
| Read one payment. Staff refund with a second staff id when the amount requires it | The payment webhook |
| Ban a customer by id | Customer search, including lookup by phone. There is no `GET /admin/customers` |
| Create a staff user and change a role | A staff profile. `GET /admin/me` does not return an email |

Version 1 is English. Money on screen is rupees with two decimals. The API stores integer paise. ₹499.00 is sent as `49900`. The app does not send a rupee float.

Coupons, settlement exceptions, shipment labels, and support tickets wait until those services exist. Do not add empty pages for them.

---

## 2. How the browser talks to the API

The dev server origin must be `http://localhost:5175`. Production origin is `https://admin.buyymart.com`. Any other origin is `403 origin_rejected`, message `That origin is not allowed.`

Every call is `fetch` with `credentials: "include"`. The session is the `bm_staff` cookie. It is `HttpOnly`, `Path=/admin`, `Max-Age=28800`. This app cannot read it and must not copy it into `localStorage`, `sessionStorage`, or a query string. The JSON body is not the session.

| Response | What the app does |
| --- | --- |
| `401 session_invalid` or `401 session_expired` | Go to Sign in. Drop every in-memory screen, including a payout and a KYC link |
| `403 forbidden` | This role cannot open the page. Go to Home. Show `message` |
| `403 origin_rejected` | Show `message`. Do not retry on another origin |
| `503 dependency_unavailable` | Show `message`. Do not clear the form. Do not clear the cookie |
| `400 invalid_body` | Show `message`. Leave the form as the person typed it |
| `404 not_found` | Show `Not found.` Do not say whether another role could open it |
| `409` with `code` and `message` | Show `message`. Branch on `code` |
| Any other 4xx with `code` and `message` | Show `message`. Branch on `code` |

`message` is safe to show. `requestId` can sit in a details disclosure. It is not the headline.

The app does not recompute tax, available stock, invoice numbers, or whether a variant is in the search index. It shows the values the API returned.

The app does not log a password, TOTP code, `mfaToken`, `mfaEnrollmentToken`, `otpauthUrl`, reset token, cookie, PAN, bank account, IFSC, account name, or a KYC `url`.

---

## 3. App flow

```text
Sign in
  → POST /admin/auth/login
       200 { mfaToken }                         Verify code
       200 { mfaEnrollmentToken, otpauthUrl }   Enrol, then confirm
       403 password_change_required             Set a new password, token in memory, no cookie
       401 credentials_invalid                  Stay on Sign in. One message
  → POST /admin/auth/mfa/verify    or    /admin/auth/mfa/confirm
       cookie set by admin-bff. JSON has no sessionToken
  → GET /admin/me
       { subjectId, role, sessionId }
       → Home, then only the nav in section 4
```

Login does not set a cookie. An enrol token and an MFA token are not sessions. Hold them in memory until verify or confirm succeeds, then drop them. Do not put them in the address bar.

`403 password_change_required` includes `passwordChangeToken` and sets no cookie. Open Set a new password with that value in memory.

Sign out is `POST /admin/auth/logout`, then Sign in. Clear in-memory screens even when logout fails. The API clears the cookie.

There is no email on `GET /admin/me`. The header shows `role` and `subjectId` from that response. Do not invent a profile call.

---

## 4. Shell

Desktop first. A tablet can use it. It is not a phone storefront.

The header shows the role, the staff id, and Sign out.

| Nav item | Who sees it | Page |
| --- | --- | --- |
| Home | Every staff role | Links only |
| Sellers | `trust`, `admin`, `super_admin` | Seller id, then review |
| Bank | `finance`, `admin`, `super_admin` | Seller id, then bank decision and payout |
| Catalogue | `catalogue`, `trust` | Category form, product decision, variant inspect |
| Orders | Every staff role | Order id, then the order |
| Customers | `trust`, `admin`, `super_admin` | Ban by customer id |
| Staff | `admin`, `super_admin` | Create a user, change a role |

`admin` and `super_admin` are not catalogue reviewers in version 1. They do not see Catalogue. A direct URL still calls the API, and `403 forbidden` returns to Home. Catalogue-service would reject them.

`finance` does not see Sellers. Bank is a separate page because `GET /admin/sellers/:id` is trust-only. A finance session on that read is `403 forbidden`.

`support` sees Home and Orders. Cancel is on the order page. Ban, seller review, catalogue, bank, and staff are hidden.

`catalogue` sees Home, Catalogue, and Orders. Order cancel is hidden for `catalogue` and `finance`. They can still read the order.

A payment read sits on the order page, because the path is the order id. The refund form is on that same page and is shown only to `finance`, `admin`, and `super_admin`.

---

## 5. Sign in

`POST /admin/auth/login`

| Field | Sent |
| --- | --- |
| Email | `email` |
| Password | `password` |

Success is `200` and no `Set-Cookie`. The body is either `{ mfaToken }` or `{ mfaEnrollmentToken, otpauthUrl }`. Follow section 6. There is no `sessionToken` in the JSON.

Unknown email and a wrong password are both `401 credentials_invalid`. The copy is the API `message`, which is `Email or password is wrong.` Do not say which of the two was wrong.

Forgot password is a link to section 7.

---

## 6. Authenticator

First sign-in for an account with no confirmed authenticator returns `mfaEnrollmentToken` and `otpauthUrl`. Show the URL as a QR code and as text the person can copy into an authenticator app. Do not log it. Do not store it after confirm succeeds.

| Step | Call | Body |
| --- | --- | --- |
| Enrol | `POST /admin/auth/mfa/confirm` | `{ token: mfaEnrollmentToken, code }` |
| Later sign-in | `POST /admin/auth/mfa/verify` | `{ token: mfaToken, code }` |

`code` is what the authenticator app shows. It is not logged.

Success is `200`, `Set-Cookie: bm_staff`, and no `sessionToken` in the JSON. Go to Home and call `GET /admin/me`.

A wrong code is identity’s error, forwarded. Show `message`. At five wrong codes identity deletes the MFA token. This app does not count the attempts. When that happens, go back to Sign in and drop the in-memory token.

The confirm and verify pages have no other fields.

---

## 7. Reset password

Forgot: `POST /admin/auth/password/forgot` with `{ email }`. Success is `202`. Show “If that email is registered, a reset link is on its way.” The link host is `https://admin.buyymart.com`. This app does not send the email.

Reset: `POST /admin/auth/password/reset` with `{ token, password }`.

| Field | Sent |
| --- | --- |
| Token | From the query string on the email link, or from `passwordChangeToken` after a forced change. Read it once into memory, then remove it from the address bar |
| New password | `password`, 12 to 128 characters. A second box checks the same value and is not sent |

The password cannot be the email. Identity returns `400 password_invalid`. Show `message`.

Success is `204` and no cookie. Go to Sign in. The token is not a session.

---

## 8. Home

`GET /admin/me` has already filled the header. Home does not call another service.

The page lists the nav items from section 4 for this role and nothing else. There is no count of KYC waiting, unconfirmed orders, payment mismatches, tickets, or settlement exceptions. Those lists are not routes on `admin-bff`.

Each link goes to an id form, except Staff and Catalogue, which open their own forms.

---

## 9. Seller review

Trust, admin, and super admin. The first screen is one field, Seller id, a UUID. Look up is `GET /admin/sellers/:id`.

A malformed id is `400 invalid_body` before the role check. Show `message`. `404 not_found` shows `Not found.`

The read is the staff review shape. It includes the operational seller, the address, masked PAN, masked bank account, grievance officer name, agreement version, document names with `attached`, `missing`, `fieldsToFix`, and `bankPending`. It does not include the full PAN, the full account, IFSC, or the account name. Those are section 10.

| Shown | Source |
| --- | --- |
| Legal name, brand, status, city, state, GSTIN | Top-level fields |
| Address | `address.line1`, `city`, `state`, `pin` |
| Customer care | `customerCarePhone`, `customerCareEmail` |
| PAN | `panMasked` only |
| Bank account | `bankAccountMasked` only |
| Agreement | `agreementVersion` |
| Documents | `documents[]` of `pan`, `gst_certificate`, `bank_proof`, each `attached` true or false |
| Still missing | `missing` |
| Sent back to the seller | `fieldsToFix` |
| Bank change waiting | `bankPending` |

There is no seller edit form. Trust does not type a new GSTIN here.

### Actions

Each action is a button on this page. Notes and reasons are at least 10 characters. A shorter value is `400 invalid_body` and seller-service is not called.

| Button | Call | Body |
| --- | --- | --- |
| Approve | `POST /admin/sellers/:id/approve` | `{ note }` |
| Reject | `POST /admin/sellers/:id/reject` | `{ note, fields }` |
| Suspend | `POST /admin/sellers/:id/suspend` | `{ reason }` |
| Reinstate | `POST /admin/sellers/:id/reinstate` | `{ note }` |
| Close | `POST /admin/sellers/:id/close` | `{ reason }` |
| Auto-approve | `PATCH /admin/sellers/:id/auto-approve` | `{ enabled }` boolean |

`fields` is one or more of `legalName`, `address`, `gstin`, `pan`, `bank`, `grievance`, `agreement`, `gst_certificate`. An empty list is `400 invalid_body`. The checkboxes start unchecked.

`409 not_reviewable` shows `message` and does not pretend the status changed. After a `200`, reload `GET /admin/sellers/:id`.

### KYC link

One button per attached document: `POST /admin/sellers/:id/kyc/:document/link`. `document` is `pan`, `gst_certificate`, or `bank_proof`.

Success is `{ url, expiresAt }`. The URL lasts 60 seconds. Show it and open it in a new tab. Do not write it to the console, do not store it, and do not offer a download that keeps a copy. When the page is left, drop the URL. This app does not call media-service.

`POST /admin/sellers/:id/bank` is not a button. That path is `404`. The seller asks for the change in Seller Centre.

---

## 10. Bank and payout

Finance, admin, and super admin. The first screen is the seller id. This page does not call `GET /admin/sellers/:id`.

| Button | Call | Body | Who |
| --- | --- | --- | --- |
| Approve bank change | `POST /admin/sellers/:id/bank/approve` | none | Finance |
| Reject bank change | `POST /admin/sellers/:id/bank/reject` | `{ note }` of at least 10 characters | Finance |
| Show payout account | `GET /admin/sellers/:id/payout` | none | Finance |

`409 bank_pending` on approve means there is nothing waiting, or the change cannot be applied. Show `message`. Do not send a second approve.

The payout body is `legalName`, `gstin`, `pan`, `bankAccount`, `ifsc`, `accountName`. Show it on this page only. Do not log it. Do not put it in `localStorage`. Clear it when the person leaves the page or signs out. A trust or catalogue session never reaches this call. The API returns `403 forbidden` and no account number.

---

## 11. Catalogue

`catalogue` and `trust` only. Admin and super admin are not in this nav.

### Category

Create: `POST /admin/categories`. Edit: `PATCH /admin/categories/:id` after the person pastes the category id. There is no category list route, so the form does not pretend to browse a tree.

| Field | Sent | Rule |
| --- | --- | --- |
| Name | `name` | Required on create |
| Slug | `slug` | Required on create |
| Parent | `parentId` | UUID or empty, sent as `null` when empty |
| HSN hint | `hsnHint` | Optional. Empty is `null`. Not copied into a product |
| GST rates | `gstRateOptions` | One or more integers from 0 to 100 |
| Return window | `returnWindowDays` | Integer 0 to 30 |
| Return shipping | `returnShippingPaidBy` | `seller` or `customer` |
| Best before | `requiresBestBefore` | Boolean |
| Licence | `licence` | `none`, `fssai`, or `bis`. Omit when unchanged on edit |

Create success is `201`. Edit success is `200`. Show the category the API returned. A partial edit sends only the fields the person changed.

### Product decision

One field, Product id. There is no product read on `admin-bff`, so this page does not show a title it does not have. The person pastes the id from outside this app.

| Button | Call | Body |
| --- | --- | --- |
| Approve | `POST /admin/products/:id/approve` | `{ reason }` |
| Reject | `POST /admin/products/:id/reject` | `{ reason }` |
| Hide | `POST /admin/products/:id/hide` | `{ reason }` |

`reason` is at least 10 characters. A shorter reason is `400 invalid_body` and catalogue-service is not called. Finance is `403 forbidden`. After `200`, show `message` from a failure or the returned product body on success. The seller still cannot approve their own listing. This page does not offer a seller Approve button.

---

## 12. Search inspect

On the Catalogue page. One field, Variant id. `GET /admin/variants/:id`.

Show `variantId`, `inIndex`, `lastEventId`, `lastEventAt`, `lastError`, and `updatedAt`. `inIndex: false` is a real answer for a variant that was never indexed. Do not draw an edit form. There is no write route. Staff cannot change the index from this app.

A malformed id on this route is `404 not_found`, not `400`. Show `Not found.`

---

## 13. Orders

Every staff role can open this page. The first screen is Order id. `GET /admin/orders/:id`.

Another id is `404 not_found`. Show `Not found.`

Show `id`, `status`, `payMode`, `payablePaise` labelled “Order total” in rupees, the address snapshot, and every seller group the body includes. Staff see the whole order, not one seller’s lines. Lines are `variantId`, `productId`, `quantity`, `pricePaise`. Do not invent a product title.

| Control | Who | Call | Body |
| --- | --- | --- | --- |
| Cancel | `support`, `trust`, `admin`, `super_admin` | `POST /admin/orders/:id/cancel` | `{ reason: "staff" }` only |
| Override a rejected return | Those same roles | `POST /admin/orders/:id/returns/:returnId/override` | No body. The return id comes from the order |
| COD refund reference | `finance`, `admin`, `super_admin` | `POST /admin/orders/:id/returns/:returnId/cod-refund` | `{ reference }` |

Cancel is one button. The app sends `staff` and does not offer `changed_mind` or `seller_unavailable`.

If the order body has no return id, show “Returns: nothing to decide until this order includes a return id.” Do not call another service to find one. Override success is `{ status: "return_accepted" }`. COD success is `{ status: "refunded" }`. Show that status. Do not invent the next state.

Confirm and Mark packed are not on this page. Those paths are `404`.

`catalogue` and `finance` see the order and do not see Cancel or Override. Finance sees the COD reference field when a return id is present.

---

## 14. Payments and refunds

On the order page, after the order has loaded. `GET /admin/payments/:orderId` uses the same id. Any staff role may read it.

Show `paymentId`, `orderId`, `status`, `amountPaise` in rupees, `currency`, `method`, `gatewayOrderId`, and `gatewayPaymentId`. A missing payment is `404 not_found`.

The refund form is finance, admin, and super admin.

| Field | Sent |
| --- | --- |
| Amount | `amountPaise`, integer, from the rupee field. Empty or a float is not sent |
| Variant | `variantId`, UUID or empty. Empty is `null` |
| Reason | Hidden. Always `staff` |
| Confirmed by | `confirmedBy`, a staff UUID, or empty as `null`. Do not fill this from `GET /admin/me` |

The request also sends header `Idempotency-Key`, a new UUID for that submit. Keep the same key if the person retries that same submit. A new amount gets a new key. A missing or non-UUID key is `400 invalid_body` and payment-service is not called.

`POST /admin/payments/:orderId/refunds` success is `202`. The order is not marked refunded by this response. Show the refund body the API returned.

`409 confirm_required` means the amount needs a second staff id. Show `message`. Leave `confirmedBy` empty until someone else types their id. `403 forbidden` when `confirmedBy` is the caller is payment-service’s rule, forwarded. Show `message`. Do not swap in the current `subjectId`.

`409 idempotency_conflict` shows `message`. Do not fire a second refund with a new key for the same click.

There is no webhook page.

---

## 15. Customer ban

Trust, admin, and super admin. One customer id and a reason of at least 10 characters.

`POST /admin/customers/:id/ban` with `{ reason }`. Success is `200`. Identity bans the customer, revokes their sessions, and writes the audit row. This app does not look the customer up and does not show their orders. There is no customer search box and no phone field.

A short reason is `400 invalid_body`. Show `message`.

---

## 16. Staff accounts

Admin and super admin.

### Create

`POST /admin/staff`

| Field | Sent | Rule |
| --- | --- | --- |
| Email | `email` | Must be on the staff email domain identity is configured with. Anything else is identity’s error. Show `message` |
| Temporary password | `password` | 12 to 128 characters. Not the email. Shown once on this form after `201`, then cleared. Not logged |
| Role | `role` | `support`, `catalogue`, `trust`, `finance`, `admin`, or `super_admin` |
| Reason | `reason` | At least 10 characters |

Success is `201`. Tell the admin that the new person must sign in and set up an authenticator. There is no second copy of the password after they leave the page.

### Change a role

The first field is the staff id. `PATCH /admin/staff/:id` with `{ role, reason }`. Reason is at least 10 characters.

A person cannot change their own role. Identity returns `403`. Show `message`. Do not hide the form just because the id matches `subjectId`. The API decides. Granting `super_admin` is also identity’s rule. Show `message` when it refuses.

---

## 17. Sign out and errors on every page

Sign out calls `POST /admin/auth/logout` and then shows Sign in.

Pages share one error line. A payout, a KYC URL, an enrol URL, and a temporary password are wiped on sign-out and on `401`.

---

## 18. Local run

Run the app at `http://localhost:5175` so `admin-bff` allows the origin. Point API calls at `http://localhost:8100`, which is admin-bff’s published port. Identity, seller, catalogue, order, payment, and search must already be running for a real sign-in. This app has no database and no `.env` secret. The service JWT stays on admin-bff.

A browser walk-through is the test. Automated checks cover routing and request bodies with an injected client. They do not need a live admin-bff.

| Check | Expected |
| --- | --- |
| Login | `POST /admin/auth/login`. No cookie is stored by the app. No `sessionToken` is read from JSON |
| Wrong password | Stay on Sign in. `message` is shown. No seller request |
| Enrol | `otpauthUrl` is shown. Confirm sends `{ token, code }` and then Home |
| Verify | `POST /admin/auth/mfa/verify`. After `200`, `GET /admin/me` fills the header |
| Forced change | Reset opens with the token in memory. After `204`, Sign in is shown and the address bar has no token |
| `finance` opens Sellers | Nav hides it. A direct URL gets `403 forbidden` and returns to Home. `GET /admin/sellers/:id` is not used as the bank page |
| `trust` opens payout | `403 forbidden`. The screen has no account number |
| `catalogue` opens Staff | `403 forbidden` returns to Home |
| Reject seller | Body has `note` and a non-empty `fields` list |
| KYC link | The URL is shown and is not written to the console |
| Hide product | `{ reason }` of at least 10 characters. No product title is invented |
| Search inspect | `inIndex: false` is shown as not indexed |
| Staff cancel | Body is `{ reason: "staff" }` |
| Confirm or pack | Those buttons do not exist |
| Refund | `Idempotency-Key` is a UUID. `reason` is `staff`. `confirmedBy` is not copied from `subjectId` |
| Refund without a key | The client does not send the POST |
| Ban | `{ reason }` only. No customer search request |
| Create staff | Email, password, role, and reason. The password is not logged |
| Sign out | In-memory payout and KYC URL are gone. Sign in is shown |

---

## 19. Later screens, not this build

Do not add these until `admin-bff` has the route.

| Screen | Why it waits |
| --- | --- |
| Home queue | No domain service has the combined list |
| Customer search | Staff can ban by id. Identity has no phone lookup |
| Shipment and label | `fulfilment-service` is not deployed |
| Settlement exceptions | `settlement-service` is not deployed. When it exists, that screen still does not show a decrypted bank account. Finance keeps using section 10 |
| Coupons | `promotion-service` is not deployed |
| Tickets near 48 hours | `support-service` is not deployed |

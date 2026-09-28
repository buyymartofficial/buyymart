# Applications

**Owns:** what each human-facing app shows. Business rules stay in the modules. The apps call their BFF and render the result.  
**Version 1:** customer storefront, Seller Centre, Admin Console.  
**Later:** Android, iOS, Warehouse Console.

---

## Customer storefront

**URL:** `https://buyymart.com`  
**Users:** guests and customers, mostly on a phone browser.  
**BFF:** `customer-bff`

| Screen | Modules it reads |
| --- | --- |
| Home | Catalogue categories, admin banners |
| Category and search | Search, catalogue fallback |
| Product | Catalogue, inventory, seller public profile, media |
| Cart | Cart, catalogue, inventory |
| Checkout | Checkout, fulfilment pin, promotions, orders, payments |
| Payment result | Orders. It does not decide paid. |
| Orders and tracking | Orders, fulfilment scans |
| Return | Returns |
| Account, addresses, delete account | Identity |
| Ticket | Support |
| Policies, grievance officer | Static content and the support configuration |

Guests can browse and fill a cart. The pay step requires OTP. English only in version 1.

The footer shows legal name, address, contact, terms, privacy, shipping, returns, and the grievance officer. Those pages exist before the first live payment.

---

## Seller Centre

**URL:** `https://seller.buyymart.com`  
**Users:** seller owner and seller staff, at a desk.  
**BFF:** `seller-bff`

| Screen | Modules |
| --- | --- |
| Register and agreement | Identity, sellers |
| KYC upload and status | Sellers, media |
| Dashboard | Orders due, late confirm, low stock |
| Product form | Catalogue, media |
| Inventory | Inventory |
| Order queue and label | Orders, fulfilment |
| Returns | Returns |
| Statement download | Settlement |
| Staff | Identity, sellers |

A seller who is not `approved` sees only registration, KYC, and status. They do not see an empty catalogue they can publish.

---

## Admin Console

**URL:** `https://admin.buyymart.com`  
**Users:** staff only. Email, password, MFA.  
**BFF:** `admin-bff`, role check on every route.

| Screen | Modules |
| --- | --- |
| Login and MFA | Identity |
| Home queue | Orders unconfirmed past SLA, KYC waiting, payment amount mismatches, tickets near 48 hours, settlement exceptions |
| Order detail | Orders, payments, fulfilment, returns |
| Customer lookup | Identity, orders. Requires an id. |
| Seller review | Sellers |
| Catalogue moderation | Catalogue |
| Coupons | Promotions |
| Refund queue | Payments, maker-checker |
| Settlement exceptions | Settlement |
| Staff and roles | Identity |
| Audit | Audit |

Refunds above the configured amount need a second staff member in finance. The person who created the request cannot confirm it.

---

## Later applications

| App | When | Rule |
| --- | --- | --- |
| Android `com.buyymart.app` | After the website can take a payment, cancel, and refund in staging | Same API. Adds push and in-app account deletion. |
| iOS `com.buyymart.app` | After Android has been stable for a release | Same API, plus a reviewer demo account. |
| Warehouse Console | Only if BuyyMart stores stock | Pick, scan, pack, inward, return QC. Until then, Seller Centre prints the label. |
| Rider app | Only if BuyyMart employs riders | Not version 1. The courier’s app does pickup and COD. |

---

## What the apps must not do

- Call the payment gateway, SMS provider, or courier with a secret from the browser.
- Recompute tax or stock differently from the API.
- Ship an admin screen inside the seller build.
- Hide the seller’s name on a marketplace invoice.

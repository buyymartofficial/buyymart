# BuyyMart — App Specification

**Brand:** BuyyMart  
**Domain:** buyymart  
**Market:** India (INR, GST invoices, pin-code delivery)  
**Document purpose:** Define what BuyyMart is, which applications the ecosystem needs, the type of each application, and what each one must do.

Company, tax, and policy paperwork lives in [STARTUP-DOCUMENTS.md](./STARTUP-DOCUMENTS.md). Service boundaries live in [SERVICES.md](./SERVICES.md). Module behaviour, data, and India-specific rules live in [modules/README.md](./modules/README.md). The path from plan through dev, stage, prod, and monitoring lives in [ROADMAP.md](./ROADMAP.md). This file is the product definition.

---

## 1. What BuyyMart is

BuyyMart is a **hybrid ecommerce platform**.

- Customers browse, buy, pay, track, and return products.
- BuyyMart can sell its own inventory (BuyyMart is the seller).
- Independent sellers can list and fulfil their own products (the seller is the seller, BuyyMart is the marketplace).
- Staff run catalogue quality, orders, refunds, sellers, and payouts from an admin console.

One order can contain items from more than one seller. Each seller’s items are fulfilled, shipped, and settled separately. The customer still sees one checkout and one payment.

---

## 2. App type

Use these labels in proposals, app-store forms, and hiring briefs.

| Layer | Type | Meaning |
| --- | --- | --- |
| Business model | B2C hybrid marketplace | Consumers buy. BuyyMart and third-party sellers supply goods. |
| Product category | Ecommerce ecosystem | Storefront, seller tools, operations, and finance. Not a single app. |
| Transaction type | Physical goods commerce | Catalogues, stock, shipping, COD, returns. Digital goods are out of version 1. |
| Geography | Domestic, India only | INR, Indian addresses, Indian payment methods. |
| Client style | Multi-client, one backend | Several apps share one API and one database. |
| System style | Modular monolith first | One backend codebase with clear modules. Split into separate services only after order volume requires it. |
| Money model | Gateway checkout, no stored cards | UPI, cards, netbanking, and wallets through a payment aggregator. No cash-out wallet in version 1. |

### Type of each application

| Application | Software type | Primary device | Who uses it |
| --- | --- | --- | --- |
| Customer storefront | Responsive web app, installable as a PWA | Phone browser | Shoppers |
| Customer Android app | Native Android app | Android phone | Shoppers, after the web flow is stable |
| Customer iOS app | Native iOS app | iPhone | Shoppers, after Android |
| Seller Centre | Responsive web app, desktop-first | Laptop | Sellers and their staff |
| Admin Console | Web app, desktop only | Laptop | BuyyMart employees |
| Warehouse Console | Web app, tablet-friendly | Tablet or desktop in the warehouse | Packers, if BuyyMart stores stock |
| Delivery app | Not built in version 1 | — | Courier partner’s own app |
| Backend API | Private HTTPS JSON API | Server | All BuyyMart apps |
| Worker service | Background jobs in the same codebase | Server | Payments, SMS, invoices, courier updates, payouts |
| Marketing site | Public website | Any browser | Visitors, payment-gateway review, app stores |

**Version 1 is three web apps plus one API:** Customer storefront, Seller Centre, and Admin Console. Android and iOS are later clients of the same API. A separate delivery app is not required while a courier partner (Shiprocket, Delhivery, or similar) already gives pickup, tracking, and COD remittance.

---

## 3. Why this set of apps

| Need | App that covers it | Why this type |
| --- | --- | --- |
| A person discovers BuyyMart from Google or Instagram | Marketing site + customer web | A link must open a fast mobile site. An app-only store loses that traffic. |
| Repeat buyers want an icon on the phone | PWA first, native Android next | A PWA can be installed from the browser. Native apps add push and store presence after checkout works. |
| A seller lists products and prints labels | Seller Centre web | Sellers work at a desk with spreadsheets, images, and GST details. A phone-only seller app is a poor version 1. |
| Staff refund, suspend a seller, or fix a stuck order | Admin Console web | Internal tool. No app store review, no mobile layout required. |
| Packers scan and hand over parcels | Warehouse Console, only if you hold stock | Tablet browser is enough. Build it when you open your own warehouse. |
| Courier picks up and collects COD | Partner app | Building your own rider network is a logistics company, not version 1 of the store. |

---

## 4. Shared rules for every app

These rules apply to customer, seller, and admin software.

- One account belongs to one role family. A customer login cannot open Seller Centre. A seller login cannot open Admin Console.
- Phone number is the customer identity. Email is optional.
- Seller and admin identities use email plus password and MFA for admin.
- All money is stored as integer paise. Prices on screen are rupees with two decimals.
- Every order item stores seller, HSN, GST rate, taxable value, tax, MRP, selling price, and country of origin.
- Order status changes are explicit. The UI shows the current status and the next legal action only.
- Admin actions that change money, KYC, price, or roles write an audit log with the staff user and a reason.
- Customer personal data shown to a seller is limited to what that seller needs to ship that order.
- Card numbers, CVV, and UPI PIN never touch BuyyMart servers.

### Order statuses

```text
created → payment_pending → paid → confirmed → packed → shipped → out_for_delivery → delivered
                ↓                ↓         ↓        ↓        ↓
            payment_failed    cancelled  cancelled cancelled cancelled
delivered → return_requested → return_picked → return_accepted → refunded
                            ↘ return_rejected
```

COD uses the same path with `payment_pending` replaced by `cod_pending` until the courier remits the cash. A refused COD becomes `cancelled` with reason `cod_refused`.

---

## 5. Customer storefront

**Type:** B2C shopping app. Responsive website first, PWA install second, native mobile later.  
**Users:** Guests and registered customers.  
**Languages in version 1:** English. Hindi can be added once copy is stable.  
**URL shape:** `https://buyymart.com` (apex and `www` redirect to the canonical host).

### What the customer can do

| Area | Capabilities |
| --- | --- |
| Discovery | Home banners, categories, search, filters (price, brand, rating, discount), sort, product suggestions |
| Product | Images, title, variant (size, colour), MRP, selling price, tax-inclusive total, delivery fee, seller name, seller GSTIN, country of origin, stock, return window, ratings from verified buyers |
| Cart | Add, change quantity, remove, see fees before payment, coupon code, serviceable pin code check |
| Checkout | Address, UPI / card / netbanking / wallet via gateway, or COD where allowed |
| Account | Orders, tracking, cancel before dispatch, return after delivery, refund status, saved addresses, profile, logout, delete account |
| Support | Ticket against an order, visible grievance contact |
| Trust | Footer links: About, Contact, Terms, Privacy, Shipping, Returns |

### Screens

1. Home
2. Category listing
3. Search results
4. Product detail
5. Cart
6. Checkout (address → delivery → payment)
7. Payment result (success, failed, pending)
8. Order list
9. Order detail and tracking
10. Return request
11. Account and addresses
12. Support ticket
13. Static policy pages

### Guest versus signed-in

Guests can browse and add to cart. Phone OTP is required before payment. The cart merges into the account after login.

### Out of version 1

Wishlist can wait. Live chat can wait. A customer wallet that pays out to a bank account is out of scope. Social features, video feed, and bargaining chat are out of scope.

---

## 6. Customer Android app

**Type:** Native Android client of the same API.  
**When:** After the website can complete a prepaid order, a cancellation, and a refund in staging.  
**Package name to reserve now:** `com.buyymart.app`

Same jobs as the storefront. Extra native pieces:

- Push notifications for shipped, out for delivery, delivered, and refunded
- Play Store listing, data safety form, and in-app account deletion
- UPI intent when the payment gateway supports it

Do not fork business rules into the Android app. Price, tax, stock, and order status come from the API.

---

## 7. Customer iOS app

**Type:** Native iOS client of the same API.  
**When:** After Android is in production and the API has not needed breaking changes for at least one release cycle.  
**Bundle id to reserve now:** `com.buyymart.app`

Same behaviour as Android, plus Apple privacy labels and a reviewer demo account. Build iOS only when there is a real iPhone customer base to serve.

---

## 8. Seller Centre

**Type:** B2B operations web app. Desktop-first, usable on a tablet, not designed as a consumer phone app.  
**Users:** Seller owner and seller staff (catalogue, orders).  
**URL shape:** `https://seller.buyymart.com`

### Seller states

```text
applied → kyc_pending → kyc_rejected
                      → approved → active
active → suspended
active → closed
```

A seller cannot publish a listing until status is `approved`. Suspended sellers stay visible on past orders and cannot add stock or receive new orders.

### What the seller can do

| Area | Capabilities |
| --- | --- |
| Onboarding | Business profile, GSTIN, PAN, bank account, address, signatory, category licences, accept seller agreement |
| Catalogue | Create product, variants, MRP, price, HSN, tax, country of origin, images, stock |
| Orders | New orders, accept or cancel within SLA, print label, mark packed, see courier AWB |
| Returns | See return requests, accept or reject with reason, track pickup |
| Money | Read-only settlements: gross, returns, commission, GST on commission, TCS, TDS, shipping, net payout |
| Account | Users for their own staff, notification email and phone |

### Screens

1. Register and verify phone/email
2. KYC form and document upload
3. Application status
4. Dashboard (today’s orders, late dispatch, low stock)
5. Add / edit product
6. Inventory
7. Order queue and order detail
8. Returns
9. Settlements and downloadable statement
10. Settings and staff

Bulk CSV upload of catalogue is phase 2. Version 1 can be a form per product.

---

## 9. Admin Console

**Type:** Internal back-office web app. Desktop only.  
**Users:** BuyyMart staff. Not sellers. Not customers.  
**URL shape:** `https://admin.buyymart.com`  
**Access:** Company email, strong password, MFA. IP allow-list when you have a fixed office network.

### Staff roles

| Role | Can do |
| --- | --- |
| Support | Look up a customer and order, add ticket notes, request a refund that finance approves |
| Catalogue | Edit content, hide a listing, approve a product that failed checks |
| Trust and safety | Approve or reject seller KYC, suspend a seller, take down a listing |
| Finance | Refunds, settlement exceptions, gateway reconciliation export |
| Admin | Create staff, change roles, site banners, coupons, configuration |
| Super admin | Break-glass access, used rarely and always audited |

Refunds above a set amount use maker-checker: one role creates the refund, finance confirms it.

### What admin can do

| Area | Capabilities |
| --- | --- |
| Orders | Search by order id, phone, AWB, or payment id. Cancel, note, trigger refund |
| Customers | Profile, orders, tickets. No browsing of data without an order or ticket id |
| Sellers | KYC review, suspend, fee override with reason |
| Catalogue | Category tree, featured banners, blocked brands |
| Promotions | Coupon create: percent or flat, cap, dates, minimum order, usage limit |
| Finance | Daily payment report, payout run exceptions, credit notes |
| Content | Policy page links are code or CMS-owned; banners are editable |
| Audit | Filter admin actions by staff user and date |

### Screens

1. Login and MFA
2. Home queue (unacked orders, KYC waiting, failed payments, open tickets)
3. Order detail
4. Customer lookup
5. Seller review
6. Catalogue moderation
7. Coupons
8. Refund queue
9. Settlement exceptions
10. Staff and roles
11. Audit log

---

## 10. Warehouse Console

**Type:** Internal web app aimed at a tablet.  
**When:** Only when BuyyMart stores its own goods or runs a fulfilment centre for sellers.  
**Users:** Packers and warehouse supervisors.

Version 1 sellers ship from their own location through the courier partner. In that model, Seller Centre’s “mark packed / print label” replaces this app.

When you do build it:

- Pick list for confirmed orders
- Scan SKU, confirm quantity
- Print invoice and shipping label
- Mark handed to courier
- Inward stock and cycle count
- Return QC: accept into stock or mark damaged

---

## 11. Delivery

**Type in version 1:** Integration, not an app.  
**BuyyMart sends:** pickup address, customer address, weight, COD amount, order id.  
**BuyyMart stores:** AWB number, scan status, delivery time, COD remittance reference.

A BuyyMart rider app (native Android for delivery partners) is a separate product. Build it only if you employ or contract your own riders. It would be a workforce app, not a shopping app, with jobs, navigation, proof of delivery, and cash collection. That is phase 3 or later.

---

## 12. Marketing site

**Type:** Public marketing website. Can be the same project as the storefront or a thin set of pages in front of it.  
**Users:** Anyone, including payment-gateway reviewers.

Required pages before the first real payment:

- Home (what BuyyMart sells and where it delivers)
- Contact (legal name, address, phone, email)
- Terms, Privacy, Shipping, Returns
- Grievance officer

The store itself (search, product, cart) can live on the same domain so customers never hit a dead brochure site.

---

## 13. Backend API

**Type:** Private JSON API over HTTPS. Versioned (`/v1`).  
**Clients:** Storefront, future mobile apps, Seller Centre, Admin Console, and webhook callers (payment gateway, courier).

### Modules inside the backend

| Module | Responsibility |
| --- | --- |
| Identity | OTP, sessions, seller login, staff login, roles |
| Catalogue | Categories, products, variants, images, search index |
| Inventory | Stock per variant per seller, reserve on payment, release on cancel |
| Cart | Cart and coupon evaluation |
| Checkout | Pin-code serviceability, shipping fee, tax breakup, order creation |
| Payments | Create gateway order, verify webhook signature, mark paid, refund |
| Orders | Status machine, cancellation rules |
| Fulfilment | Courier booking, label, tracking webhooks |
| Returns | Return window check, pickup, QC result, refund trigger |
| Sellers | Onboarding, KYC files, suspension |
| Ledger | Append-only lines for sale, commission, tax, TCS, TDS, shipping, refund, payout |
| Notifications | SMS, email, and later push, from templates |
| Support | Tickets linked to orders |
| Audit | Staff actions |
| Files | Product images in a public bucket. KYC in a private bucket |

### Background jobs

- Send OTP and transactional SMS
- Generate invoice PDF
- Book courier after `confirmed`
- Apply tracking updates
- Expire unpaid orders and release stock
- Build the settlement statement
- Reconcile gateway settlements to orders

### Data the API must keep

User, Address, Seller, SellerKyc, StaffUser, Category, Product, Variant, Inventory, Cart, CartItem, Order, OrderItem, Payment, Refund, Shipment, Return, Coupon, LedgerEntry, Payout, Ticket, ConsentLog, AdminAuditLog.

Orders, payments, invoices, and ledger lines are retained for tax. They are not hard-deleted when a user deletes an account. The account deletion flow removes or anonymises profile fields that tax law does not require.

---

## 14. How the apps talk to each other

```text
Customer web / Android / iOS ──┐
Seller Centre ─────────────────┼──► BuyyMart API ──► Database
Admin Console ─────────────────┘         │
                                         ├── Payment gateway
                                         ├── SMS and email
                                         ├── Courier API
                                         └── Image and KYC storage
```

- The browser and the mobile apps never call the payment gateway’s secret APIs directly. They open the gateway checkout that the API creates.
- The gateway and the courier call webhooks on the API. Webhooks are authenticated with the vendor’s signature.
- Seller Centre and Admin Console are separate frontends so a seller build cannot ship an admin screen.

---

## 15. Feature phases

### Phase 0 — Foundation

Company, GST, policies on the domain, payment gateway in test mode, cloud account, and this specification frozen for version 1 categories and cities.

### Phase 1 — Web ecosystem (this is the first release)

| Included | Deferred |
| --- | --- |
| Customer web: browse, search, cart, prepaid checkout, COD, orders, cancel, return | Native Android and iOS |
| One or two categories, selected pin codes | All-India promise |
| BuyyMart’s own listings and a small set of sellers | Open self-serve seller sign-up with no review |
| Seller Centre: KYC, products, orders, labels, settlement view | Bulk catalogue CSV, advertising console |
| Admin: KYC review, order support, refunds, coupons, banners | Fully automated payouts with no human check |
| Courier integration and basic tracking | Own rider app |
| English UI | Hindi and other languages |
| SMS for OTP and order updates | Push, WhatsApp as a required channel |

Launch city and category are a business choice. The software should still store state, pin code, and category so you can open the next city without a rewrite.

### Phase 2 — Mobile and scale

- Android app
- iOS app
- Hindi
- Bulk product upload
- Better search (typos, synonyms)
- Automated settlement files to the bank, still with a finance approval step
- Warehouse Console if you lease a warehouse
- Reviews with photo
- Basic seller analytics

### Phase 3 — Ecosystem extras

Only after phase 1 orders reconcile to the bank every day:

- Own delivery app and rider payouts
- Seller advertising
- Subscription or membership
- Multi-warehouse routing
- Closed promotional balance usable only on BuyyMart
- More categories that need extra licences (food needs FSSAI; medicines stay off until a specialist clears them)

---

## 16. End-to-end flows the apps must share

### Prepaid order

1. Customer sets a pin code. The API says whether it is serviceable and what delivery fee applies.
2. Customer checks out. The API creates an order in `payment_pending` and reserves stock.
3. Customer pays in the gateway.
4. Gateway webhook marks the order `paid`. A duplicate webhook does not create a second order or a second charge.
5. Seller (or BuyyMart warehouse) sees the order, confirms, packs, and the API books a courier.
6. Tracking moves the order to `delivered`.
7. After the return window, the ledger can include the line in a payout.

### COD order

Same path. Stock is reserved at order placement. Payment is collected by the courier. The order stays financially open until the remittance file matches that AWB. If the customer refuses, stock reservation ends and the order is `cancelled`.

### Return and refund

1. Customer requests a return on a delivered item inside the category window.
2. Seller or admin accepts. Courier pickup is booked.
3. QC accepts or rejects.
4. On accept, the API calls the gateway refund for prepaid orders, or creates a payout deduction and a manual or automated COD refund.
5. Customer sees `refunded` with the payment reference.

### Seller payout

For a period, the statement shows item sales, returns, commission, GST on commission, TCS, TDS, shipping adjustments, and net amount. Finance exports it. Version 1 can pay sellers by bank transfer outside the app as long as the statement in Seller Centre matches the transfer.

---

## 17. Suggested technical type (so hiring matches the apps)

The cloud design is a microservice system on Amazon EKS. Every service is listed in [SERVICES.md](./SERVICES.md). AWS accounts, networking, data stores, and Kubernetes deployment are specified in [AWS-ARCHITECTURE.md](./AWS-ARCHITECTURE.md).

| Piece | Type |
| --- | --- |
| Customer, seller, and admin frontends | TypeScript web apps (one design system, three apps) |
| Android | Kotlin, calling the customer BFF |
| iOS | Swift, calling the customer BFF |
| Backend | Microservices on Kubernetes (EKS), one database per service |
| Database | Amazon Aurora PostgreSQL |
| Cache and events | ElastiCache Redis, Amazon EventBridge and SQS |
| Images and KYC | S3, with CloudFront for public product images |
| Secrets | AWS Secrets Manager, injected into pods by External Secrets |
| Environments | Dev, stage, and production. Stage uses test payments only. The promotion path is in [ROADMAP.md](./ROADMAP.md) |

---

## 18. Integrations by app

| System | Used by | Version |
| --- | --- | --- |
| Payment gateway (UPI, cards, netbanking, refunds, webhooks) | Customer checkout, Admin refunds, API | 1 |
| SMS OTP and order SMS | API | 1 |
| Email for seller and staff | API | 1 |
| Courier or shipping aggregator | Seller labels, tracking, COD | 1 |
| Map or pin-code master for city, state, serviceability | Checkout | 1 |
| GST e-invoice | API, when turnover crosses the notified limit | Later |
| Push (FCM, APNs) | Android and iOS | Phase 2 |
| WhatsApp | Optional notifications | Phase 2 |

---

## 19. Non-goals

BuyyMart version 1 is not:

- A super-app (chat, reels, games, bill payments)
- A food-delivery or restaurant app
- A pharmacy
- A lending or buy-now-pay-later lender (a partner BNPL button can be revisited later with that partner’s compliance)
- A cryptocurrency or investment product
- A cross-border store
- A cash-out wallet or prepaid instrument
- A rider payroll system

---

## 20. Success of version 1

The ecosystem is working when all of these are true:

1. A customer on a phone browser can search, pay, and receive a shipment in the launch pin codes.
2. A seller can pass KYC, list a product, and ship it with a courier label.
3. A support agent can find that order and issue a refund that appears in the gateway and on the order.
4. A finance export for a day matches the payment gateway settlement and the bank.
5. The public site shows legal name, policies, and the grievance officer.

Downloads, social followers, and a native app are not the success test for the first release.

---

## 21. Build order

1. API skeleton, database, staff login, and audit log
2. Catalogue and search on the customer web
3. Cart, checkout, test-mode payments, order status
4. Seller onboarding, KYC review in admin, seller orders and labels
5. Courier tracking and customer order page
6. Returns, refunds, and ledger statement
7. Policies, banners, coupons, and production gateway
8. Android, then iOS, then warehouse or rider apps only if the operating model needs them

---

## 22. Decisions still open

Fill these in before UI design starts. They change screens and licences, not the app types above.

| Decision | Options | Affects |
| --- | --- | --- |
| Launch categories | Fashion, home, electronics, grocery, other | Filters, return rules, FSSAI or BIS |
| Launch geography | City list or pin-code list | Serviceability and GST places of business |
| COD | On, off, or above/below an amount | Checkout and reconciliation |
| Who ships in month one | Only sellers, only BuyyMart warehouse, or both | Whether Warehouse Console is in phase 1 |
| Seller entry | Invite-only or open applications | Seller Centre registration |
| Company legal name | May differ from the brand BuyyMart | Invoices, footer, app stores |

# BuyyMart — Services

**Style:** Domain microservices. One database per service. Events between services.  
**Clients talk only to a BFF.** Domain services are ClusterIP inside the cluster.  
**Related:** [APP-SPECIFICATION.md](./APP-SPECIFICATION.md), [AWS-ARCHITECTURE.md](./AWS-ARCHITECTURE.md), [ROADMAP.md](./ROADMAP.md)

Public clients (customer web, later Android and iOS, Seller Centre, Admin Console) call their own BFF. A BFF calls domain services. A domain service does not call another service’s database.

`order-service` is the source of truth for order status. Other services store their own ids (`paymentId`, `awb`, `ledgerEntryId`) and the order id. They do not update the order row.

---

## 1. All services

Seventeen deployable services. Three are edge BFFs. Fourteen are domain services.

| Service | Kind | Namespace | Store | Public entry |
| --- | --- | --- | --- | --- |
| `customer-bff` | BFF | `edge` | None. Short response cache in Redis | `api.buyymart.com` `/v1/*` |
| `seller-bff` | BFF | `edge` | None | `api.buyymart.com` `/seller/*` |
| `admin-bff` | BFF | `edge` | None | `api.buyymart.com` `/admin/*` |
| `identity-service` | Domain | `identity` | Postgres `identity` | — |
| `seller-service` | Domain | `sellers` | Postgres `seller` | — |
| `catalogue-service` | Domain | `catalogue` | Postgres `catalogue` | — |
| `search-service` | Domain | `catalogue` | OpenSearch index `products` | — |
| `media-service` | Domain | `catalogue` | Postgres `media` metadata, S3 | — |
| `inventory-service` | Domain | `commerce` | Postgres `inventory` | — |
| `cart-service` | Domain | `commerce` | Redis `cart:{userId}` plus Postgres snapshot | — |
| `promotion-service` | Domain | `commerce` | Postgres `promotion` | — |
| `order-service` | Domain | `commerce` | Postgres `order` | — |
| `payment-service` | Domain | `money` | Postgres `payment` | `api.buyymart.com` `/webhooks/payments` |
| `settlement-service` | Domain | `money` | Postgres `settlement` | — |
| `fulfilment-service` | Domain | `sellers` | Postgres `fulfilment` | `api.buyymart.com` `/webhooks/courier` |
| `notification-service` | Domain | `engagement` | Postgres `notification` | — |
| `support-service` | Domain | `engagement` | Postgres `support` | — |

Later, if BuyyMart holds its own warehouse stock, a `warehouse-bff` can sit in `edge` and call the same domain services. It is not part of version 1.

---

## 2. Edge services (BFFs)

BFFs authenticate the caller, validate the request, shape the response, and apply timeouts. They do not own a database. Search can fail open to a catalogue keyword query. Payment never fails open.

| Service | Owns | Calls |
| --- | --- | --- |
| `customer-bff` | Mobile and web API for shoppers. Aggregates product, price, stock, delivery estimate | search, catalogue, inventory, cart, promotion, order, payment, fulfilment, support, identity |
| `seller-bff` | Seller Centre API | identity, seller, catalogue, inventory, order, fulfilment, settlement |
| `admin-bff` | Admin API. Enforces staff role on every route | All domain services through internal admin endpoints |

| Caller | Callee | Timeout | If it fails |
| --- | --- | --- | --- |
| customer-bff | search-service | 400 ms | Fall back to catalogue keyword query |
| customer-bff | inventory-service | 300 ms | Show “stock unavailable”, block add-to-cart |
| customer-bff | payment-service | 3 s | Show payment error. Order remains `payment_pending` |

Redis prefix `bff:` holds product-page fragments for 30–60 seconds. Production minimum is 3 replicas for `customer-bff`. HorizontalPodAutoscaler max is 20 for BFFs.

### customer-bff

Shopper API for the storefront and, later, Android and iOS. Checkout orchestration lives here: read the cart, price the coupon, ask for the pin-code fee, create the order, then open the payment session. The browser return URL only displays status. It never marks an order paid.

### seller-bff

Seller Centre API. A seller login cannot reach admin routes. Catalogue writes, stock, labels, and payout statements go through this BFF to the owning domain service.

### admin-bff

Staff API. Every route checks a staff role. Staff actions are written as audit rows in the domain service that owns the change. Warehouse Console, when built, uses admin routes or a future `warehouse-bff`.

---

## 3. Domain services

| Service | Responsibility | Database | Publishes | Consumes |
| --- | --- | --- | --- | --- |
| `identity-service` | Customer OTP, sessions, seller login, staff login, MFA, roles, consent log | `identity` | `UserRegistered`, `UserDeleted` | — |
| `seller-service` | Seller profile, KYC status, suspension, staff of that seller | `seller` | `SellerApproved`, `SellerSuspended` | `UserRegistered` |
| `catalogue-service` | Categories, products, variants, MRP, HSN, tax rate, country of origin, images metadata | `catalogue` | `ProductChanged`, `ProductHidden` | `SellerSuspended` |
| `search-service` | Product search and filters | OpenSearch index `products` | — | `ProductChanged`, `InventoryChanged` |
| `inventory-service` | Stock on hand, reservation, release | `inventory` | `InventoryChanged`, `StockReserved`, `StockRejected` | `OrderCancelled`, `PaymentFailed` |
| `cart-service` | Cart per customer, merge on login | Redis `cart:{userId}` plus a Postgres snapshot | `CartCheckedOut` | — |
| `promotion-service` | Coupons, validity, usage count | `promotion` | `CouponRedeemed` | `OrderCancelled` (restore usage) |
| `order-service` | Order aggregate and status machine | `order` | `OrderPlaced`, `OrderPaid`, `OrderConfirmed`, `OrderCancelled`, `ReturnRequested`, `ReturnAccepted` | `StockReserved`, `PaymentCaptured`, `PaymentFailed`, `ShipmentUpdated` |
| `payment-service` | Gateway order, webhook verification, capture, refund, idempotency | `payment` | `PaymentCaptured`, `PaymentFailed`, `RefundCompleted` | `OrderPlaced`, `RefundRequested` |
| `fulfilment-service` | Serviceability, shipping fee, courier booking, labels, tracking, COD remittance match | `fulfilment` | `ShipmentBooked`, `ShipmentUpdated`, `CodRefused` | `OrderConfirmed` |
| `settlement-service` | Ledger lines, commission, GST on commission, TCS, TDS, payout statements | `settlement` | `PayoutPrepared` | `OrderPaid`, `RefundCompleted`, `ShipmentUpdated` |
| `notification-service` | SMS, email, later push. Templates | `notification` | — | All customer-facing domain events |
| `support-service` | Tickets linked to an order id | `support` | `TicketOpened` | — |
| `media-service` | Presigned upload, image processing job, public vs KYC bucket policy | `media` metadata | `MediaReady` | — |

Sync calls that must finish while the user waits:

| Caller | Callee | Timeout | If it fails |
| --- | --- | --- | --- |
| order-service | inventory-service | 800 ms | Order stays `created` and returns an error. No payment session |
| order-service | promotion-service | 500 ms | Reject coupon. Do not place a discounted order you cannot prove |
| fulfilment-service | courier API | 5 s | Retry via queue. Seller sees “label pending” |

Production minimum is 3 replicas for `order-service` and `payment-service`. Settlement autoscales to a max of 4.

### identity-service

Customer identity is a phone number and OTP. Seller identity is email and password. Staff identity is email, password, and MFA. Sessions are opaque tokens stored hashed in Redis (`id:` prefix: OTP 5 minutes, session 30 days sliding). Consent is an append-only log. Account deletion publishes `UserDeleted` and anonymises profile fields. Orders, payments, invoices, and ledger lines stay, because tax retention requires them.

Database `identity` is the only database on Aurora cluster `bm-identity`.

### seller-service

Seller profile, KYC status, suspension, and the staff who work for that seller. KYC files live in the private bucket `buyymart-prod-kyc`. Only this service’s task role can read that bucket. `SellerSuspended` tells catalogue to hide that seller’s products. `SellerApproved` is the gate before a seller can list.

### catalogue-service

Categories, products, variants, MRP, HSN, GST rate, and country of origin. Image metadata points at objects written by `media-service`. This service is the product source of truth. Search is a projection.

### search-service

The only writer of the OpenSearch `products` index. A result card has name, brand, category, price in paise, rating, seller id, in-stock flag, and country of origin. The index has no customer data. It rebuilds from `ProductChanged` and `InventoryChanged`.

### media-service

Issues a presigned PUT. The browser uploads straight to S3. A worker writes WebP renditions and publishes `MediaReady`. Public product images go to `buyymart-prod-images` (CloudFront only). KYC objects stay on the private bucket.

### inventory-service

Stock on hand per variant per seller, plus reservation and release. Available-to-sell is cached in Redis under `inv:` for 10 seconds. Reservation is synchronous from `order-service` at order creation. `OrderCancelled` and `PaymentFailed` release the hold. `InventoryChanged` keeps search in step.

### cart-service

One cart per customer. A guest cart merges on login. Live cart JSON is Redis key `cart:{userId}` with a 14-day TTL, plus a Postgres snapshot so a Redis flush does not drop the cart. Checkout reads the cart through `customer-bff`, then this service publishes `CartCheckedOut`.

### promotion-service

Coupons, validity windows, and usage counts. `order-service` asks it to price a coupon before the order is created. If that call fails, the coupon is rejected. `OrderCancelled` restores usage. `CouponRedeemed` records a successful use.

### order-service

The order aggregate and the only place status changes. Statuses from the product spec (`payment_pending`, `paid`, `confirmed`, `packed`, `shipped`, `delivered`, returns, refunds) are enforced here. Other services react to the events this status machine publishes.

Stored on the order: order id, customer id, seller id per line, amounts in paise, GST breakup, status, address snapshot. Invoice PDFs can be written to `buyymart-prod-invoices` with object lock for 8 years.

Idempotency responses share the Redis prefix `idem:` with payment, TTL 24 hours.

### payment-service

Creates the gateway order, verifies the webhook signature, captures, refunds, and stores idempotency. The public route `POST /webhooks/payments` skips the BFF. A duplicate webhook returns the first result from the idempotency table.

Stored: order id, gateway payment id, status, method, refund ids. No card number, CVV, or UPI PIN.

External system: the payment gateway. No other service calls it. A second reviewer is required on the gitops pull request. Database `payment` sits on Aurora cluster `bm-money` (35-day point-in-time recovery, deletion protection).

### fulfilment-service

Pin-code serviceability, shipping fee, courier booking, labels, tracking scans, and COD remittance match. Books a shipment when it consumes `OrderConfirmed`. The public route `POST /webhooks/courier` skips the BFF.

Stored: order id, warehouse or seller pickup address, AWB, scans, COD amount.

External system: the courier aggregator. No other service calls it.

### settlement-service

Append-only ledger lines: sale, commission, GST on commission, TCS, TDS, shipping adjustments, refund, payout. Commission is final after delivery. Consumes `OrderPaid` (or `PaymentCaptured` on the checkout path), `RefundCompleted`, and `ShipmentUpdated`. Publishes `PayoutPrepared` when a statement is ready. Sellers are paid by bank transfer outside the app in version 1. The statement in Seller Centre must match the transfer.

GST e-invoice, when turnover crosses the notified limit, is called only from this service.

Database `settlement` sits on `bm-money` with the same backup rules as payment. Gitops changes need a second reviewer. Autoscaler max is 4.

### notification-service

SMS and email from templates. Push is later. It consumes customer-facing domain events (`OrderConfirmed`, `ShipmentBooked`, and the rest of that set) and does not publish domain events. OTP send is a job for this service, using the template and the phone from identity.

Stored: template, phone or email, provider message id, delivery status.

External systems: SMS provider, and email (SES or a provider). No other service calls them.

### support-service

Tickets linked to an order id. Publishes `TicketOpened`. Stored: ticket id, order id, messages.

---

## 4. What each service stores about an order

| Service | Stores |
| --- | --- |
| order | Order id, customer id, seller id per line, amounts in paise, GST breakup, status, address snapshot |
| payment | Order id, gateway payment id, status, method, refund ids. No card number, CVV, or UPI PIN |
| inventory | Variant id, seller id, reservation id, order id |
| fulfilment | Order id, warehouse or seller pickup address, AWB, scans, COD amount |
| settlement | Order id, ledger lines in paise, payout batch id |
| notification | Template, phone or email, provider message id, delivery status |
| support | Ticket id, order id, messages |

---

## 5. External systems

| External system | Only this service calls it |
| --- | --- |
| Payment gateway | `payment-service` |
| SMS provider | `notification-service` |
| Email (SES or provider) | `notification-service` |
| Courier aggregator | `fulfilment-service` |
| GST e-invoice (later) | `settlement-service` |

---

## 6. Databases and caches

One Aurora PostgreSQL database per service. Credentials differ per service. No service has a connection string to another service’s database.

| Cluster | Databases |
| --- | --- |
| `bm-identity` | `identity` |
| `bm-commerce` | `catalogue`, `inventory`, `cart`, `promotion`, `order` |
| `bm-money` | `payment`, `settlement` |
| `bm-ops` | `seller`, `fulfilment`, `notification`, `support`, `media` |

`search-service` uses OpenSearch, not Aurora. `cart-service` uses Redis for the live cart and Postgres for the snapshot. `customer-bff`, `identity-service`, `inventory-service`, `payment-service`, and `order-service` also use Redis prefixes (`bff:`, `id:`, `inv:`, `idem:`, `cart:`). Orders, payments, and the ledger are in Aurora only.

| Bucket | Which service |
| --- | --- |
| `buyymart-prod-images` | `media-service` writes. CloudFront reads |
| `buyymart-prod-kyc` | `seller-service` task role only |
| `buyymart-prod-invoices` | `settlement-service` and `order-service` |

---

## 7. Product modules mapped to services

The app specification lists backend modules. They land on these services.

| Product module | Service |
| --- | --- |
| Identity | `identity-service` |
| Catalogue | `catalogue-service` |
| Search index | `search-service` |
| Inventory | `inventory-service` |
| Cart | `cart-service` |
| Coupon evaluation | `promotion-service` |
| Checkout | `customer-bff` plus `fulfilment-service` (fee and pin code) and `order-service` |
| Payments | `payment-service` |
| Orders | `order-service` |
| Returns | `order-service` (`ReturnRequested`, `ReturnAccepted`) with refund on `payment-service` |
| Fulfilment | `fulfilment-service` |
| Sellers | `seller-service` |
| Ledger | `settlement-service` |
| Notifications | `notification-service` |
| Support | `support-service` |
| Files | `media-service` |
| Audit | Written by `admin-bff` and the domain service that owns the change |

---

## 8. Checkout, and who does what

```text
Customer
  → customer-bff
      → cart-service            read cart
      → promotion-service       price the coupon
      → fulfilment-service      fee and serviceability for the pin code
      → order-service           create order (payment_pending)
            → inventory-service reserve stock (sync)
            → publish OrderPlaced
  → payment-service             create gateway session (sync, from BFF)
Customer pays at the gateway
Gateway
  → POST /webhooks/payments  → payment-service
        verify signature, store payment, publish PaymentCaptured
order-service consumes PaymentCaptured
        status = paid, then confirmed for in-stock items
        publish OrderConfirmed
fulfilment-service consumes OrderConfirmed
        book courier, publish ShipmentBooked
notification-service consumes OrderConfirmed and ShipmentBooked
        send SMS
settlement-service consumes PaymentCaptured
        append ledger lines (sale, tax). Commission is final after delivery
search-service and inventory stay consistent through InventoryChanged
```

Payment webhooks are the source of truth for “money captured”.

---

## 9. Build order

| Step | Services |
| --- | --- |
| 2 | `identity-service`, `customer-bff` |
| 3 | `catalogue-service`, `media-service`, `search-service` |
| 4 | `inventory-service`, `cart-service` |
| 5 | `order-service`, `payment-service` |
| 6 | `seller-service`, `seller-bff`, `admin-bff` |
| 7 | `fulfilment-service`, `notification-service` |
| 8 | `promotion-service`, `support-service` |
| 9 | `settlement-service` |

Step 1 is platform only (VPC, EKS, ingress, secrets, gitops). It deploys no BuyyMart service.

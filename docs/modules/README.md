# BuyyMart — Module documents

**Market:** India, INR, physical goods.  
**Version:** 1 web ecosystem (customer storefront, Seller Centre, Admin Console).  
**Related:** [APP-SPECIFICATION.md](../APP-SPECIFICATION.md), [SERVICES.md](../SERVICES.md)

These documents are the build spec for every module. Each one states who uses it, the data it stores, the rules it enforces, and the failure cases. Money is integer paise. The service that owns the data is named in [SERVICES.md](../SERVICES.md).

This is product design informed by public rules and gateway practice. It is not a legal opinion. A chartered accountant and an ecommerce lawyer confirm rates, invoice wording, and policy text before the first live order.

---

## Modules

| Document | Module | Owns |
| --- | --- | --- |
| [01-identity](./01-identity.md) | Identity | Customer OTP, seller login, staff MFA, consent, account deletion |
| [02-catalogue](./02-catalogue.md) | Catalogue | Categories, products, variants, price, HSN, tax, origin |
| [03-search](./03-search.md) | Search | Product index, filters, ranking disclosure |
| [04-media](./04-media.md) | Files | Product images and private KYC objects |
| [05-inventory](./05-inventory.md) | Inventory | Stock, reservation, release |
| [06-cart](./06-cart.md) | Cart | Guest cart, merge on login |
| [07-promotions](./07-promotions.md) | Promotions | Coupons, who funds the discount |
| [08-checkout](./08-checkout.md) | Checkout | Pin code, fee, tax breakup, consent, order create |
| [09-orders](./09-orders.md) | Orders | Status machine, cancellation, multi-seller lines |
| [10-payments](./10-payments.md) | Payments | Gateway session, webhook, refund |
| [11-fulfilment](./11-fulfilment.md) | Fulfilment | Serviceability, labels, tracking, COD remittance |
| [12-returns](./12-returns.md) | Returns | Window, pickup, QC, refund trigger |
| [13-sellers](./13-sellers.md) | Sellers | Onboarding, KYC, suspension |
| [14-settlement](./14-settlement.md) | Ledger and tax | Commission, GST on fees, TCS, TDS, payout statement |
| [15-notifications](./15-notifications.md) | Notifications | SMS and email templates |
| [16-support](./16-support.md) | Support | Order tickets and grievance clocks |
| [17-audit](./17-audit.md) | Audit | Staff actions that change money, KYC, price, or roles |
| [18-applications](./18-applications.md) | Applications | Storefront, Seller Centre, Admin Console screens |

---

## What the research changed in the design

Public material checked on 28 September 2026:

| Topic | What BuyyMart implements | Source to re-check |
| --- | --- | --- |
| Marketplace disclosures | Product page shows seller legal name, whether registered, address, customer-care contact, GSTIN when held, country of origin, MRP, selling price, tax, delivery fee after pin code, return window, and who pays return shipping. Total price is one figure plus a breakup. | [Consumer Protection (E-Commerce) Rules, 2020](https://taxguru.in/corporate-law/consumer-protection-e-commerce-rules-2020.html) |
| 2026 amendment | Law-firm briefings describe an amendment notified in September 2026 (reported as G.S.R. 789(E)) that adds best-before where relevant, government identifiers, a copy of the complaint as recorded, ranking and sponsored-label duties, invoice seller name at the same visual weight as the platform, and mandatory National Consumer Helpline participation. Confirm the gazette and the commencement date before treating every new line as already in force. The data model stores the fields now. | [ICLG briefing](https://iclg.com/briefing/transparency-by-design-indias-2026-e-commerce-rules-amendment/), [Bar & Bench](https://www.barandbench.com/law-firms/view-point/consumer-protection-e-commerce-amendment-rules-2026-from-consumer-disclosure-to-platform-governance) |
| Grievance | Ticket number is visible. Acknowledge within 48 hours. Resolve within one month. Footer names the grievance officer. | 2020 Rules, rule 4 |
| Purchase consent | The pay button is an explicit checkbox. It is not pre-ticked. | 2020 Rules |
| Cards and UPI | BuyyMart never stores card number, CVV, or UPI PIN. Saved cards, if offered later, are network tokens held by the gateway. | [RBI payment aggregator guidelines](https://www.rbi.org.in/Scripts/NotificationUser.aspx?Id=11822&Mode=0), [card-on-file tokenisation](https://www.rbi.org.in/Scripts/NotificationUser.aspx?Id=12159&Mode=0) |
| Webhooks | Signature check, store the gateway event id, return 2xx only after the row is saved, ignore duplicates. | [Razorpay webhook practices](https://razorpay.com/docs/webhooks/best-practices/) |
| TCS and TDS | Separate ledger lines. TCS is GST section 52 on other sellers’ net taxable supplies, reported later in GSTR-8. TDS is income-tax section 194-O on facilitated gross sales, reported later in Form 26Q. BuyyMart’s own inventory is not treated as a third-party participant sale. **Rates are configuration.** Published guides in 2026 disagree (194-O cited as both 0.1% and 1%; section 52 cited as both 0.5% and 1% split across CGST and SGST). Finance locks the live rate from incometax.gov.in and the CBIC notification before the first payout. | [194-O explainer](https://www.patronaccounting.com/blog/section-194o-tds-ecommerce-guide), [TCS vs TDS](https://finin2min.com/articles/gst-tcs-vs-income-tax-tds-under-section-194-o-seller-reconciliation.html), [operator guide](https://masllp.com/compliance-with-tds-and-tcs-for-e-commerce-operators/) |
| SMS | Transactional SMS uses a DLT-registered sender and template. OTP values are not written to logs. | TRAI commercial-communication rules; template ids stored on the notification record |

---

## Rules that apply to every module

- One account belongs to one role family: customer, seller, or staff.
- A client never sends the price, tax, or stock that the server will charge. The server recomputes at checkout.
- An order can contain lines from more than one seller. The customer pays once. Each seller’s lines ship and settle on their own.
- `order-service` is the only writer of order status.
- Prepaid “paid” comes from a verified payment webhook, not from the browser return URL.
- Staff changes to money, KYC, price, or roles write an audit row with the person and a reason.
- Orders, payments, invoices, and ledger lines stay when a customer deletes an account. Profile fields that tax law does not require are removed or anonymised.

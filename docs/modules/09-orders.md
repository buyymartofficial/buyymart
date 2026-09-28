# Orders

**Owns:** the order aggregate and the only status machine.  
**Service:** `order-service`.  
**Does not own:** the gateway payment row, the AWB, or the ledger. Those services store the order id and their own id.

Other services do not update the order row. They publish events. Order-service decides the next status.

---

## Shape

One customer checkout creates one order and one or more seller groups. A group is the set of lines with the same seller. Each group ships and settles on its own. The customer still sees one order and one payment.

| Order field | Meaning |
| --- | --- |
| `id` | Shown to the customer as the order number |
| `customerId` | |
| `status` | The customer-facing status, derived from the groups |
| `addressSnapshot` | Copy of the checkout address |
| `itemsPaise`, `discountPaise`, `deliveryPaise`, `payablePaise` | |
| `payMode` | `prepaid` or `cod` |
| `consentAt` | When they ticked the unticked checkbox |
| `policyVersion` | Terms version they saw |

| Line field | Meaning |
| --- | --- |
| `sellerId`, `variantId`, `quantity` | |
| `title`, `hsn`, `gstRate`, `countryOfOrigin` | Snapshot |
| `mrpPaise`, `pricePaise`, `taxablePaise`, `cgstPaise`, `sgstPaise`, `igstPaise` | Snapshot |
| `discountPaise`, `fundedBy` | From promotions |
| `groupStatus` | This seller’s progress |

The customer-facing status is the least advanced group that is still open. If one seller has shipped and another has not, the order shows the earlier status and the detail page shows both groups.

---

## Status

```text
created → payment_pending → paid → confirmed → packed → shipped → out_for_delivery → delivered
                ↓                ↓         ↓        ↓        ↓
            payment_failed    cancelled  cancelled cancelled cancelled
delivered → return_requested → return_picked → return_accepted → refunded
                            ↘ return_rejected
```

COD uses `cod_pending` in place of `payment_pending` until the seller confirms. A refused COD is `cancelled` with reason `cod_refused`.

| From | To | Who may do it |
| --- | --- | --- |
| `created` | `payment_pending` or `cod_pending` | Order service, as soon as stock is reserved |
| `payment_pending` | `paid` | Only `PaymentCaptured` |
| `payment_pending` | `payment_failed` | `PaymentFailed`, or the 15-minute hold expired |
| `paid` or `cod_pending` | `confirmed` | Seller, or auto-confirm when the seller flag says so |
| `confirmed` | `packed` | Seller, after they print the label |
| `packed` | `shipped` | Courier scan, via fulfilment |
| `shipped` | `out_for_delivery`, then `delivered` | Courier scans |
| Before `shipped` | `cancelled` | Customer, seller, or staff, with a reason |
| `delivered` | return states | Returns module |
| `return_accepted` | `refunded` | `RefundCompleted`, or a recorded COD refund |

A seller must confirm or cancel within the SLA admin sets (default 24 hours). A breach shows on the seller dashboard and the admin queue. Version 1 does not auto-cancel on SLA breach. Staff do.

Cancellation before dispatch releases stock, restores a coupon use, and, if the order was prepaid and captured, asks payments for a refund. Cancellation after ship is not a cancel. It is a return, after delivery, or a support ticket if the parcel is already with the courier.

---

## Customer-facing status

The UI shows the current status and the next legal action only.

| Status | Next action |
| --- | --- |
| `payment_pending` | Pay again, or wait |
| `paid`, `cod_pending`, `confirmed` | Cancel |
| `packed`, `shipped`, `out_for_delivery` | Track. Cancel is not offered. |
| `delivered` and inside the return window | Request return |
| `delivered` and outside the window | No return button. Support can still open a ticket. |

---

## Events published

`OrderPlaced`, `OrderPaid`, `OrderConfirmed`, `OrderCancelled`, `ReturnRequested`, `ReturnAccepted`.

`OrderPlaced` is the create. `OrderPaid` follows payment capture. Fulfilment books a courier on `OrderConfirmed`, not on `OrderPaid`, so a seller who is out of stock can cancel before a label is bought.

---

## Invoice snapshot

When a group is confirmed, the order service generates an invoice PDF for that seller’s lines into the invoices bucket. The seller’s legal name is printed at the same size as “BuyyMart”, which is the direction of the 2026 amendment briefings and is safe under the 2020 rules as well. The invoice includes GSTIN, invoice number, date, place of supply, HSN, taxable value, and CGST/SGST or IGST. BuyyMart’s own goods use BuyyMart’s GSTIN. A marketplace seller’s goods use that seller’s GSTIN.

Invoice numbers are per seller, increasing, and never reused.

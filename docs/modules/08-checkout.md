# Checkout

**Owns:** the steps that turn a cart into an order: pin code, delivery fee, tax breakup, explicit consent, and the create-order call.  
**Runs in:** `customer-bff`, calling fulfilment, promotions, cart, and `order-service`.  
**Does not own:** payment capture (payments) or shipment booking (fulfilment, after the order is confirmed).

The customer sees one checkout and one payment. The server splits lines by seller.

---

## Steps

1. Customer enters or selects an Indian address with a 6-digit pin code.
2. Fulfilment says whether that pin is serviceable and the delivery fee in paise. An unserviceable pin stops checkout. The message is “We don’t deliver to this pin code yet.”
3. Customer may enter a coupon. Promotions prices it.
4. The BFF shows one total and a breakup: items, discount, delivery, and tax. Tax is already inside the item prices. The breakup still shows CGST and SGST, or IGST, so the invoice can be explained.
5. The customer ticks “I agree to the terms and to place this order.” The box starts unticked. A pre-ticked box is not valid consent under the 2020 e-commerce rules.
6. `order-service` creates the order, reserves stock, and returns `payment_pending` for prepaid or `cod_pending` for COD.
7. For prepaid, the BFF asks payment-service for a gateway session and sends the customer to the gateway. For COD, there is no gateway session.

The browser return URL only displays status. It does not mark the order paid.

---

## Address

| Field | Rule |
| --- | --- |
| `name`, `phone`, `line1`, `city`, `state`, `pin` | Required |
| `line2`, `landmark` | Optional |
| `state` | Must match the pin code’s state in the pin master. A mismatch is corrected to the master, and the customer sees the corrected state. |

The order stores a snapshot. Later edits to the address book do not change an order that was already placed.

The seller who ships a group sees the name, phone, and address for that shipment. They do not see the customer’s other addresses or their email.

---

## Tax breakup

Version 1 physical goods use the delivery address as the place of supply.

| Seller state and delivery state | Tax |
| --- | --- |
| Same | CGST and SGST, each half of the product GST rate |
| Different | IGST at the product GST rate |

Each line stores seller id, HSN, GST rate, taxable paise, tax paise, MRP, selling price, and country of origin. Delivery fee tax, if the business decides the fee is taxable, is its own line with its own SAC. Finance sets that flag before launch. Until they do, the fee is stored and the tax on the fee is zero, and the invoice does not pretend otherwise.

A chartered accountant confirms bill-to and ship-to edge cases. Version 1 ships to the address on the order. There is no separate billing state.

---

## COD

COD is a checkout choice only when all of these are true: the launch decision turns COD on, the pin is COD-serviceable, every line’s category allows COD, and the order total is inside the minimum and maximum finance sets. Stock is still reserved at order placement. The order is `cod_pending` until the seller confirms. Money is not captured by the payment gateway. Fulfilment records the COD amount on the shipment and matches it when the courier remits.

---

## Totals

```text
itemsPaise        = sum of variant pricePaise * quantity
discountPaise     = from promotions
deliveryPaise     = from fulfilment
payablePaise      = itemsPaise - discountPaise + deliveryPaise
```

All four are stored on the order. The client total is not accepted if it differs.

---

## Failure

| Failure | Result |
| --- | --- |
| Pin not serviceable | No order |
| Coupon service down | Coupon rejected. Full price remains available. |
| Stock reserve fails | No order. Cart keeps the line with “Only N left” or “Out of stock.” |
| Gateway session fails | Order stays `payment_pending`. Customer sees a payment error and can retry the session. |
| Customer closes the gateway | Order stays `payment_pending` until the 15-minute hold expires. |

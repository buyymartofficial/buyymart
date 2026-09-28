# Returns

**Owns:** whether a delivered line may come back, the pickup, the QC result, and the request that payments or settlement refund the customer.  
**Service:** return states live on `order-service`. Pickup booking is fulfilment. The refund of a prepaid capture is payments.  
**Events:** `ReturnRequested`, `ReturnAccepted`.

A return is per line, inside one seller group. The customer can return one item from a multi-item order.

---

## Window

Each line has `returnWindowDays` copied from the category at purchase, and `returnShippingPaidBy` copied the same way. The product page showed both before payment. The return button uses the copy on the line, not today’s category setting.

The window starts at the group’s `delivered` time. On the last day the button works until 23:59 IST. After that, the button is gone. Support can still open a ticket. A ticket is not itself a return.

Categories with a zero-day window show “No returns” on the product page and have no return button. Hygiene and made-to-order lines, if a category is marked non-returnable, use zero.

---

## Flow

1. Customer picks the line, a reason code, and optional photos (media upload, private to the return).
2. Order becomes `return_requested` for that line and publishes `ReturnRequested`.
3. The seller, or staff, accepts or rejects with a reason within 48 hours. Silence shows on the admin queue. It does not auto-accept in version 1.
4. On accept, fulfilment books a reverse pickup to the same address snapshot. The customer sees the return AWB.
5. The seller or warehouse does QC: `sellable`, `damaged`, or `wrong_item`.
6. `sellable` tells inventory to add one unit. `damaged` does not.
7. QC accept publishes the refund request. QC reject moves the line to `return_rejected` and the item is not refunded. The customer sees the reason.

Customer-facing statuses: `return_requested`, `return_picked`, `return_accepted`, `return_rejected`, `refunded`.

---

## Money

| Pay mode | Refund |
| --- | --- |
| Prepaid | Payment-service refunds the line’s `pricePaise` times quantity, minus any seller-funded discount already on the line, back to the original instrument. Delivery is refunded only when every line in the order is returned before dispatch, or when the policy for that category says a delivered return includes delivery. Version 1 refunds delivery on a delivered return only if `returnShippingPaidBy` is `seller` and finance has set “refund outbound delivery” off. Default: item amount only. |
| COD | No gateway call. Settlement records a deduction from the seller’s next payout, or finance records a manual bank refund. The customer sees `refunded` only after that payout line or bank reference exists. |

Return shipping: if the customer pays it, the refund is reduced by the disclosed return-shipping amount. If the seller pays it, the refund is not reduced and the seller statement gets a return-shipping line. The amount was on the product page before purchase.

A partial refund cannot exceed the captured remainder on that payment.

---

## Rules

- A line that is not `delivered` cannot be returned. The customer cancels instead, and only before ship.
- Two open returns for the same line are rejected.
- Rejecting a return requires a reason from a fixed list plus a note: outside window (should already be blocked), used or damaged contrary to the category, wrong photos, or not the item we shipped.
- Staff trust can override a seller rejection. That writes an audit row.

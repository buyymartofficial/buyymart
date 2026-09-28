# Promotions

**Owns:** coupons, validity, usage counts, and who funds the discount.  
**Service:** `promotion-service`.  
**Does not own:** the order total (orders copies the result) or the payment (payments).

Version 1 is a coupon code the customer types. Automatic site-wide sales, bank offers, and seller advertising are out of scope.

---

## Coupon

| Field | Meaning |
| --- | --- |
| `code` | Stored uppercase. Customer input is trimmed and uppercased. |
| `kind` | `percent` or `flat` |
| `value` | Percent, or flat amount in paise |
| `capPaise` | Maximum discount for a percent coupon. Required for percent. |
| `minOrderPaise` | Cart merchandise total before delivery |
| `startsAt`, `endsAt` | Inclusive window, UTC |
| `usageLimit` | Total redemptions. Null means no global cap. |
| `perCustomerLimit` | Default 1 |
| `fundedBy` | `platform` or `seller` |
| `sellerId` | Required when the seller funds it. The coupon applies only to that seller’s lines. |

A platform coupon can apply to the whole merchandise total. A seller coupon applies only to that seller’s lines. Delivery fee is not discounted in version 1.

---

## Who funds it, and why it matters

Settlement treats the two funders differently. A seller-funded discount reduces that seller’s gross. A platform-funded discount is a BuyyMart cost and does not reduce the seller’s gross. The order stores `discountPaise` and `fundedBy` on each affected line. If this is not stored, TCS and TDS are computed on the wrong base. See settlement.

---

## Evaluate and redeem

Checkout calls promotions with the cart lines, the customer id, and the code. Timeout is 500 ms. If the call fails, the coupon is rejected. BuyyMart does not place a discounted order it cannot prove.

| Result | What happens |
| --- | --- |
| Valid | Discount paise returned. Order stores it. |
| Expired, unknown, below minimum, or over the cap of uses | Checkout shows the reason. The customer can pay without the code. |
| Valid, then the order is cancelled before dispatch | `OrderCancelled` restores one usage. |

`CouponRedeemed` is published when the order is placed with that code, not when the customer types it. Typing a code does not consume a use. Closing the browser does not consume a use.

---

## Admin

Staff with the admin role create coupons. Creating or disabling a coupon writes an audit row. Support cannot invent a code from the ticket screen. They ask admin, or they refund.

---

## Examples

| Code | Cart | Result |
| --- | --- | --- |
| 10% cap ₹100, merchandise ₹2,000 | Platform funded | Discount ₹100, not ₹200 |
| Flat ₹50, merchandise ₹40 | Below a ₹299 minimum | Rejected |
| Seller A funds ₹50, cart has seller A and seller B | | ₹50 comes off seller A’s lines only |

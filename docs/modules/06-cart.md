# Cart

**Owns:** the set of variants a customer intends to buy, and the quantity.  
**Service:** `cart-service`.  
**Does not own:** the price (catalogue), the coupon decision (promotions), or stock (inventory).

The live cart is Redis key `cart:{userId}` with a 14-day TTL, plus a Postgres snapshot so a Redis flush does not drop it. Guest carts use an anonymous id in a cookie until login.

---

## Lines

| Field | Meaning |
| --- | --- |
| `variantId` | What they want |
| `sellerId` | Copied from the catalogue so the cart can group by seller |
| `quantity` | Integer, at least 1 |
| `addedAt` | Used only for display order |

The cart does not store price. Every read joins catalogue for the current price and inventory for the current available quantity. If the price changed since add, the cart shows the new price. The customer pays the price at checkout, not the price at add time.

---

## Guest and login

Guests can add, change quantity, and remove. Payment requires a customer session. On OTP success the guest cart merges into the account cart:

- Same variant: quantities add, then cap at available stock.
- Different variants: both lines remain.
- The guest cart is then deleted.

---

## Checkout handoff

The customer BFF reads the cart, asks promotions to price a coupon, asks fulfilment whether the pin code is serviceable, and asks order-service to create the order. After the order is created, cart publishes `CartCheckedOut` and clears those lines. Lines that failed stock reserve stay in the cart with an error.

A cart with lines from two sellers is still one cart and becomes one customer order with two seller groups.

---

## Rules

- Maximum 30 lines and 10 units of one variant, unless available stock is lower.
- A hidden or rejected product cannot be added. If it is hidden after add, the line remains visible as unavailable and is excluded from checkout.
- The client cannot submit a price with the add request. A price field on the request is ignored.

---

## Errors

| Case | Message |
| --- | --- |
| Quantity above stock | “Only N left.” The line is reduced to N. |
| Product hidden | “This item is no longer available.” |
| Empty cart at checkout | Checkout does not open. |

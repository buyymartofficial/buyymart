# Inventory

**Owns:** stock on hand, reservations, and the available-to-sell number.  
**Service:** `inventory-service`.  
**Does not own:** the price (catalogue) or the order status (orders).

Available to sell is `onHand - reserved`. Version 1 has one stock pool per variant per seller. A warehouse bin is out of scope until BuyyMart stores goods itself.

---

## Records

| Field | Meaning |
| --- | --- |
| `variantId`, `sellerId` | The stock pool |
| `onHand` | Units the seller says are in their location |
| `reserved` | Units held for orders that are not finished |
| `reservationId` | One hold, tied to an order id and a quantity |
| `expiresAt` | Unpaid holds expire |

Redis key `inv:` caches available-to-sell for 10 seconds so the product page is fast. Checkout and reserve read the database, not the cache.

---

## Seller stock edits

The seller sets `onHand` from Seller Centre. They cannot set `reserved`. A change that would make `onHand` less than `reserved` is rejected. The seller must cancel or wait for those orders.

Every successful change publishes `InventoryChanged` with the variant id and whether available-to-sell is above zero. Search uses that flag. The exact number stays out of the public index.

---

## Reservation

`order-service` calls inventory synchronously while creating the order.

| Caller | Timeout | If it fails |
| --- | --- | --- |
| `order-service` | 800 ms | The order stays unplaced. No payment session is created. |

The reserve is one database transaction: check available, insert the reservation, increment `reserved`. Two customers cannot take the last unit. The second call gets `StockRejected`.

A prepaid reservation expires 15 minutes after the order is created if payment has not been captured. The expiry job releases the hold and the order moves to `payment_failed` if it is still `payment_pending`. A COD reservation does not use the 15-minute payment timer. It stays until the order is cancelled, refused, or delivered.

Release happens when inventory consumes `OrderCancelled` or `PaymentFailed`, and when a COD refusal is recorded. Delivery does not release the reservation back to available stock. Delivery converts the reservation into a permanent reduction of `onHand` and clears `reserved`, so the unit is gone.

Returns do not automatically add stock. The returns module tells inventory to add a unit only when QC accepts the item as sellable.

---

## Events

| Event | When |
| --- | --- |
| `InventoryChanged` | Available-to-sell crosses or leaves zero, or onHand changes |
| `StockReserved` | Hold created |
| `StockRejected` | Not enough units |

---

## What the customer sees

| State | Product page | Cart |
| --- | --- | --- |
| Available > 0 | Can add, up to available quantity | Quantity stepper stops at available |
| Available = 0 | “Out of stock”, add is disabled | Line shows out of stock and cannot check out |
| Inventory call fails | “Stock unavailable” | Add and checkout are blocked |

The customer BFF waits at most 300 ms for a stock read on the product page.

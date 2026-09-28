# Fulfilment

**Owns:** pin-code serviceability, delivery fee, courier booking, labels, tracking scans, and COD remittance matching.  
**Service:** `fulfilment-service`.  
**Public route:** `POST /webhooks/courier`, which skips the BFF.  
**Does not own:** order status. It publishes `ShipmentBooked`, `ShipmentUpdated`, and `CodRefused`. Order-service moves status from those events.

Version 1 uses a courier aggregator (Shiprocket, Delhivery, or a similar contract). BuyyMart does not run a rider app.

---

## Pin code and fee

A pin master maps every 6-digit pin BuyyMart will serve to city, state, serviceable yes or no, COD yes or no, and a fee in paise. Launch can be a short list of pins. The table is still the full shape so the next city is data, not a code change.

Checkout calls this synchronously. An unknown pin is not serviceable. The fee depends on the seller group’s chargeable weight. Version 1 uses the variant’s declared weight, summed for the group, and a slab table in the courier contract. If weight is missing, the product cannot go live.

Each seller group can have its own fee. The customer sees one delivery total, which is the sum.

The courier API timeout for a live serviceability call is 5 seconds. If the partner is down, checkout uses the pin master only and does not invent a fee. If the pin is absent from the master, checkout stops.

---

## Booking

Fulfilment consumes `OrderConfirmed` and books one shipment per seller group.

| Sent to the courier | Stored |
| --- | --- |
| Pickup address (seller or BuyyMart warehouse) | Shipment id |
| Customer address snapshot | Order id and group id |
| Weight and dimensions | AWB |
| COD amount, or zero for prepaid | Label URL in the private docs sense, or a reprint call |
| Order id as the merchant reference | Scan history |

If the courier call fails, the seller sees “Label pending” and the job retries from the queue. The order stays `confirmed`. Staff can see the last error.

`ShipmentBooked` tells the seller to print the label and tells notifications to send the AWB to the customer. The seller marks `packed` in Seller Centre after the label is printed. The first in-transit scan moves the group to `shipped`.

---

## Tracking

Courier webhooks are signature-checked the same way as payments: raw body, secret, event id, ignore duplicates, then append a scan. `ShipmentUpdated` carries the normalised status:

| Courier state | Order group |
| --- | --- |
| Picked up, in transit | `shipped` |
| Out for delivery | `out_for_delivery` |
| Delivered | `delivered` |
| RTO / undelivered back to seller | Stays on the shipment as `rto`. Staff decide cancel or reattempt. Version 1 does not auto-refund an RTO. |
| Customer refused COD | `CodRefused`. Order-service cancels with reason `cod_refused` and inventory releases stock. |

---

## COD remittance

The shipment stores `codAmountPaise`, equal to that group’s share of `payablePaise`. The courier later sends a remittance file: AWB, amount, UTR, date. Fulfilment matches the AWB.

| Match | Result |
| --- | --- |
| AWB and amount match | Shipment `codRemitted` with the UTR. Settlement can treat the cash as collected. |
| Amount differs | Exception queue for finance. No automatic ledger line. |
| AWB unknown | Exception queue. |

Until the match, a delivered COD order is delivered to the customer and still financially open. The seller statement does not include that cash as collected.

---

## What a seller sees

Pickup address on their profile, a button to reprint the label, the AWB, and the latest scan. They do not see other sellers’ shipments on the same customer order, except the fact that the customer paid once.

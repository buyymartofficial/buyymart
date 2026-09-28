# Payments

**Owns:** the gateway session, the verified result, refunds, and idempotency.  
**Service:** `payment-service`.  
**Public route:** `POST /webhooks/payments`, which skips the BFF.  
**Does not own:** order status. It publishes `PaymentCaptured`, `PaymentFailed`, and `RefundCompleted`.

BuyyMart is a merchant of a payment aggregator. It is not a bank and it does not store cards.

---

## What is never stored

Card number, CVV, expiry, UPI PIN, and netbanking password never touch BuyyMart servers or logs. The RBI payment-aggregator guidelines say the merchant site does not save customer card data. If saved cards are added later, the gateway or card network holds a token. BuyyMart stores the token reference only, and the customer can remove it.

Stored fields are: order id, gateway order id, gateway payment id, status, method (`upi`, `card`, `netbanking`, `wallet`), amount paise, and refund ids.

---

## Create a session

1. The order exists in `payment_pending` and stock is reserved.
2. Customer BFF calls payment-service. Timeout 3 seconds.
3. Payment-service creates a gateway order for `payablePaise` in INR and returns the session the browser opens.
4. The customer pays on the gateway page or UPI intent.

If this call fails, the order stays `payment_pending` and the screen says the payment could not be started.

COD does not create a session.

---

## Webhook

The gateway delivers at least once. Razorpay’s published practice is typical: retries for about 24 hours, and a header event id that is unique per event. BuyyMart’s handler:

1. Reads the raw body and checks the signature with the webhook secret. A bad signature is a 400 and nothing is written.
2. Inserts the event id. If that id already exists, it returns 200 and does no work.
3. Writes the payment row and publishes `PaymentCaptured` or `PaymentFailed` in the same transaction as the idempotency row.
4. Returns 200. Work that can wait (email, ledger) happens off the webhook.

The browser return URL is not a payment. A reconciliation job asks the gateway for any `payment_pending` order older than 2 minutes, every 2 minutes, for 30 minutes. That covers a lost webhook. The job uses the same idempotency row, so a late webhook does not double-apply.

---

## Refunds

A refund exists only for a captured prepaid payment, and only for an amount up to the remaining captured amount.

| Who starts it | Rule |
| --- | --- |
| Customer cancel before ship | Full remaining capture for the cancelled lines, including that share of delivery if nothing has shipped |
| Return accepted | The accepted lines, plus return-shipping rules from the returns module |
| Support | Creates a refund request. Finance confirms it. Above the maker-checker amount, finance is a second person. |

The gateway refund call uses an idempotency key of the refund id. `RefundCompleted` is published from the gateway refund webhook or a reconcile poll, not from the moment staff clicked the button. The order becomes `refunded` only after that event.

COD is not refunded through the gateway. The returns and settlement modules record a bank refund or a deduction from the seller payout. The customer still sees `refunded` and a reference number.

---

## Failure cases

| Case | Behaviour |
| --- | --- |
| Customer pays twice because they retried | The second gateway payment is refunded automatically when the order is already `paid`. |
| Webhook arrives before the local payment row commits | The handler retries from the queue. The event id stops a double publish. |
| Amount on the webhook differs from `payablePaise` | Do not mark paid. Alert finance. Leave the order `payment_pending`. |
| Partial capture | Not used in version 1. The gateway is asked for the full payable amount. |

---

## Environments

| Environment | Keys |
| --- | --- |
| Dev | Sandbox, or no gateway until checkout exists |
| Stage | Test mode only |
| Prod | Live keys |

Live keys exist only in the production account.

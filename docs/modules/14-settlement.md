# Settlement and tax

**Owns:** the append-only ledger and the seller payout statement.  
**Service:** `settlement-service`.  
**Does not own:** the payment gateway balance. Finance reconciles the statement to the gateway and the bank.

Version 1 pays sellers by a bank transfer outside the app. The statement in Seller Centre is the instruction for that transfer. If the two differ, the statement is wrong and must be fixed before anyone treats the bank amount as the books.

This module is where a chartered accountant has to confirm rates before go-live. The software stores a rate table. It does not hard-code a percentage that blog posts currently disagree on.

---

## Two kinds of sale

| `seller.kind` | What BuyyMart is | TCS section 52 | TDS section 194-O |
| --- | --- | --- | --- |
| `owned` | The seller. GSTIN on the invoice is BuyyMart’s. | Not collected from itself. Guides describe section 52 as tax on supplies by other suppliers through the operator. | Not a deduction from itself. Section 194-O applies to a participant who sells through the operator. |
| `marketplace` | The operator. The invoice names the seller, at the same visual size as BuyyMart. | Collected on that seller’s net taxable supplies and later reported in GSTR-8. | Deducted on the facilitated gross and later reported in Form 26Q against the seller PAN. |

Both can apply to the same marketplace order. They are different laws, different bases, and different credits. TCS is the seller’s GST cash-ledger credit after GSTR-8. TDS is the seller’s income-tax credit in Form 26AS. Neither is commission.

---

## Ledger lines

Every line is append-only: id, order id, seller id, type, amount paise, and a pointer to the event that caused it. Corrections are new lines, not edits.

| Type | When it is written | Base |
| --- | --- | --- |
| `sale` | Prepaid: `PaymentCaptured`. COD: COD remittance matched. | Item taxable value plus GST, after seller-funded discount |
| `platform_discount` | Same moment, if BuyyMart funded the coupon | The discount paise. A BuyyMart cost. Does not reduce seller gross. |
| `commission` | After delivery, when the return window for that line has ended without an open return | Percent of the seller’s item total, from the seller’s fee schedule |
| `gst_on_commission` | With commission | GST on BuyyMart’s commission invoice. Finance sets the rate and SAC. Guides commonly cite 18% for this service. Confirm. |
| `tcs` | With the marketplace sale, adjusted when a return reduces the month’s net | Net taxable supplies for that line. Rate from the section 52 config. |
| `tds` | When the marketplace amount is credited, subject to the PAN threshold finance configures | Gross facilitated amount. Rate from the section 194-O config. Missing PAN uses the higher section 206AA rate finance configures. |
| `shipping_adjustment` | Courier chargeback or weight dispute | As finance enters it |
| `return` | `ReturnAccepted` | Reverses the sale amount refunded |
| `payout` | When finance marks a statement paid | The net of the statement. Stores the bank UTR. |

Commission waits until the return window closes so a returned line is not commissioned and then reversed as the normal path. A return after commission was already taken writes a reversing commission line.

---

## Rates are configuration

Published explanations checked on 28 September 2026 do not agree:

- Section 194-O is described as 1% in some operator guides and as 0.1% in later seller guides, after a reduction. A missing PAN is described as 5% under section 206AA. An individual or HUF below a yearly threshold (guides cite ₹5 lakh) may be exempt when PAN or Aadhaar is on file.
- Section 52 is described as 0.5% CGST plus 0.5% SGST, or 1% IGST, and also as a lower collected rate of about 0.5% total. The base is the net value of taxable supplies: sales minus returns, GST excluded from that base in the standard reading. Exempt goods are outside TCS.

Before the first marketplace payout, finance writes the live rates into configuration and cites the notification. The statement prints the rate it used. Changing the rate does not rewrite old lines.

---

## Statement

For a period, per seller:

```text
sales
− returns
− seller-funded discounts already inside sales
− commission
− GST on commission
− TCS
− TDS
± shipping adjustments
= net payout
```

The seller can download this statement. They cannot edit it. Finance exports the same file, pays the bank, and enters the UTR. `PayoutPrepared` is published when the statement is ready for that export.

Version 1 does not file GSTR-8 or Form 26Q by itself. It exports the lines those returns need: seller GSTIN, PAN, taxable value, TCS, gross for TDS, and the month. An accountant files them. E-invoice (IRN) starts when BuyyMart’s turnover crosses the CBIC-notified limit, and only this service calls that API.

---

## Reconciliation

A day is closed when all three match:

- Gateway settlement total for prepaid captures and refunds.
- Ledger `sale` and `return` lines for those payment ids.
- Bank credit for that gateway settlement.

COD adds a fourth match: courier remittance file to `codRemitted` shipments. A mismatch stays on the finance exception queue and is not forced into the seller payout.

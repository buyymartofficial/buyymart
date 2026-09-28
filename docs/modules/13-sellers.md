# Sellers

**Owns:** the seller profile, KYC status, suspension, and the staff who work for that seller.  
**Service:** `seller-service`.  
**Does not own:** the product catalogue, stock, or the payout calculation. It stores the bank account and tax ids that settlement reads.

BuyyMart’s own inventory is a seller row with `kind = owned`. It skips marketplace TCS and TDS. Every other seller is `kind = marketplace`.

---

## States

```text
applied → kyc_pending → kyc_rejected
                      → approved → active
active → suspended
active → closed
```

`approved` is the gate for the first listing. `active` is set when the first product goes live. A suspended seller cannot add stock or receive new orders. Past orders stay visible. Catalogue consumes `SellerSuspended` and hides that seller’s products.

---

## Profile the marketplace must hold

The 2020 rules require the marketplace to display seller identity and require a written agreement. The seller record therefore has:

| Field | Rule |
| --- | --- |
| `legalName` | Shown on the product page and the invoice |
| `brandName` | Optional display name |
| `registered` | Whether the business is a registered entity |
| `address` | Principal address, city, state, pin |
| `customerCarePhone`, `customerCareEmail` | Shown to customers for that seller’s goods |
| `gstin` | Required for a taxable seller. Validated for format and that the state code matches the address state. |
| `pan` | Required. Stored encrypted. Shown to finance, not on the product page. |
| `bankAccount`, `ifsc`, `accountName` | Encrypted. Used for payouts. Version 1 pays by a bank transfer outside the app. The statement must match. |
| `grievanceOfficerName`, `grievancePhone`, `grievanceEmail` | The seller’s own grievance contact, required before approval |
| `agreementVersion`, `agreementAcceptedAt` | The seller agreement. Publish is blocked without it. |
| `licence` | Empty in version 1. FSSAI or BIS later. |

Customers see legal name, registered flag, city, customer-care phone, GSTIN, and ratings when ratings exist. They do not see PAN, bank account, or KYC images.

---

## KYC review

The seller uploads PAN, GST certificate, and bank proof through the private KYC bucket. Status moves to `kyc_pending`. Trust staff open a 60-second presigned link, then approve or reject with a reason. Rejection returns the seller to `kyc_rejected` with the fields they must fix. Approval publishes `SellerApproved`.

Version 1 is invite-only or open applications, whichever the launch decision says. Open applications still do not go live without this review.

A bank-account change after approval is a new KYC item. Payouts stay on the old account until finance approves the new one.

---

## Seller staff

The owner invites staff by email. Roles are catalogue, orders, or owner. Staff see only this seller’s products, orders, and statements. They see customer name, phone, and address for orders they must ship, and nothing else.

---

## Suspension and closing

Trust can suspend with a reason. That is audited. New checkouts exclude the seller’s variants. In-flight orders continue unless staff cancel them one by one.

`closed` is a seller who has finished returns and payouts. It is not a delete. Ledger rows stay.

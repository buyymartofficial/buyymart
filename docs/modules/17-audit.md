# Audit

**Owns:** a durable record of staff actions that change money, KYC, price, roles, or a customer’s data.  
**Written by:** `admin-bff` and the domain service that owns the change.  
**Not a replacement for:** the ledger. The ledger is the money. The audit log is who changed something, and why.

CloudTrail still records AWS API calls. This module records product actions.

---

## What must be audited

| Action | Service that writes the row |
| --- | --- |
| Approve or reject seller KYC | Seller |
| Suspend or close a seller | Seller |
| Change a bank account | Seller |
| Hide, reject, or edit a live price | Catalogue |
| Create, disable, or change a coupon | Promotions |
| Confirm a refund, or override a return rejection | Payments or orders |
| Change a staff role, or use `super_admin` | Identity |
| Export a payout file | Settlement |
| Read a KYC document | Seller, including the staff id and the time |

Customer OTP login is not an audit row. It is an identity log with a short life. KYC reads are audited because the files are sensitive.

---

## Row

| Field | Meaning |
| --- | --- |
| `id` | |
| `at` | UTC |
| `staffId` | Who |
| `role` | Their role at that moment |
| `action` | A fixed verb, for example `seller.suspend` |
| `subjectType`, `subjectId` | Seller, order, coupon, staff user |
| `reason` | Required for money, KYC, suspension, and role changes. Free text, at least 10 characters. |
| `before`, `after` | JSON of the fields that changed. Secrets are not copied. A bank account is stored as the last four digits. |
| `requestId` | Ties the row to the HTTP log |

Rows are insert-only. There is no edit and no delete in the admin UI. A mistaken action is a new action that reverses it, with its own reason.

---

## Who can read it

The audit screen is limited to admin and super admin. Filters are staff user, action, subject id, and date. Support does not get a global audit browser. They can see the audit entries attached to an order they already opened.

Exports are themselves audited.

---

## Retention

Audit rows follow the same retention idea as tax records for financial actions: keep them with the order and the ledger. Non-financial rows (a banner edit) can follow a shorter admin policy. Version 1 keeps all audit rows. A deletion job waits until the lawyer sets the schedule.

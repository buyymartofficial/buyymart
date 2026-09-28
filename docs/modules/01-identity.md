# Identity

**Owns:** customer phone login, seller login, staff login, sessions, roles, consent, account deletion.  
**Service:** `identity-service`.  
**Does not own:** seller KYC documents (sellers module), order history (orders module).

A customer, a seller, and a staff user are different accounts. A customer session cannot open Seller Centre. A seller session cannot open Admin Console.

---

## Actors

| Actor | How they sign in | Session |
| --- | --- | --- |
| Customer | Indian mobile number and OTP | Opaque token, stored only as a hash. Sliding 30 days. |
| Seller owner and seller staff | Email and password | Opaque token, 12 hours, idle timeout 2 hours |
| BuyyMart staff | Company email, password, and TOTP MFA | Opaque token, 8 hours. MFA required on every new browser. |

Password hashes use Argon2id. OTP and session tokens are never stored in clear text. OTP lives in Redis under `id:` for 5 minutes. A session lives in Redis for its lifetime and has a Postgres row so a Redis flush logs everyone out cleanly rather than accepting a forged token.

---

## Customer OTP

1. Customer enters a 10-digit Indian mobile number.
2. The API rejects a number that is not a valid Indian mobile pattern.
3. Notification module sends a 6-digit OTP by SMS using a DLT template. Dev logs the code and does not send it. Stage sends only to an allow-list. Prod sends to the number.
4. Five wrong attempts, or three OTP requests in 15 minutes, lock that number for 15 minutes. The WAF also rate-limits the OTP route.
5. A correct OTP creates the customer if the number is new, writes a consent row if they accepted the current privacy notice, and returns a session.
6. Email is optional and is not the login.

The SMS body is a registered template. The log line records the provider message id and the last two digits of the phone. It does not record the OTP.

---

## Consent

The Digital Personal Data Protection Act, 2023 requires a purpose and a way to withdraw consent. BuyyMart records consent instead of inferring it.

| Field | Meaning |
| --- | --- |
| `subjectId` | Customer, seller, or staff id |
| `purpose` | `account`, `order_sms`, `marketing` |
| `policyVersion` | Version of the privacy notice they saw |
| `granted` | True or false |
| `channel` | `web`, later `android` or `ios` |
| `at` | UTC timestamp |
| `requestId` | The HTTP request |

`account` and `order_sms` are required to place an order. `marketing` is off unless the customer turns it on. Withdrawal of marketing stops promotional messages and does not delete orders. Withdrawal of `account` starts the deletion flow below.

A purchase also requires a separate checkout checkbox. That checkbox is an order consent, stored on the order, and it is not pre-ticked. See checkout.

---

## Account deletion

The customer can delete the account from the storefront. Play Store and, later, App Store rules require the same path in the native apps.

Deletion does:

- Revoke all sessions.
- Remove name, email, and saved addresses.
- Replace the phone with a one-way hash so the same number cannot immediately reopen a banned account, and so support can still see that an account was deleted.
- Keep orders, payments, invoices, shipment records, and ledger lines. Tax and consumer-dispute records need them.
- Keep the consent log.

Deletion does not cancel an order that is already `paid` or later. Those orders finish or follow the return path. The customer is told that before they confirm.

---

## Roles

| Role family | Roles inside it |
| --- | --- |
| Customer | `customer` |
| Seller | `seller_owner`, `seller_catalogue`, `seller_orders` |
| Staff | `support`, `catalogue`, `trust`, `finance`, `admin`, `super_admin` |

Seller staff exist only inside one seller. Staff roles are global. `super_admin` is break-glass, always audited, and not used for daily refunds.

---

## Data

| Record | Fields |
| --- | --- |
| Customer | `id`, phone hash, phone (until deletion), name, email, status (`active`, `deleted`), created at |
| Seller user | `id`, seller id, email, password hash, role, status |
| Staff user | `id`, email, password hash, TOTP secret (encrypted), role, status |
| Session | `id`, subject id, family, token hash, issued at, expires at, revoked at |
| Consent | fields in the table above |

---

## Rules

- Login responses use the same timing for “unknown number” and “wrong OTP” so the API is not a phone-number oracle beyond the SMS itself.
- Staff without MFA cannot call admin routes. Admin BFF checks the role on every request.
- A password reset for sellers and staff is an emailed one-time link, 30 minutes, single use. Customers do not have passwords.
- Refresh does not trust the browser to extend a revoked session.

---

## Events

| Event | When |
| --- | --- |
| `UserRegistered` | First successful customer OTP, or seller owner created |
| `UserDeleted` | Customer deletion confirmed |

Seller service consumes `UserRegistered` when the subject is a seller owner. Catalogue does not.

---

## Errors the UI shows

| Case | What the customer sees |
| --- | --- |
| Bad number | “Enter a valid Indian mobile number.” |
| Too many attempts | “Try again in 15 minutes.” |
| Wrong OTP | “That code is wrong or expired.” |
| Staff without MFA | Login stops on the authenticator step. |
